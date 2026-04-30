# Golden Path Platform

A Helm library chart that standardizes Kubernetes deployments across teams. Service teams depend on the library and only provide their own values (image, port, env). The library enforces org standards — labels, security context, probes, PDBs, topology spread, secrets — so every deployment is production-grade by default.

New service onboarding is a single Git commit. ArgoCD ApplicationSets detect the new folder and deploy automatically.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     Git Repository                       │
│                                                          │
│  charts/platform-library/       Library chart (v1.0.0)   │
│    _deployment.yaml             Standardized templates   │
│    _service.yaml                                         │
│    _ingress.yaml                                         │
│    _pdb.yaml                                             │
│    _externalsecret.yaml                                  │
│                                                          │
│  services/                                               │
│    catalog/  ──┐                                         │
│    orders/   ──┼── Consumer charts (values only)         │
│    payments/ ──┤                                         │
│    reports/  ──┘                                         │
│                                                          │
│  apps/applicationset.yaml       Git generator            │
└────────────────────┬─────────────────────────────────────┘
                     │ ArgoCD watches main branch
                     ▼
┌─────────────────────────────────────────────────────────┐
│                    EKS Cluster                          │
│                                                         │
│  ArgoCD ApplicationSet                                  │
│    ├── catalog   (Synced + Healthy)                     │
│    ├── orders    (Synced + Healthy)                     │
│    ├── payments  (Synced + Healthy)                     │
│    └── reports   (Synced + Healthy)                     │
│                                                         │
│  External Secrets Operator                              │
│    └── ClusterSecretStore → AWS Secrets Manager         │
└─────────────────────────────────────────────────────────┘
```

## What the library provides

Every service deployed through the library gets:

- **Deployment** — readiness/liveness probes, security context (runAsNonRoot, readOnlyRootFilesystem, no privilege escalation), topology spread (zone + hostname), emptyDir at /tmp
- **Service** — ClusterIP default, named port indirection
- **Ingress** — optional, enabled per service, requires host when enabled
- **PodDisruptionBudget** — on by default, maxUnavailable: 1, opt-out with `pdb.enabled: false`
- **ExternalSecret** — AWS Secrets Manager integration via ESO, one ExternalSecret per secret entry
- **Standardized labels** — `app.kubernetes.io/*` for K8s tooling + `platform.io/*` for ownership and SLO tier
- **Required fields** — `team`, `slo_tier`, `on_call_channel` — chart won't render without them

## Repo structure

```
platform-golden-path/
├── charts/
│   └── platform-library/          # Library chart (type: library, v1.0.0)
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── _deployment.yaml
│           ├── _service.yaml
│           ├── _ingress.yaml
│           ├── _pdb.yaml
│           └── _externalsecret.yaml
├── services/                      # One folder per service
│   ├── catalog/
│   ├── orders/
│   ├── payments/
│   └── reports/
├── apps/
│   └── applicationset.yaml        # ArgoCD Git generator
└── docs/
    ├── decisions.md               # Architecture Decision Records
    ├── migration.md               # How to migrate a drifted chart
    └── versioning.md              # Semver policy for the library
```

## Quickstart: onboard a new service

### 1. Create a folder under `services/`

```bash
mkdir -p services/my-service/templates
```

### 2. Add Chart.yaml with library dependency

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

### 3. Add values.yaml

```yaml
name: my-service
team: my-team
slo_tier: "2"
on_call_channel: my-team-oncall

image:
  repository: my-registry/my-image
  tag: "1.0.0"

port: 8080
replicaCount: 2

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi

securityContext: {}
probes:
  readiness: {}
  liveness: {}
service:
  type: ClusterIP
  port: 80
ingress:
  enabled: false
```

### 4. Add templates/resources.yaml

```yaml
{{- include "platform.deployment" . }}
---
{{- include "platform.service" . }}
---
{{- include "platform.pdb" . }}
```

### 5. Verify locally

```bash
helm dependency update services/my-service
helm template my-service services/my-service
```

### 6. Commit and push

ArgoCD detects the new folder and deploys automatically. No manual config needed.

## Adding secrets from AWS Secrets Manager

Add a `secrets` list to your values and include the ExternalSecret template:

```yaml
# values.yaml
secrets:
  - name: db-password
    remoteKey: platform/my-service/db-password
  - name: api-key
    remoteKey: platform/my-service/api-key
```

```yaml
# templates/resources.yaml — add this line
{{- include "platform.externalsecret" . }}
```

Each entry creates an ExternalSecret → K8s Secret → injected into the pod via `envFrom`.

## Dev loop

```bash
# After any library change
helm dependency update services/<service>
helm template <release> services/<service>
```

`helm dependency update` rebuilds the library tarball from source. Without it, you render against a stale cached copy.

## Key decisions

| Decision | Rationale | Doc |
|---|---|---|
| Library chart over umbrella | Umbrella couples all service lifecycles. Library lets each service own its release. | [decisions.md](docs/decisions.md) |
| CPU limit optional | CPU limits throttle silently. Memory limits OOM-kill visibly. | [decisions.md](docs/decisions.md) |
| PDB on by default | Only 1 of 3 drifted services had a PDB. maxUnavailable over minAvailable to avoid blocking HPA. | [decisions.md](docs/decisions.md) |
| file:// dependency | ArgoCD clones the full repo, so relative paths resolve. No chart museum needed. | [decisions.md](docs/decisions.md) |
| ApplicationSet Git generator | Folder structure = deployment topology. No per-service ArgoCD config. | [decisions.md](docs/decisions.md) |

## Related repos

- [platform-foundation](https://github.com/rinatmyrzaliev/platform-foundation) — EKS cluster, VPC, Terraform, bootstrap addons
- [platform-observability](https://github.com/rinatmyrzaliev/platform-observability) — SLO and observability platform (Project 2)

## Docs

- [Architecture Decision Records](docs/decisions.md)
- [Migration guide](docs/migration.md)
- [Versioning policy](docs/versioning.md)