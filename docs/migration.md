# Migrating an Existing Service to the Platform Library Chart

## Problem

Your team has a custom Helm chart with its own Deployment, Service, PDB templates.
Over time it drifted — missing labels, no security context, no topology spread.
You want to move to the platform library chart so you get all standards for free.

---

## Steps

### 1. Inventory your current chart

List what your chart produces:

```bash
helm template my-release ./my-chart | grep "^kind:"
```

Map each resource to a library equivalent:

- Deployment → `platform.deployment`
- Service → `platform.service`
- PDB → `platform.pdb`
- ExternalSecret → `platform.externalsecret`
- Ingress → `platform.ingress`

### 2. Create a new service folder

```bash
mkdir -p services/my-service/templates
```

### 3. Write Chart.yaml with library dependency

```yaml
apiVersion: v2
name: my-service
version: 0.1.0
type: application
dependencies:
  - name: platform-library
    version: "1.0.0"
    repository: "file://../../charts/platform-library"
```

### 4. Translate values

Map your old values to library values format:

| Old chart value    | Library value    |
| ------------------ | ---------------- |
| `image.repository` | `image.repository` |
| `image.tag`        | `image.tag`        |
| `replicaCount`     | `replicaCount`     |
| `service.port`     | `service.port`     |
| `containerPort`    | `port`             |
| (missing)          | `team` (required)          |
| (missing)          | `slo_tier` (required)      |
| (missing)          | `on_call_channel` (required) |

### 5. Write the templates file

```yaml
{{- include "platform.deployment" . }}
{{- include "platform.service" . }}
{{- include "platform.pdb" . }}
```

Add `platform.ingress` or `platform.externalsecret` if needed.

### 6. Compare output

```bash
# Old chart
helm template my-release ./old-chart > old.yaml

# New chart
helm dependency update services/my-service
helm template my-release services/my-service > new.yaml

# Compare
diff old.yaml new.yaml
```

Review the diff. Expected differences:

- New labels (platform.io/team, slo-tier, on-call)
- Security context added
- Topology spread constraints added
- PDB added
- emptyDir /tmp volume added

### 7. Deploy and cutover

**Option A — Clean cutover (lab/staging):**

```bash
helm uninstall my-release -n my-namespace
```

Commit new chart to `services/my-service`, ArgoCD picks it up.

**Option B — Zero-downtime (production):**

- Deploy new chart to a staging namespace first
- Verify all probes pass, secrets sync, traffic works
- Switch production namespace by updating values
- ArgoCD self-heal handles the rest

### 8. Delete old chart

Remove the old chart directory. The service now gets all future
platform improvements (new security policies, updated defaults)
automatically through library version bumps.

---

## Common Issues During Migration

- **Missing required fields** — Library requires `team`, `slo_tier`, `on_call_channel`. Add them.
- **Port mismatch** — Library uses `port` for container port, not `containerPort`.
- **Security context breaks app** — If app needs root, override `securityContext` in values. Better: switch to a non-root image.
- **Nil pointer errors** — Define empty parent maps (`securityContext: {}`, `probes: readiness: {}`) for fields your app doesn't customize.