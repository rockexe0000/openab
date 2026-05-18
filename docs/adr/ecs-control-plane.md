# ADR: ECS Control Plane (CRD + Operator Pattern on ECS)

- **Status:** Proposed
- **Date:** 2026-05-18
- **Author:** @pahud.hsieh
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
│  │ (S3 + DDB) │◄─│  Controller  │─►│  CloudMap   │  │
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
4. Apply changes: create, update, or delete ECS resources
5. Write status back to DynamoDB
6. Sleep / wait for next event

---

## 3. Manifest Schema

```yaml
apiVersion: oab.dev/v1
kind: OABService
metadata:
  name: my-agent
  namespace: prod
spec:
  replicas: 2
  taskDefinition:
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/openab:latest
    cpu: 256
    memory: 512
    environment:
      - name: BACKEND_TYPE
        value: bedrock
  backend:
    type: bedrock
    model: anthropic.claude-sonnet-4-20250514
  networking:
    subnets: [subnet-abc, subnet-def]
    securityGroups: [sg-123]
    assignPublicIp: false
status:
  phase: Running
  observedGeneration: 3
  conditions:
    - type: Available
      status: "True"
      lastTransitionTime: "2026-05-18T10:00:00Z"
```

Key fields:
- `spec.taskDefinition` — maps directly to ECS RegisterTaskDefinition
- `spec.backend` — OAB-specific config injected as environment/secrets
- `spec.networking` — ECS awsvpc configuration
- `status` — written back by the controller (stored in DDB, not in the S3 YAML)

---

## 4. State Store Design

| Concern | Store | Rationale |
|---------|-------|-----------|
| Desired state (manifests) | S3 | Human-readable, versioned, git-syncable |
| Status + generation tracking | DynamoDB | Fast reads, conditional writes for optimistic concurrency |
| Leader election lock | DynamoDB | TTL-based lease via conditional PutItem |

S3 event notifications (→ EventBridge → SQS) trigger the controller on manifest changes. A periodic full-reconciliation (every 30–60s) catches drift from out-of-band changes.

---

## 5. Controller Lifecycle

### Reconcile Actions

| Diff | Action |
|------|--------|
| Manifest exists, no ECS service | RegisterTaskDefinition → CreateService |
| Manifest spec changed | RegisterTaskDefinition (new revision) → UpdateService |
| Manifest deleted | UpdateService (desiredCount=0) → DeleteService → DeregisterTaskDefinition |
| ECS state drifted from spec | UpdateService to re-converge |

### Leader Election

When running multiple controller replicas for HA:
- Use DynamoDB conditional writes with a TTL-based lease
- Only the leader performs reconciliation; standbys poll for lease expiry
- Single-replica deployments skip leader election entirely

### Finalizers

Before deleting an ECS service, the controller:
1. Drains active connections (set desiredCount=0, wait for task stop)
2. Cleans up CloudMap service discovery entries
3. Removes the DynamoDB status record
4. Only then acknowledges deletion

---

## 6. MVP Scope

Phase 1 (minimal viable control plane):

1. **S3 bucket** with YAML manifests (one file per OABService)
2. **Single-instance ECS task** running the controller (no leader election)
3. **Poll S3 every 30s**, reconcile against ECS DescribeServices
4. **Write status** to DynamoDB table
5. Support `create` and `update` operations only (manual delete via console)

Phase 2 additions:
- Event-driven triggers (S3 → EventBridge → controller)
- Multi-replica with DDB leader election
- Delete reconciliation with finalizers
- CLI tool for `oab apply -f manifest.yaml` (uploads to S3)
- Rollback via manifest generation history

---

## 7. Alternatives Considered

| Alternative | Why not chosen |
|-------------|---------------|
| AWS Proton | Opinionated, limited customization for OAB-specific logic |
| AWS Copilot | Good for simple apps, no custom reconciliation loop |
| CDK Pipelines | Deployment tool, not a runtime controller with drift detection |
| Step Functions orchestrator | Stateless execution model, no continuous reconciliation |
| Run K8s anyway (EKS) | Valid but adds operational overhead for teams that chose ECS |

---

## 8. Open Questions

1. **Secrets management** — inject via ECS Secrets (SSM/SecretsManager) or controller-managed env vars?
2. **Multi-region** — single controller per region, or a global controller with regional reconcilers?
3. **Observability** — CloudWatch metrics from the controller, or push to a shared OAB dashboard?
4. **Upgrade strategy** — how does the controller upgrade itself without downtime?

---

## 9. Decision

We adopt the CRD + Operator pattern on ECS as described above. The controller will be implemented as a standalone ECS service that reconciles OABService manifests stored in S3 against actual ECS state. This gives ECS-native teams the same declarative, self-healing deployment experience that K8s operators provide, without requiring a Kubernetes cluster.
