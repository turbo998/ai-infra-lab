# Enterprise stateful-agent memory revocation and temporal-RAG governance selection FAQ (Cosmos Change Feed vs Redis lease vs AI Search tombstones)

> Audience: Cloud & AI Solution Architects, data-platform leaders, and compliance/security owners.
>
> Core question: how to deliver retrievable, revocable, and auditable long-term memory governance in enterprise multi-tenant agent platforms without stale-memory resurrection across caches and vector indexes.

## 1. Executive summary

Enterprise memory governance must be designed around revocation, not just ingestion. Recommended pattern:

- **Route A (backbone)**: `Cosmos DB + Change Feed + AI Search push updates` for event-driven index consistency.
- **Route B (acceleration guardrail)**: `Redis lease + RYW` controls for session-level immediate revocation effect.
- **Route C (audit layer)**: temporal validity (`valid_from/valid_until`) + append-only tombstone ledger for replayability.
- **Route D (anti-pattern)**: update source DB only while ignoring cache/index invalidation and audit proof.

One-line takeaway: **memory governance success is measured by provable deletion correctness, not just write throughput.**

## 2. Why revocation is harder than ingestion

- Retrieval stacks usually include multiple replicas: transactional store, cache, vector index, and offline lakehouse copies.
- Revocation events can arrive out-of-order and create ghost-hit windows.
- Session context can survive policy updates if version controls are missing.
- Regulatory audits require timestamped explainability of retrieval outcomes.

## 3. Route definitions

### Route A: Cosmos Change Feed consistency backbone

- Persist memory entities in Cosmos and emit mutation/revocation events.
- Push tombstones and updates to AI Search and derived layers.
- Suitable for large-scale, multi-region, event-driven data planes.

### Route B: Redis lease and session-version guardrails

- Maintain short-lived per-tenant lease versions.
- On revocation, increment lease so stale sessions are forced to refresh.
- Suitable when sub-second revocation effect is required.

### Route C: temporal semantics + tombstone audit ledger

- Persist `valid_from`, `valid_until`, and `revoked_at` semantics.
- Keep append-only evidence for replay and regulator review.
- Suitable for high-assurance compliance domains.

### Route D: anti-pattern baseline

- Source-store updates only, no cache/index invalidation discipline.
- No immutable revocation evidence chain.
- Produces “deleted but still retrievable” incidents.

## 4. Quantitative models

### 4.1 Revocation propagation window

Definitions:

- `T_emit`: revocation event emission time
- `T_cache`: cache invalidation completion time
- `T_index`: index invalidation completion time
- `T_client`: session context refresh completion time

Window model:

`W_revoke = max(T_cache, T_index, T_client) - T_emit`

### 4.2 Ghost-hit probability

`P_ghost = P(cache_stale ∪ index_stale ∪ session_stale)`

The target is to constrain `P_ghost` through versioned guardrails and deterministic event handling.

### 4.3 RYW consistency overhead

`Cost_RYW ≈ N_read * C_version_check + N_invalidate * C_invalidate`

Use this to tune lease checks by risk tier.

### 4.4 Compliance replay coverage

`Replay_coverage = Replayed_calls / Total_sensitive_calls`

High-regulation domains should target near-complete replay coverage.

## 5. Twelve-dimension comparison matrix

| Dimension | Route A event backbone | Route B lease guard | Route C audit ledger | Route D anti-pattern |
|---|---|---|---|---|
| Sub-second revocation effect | Medium-high | High | Medium | Low |
| End-to-end consistency | High | Medium-high | Medium-high | Low |
| Audit replayability | Medium | Medium | High | Low |
| Engineering complexity | Medium | Medium | Medium | Low |
| Runtime cost | Medium | Medium | Medium-high | Low (apparent) |
| Multi-region scalability | High | Medium-high | High | Low |
| Cache contamination control | Medium-high | High | Medium | Low |
| Index drift resilience | High | Medium | Medium-high | Low |
| Compliance risk reduction | High | Medium-high | High | Low |
| Explainability | High | Medium | High | Low |
| Evolution flexibility | High | Medium-high | High | Low |
| Recommendation | ✅ backbone | ✅ acceleration | ✅ audit | ⛔ |

## 6. Deep quantitative Q&A (Q1-Q8)
### Q1: Why do deleted memories still appear in answers?
Because source deletion does not automatically invalidate caches, indexes, and active session context.

### Q2: Does Change Feed solve all consistency problems?
No. It solves propagation, but session-level staleness still needs lease/version checks.

### Q3: Does lease validation hurt latency?
Slightly. In critical tiers, the risk reduction usually outweighs the overhead.

### Q4: Why is a tombstone ledger required?
Without immutable evidence, revocation timing and causality cannot be proven during audits.

### Q5: How to handle cross-region out-of-order events?
Use monotonic versions and idempotent handlers so stale events cannot overwrite newer state.

### Q6: How to validate GDPR-style deletions?
Use replay sampling and index scan assertions that revoked entities are no longer retrievable.

### Q7: Can TTL replace explicit revocation?
No. TTL handles expiration, not immediate withdrawal rights and legal evidence obligations.

### Q8: When is A+B+C mandatory?
When business requires sub-second revocation, low ghost-hit tolerance, and strict replayable compliance.

## 7. Architecture diagrams (Mermaid)

```mermaid
flowchart LR
    W[Write Update Delete] --> C[Cosmos DB]
    C --> F[Change Feed Processor]
    F --> S[AI Search Index Updater]
    F --> R[Redis Lease Version Guard]
    F --> L[Tombstone Ledger]
    S --> Q[RAG Query Path]
    R --> Q
```

```mermaid
flowchart TD
    D[Revoke request] --> E[Emit tombstone event]
    E --> I[Index invalidate]
    E --> K[Cache version bump]
    E --> A[Audit ledger append]
    I --> V[Verification scan]
    K --> V
    A --> V
```

## 8. Production snippet: change-feed handler (pseudo-code)

```python
def on_change(event):
    key = event["memory_id"]
    version = event["version"]
    if event["op"] == "revoke":
        search.delete(key)
        redis.set(f"lease:{event['tenant_id']}", version)
        ledger.append({"memory_id": key, "revoked_at": event["ts"], "version": version})
    else:
        search.upsert(event["payload"])
```

## 9. Production snippet: query-path lease guard (pseudo-code)

```python
def guarded_query(tenant_id, session_version, query):
    lease = redis.get(f"lease:{tenant_id}")
    if lease and session_version < lease:
        raise RetryWithRefresh("session context stale")
    return rag.search(query)
```

## 10. Enterprise scenarios (2+)

### Scenario A: regulated financial support assistant

- Policy updates require near-real-time withdrawal of obsolete guidance.
- Route A drives full propagation, Route B enforces immediate session guards.
- Route C preserves auditable evidence for regulator inquiries.

### Scenario B: healthcare follow-up copilot

- Consent withdrawal must invalidate patient-linked memory retrieval quickly.
- Lease versioning blocks stale sessions from replaying old context.
- Replay sampling demonstrates revocation closure to governance teams.

### Scenario C: global manufacturing knowledge hub

- Multi-region replication requires robust out-of-order event handling.
- Idempotent processors and tombstone ledgers limit drift risk.
- Revocation-window SLO is treated as a first-class reliability metric.

## 11. Three-tier 3-year TCO (USD)

| Scale tier | Route A | Route A+B | Route A+C | Route A+B+C |
|---|---:|---:|---:|---:|
| 500K memories | 1,450,000 | 1,620,000 | 1,780,000 | 1,930,000 |
| 50M memories | 8,900,000 | 9,800,000 | 10,700,000 | 11,600,000 |
| 500M memories | 49,500,000 | 53,400,000 | 57,800,000 | 61,900,000 |

Interpretation:

- Composite controls cost more, but reduce compliance and reputation-loss exposure.
- In high-risk sectors, provable deletion correctness outweighs minimal storage cost.

## 12. Day-0 / Day-1 / Day-2 checklist

### Day-0 (contract and model baseline)
- define memory schema, version semantics, revocation policy, and audit retention requirements.

### Day-1 (engineering rollout)
- wire change-feed processors, index invalidation, lease checks, and alert baselines.

### Day-2 (operations and governance)
- monitor revocation window, ghost-hit rate, replay coverage, and regional drift indicators.

## 13. Good fit vs Not for

### ✅ Good fit

- Regulated RAG workloads with strict withdrawal and audit obligations.
- Large platforms with multi-layer cache and index paths.
- Multi-region environments requiring revocation resilience drills.

### ⛔ Not for

- Single-node PoCs with no durable memory requirements.
- Low-risk experiments without revocation evidence requirements.
- Teams lacking minimum data-governance engineering capacity.

## 14. Five-step rollout guide

1. Normalize memory entity schema and versioning policy.
2. Integrate mutation and revocation events through Change Feed.
3. Enforce lease-based session-version checks.
4. Build immutable tombstone ledgers and replay tooling.
5. Continuously reduce revocation window via drill-driven tuning.

## 15. Core KPIs

- `revoke_window_p95_seconds`
- `ghost_hit_rate`
- `session_stale_block_rate`
- `replay_coverage`
- `index_drift_incidents`
- `revocation_audit_pass_rate`

## 16. Final recommendation

Use Route A by default, add Route B for critical low-latency withdrawal domains, and add Route C wherever compliance replayability is mandatory; explicitly prohibit Route D.

One-line conclusion:

**The real durability of enterprise memory systems is measured by provable revocation correctness under load, not by ingestion throughput peaks.**
