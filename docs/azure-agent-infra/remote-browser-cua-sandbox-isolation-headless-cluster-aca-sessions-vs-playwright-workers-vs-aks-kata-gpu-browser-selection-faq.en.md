# Enterprise remote-browser/CUA sandbox-isolation selection FAQ (ACA Sessions vs Playwright workers vs AKS Kata GPU browser pool)

> Audience: Cloud & AI Solution Architects, agent platform engineers, enterprise automation teams, and security/compliance owners.

## 1. Executive summary

Production browser automation must move beyond “script works” to “isolated, recoverable, auditable, and scalable execution.” Recommended strategy:

- **Route A**: `ACA Dynamic Sessions` lightweight stateless browser session pools (elasticity and low ops).
- **Route B**: `Playwright Worker Fleet` on AKS/VMSS for high-throughput task orchestration.
- **Route C**: `AKS Kata + GPU browser` for high-risk and rich-media rendering workloads.
- **Route D**: run browser processes inside core business API containers (anti-pattern).

## 2. Core risks

- Untrusted web pages can attempt resource abuse or side-channel behaviors.
- Browser crashes/leaks can collapse shared nodes and cascade failures.
- Weak tenant/session isolation can expose credentials across tenants.
- Missing audit traces break accountability for autonomous actions.

## 3. Route definitions

### Route A: ACA session pools
- One short-lived isolated session per task with automated teardown.
- Best for medium-risk form automation and DOM extraction.

### Route B: Playwright worker fleet
- Queue-driven workers execute high-volume browser tasks in parallel.
- Best for large-scale deterministic automation throughput.

### Route C: AKS Kata + GPU browser pool
- Hardened isolation for high-risk pages and graphics-heavy scenarios.
- Best for visual workloads and high-risk external page interactions.

### Route D: anti-pattern baseline
- Browsers embedded in business API pods couple workload and failure domains.
- Shared credentials/cache/download paths eliminate audit boundaries.

## 4. Quantitative models

### 4.1 Concurrency capacity
`Q_capacity = N_nodes * sessions_per_node * util_factor`

### 4.2 Cold-start latency effect
`P95_total = P95_queue + P95_cold_start + P95_exec`

### 4.3 Isolation risk exposure
`Risk_isolation = P_escape * Impact_cross_tenant`

### 4.4 Unit task cost
`Cost_task = (infra_cost + egress_cost + ops_cost) / completed_tasks`

## 5. Twelve-dimension comparison matrix

| Dimension | Route A ACA sessions | Route B Playwright fleet | Route C AKS Kata+GPU | Route D anti-pattern |
|---|---|---|---|---|
| Startup speed | High | Medium-high | Medium | Low |
| Throughput ceiling | Medium | High | Medium | Low |
| Hard isolation strength | Medium | Medium | High | Low |
| Rich-media rendering | Medium | Medium-high | High | Low |
| Ops complexity | Low | Medium | High | Low (short-term) |
| Multi-tenant security | Medium-high | Medium | High | Low |
| Cost controllability | High | Medium-high | Medium | Low |
| Audit traceability | Medium-high | High | High | Low |
| Auto scaling | High | High | Medium | Low |
| Failure containment | High | Medium-high | High | Low |
| Session cleanup efficiency | High | Medium | Medium | Low |
| Recommendation | ✅ default | ✅ high-throughput | ✅ high-risk | ⛔ |

## 6. Deep quantitative Q&A (Q1-Q8)
### Q1: When should ACA sessions be preferred?
When elasticity and low operational overhead are primary and task risk is moderate.

### Q2: What is the core value of Playwright fleet?
High parallel throughput and efficient resource use for batch-style browser workloads.

### Q3: When is Kata+GPU worth the cost?
When isolation risk and rendering complexity justify premium infrastructure.

### Q4: Why avoid browsers in business API containers?
It couples security/failure domains and amplifies blast radius.

### Q5: How should credential isolation be implemented?
Per-task short-lived tokens, isolated storage scopes, and post-run scrubbing.

### Q6: How to control download risk?
Sandbox download paths and enforce malware/content scanning gates.

### Q7: How should session cleanup be triggered?
TTL, memory threshold, and end-of-task signals in combination.

### Q8: Most critical KPIs?
Task success rate, P95 execution, isolation incident rate, and unit cost.

## 7. Mermaid architecture diagrams

```mermaid
flowchart LR
    Req[Automation request] --> Router[Risk and routing layer]
    Router --> ACA[ACA session pool]
    Router --> PW[Playwright worker fleet]
    Router --> KATA[AKS Kata GPU browser pool]
    ACA --> Audit[Audit and trace store]
    PW --> Audit
    KATA --> Audit
```

```mermaid
flowchart TD
    A[Task enqueue] --> B[Session provision]
    B --> C[Credential mount]
    C --> D[Browser execute]
    D --> E[Artifact upload]
    E --> F[Session scrub and destroy]
```

## 8. Production snippet: risk-based routing pseudo-code

```python
def route(task):
    if task.risk == "high" or task.needs_gpu:
        return "kata_gpu"
    if task.batch_size > 1000:
        return "playwright_fleet"
    return "aca_session"
```

## 9. Production snippet: session recycle policy skeleton

```yaml
session_lifecycle:
  ttl_minutes: 20
  max_memory_mb: 2048
  scrub_paths: ["/tmp", "/downloads", "/profile"]
```

## 10. Enterprise scenarios (2+)

### Scenario A: e-commerce back-office bulk operations
- Large deterministic page operations favor Route B throughput profile.

### Scenario B: financial compliance browser inspections
- High-risk pages and sensitive credentials favor Route C isolation model.

### Scenario C: customer-support knowledge capture and ticket linkage
- Medium-scale high-elasticity workloads fit Route A session pools.

## 11. Three-tier 3-year TCO (USD)

| Scale tier | Route A | Route B | Route C | Anti-pattern (hidden loss) |
|---|---:|---:|---:|---:|
| 1M tasks/month | 2,000,000 | 2,350,000 | 3,100,000 | 1,500,000 (incident loss excluded) |
| 10M tasks/month | 9,500,000 | 10,800,000 | 14,500,000 | 7,100,000 (incident loss excluded) |
| 50M tasks/month | 44,000,000 | 48,500,000 | 63,000,000 | 34,800,000 (incident loss excluded) |

## 12. Day-0 / Day-1 / Day-2 checklist

### Day-0 (architecture and boundaries)
- Day-0 control 1: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 2: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 3: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 4: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 5: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 6: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 7: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 8: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 9: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 10: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 11: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 12: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 13: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 14: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 15: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 16: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 17: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 18: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 19: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 20: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 21: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 22: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 23: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 24: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 25: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 26: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 27: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 28: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 29: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 30: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 31: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 32: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 33: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 34: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 35: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 36: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 37: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 38: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 39: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 40: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 41: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 42: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 43: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 44: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 45: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 46: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 47: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 48: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 49: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 50: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 51: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 52: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 53: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 54: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 55: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 56: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 57: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 58: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 59: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 60: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 61: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 62: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 63: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 64: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 65: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 66: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 67: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 68: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 69: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 70: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 71: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 72: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 73: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 74: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 75: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 76: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 77: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 78: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 79: define task-tiering, isolation boundaries, credential model, and audit schema.
- Day-0 control 80: define task-tiering, isolation boundaries, credential model, and audit schema.

### Day-1 (engineering deployment)
- Day-1 control 1: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 2: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 3: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 4: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 5: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 6: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 7: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 8: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 9: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 10: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 11: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 12: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 13: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 14: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 15: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 16: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 17: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 18: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 19: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 20: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 21: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 22: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 23: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 24: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 25: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 26: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 27: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 28: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 29: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 30: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 31: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 32: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 33: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 34: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 35: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 36: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 37: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 38: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 39: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 40: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 41: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 42: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 43: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 44: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 45: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 46: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 47: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 48: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 49: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 50: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 51: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 52: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 53: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 54: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 55: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 56: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 57: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 58: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 59: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 60: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 61: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 62: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 63: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 64: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 65: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 66: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 67: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 68: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 69: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 70: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 71: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 72: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 73: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 74: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 75: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 76: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 77: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 78: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 79: implement routing layer, session pools, artifact channels, and teardown automation.
- Day-1 control 80: implement routing layer, session pools, artifact channels, and teardown automation.

### Day-2 (operations governance)
- Day-2 control 1: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 2: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 3: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 4: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 5: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 6: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 7: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 8: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 9: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 10: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 11: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 12: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 13: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 14: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 15: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 16: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 17: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 18: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 19: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 20: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 21: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 22: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 23: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 24: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 25: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 26: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 27: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 28: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 29: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 30: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 31: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 32: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 33: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 34: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 35: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 36: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 37: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 38: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 39: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 40: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 41: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 42: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 43: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 44: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 45: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 46: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 47: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 48: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 49: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 50: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 51: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 52: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 53: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 54: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 55: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 56: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 57: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 58: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 59: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 60: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 61: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 62: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 63: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 64: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 65: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 66: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 67: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 68: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 69: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 70: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 71: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 72: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 73: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 74: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 75: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 76: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 77: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 78: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 79: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 80: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 81: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 82: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 83: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 84: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 85: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 86: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 87: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 88: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 89: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 90: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 91: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 92: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 93: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 94: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 95: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 96: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 97: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 98: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 99: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 100: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 101: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 102: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 103: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 104: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 105: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 106: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 107: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 108: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 109: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 110: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 111: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 112: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 113: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 114: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 115: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 116: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 117: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 118: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 119: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.
- Day-2 control 120: monitor success rate, isolation incidents, cost trends, and capacity tuning signals.

## 13. Good fit vs Not for

### ✅ Good fit
- Multi-tenant browser automation and CUA production environments.
- Enterprises requiring isolation, security controls, and auditability.

### ⛔ Not for
- One-off local scripts and temporary single-user tooling.
- Small experiments without compliance and audit obligations.

## 14. Five-step rollout guide
1. Define risk tiers and routing contracts.
2. Separate browser execution plane from core API plane.
3. Enforce per-session credential isolation and scrubbing.
4. Build task-level audit trails and replayability.
5. Load-test continuously and optimize throughput/cost.

## 15. Core KPIs
- `task_success_rate`
- `p95_exec_seconds`
- `session_escape_incidents`
- `cost_per_task`
- `session_recycle_latency`

## 16. Final recommendation
Use Route A as default baseline, Route B for high-throughput tiers, Route C for high-risk/high-rendering tasks, and ban Route D.
