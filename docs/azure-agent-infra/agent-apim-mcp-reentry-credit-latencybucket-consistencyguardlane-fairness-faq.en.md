# APIM MCP Reentry Consistency Guard Lane Latency Bucket FAQ (Cycle 2152)

## 1) Three-minute decision
- **Core issue:** under reentry peaks with configuration inconsistency, latency-bucket recovery ordering drifts across tenants.
- **Recommendation:** deploy a closed loop of consistency guard lanes, latency-bucket allocation, tiered downgrade gates, and replay audit.
- **Goal:** preserve tenant fairness while bounding jitter and rollback blast radius during consistency windows.

## 2) Q1 / Q2 / Q3
### Q1: Why add a consistency guard lane?
1) unified throttling cannot constrain inconsistency-driven skew,
2) consistency lanes isolate high-risk sessions and verify config state first,
3) with latency buckets, capacity, validity, ordering, and consistency boundaries are jointly governed.

### Q2: How should production layering work?
- **Consistency guard layer:** map tenant class + consistency state into recovery lanes,
- **Allocation layer:** issue lane-scoped bucket credit and renewal windows,
- **Reclaim layer:** reclaim overdraft leases with anti-oscillation windows,
- **Downgrade layer:** trigger tiered downgrade and human replay queue during peaks.

### Q3: What is the go-live bar?
- unfenced drift rate keeps declining,
- consistency-lane variance remains stable and traceable,
- recovery gain (percentage points) stays reproducible across windows.

## 3) Reference architecture
```mermaid
flowchart LR
  A[MCP Reentry Ingress] --> B[Consistency Guard Lane Classifier]
  B --> C[Credit Latency Bucket Controller]
  C --> D[Consistency-aware Recovery Scheduler]
  D --> E[Overload Downgrade Gate]
  E --> F[Consistency Priority Lane]
  E --> G[Standard Lane]
  F --> H[Gateway Response]
  G --> H
  H --> I[Audit Replay Timeline]
```

## 4) Decision matrix
| Dimension | Unified latency bucket | Consistency Guard Lane + Latency Bucket (Recommended) |
|---|---:|---:|
| Consistency drift control | Medium | High |
| Cross-tenant jitter suppression | Medium | High |
| Consistency replay explainability | Medium | High |
| Peak downgrade stability | Medium | High |
| Implementation complexity | Medium | Medium-High |

## 5) Evidence and reproducibility
- Evidence files:
  - `workshops/azure-agent-infra/evidence/cycle-2152-benchmark.json`
  - `workshops/azure-agent-infra/evidence/cycle-2152-latency.csv`
  - `workshops/azure-agent-infra/evidence/cycle-2152-throughput.csv`
  - `workshops/azure-agent-infra/evidence/cycle-2152-cost.csv`
- Contract keys:
  - `percentage_contract.unfenced_reentry_consistencyguardlane_drift_rate`
  - `percentage_contract.reentry_consistencyguardlane_variance_rate`
  - `percentage_contract.reentry_consistencyguardlane_recovery_gain_pts`
- Labels: `measured / synthetic / anonymized_reported`.

## 6) Production checklist
- [ ] Consistency-lane classification is tenant-class replayable.
- [ ] Bucket grant/renew/reclaim is consistency-lane consistent.
- [ ] Consistency-priority lanes stay starvation-free during peak downgrade.
- [ ] Audit trails bind policy version, consistency class, and recovery path.
- [ ] FAQ/H5/SVG/7-card/evidence stay cycle-topic consistent.
