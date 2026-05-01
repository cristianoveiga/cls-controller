# Version Resolution Adapter — Design Document

> **Status:** Draft
>
> **Context:** This design describes a Version Resolution Adapter for CLM (Cluster Lifecycle Manager) based on the HyperFleet adapter framework. It replaces the CLS version resolution controller with a configuration-driven, compute-only adapter.

---

## Background

### CLS Implementation (Current)

The CLS version resolution controller is a purpose-built Go service that:
1. Receives cluster events via Pub/Sub
2. Reads `spec.release.version` and `spec.release.channelGroup` from the cluster spec
3. Queries the Cincinnati API to resolve the version to a release image pullspec
4. Reports the resolved image, version, channel, and channel group as status metadata

This works but requires custom Go code. The HyperFleet adapter framework enables the same behavior through YAML configuration.

### Key Decisions from F2F (April 20, 2026)

The following decisions were made at the GCP HCP face-to-face regarding version resolution and upgrades:

**Cincinnati as Source of Truth:**
- Consume versions directly from Cincinnati — no extra layers, no cluster image sets
- Use conditional risk mechanisms to influence available versions for installation and upgrades
- The CVO provides available updates via the HostedCluster CR status, resolved through Cincinnati

**Conditional Updates and Risk Management:**
- When offering updates, exclude versions where the "recommended" status is false (risk applies to the cluster)
- Available updates = union of "available updates" (no risks) + "conditional updates" where recommendation is true (risk does not apply)
- Auto-upgrades must apply the same risk evaluation logic
- A process needs to be documented for adding risks to Cincinnati for GCP managed services
- Managed clusters are identified via console branding metrics; PromQL queries can target managed clusters specifically

**Node Pool Upgrades:**
- Node pools do not have available/conditional updates in their CR — they only validate version resolution and version skew (N-2)
- Node pool upgrades should match the control plane version (not independent version selection)
- If node pool upgrade matches control plane version, the risk is already accepted by the control plane upgrade
- Propose to PM: node pool upgrade = "upgrade button" that matches control plane version

**Pre-production Version Support:**
- Need ability to install specific versions not in Cincinnati (nightlies, CI builds)
- Release controller provides its own Cincinnati-compatible graph for nightlies
- Candidate channel is most useful for e2e testing (earliest build with QE sign-off)
- API flag to inject a specific release image URL for development/testing

**Upgrade Blocking Mechanisms:**
- Conditional risks in Cincinnati (PromQL-based)
- `Upgradeable` condition on cluster operators (minor version only)
- Managed service-specific: publish a cluster operator to set `Upgradeable=False` based on pre-upgrade checks (e.g., updated WIF configs)
- CLM must NOT check customer projects directly — information must come through HyperShift operator or component operators

---

## Proposed Design

### Adapter Type: Compute-Only

The version resolution adapter is a **compute-only adapter** — it does not create Kubernetes resources or ManifestWorks. It:
1. Fetches the cluster spec from the HyperFleet API
2. Queries Cincinnati to resolve the version to a release image
3. Reports the result as status metadata back to the HyperFleet API

This is a supported pattern in the HyperFleet adapter framework (Phase 3 resources can be empty).

### Architecture

```
                  ┌──────────────┐
                  │  Sentinel    │
                  └──────┬───────┘
                         │ CloudEvent (cluster.id, generation)
                         ▼
                  ┌──────────────┐
                  │   Broker     │
                  │  (Pub/Sub)   │
                  └──────┬───────┘
                         │
                         ▼
          ┌──────────────────────────┐
          │ Version Resolution       │
          │ Adapter                  │
          └──────┬───────────────────┘
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
  ┌──────────┐ ┌────────┐ ┌──────────────┐
  │HyperFleet│ │Cincinn.│ │ HyperFleet   │
  │ API      │ │  API   │ │ API          │
  │ (GET)    │ │ (GET)  │ │ (POST status)│
  └──────────┘ └────────┘ └──────────────┘
```

### Adapter Configuration

#### adapter-config.yaml

```yaml
adapter:
  name: version-resolution-adapter
  version: "0.1.0"

log:
  level: info

clients:
  hyperfleet_api:
    base_url: http://hyperfleet-api:8000/api/hyperfleet
    version: v1
    timeout: 10s
    retry_attempts: 3
    retry_backoff: exponential
```

No Maestro or Kubernetes client needed — this adapter only makes HTTP calls.

#### adapter-task-config.yaml

```yaml
# ============================================================================
# Phase 1: Parameter Extraction
# ============================================================================
params:
  - name: clusterId
    source: event.id
    type: string
    required: true

  - name: generation
    source: event.generation
    type: int
    required: true

  - name: cincinnatiBaseUrl
    source: env.CINCINNATI_BASE_URL
    type: string
    default: "https://api.openshift.com/api/upgrades_info/v1/graph"

# ============================================================================
# Phase 2: Preconditions — fetch cluster spec + resolve version via Cincinnati
# ============================================================================
preconditions:
  # Step 1: Fetch cluster details from HyperFleet API
  - name: clusterDetails
    api_call:
      method: GET
      url: "/clusters/{{ .clusterId }}"
      timeout: 10s
      retry_attempts: 3
      retry_backoff: exponential
    capture:
      - name: version
        expression: "clusterDetails.spec.release.version.orValue('')"
      - name: channelGroup
        expression: "clusterDetails.spec.release.channelGroup.orValue('stable')"

  # Step 2: Gate — only proceed if version is set
  - name: versionExists
    expression: "version != ''"

  # Step 3: Derive channel name from version + channelGroup
  # e.g., version "4.22.0-ec.4" + channelGroup "candidate" → "candidate-4.22"
  - name: deriveChannel
    capture:
      - name: majorMinor
        expression: |
          version.split('.')[0] + '.' + version.split('.')[1]
      - name: channel
        expression: |
          channelGroup + '-' + version.split('.')[0] + '.' + version.split('.')[1]

  # Step 4: Query Cincinnati API for release image
  # NOTE: api_call supports absolute URLs — the framework detects the https:// scheme
  # and uses the URL as-is instead of prepending the HyperFleet API base URL
  - name: cincinnatiResolution
    api_call:
      method: GET
      url: "{{ .cincinnatiBaseUrl }}?channel={{ .channel }}&arch=amd64"
      headers:
        - name: Accept
          value: application/json
      timeout: 30s
      retry_attempts: 3
      retry_backoff: exponential
    capture:
      - name: releaseImage
        expression: |
          cincinnatiResolution.nodes.filter(n, n.version == version)[0].payload
    conditions:
      - field: releaseImage
        operator: notEquals
        value: ""

# ============================================================================
# Phase 3: Resources — None (compute-only adapter)
# ============================================================================
resources: []

# ============================================================================
# Phase 4: Post-Actions — Report resolved version as status metadata
# ============================================================================
post:
  payloads:
    - name: statusPayload
      build:
        adapter: version-resolution-adapter
        observed_generation: "{{ .generation }}"
        conditions:
          - type: Applied
            status: "{{ if .releaseImage }}True{{ else }}False{{ end }}"
            reason: "{{ if .releaseImage }}VersionResolved{{ else }}ResolutionFailed{{ end }}"
            message: "{{ if .releaseImage }}Resolved {{ .version }} to {{ .releaseImage }}{{ else }}Failed to resolve version{{ end }}"
          - type: Available
            status: "{{ if .releaseImage }}True{{ else }}False{{ end }}"
            reason: "{{ if .releaseImage }}VersionResolved{{ else }}ResolutionFailed{{ end }}"
            message: "{{ if .releaseImage }}Version resolution complete{{ else }}Version resolution failed{{ end }}"
        data:
          release_image: "{{ .releaseImage }}"
          release_version: "{{ .version }}"
          release_channel: "{{ .channel }}"
          release_channel_group: "{{ .channelGroup }}"

  post_actions:
    - name: reportStatus
      api_call:
        method: POST
        url: "/clusters/{{ .clusterId }}/statuses"
        body: "{{ .statusPayload }}"
```

### How it Works

1. **Event arrives** — Sentinel detects a cluster spec change, publishes CloudEvent with cluster ID and generation
2. **Phase 1** — adapter extracts `clusterId`, `generation`, and `cincinnatiBaseUrl` (configurable per environment)
3. **Phase 2** — adapter fetches the cluster spec, extracts `version` and `channelGroup`, derives the Cincinnati channel name, and queries Cincinnati with an absolute URL (`https://api.openshift.com/...`). CEL filters the response nodes to find the matching version and extract the payload (release image pullspec)
4. **Phase 3** — skipped (no resources)
5. **Phase 4** — reports `Applied: True`, `Available: True`, and metadata (`release_image`, `release_version`, `release_channel`, `release_channel_group`) to the HyperFleet API

### External URL Support (Verified)

The adapter framework's `buildHyperfleetAPICallURL` function (in `internal/executor/utils.go`) checks if the URL has a scheme (`http://` or `https://`). If absolute, it uses the URL as-is. This means Cincinnati API calls work natively:

```go
// From hyperfleet-adapter/internal/executor/utils.go
parsedURL, err := url.Parse(apiCallURL)
if parsedURL.Scheme != "" {
    // Absolute URL — use as-is
}
```

### CEL Expression Support (To Verify)

The design relies on CEL for:
- **String splitting**: `version.split('.')[0]` — to derive major.minor from version
- **Array filtering**: `nodes.filter(n, n.version == version)[0].payload` — to find the matching node in Cincinnati's graph response
- **Optional chaining**: `.orValue('')` — for safe null handling

CEL supports `filter()` and `split()` natively. The adapter framework uses `criteria.NewEvaluator` with standard CEL capabilities.

**Open question:** Does the adapter's CEL environment register the `split` and `filter` macros? This needs verification against the `criteria` package.

---

## Pre-production Version Support

For development and testing environments, the adapter supports alternate Cincinnati endpoints:

```yaml
# values.yaml for dev/int environments
env:
  - name: CINCINNATI_BASE_URL
    value: "https://amd64.ocp.releases.ci.openshift.org/api/v1/graph"
```

For injecting a specific release image URL directly (bypassing Cincinnati):
- The cluster spec could include `spec.release.image` as an override
- If `release.image` is set, the adapter skips Cincinnati resolution and reports it directly
- This supports custom assemblies and nightly builds

---

## HC Adapter Integration

The HC adapter (or equivalent HostedCluster adapter in CLM) would gate on the version resolution adapter's status:

```yaml
# In the HC adapter's preconditions
preconditions:
  - name: adapterStatuses
    api_call:
      method: GET
      url: "/clusters/{{ .clusterId }}/statuses"
    capture:
      - name: releaseImage
        expression: |
          adapterStatuses.filter(s, s.adapter == 'version-resolution-adapter')[0]
            .data.release_image.orValue('')
      - name: releaseChannel
        expression: |
          adapterStatuses.filter(s, s.adapter == 'version-resolution-adapter')[0]
            .data.release_channel.orValue('')
    conditions:
      - field: releaseImage
        operator: notEquals
        value: ""
```

This is the same pattern as the CLS precondition on `release_image` existing in VRC status.

---

## Comparison: CLS vs CLM

| Aspect | CLS (Current) | CLM (Proposed) |
|--------|---------------|----------------|
| Implementation | Custom Go service | YAML configuration |
| Cincinnati client | Go `http.Client` | Adapter framework `api_call` |
| Channel derivation | Go `strings.SplitN` | CEL `split()` expression |
| Version matching | Go `for` loop | CEL `filter()` expression |
| Status reporting | Go SDK `ReportStatus()` | Adapter post-action `POST` |
| Deployment | Helm chart + Deployment | Adapter chart + ConfigMap |
| Code changes for new logic | Go code + build + deploy | YAML config update |

---

## Open Questions

1. **CEL string operations**: Does the adapter's CEL environment support `split()` on strings? If not, the channel derivation would need an alternative approach (e.g., a custom CEL function or a pre-computed parameter).

2. **CEL array filtering on external API responses**: Can CEL `filter()` operate on the Cincinnati response (which is a JSON array of nodes)? The response format is `{"nodes": [...], "edges": [...]}` — need to verify CEL can navigate this structure.

3. **Error handling**: What happens when Cincinnati returns an error or the version is not found in the graph? The CLS controller reports `Applied: False` with the error. The adapter's precondition failure would result in `ResourcesSkipped=true` — is that sufficient, or does it need explicit error reporting?

4. **Reconciliation frequency**: The CLS controller resolves on every reconcile event. Should the CLM adapter cache the resolution and skip re-resolution if generation hasn't changed? The framework's generation-based idempotency should handle this.

5. **Cincinnati availability**: What happens when Cincinnati is down? The adapter would fail the precondition and report skipped. Should there be a fallback or retry strategy beyond the configured `retry_attempts`?