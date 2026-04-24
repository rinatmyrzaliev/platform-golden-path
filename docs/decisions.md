# Architecture Decision Records — Golden Path Platform

Each ADR captures a design decision made during the Golden Path library chart
project. Decisions are numbered and include context, rationale, and
consequences.

---

## ADR-001: Library chart over umbrella chart

**Status:** Accepted
**Ticket:** PLAT-001

**Context:** We needed a way to standardize Kubernetes resources across
multiple services. Two patterns exist: umbrella charts (one parent chart
that contains all services as subcharts) and library charts (shared template
code that each service imports independently).

**Decision:** Library chart. Each service has its own consumer chart that
declares a dependency on `platform-library` and calls `include` to render
standardized templates.

**Rationale:**
- Umbrella charts couple all service lifecycles into one Helm release. A
  failed deploy of service A blocks service B.
- Library charts let each service own its release cycle. Teams update the
  library version on their own schedule.
- Same pattern Bitnami uses with its `common` chart and most mature platform
  teams.

**Consequences:**
- Each consumer chart must declare the library dependency and run
  `helm dependency update` after library changes.
- Library version bumps must be coordinated — consumers pin a version and
  upgrade explicitly.

---

## ADR-002: Required ownership fields (team, slo_tier, on_call_channel)

**Status:** Accepted
**Ticket:** PLAT-001, PLAT-003

**Context:** In the drift study, label conventions were inconsistent across
all 3 sample services. Some had team labels, none had SLO tier, and on-call
routing was guesswork.

**Decision:** The library uses Helm's `required` function to fail rendering
if `team`, `slo_tier`, or `on_call_channel` are missing. These become
`platform.io/*` labels on every resource.

**Rationale:**
- Catches drift at render time, not at review time or in production.
- Enables querying for all services by team, SLO tier, or on-call channel
  using label selectors.
- Invalid states become unrepresentable — you literally cannot deploy
  without declaring ownership.

**Consequences:**
- Every consumer chart must set these three values. No defaults provided.
- Adding a new required field is a breaking change (major version bump).

---

## ADR-003: Immutable selectors — only name + instance in matchLabels

**Status:** Accepted
**Ticket:** PLAT-001

**Context:** Kubernetes Deployment selectors are immutable after creation.
If a mutable label (like `team`) is in the selector, changing the team
requires deleting and recreating the Deployment.

**Decision:** matchLabels contains only `app.kubernetes.io/name` and
`app.kubernetes.io/instance`. All other labels (team, slo-tier, on-call)
are on the pod template but not in the selector.

**Rationale:**
- Team reassignments and SLO-tier changes become label updates, not
  Deployment recreations.
- Service selectors match the same minimal set, so traffic routing is
  unaffected by metadata changes.

**Consequences:**
- Pod template labels and selector labels are intentionally different sets.
- New labels can be added to pods without breaking existing Deployments.

---

## ADR-004: PDB on by default, maxUnavailable over minAvailable

**Status:** Accepted
**Ticket:** PLAT-003

**Context:** In the drift study, only 1 of 3 services had a PDB. During
node drains, the other 2 would lose all pods simultaneously.

**Decision:** PDB is rendered by default with `maxUnavailable: 1`. Teams
can opt out with `pdb.enabled: false`.

**Rationale:**
- `maxUnavailable: 1` means at most 1 pod can be down during voluntary
  disruptions (drains, upgrades).
- `minAvailable` was rejected because it blocks evictions when the HPA is
  at its floor — if you have 2 replicas and minAvailable is 2, no pod can
  be drained.
- Default-on ensures safety; opt-out requires an explicit decision.

**Consequences:**
- Single-replica services get a PDB that blocks all voluntary disruption.
  This is intentional — it forces teams to either add replicas or
  explicitly opt out.

---

## ADR-005: Topology spread with ScheduleAnyway

**Status:** Accepted
**Ticket:** PLAT-003

**Context:** Pods should spread across zones and nodes for resilience, but
hard constraints can block scheduling in small clusters or during failures.

**Decision:** Two topology spread constraints (zone + hostname) with
`whenUnsatisfiable: ScheduleAnyway` and `maxSkew: 1`.

**Rationale:**
- `ScheduleAnyway` is a soft preference — the scheduler tries to spread
  but won't leave pods Pending if it can't achieve perfect balance.
- `DoNotSchedule` would block scheduling in single-zone labs or when nodes
  are at capacity.
- In production multi-AZ clusters, `ScheduleAnyway` still achieves good
  spread because the scheduler prefers balanced placement.

**Consequences:**
- In a 2-node single-AZ lab, both pods may land on the same node. This is
  acceptable — the constraint still works correctly in production.

---

## ADR-006: ExternalSecret via library template, not per-service

**Status:** Accepted
**Ticket:** PLAT-004

**Context:** Services need secrets from AWS Secrets Manager. Two approaches:
each service writes its own ExternalSecret YAML, or the library provides a
template that generates ExternalSecrets from a values list.

**Decision:** Library template. Teams declare secrets in values:
```yaml
secrets:
  - name: db-password
    remoteKey: platform/orders/db-password
```
The library generates one ExternalSecret per entry and wires `envFrom` into
the Deployment.

**Rationale:**
- Reduces boilerplate — teams only specify name and remote key.
- Enforces consistent secret naming, refresh interval, and store reference.
- Adding a new secret is a values change, not a template change.

**Consequences:**
- All secrets go through `envFrom` (every key becomes an env var). Teams
  that need selective key mapping would need a library extension.
- ClusterSecretStore is a shared dependency — must be provisioned before
  any service can use secrets.

---

## ADR-007: CPU limit deliberately optional

**Status:** Accepted
**Ticket:** PLAT-003

**Context:** The library chart enforces memory limits but makes CPU limits
optional. Teams can set CPU limits if they choose, but the library does not
require them.

**Decision:** CPU limits are optional. Memory limits are required.

**Rationale:**
- CPU limits cause **throttling**. When a container hits its CPU limit, the
  kernel throttles it — the app slows down silently even when the node has
  spare CPU. This is hard to debug.
- Memory limits cause **OOM kills**. When a container exceeds its memory
  limit, the kernel kills it. This is visible, alertable, and restartable.
- CPU **requests** are still required. The scheduler uses requests for
  placement decisions.
- Many production teams (including Google's internal guidance) recommend
  setting CPU requests but not CPU limits.

**Consequences:**
- Teams must set `resources.requests.cpu`, `resources.requests.memory`, and
  `resources.limits.memory`.
- Teams may optionally set `resources.limits.cpu` if their workload requires
  strict CPU isolation.

---

## ADR-008: file:// dependency for library chart

**Status:** Accepted
**Ticket:** PLAT-005

**Context:** Consumer charts need to reference the library. Options: publish
to a Helm registry (ChartMuseum, OCI), or use `file://` relative paths
within the same Git repo.

**Decision:** `file://../../charts/platform-library` — relative path from
each service folder to the library.

**Rationale:**
- ArgoCD clones the full Git repo, so relative paths resolve correctly.
- No external infrastructure needed (no chart museum, no OCI registry).
- Git is the single source of truth for both app config and library code.
- Library changes and consumer updates can be in the same commit/PR.

**Consequences:**
- All consumer charts must live in the same repo as the library (monorepo).
- If the library is ever extracted to its own repo, all consumers need to
  switch to a registry reference — a breaking change in workflow.

---

## ADR-009: ApplicationSet Git generator for service onboarding

**Status:** Accepted
**Ticket:** PLAT-005

**Context:** With 4 services and growing, manually creating ArgoCD
Application resources per service doesn't scale and creates a bottleneck
on the platform team.

**Decision:** One ApplicationSet with a Git directory generator scanning
`services/*`. Each folder becomes an ArgoCD Application automatically.

**Rationale:**
- Onboarding is a Git commit — add a folder, push, done.
- Platform team controls the template (sync policy, destination, project).
  Service teams control their folder (values, version).
- `automated.selfHeal` reverts manual kubectl changes. `automated.prune`
  deletes resources removed from Git.
- `CreateNamespace=true` means services don't need pre-created namespaces.

**Consequences:**
- Every folder under `services/` becomes a deployment. No way to have a
  "draft" folder without it being deployed (could add an ignore pattern).
- Namespace name equals folder name — naming convention is enforced by
  directory structure.

  ---

## ADR-010: Opt-in Rollout template in platform-library v1.1.0

### Date
2026-04-23

### Status
Accepted

### Context
We want to offer progressive delivery (canary rollouts) alongside the existing Deployment pattern in the platform library. A Deployment's rolling update can't check real user-facing metrics before promoting — it only checks readiness probes. We want promotions gated on SLIs from Project 2 (latency, availability, 5xx rate) so a bad version is caught at small blast radius instead of full fleet.

Argo Rollouts provides this: a canary version takes a small weight of traffic (e.g., 20%), pauses, gets promoted step by step until it reaches 100% only if metrics stay healthy.

We make this change in the library chart because it should be the platform-wide standard. Adding Rollouts service-by-service outside the library would duplicate work and drift over time.

### Decision
- Added an opt-in Rollout template to the library, gated by `rollout.enabled` (default `false`). Existing consumers on v1.0.0 behavior are unaffected.
- Two modes supported:
  - **Mode A (inline):** Rollout owns its pod spec directly. For greenfield services starting on Rollouts.
  - **Mode B (`workloadRef` takeover):** Rollout points at an existing Deployment and adopts its ReplicaSets for zero-downtime migration. Temporary state — flipped off after migration completes.
- **Library owns the shape** of a Rollout (structure, labels, pod spec via shared `_podspec.yaml` partial). **Consumer chart owns the choice** of whether to render a Rollout (branching lives in `services/*/templates/rollout.yaml` and `deployment.yaml`, not inside the library).
- Versioned as **v1.1.0 (minor bump)** because the change is purely additive — no existing consumer needs to change code or values to upgrade.

### Alternatives considered

**1. Replace Deployment template with Rollout (major bump, v2.0.0)**

Rejected. Making Rollout the default would force every existing consumer to start rendering a new resource type without asking. That's a breaking change under semver — it would require a v2.0.0 major bump and a coordinated migration across all services. Opt-in avoids the forced migration entirely.

**2. Library-side branching (single partial picks Deployment or Rollout internally)**

Rejected. Branching belongs in the consumer chart, not hidden inside the library. When a service developer reads their own `templates/` directory, they should see what their chart produces. Library-side branching forces them to read library internals to know whether they're shipping a Deployment or a Rollout — that's bad for both onboarding and debugging.

**3. Skip the `_podspec` refactor, duplicate pod spec in both templates**

Rejected. Two inline copies of the same pod spec would drift over time — someone adds an env var to Deployment but forgets Rollout, or updates probes in Rollout but not Deployment. Extracting the shared pod spec into `_podspec.yaml` makes drift impossible by construction: there is only one source of truth for the pod spec, and both templates include it.

### Consequences

**Positive:**
- Service teams can adopt Rollouts incrementally, one service at a time, by flipping a single values flag.
- Library upgrade from v1.0.0 → v1.1.0 requires zero changes from existing consumers.
- Pod spec lives in exactly one place (`_podspec.yaml`), shared by both Deployment and Rollout templates — drift-proof by construction.
- Platform team ships opinionated canary defaults (20 → manual gate → 50 → 2m pause → 100) in library `values.yaml`. Teams that want different behavior override explicitly, making the choice visible in Git.

**Negative / trade-offs:**
- Two templates to maintain (`_deployment.yaml` and `_rollout.yaml`). Future pod-spec features (init containers, sidecars, etc.) touch the shared partial — but outer-spec features may need to be added to both.
- Opt-in adoption is not automatic. Services that never set `rollout.enabled: true` stay on Deployments indefinitely. The platform gets the capability, but organization-wide adoption requires team buy-in.
- `workloadRef` is a migration state, not a permanent one. A service left with `workloadRef.enabled: true` after migration completes will render both a Deployment and a Rollout in the cluster — the stale Deployment wastes resources and creates confusion. Requires discipline, documentation, or automated linting to catch.

### Related decisions
- ADR-001 (library vs umbrella chart)
- ADR-004 (required fields enforcement)