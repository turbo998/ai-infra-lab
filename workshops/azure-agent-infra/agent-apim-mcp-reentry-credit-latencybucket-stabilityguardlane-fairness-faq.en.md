# APIM MCP Reentry Stability Guard Lane Latency Bucket FAQ (Cycle 2153)

## 1) Three-minute decision
- **Core issue:** under reentry peaks with policy oscillation, latency-bucket recovery ordering drifts in stability.
- **Recommendation:** deploy a closed loop of stability guard lanes, latency-bucket allocation, tiered downgrade gates, and replay audit.
- **Goal:** preserve tenant fairness while bounding jitter and rollback blast radius during stability windows.

## 2) Q1 / Q2 / Q3
### Q1: Why add a stability guard lane?
1) unified throttling cannot constrain oscillation-driven skew,
2) stability lanes isolate high-risk sessions and stabilize critical recovery first,
3) with latency buckets, capacity, validity, ordering, and stability boundaries are jointly governed.

### Q2: How should production layering work?
- **Stability guard layer:** map tenant class + stability state into recovery lanes,
- **Allocation layer:** issue lane-scoped bucket credit and renewal windows,
- **Reclaim layer:** reclaim overdraft leases with anti-oscillation windows,
- **Downgrade layer:** trigger tiered downgrade and human replay queue during peaks.

### Q3: What is the go-live bar?
- unfenced drift rate keeps declining,
- stability-lane variance remains stable and traceable,
- recovery gain (percentage points) stays reproducible across windows.

## 3) Reference architecture
```mermaid
flowchart LR
  A[MCP Reentry Ingress] --> B[Stability Guard Lane Classifier]
  B --> C[Credit Latency Bucket Controller]
  C --> D[Stability-aware Recovery Scheduler]
  D --> E[Overload Downgrade Gate]
  E --> F[Stability Priority Lane]
  E --> G[Standard Lane]
  F --> H[Gateway Response]
  G --> H
  H --> I[Audit Replay Timeline]
```

## 4) Decision matrix
| Dimension | Unified latency bucket | Stability Guard Lane + Latency Bucket (Recommended) |
|---|---:|---:|
| Stability drift control | Medium | High |
| Cross-tenant jitter suppression | Medium | High |
| Stability replay explainability | Medium | High |
| Peak downgrade stability | Medium | High |
| Implementation complexity | Medium | Medium-High |

## 5) Evidence and reproducibility
- Evidence files:
  - `workshops/azure-agent-infra/evidence/cycle-2153-benchmark.json`
  - `workshops/azure-agent-infra/evidence/cycle-2153-latency.csv`
  - `workshops/azure-agent-infra/evidence/cycle-2153-throughput.csv`
  - `workshops/azure-agent-infra/evidence/cycle-2153-cost.csv`
- Contract keys:
  - `percentage_contract.unfenced_reentry_stabilityguardlane_drift_rate`
  - `percentage_contract.reentry_stabilityguardlane_variance_rate`
  - `percentage_contract.reentry_stabilityguardlane_recovery_gain_pts`
- Labels: `measured / synthetic / anonymized_reported`.

## 6) Production checklist
- [ ] Stability-lane classification is tenant-class replayable.
- [ ] Bucket grant/renew/reclaim is stability-lane consistent.
- [ ] Stability-priority lanes stay starvation-free during peak downgrade.
- [ ] Audit trails bind policy version, stability class, and recovery path.
- [ ] FAQ/H5/SVG/7-card/evidence stay cycle-topic consistent.
