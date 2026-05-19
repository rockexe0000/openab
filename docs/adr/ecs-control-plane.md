# ADR: ECS Control Plane (CRD + Operator Pattern on ECS)

- **Status:** Proposed
- **Date:** 2026-05-18
- **Author:** @pahud.hsieh
- **Reviewers:** 擺渡法師(Codex), 普渡法師(CC), 口渡法師(Copilot), 超渡法師(Kiro)
- **Related:** [Multi-Platform Adapters](./multi-platform-adapters.md), [Basic CronJob](./basic-cronjob.md)

---

## 1. Context & Motivation

OpenAB currently deploys on Kubernetes using Helm charts. While K8s provides a mature operator pattern (CRD + Controller), many teams prefer or require **Amazon ECS** for its operational simplicity and tighter AWS integration.

We want to bring the same declarative, self-healing deployment model to ECS:

- Operators declare desired state in YAML manifests (analogous to CRDs)
- A controller reconciles desired state against actual ECS resources
- OAB instances + arbitrary backends are deployed and maintained automatically

This enables a "GitOps for ECS" workflow where pushing a YAML change triggers the controller to converge the cluster to the new desired state.

---

## 2. Design Overview

```
┌──────────────────────────────────────────────────────┐
│  ECS Control Plane (runs as ECS Service)             │
│                                                      │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │ State Store│  │  Reconciler  │  │  ECS API /  │  │
│  │   (S3)     │◄─│  Controller  │─►│  CloudMap   │  │
│  │            │  │              │  │             │  │
│  └────────────┘  └──────────────┘  └─────────────┘  │
│                        ▲                             │
│                        │ events / poll               │
│                        ▼                             │
│                 ┌──────────────┐                     │
│                 │ S3 Events /  │                     │
│                 │ EventBridge  │                     │
│                 └──────────────┘                     │
└──────────────────────────────────────────────────────┘
```

### Core Loop (Reconciliation)

1. Load all YAML manifests from S3 (desired state)
2. Describe current ECS services/tasks (observed state)
3. Compute diff
4. Render config artifact → upload to S3
5. Apply changes: create, update, or delete ECS resources
6. Write status back to S3 (with observedGeneration)
7. Sleep / wait for next event

---

## 3. API Identity

| Field | Value |
|-------|-------|
| API Group | `oab.dev` |
| API Version | `oab.dev/v1` |
| Kind | `OABService` |
| Plural resource | `oabservices` |
| CLI noun | `oabservice` |

All paths (S3, SSM, IAM) use this identity consistently:
- S3: `manifests/{namespace}/{name}.yaml`
- SSM: `/oab/{namespace}/{name}/{secret-name}`
- ECS: service name = `oab-{namespace}-{name}`

---

## 4. Manifest Schema

```yaml
apiVersion: oab.dev/v1
kind: OABService
metadata:
  name: kiro-01
  namespace: prod
  generation: 4                        # incremented by oabctl on each apply
spec:
  replicas: 1                          # see "Replicas & HA" section
  capacityProvider: FARGATE_SPOT       # FARGATE (default) or FARGATE_SPOT
  cpu: 256                             # vCPU units (256 = 0.25 vCPU)
  memory: 512                          # MB
  taskDefinition:
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/openab:latest
  bootstrapFrom: s3://oab-backups/agents/kiro-01/latest.tar.gz
  networking:
    subnets: [subnet-aaa, subnet-bbb]
    securityGroups: [sg-oab]
    assignPublicIp: false
  secrets:
    - name: DISCORD_TOKEN
      valueFrom: /oab/prod/kiro-01/discord-token
    - name: OPENAI_API_KEY
      valueFrom: /oab/prod/kiro-01/openai-key
  config:
    channels:
      - type: discord
        guild_id: "1490282656913559673"
    backend:
      type: bedrock
      model_id: anthropic.claude-sonnet-4-20250514
      region: us-east-1
    steering:
      system_prompt: "You are 超渡法師(Kiro)..."
    features:
      stt: true
      cronjob: true
status:
  phase: Running
  observedGeneration: 4
  conditions:
    - type: Available
      status: "True"
      lastTransitionTime: "2026-05-18T10:00:00Z"
```

### Field Responsibilities

| Field | Purpose | Who writes |
|-------|---------|-----------|
| `spec.bootstrapFrom` | Mutable HOME state (memory, runtime data, scripts) | Operator |
| `spec.config` | Desired agent configuration (channels, backend, steering) | Operator |
| `spec.secrets` | SSM/Secrets Manager references for tokens & keys | Operator |
| `metadata.generation` | Monotonic counter, incremented on each apply | oabctl |
| `status.observedGeneration` | Last generation the controller successfully reconciled | Controller |

### Separation of Concerns

- **`bootstrapFrom`** = mutable state (memory, knowledge base, scripts). Does NOT contain secrets or config.
- **`spec.config`** = desired configuration. Controller renders this to `config.toml` artifact.
- **`spec.secrets`** = all credentials. Always SSM/Secrets Manager references, never in archive or manifest values.

---

## 5. Replicas & HA

Bot agents (Discord, Telegram, Slack) are inherently **single-instance** — the same bot token cannot have multiple consumers without duplicate responses.

Rules:
- `replicas: 1` — default and only valid value for bot-type agents
- Schema validation rejects `replicas > 1` when `spec.config.channels` contains bot-type channels
- Future: `replicas > 1` allowed for stateless HTTP/webhook agents with a load balancer

For HA of bot agents, the controller relies on ECS service auto-recovery (task restart on failure) rather than multiple replicas.

---

## 6. State Store Design (S3-Only, Phase 1)

```
s3://oab-control-plane/
  ├── manifests/{namespace}/{name}.yaml       ← desired state (oabctl writes)
  ├── config/{namespace}/{name}/config.toml   ← rendered config (controller writes)
  └── status/{namespace}/{name}.json          ← observed state (controller writes)
```

| Concern | Mechanism | Rationale |
|---------|-----------|-----------|
| Desired state | `manifests/` | Human-readable, git-syncable, versioned via S3 versioning |
| Rendered config | `config/` | Controller generates config.toml from spec.config |
| Status | `status/` | Contains observedGeneration, phase, conditions |
| Generation tracking | Explicit `metadata.generation` counter in manifest | Unambiguous; S3 VersionId alone is insufficient for reconcile tracking |
| Change detection | S3 Event Notifications → EventBridge (Phase 2) | Phase 1 uses polling |
| Consistency | S3 strong read-after-write | Sufficient for single-controller |

---

## 7. Controller Lifecycle

### Task Startup (Entrypoint Wrapper)

ECS/Fargate has no init container. The OAB container uses an **entrypoint wrapper script**:

```bash
#!/bin/bash
# 1. Download bootstrap (mutable state)
aws s3 cp "${BOOTSTRAP_FROM}" /tmp/bootstrap.tar.gz
tar xzf /tmp/bootstrap.tar.gz -C $HOME

# 2. Download rendered config (controller-generated)
aws s3 cp "s3://oab-control-plane/config/${NAMESPACE}/${NAME}/config.toml" $HOME/config.toml

# 3. Start OAB (secrets injected via ECS task definition)
exec /usr/bin/openab
```

The controller passes `BOOTSTRAP_FROM`, `NAMESPACE`, `NAME` as environment variables in the task definition.

### Reconcile Actions

| Diff | Controller Action |
|------|-------------------|
| Manifest exists, no ECS service | Render config → upload to S3 → RegisterTaskDefinition → CreateService |
| `spec.config` changed | Render new config.toml → upload to S3 → restart task (new config on next start) |
| `spec.cpu/memory/image` changed | New task definition revision → UpdateService (rolling restart) |
| `spec.secrets` changed | Update task definition secrets → UpdateService |
| Deletion marker present | desiredCount=0 → wait for drain → DeleteService → cleanup status/config |
| ECS state drifted | UpdateService to re-converge |
| `status.observedGeneration < metadata.generation` | Reconcile needed |

### Deletion (Tombstone Pattern)

`oabctl delete` does NOT remove the manifest from S3. Instead:

```yaml
metadata:
  name: kiro-01
  namespace: prod
  generation: 5
  deletionTimestamp: "2026-05-19T00:10:00Z"   # ← marks intent to delete
```

Controller sees `deletionTimestamp`:
1. Scale to 0, wait for task drain
2. Delete ECS service
3. Remove `config/{ns}/{name}/` from S3
4. Remove `status/{ns}/{name}.json` from S3
5. Remove `manifests/{ns}/{name}.yaml` from S3 (final cleanup)

---

## 8. CLI UX (`oabctl`)

### Core Commands

```bash
oabctl apply -f agent.yaml          # declare/update desired state
oabctl get oabservice               # list all services + status
oabctl get oabservice kiro-01       # single service detail
oabctl delete oabservice kiro-01    # set deletionTimestamp (tombstone)
oabctl diff -f agent.yaml           # show local vs remote diff
oabctl logs kiro-01                 # shortcut to ECS task logs
oabctl snapshot kiro-01             # backup current HOME to S3
```

### `apply` Flow

```
$ oabctl apply -f prod/kiro-01.yaml

✓ Schema validated (oab.dev/v1 OABService)
✓ Secrets verified (SSM paths exist)
✓ Uploaded to s3://oab-control-plane/manifests/prod/kiro-01.yaml
✓ Generation: 3 → 4
⏳ Waiting for reconciliation...
✓ Service kiro-01 reconciled (observedGeneration=4, phase=Running)
```

### Directory Apply

```bash
$ oabctl apply -f agents/

✓ kiro-01  applied (FARGATE_SPOT, 256cpu/512mem)
✓ kiro-02  applied (FARGATE_SPOT, 256cpu/512mem)
✓ kiro-03  applied (FARGATE_SPOT, 256cpu/512mem)
✓ kiro-04  applied (FARGATE_SPOT, 256cpu/512mem)
✓ kiro-05  applied (FARGATE_SPOT, 256cpu/512mem)
✓ cc-01    applied (FARGATE_SPOT, 512cpu/1024mem)
✓ cc-02    applied (FARGATE_SPOT, 512cpu/1024mem)
✓ cc-03    applied (FARGATE_SPOT, 512cpu/1024mem)
✓ codex-01 applied (FARGATE_SPOT, 1024cpu/2048mem)
✓ codex-02 applied (FARGATE_SPOT, 1024cpu/2048mem)

10 services reconciled.
```

```bash
$ oabctl get oabservice

NAME       NAMESPACE  CPU   MEM   CAPACITY      STATUS   GEN  OBS-GEN  AGE
kiro-01    prod       256   512   FARGATE_SPOT  Running  4    4        2m
kiro-02    prod       256   512   FARGATE_SPOT  Running  4    4        2m
kiro-03    prod       256   512   FARGATE_SPOT  Running  4    4        2m
kiro-04    prod       256   512   FARGATE_SPOT  Running  4    4        2m
kiro-05    prod       256   512   FARGATE_SPOT  Running  4    4        2m
cc-01      prod       512   1024  FARGATE_SPOT  Running  4    4        2m
cc-02      prod       512   1024  FARGATE_SPOT  Running  4    4        2m
cc-03      prod       512   1024  FARGATE_SPOT  Running  4    4        2m
codex-01   prod       1024  2048  FARGATE_SPOT  Running  4    4        1m
codex-02   prod       1024  2048  FARGATE_SPOT  Running  4    4        1m
```

---

## 9. Schema Versioning & Validation

- `oabctl` validates manifests against a built-in JSON Schema for `oab.dev/v1`
- Controller also validates on reconcile (defense in depth)
- Unknown fields are rejected (strict mode) to prevent silent misconfiguration
- Schema version is tied to `oabctl` / controller version; version skew policy:
  - Controller must be >= oabctl version
  - Controller accepts current version and one prior version

---

## 10. MVP Scope

### Phase 1 (In Scope)

1. **S3 bucket** — manifests/, config/, status/ prefixes, versioning enabled
2. **Single-instance ECS task** running the controller
3. **Poll-based** — ListObjects + GetObject every 30s
4. **`oabctl apply`** — validates, increments generation, uploads to S3
5. **`oabctl get`** — reads status/, displays table with generation tracking
6. **`oabctl delete`** — writes deletionTimestamp (tombstone), controller cleans up
7. **Config rendering** — controller generates config.toml from spec.config → S3
8. **Entrypoint wrapper** — downloads bootstrap + config on task start
9. **Secrets via SSM** — referenced in task definition, never in S3

### Phase 1 (Out of Scope)

- Event-driven triggers (EventBridge)
- Multi-replica controller / leader election (DynamoDB)
- `oabctl diff`, `oabctl logs`, `oabctl snapshot`
- Rollback via generation history
- Secret rotation automation
- Multi-region
- OABFleet kind (template for N agents)

### Phase 2

- Event-driven triggers (S3 → EventBridge → controller wakes immediately)
- DynamoDB for leader election (multi-replica controller HA)
- `oabctl diff` (spec-only vs rendered vs runtime)
- `oabctl snapshot` (backup HOME → S3)
- Rollback via generation history
- GitOps integration (GitHub Actions → `oabctl apply` on push)

### Phase 3

- Multi-region (controller per region, S3 cross-region replication)
- OABFleet kind (deploy N agents from a template)
- Secret rotation automation
- Auto-scaling policies
- Controller self-upgrade (blue/green deployment)

---

## 11. Alternatives Considered

| Alternative | Why not chosen |
|-------------|---------------|
| AWS Proton | Opinionated, limited customization for OAB-specific logic |
| AWS Copilot | Good for simple apps, no custom reconciliation loop |
| CDK Pipelines | Deployment tool, not a runtime controller with drift detection |
| Step Functions orchestrator | Stateless execution model, no continuous reconciliation |
| Run K8s anyway (EKS) | Valid but adds operational overhead for teams that chose ECS |
| DynamoDB as primary store | Adds infra; S3 sufficient for single-controller Phase 1 |
| Secrets in bootstrap archive | No per-secret audit trail, broad S3 access = all tokens exposed |

---

## 12. Open Questions

1. **Controller upgrade strategy** — ECS rolling deployment sufficient for Phase 1; need blue/green for Phase 2 multi-replica
2. **Runtime state persistence** — how/when does agent snapshot HOME back to S3? (periodic cron? on graceful shutdown?)
3. **Networking isolation** — per-service SG rules, egress policy, VPC endpoint restrictions
4. **Observability** — CloudWatch metrics from controller, or push to shared dashboard?
5. **`oabctl diff` granularity** — spec-only, rendered taskdef, or runtime status?

---

## 13. Decision

We adopt the CRD + Operator pattern on ECS with:
- **`oab.dev/v1` / `OABService`** as the fixed API identity
- **S3-only state store** (manifests + rendered config + status)
- **`oabctl` CLI** with kubectl-like UX
- **Explicit generation/observedGeneration** for reconcile tracking
- **Secrets always in SSM/Secrets Manager**, never in S3 archives
- **Entrypoint wrapper** pattern for bootstrap + config download
- **Tombstone deletion** pattern for safe cleanup
- **`replicas: 1`** enforced for bot-type agents

Phase 1 targets a single-controller, poll-based MVP that can deploy and manage a fleet of OAB agents on ECS Fargate with full declarative lifecycle management.
