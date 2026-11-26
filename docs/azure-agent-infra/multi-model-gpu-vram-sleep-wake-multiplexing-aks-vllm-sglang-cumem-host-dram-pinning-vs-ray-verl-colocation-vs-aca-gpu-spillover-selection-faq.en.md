# Enterprise multi-model GPU VRAM sleep-wake multiplexing selection FAQ: AKS vLLM/SGLang cuMem host-DRAM pinning vs Ray+veRL co-location vs ACA GPU spillover

> 🌐 **Language / 语言切换**: **English (Current)** | **[中文](multi-model-gpu-vram-sleep-wake-multiplexing-aks-vllm-sglang-cumem-host-dram-pinning-vs-ray-verl-colocation-vs-aca-gpu-spillover-selection-faq.md)** | **[Back to catalog](README.en.md)**

## 1. Executive summary

- **Primary path**: enable `vLLM/SGLang` sleep mode on `AKS ND/NC H100` pools and offload weights to host DDR5 via `cuMemCreate/cuMemUnmap` while preserving CUDA Graph and NCCL context for `<0.6~1.5s` wake.
- **Secondary path**: for RL rollout plus training co-location, use `Ray + veRL/OpenRLHF` so inference engines enter Level-1 sleep during optimizer steps.
- **Spillover path**: route low-duty overnight traffic to `ACA Serverless GPU` or `APIM -> Foundry MaaS`, accepting `8~25s` cold starts for near-zero idle spend.
- **Anti-pattern warning**: naive MPS time slicing across multiple full-weight active engines without `cuMemUnmap` causes deterministic HBM OOM under bursty KV growth.
- **Target metrics**: `WakeP95 <= 1.5s`, `HBMHit >= 0.9`, `GPUUtil >= 0.55`, and declining hourly idle cost.

## 2. Route definitions

- **Route A**: `AKS + vLLM/SGLang Sleep Mode` plus Envoy hold-queue orchestrator; multiplex `4~8` heterogeneous base models per GPU pool.
- **Route B**: `Ray + veRL/OpenRLHF` same-GPU temporal co-location: inference sleeps during training steps and wakes between windows.
- **Route C**: `ACA Serverless GPU` or `KServe` triggers with `APIM` spillover to `Foundry MaaS`.
- **Route D (forbidden)**: static dedicated GPU pools per low-duty model or naive shared MPS co-packing.

## 3. Reference architecture (Mermaid 1)

```mermaid
flowchart LR
    U[Client Traffic] --> G[APIM plus Envoy Queue]
    G --> A[AKS vLLM or SGLang Sleep Pool]
    G --> C[ACA or Foundry Spillover]
    A --> M[(Model State Registry)]
    A --> P[(Prometheus and Kusto)]
    B[Ray Training Plane] <--> A
    B --> L[(veRL Rollout Ledger)]
```

## 4. Sleep-wake orchestration timeline (Mermaid 2)

```mermaid
sequenceDiagram
    participant Q as Envoy Hold Queue
    participant C as Controller
    participant V as vLLM Engine
    participant H as Host DRAM Tier
    Q->>C: backlog and SLO pressure
    C->>V: sleep(level1) or wake(modelId)
    V->>H: map or unmap weight pages
    V-->>C: wake latency telemetry
    C-->>Q: admit or defer decision
```

## 5. Quantitative models

1. **HBM budget**: `M_base + M_kv(active) + M_fragment + M_guard <= M_hbm`.
2. **Sleep gain**: `Gain_sleep = (Idle_hours * GPU_hourly_price) - Wake_penalty_cost`.
3. **Wake latency**: `T_wake = T_map + T_prefetch + T_graph_rebind + T_first_token`.
4. **Co-location efficiency**: `Eff_colocate = (Util_infer + Util_train - overlap_penalty)`.
5. **Spillover trigger**: `if queue_wait_p95 > SLO and wake_slots_exhausted => spillover`.

## 6. Twelve-dimension comparison matrix

| Dimension | Route A AKS Sleep | Route B Ray+veRL Co-location | Route C ACA/Foundry Spillover | Route D Anti-pattern |
|---|---|---|---|---|
| HBM multiplexing efficiency | High | High | Medium | Low |
| Wake latency | Very low | Low to medium | High | Unstable |
| Training friendliness | Medium | Very high | Low | Low |
| Operational complexity | Medium | High | Low | Low surface, high hidden risk |
| Cost elasticity | High | High | Medium | Low |
| OOM risk | Low | Medium | Low | Critical |
| Multi-model concurrency | High | Medium | Medium | Low |
| SLA controllability | High | Medium-high | Medium | Low |
| Observability depth | High | High | Medium | Low |
| Supply chain dependency | Medium | High | Medium | Low |
| Three-year TCO | Low to medium | Medium | Medium | High |
| Anti-pattern exposure | Low | Medium | Medium | Critical |

## 7. Deep Q&A (Q1–Q8)

### Q1. When should Route A be prioritized?
- Prioritize Route A when model variety is high and per-model duty cycle is below 35 percent.
- Use Route A when sub-second or near-sub-second wake is required.
- Route A is strong when preserving CUDA Graph and NCCL context is critical.
- Adopt staged rollout by model heat tiers.

### Q2. Where does Route B add value?
- It removes dedicated dual-cluster waste for RL rollout and training loops.
- Inference sleeps during optimizer-heavy intervals and wakes between windows.
- It is effective for cyclical RLHF workloads with clear peaks and troughs.
- It requires robust scheduling and rollback controls.

### Q3. When is Route C spillover needed?
- When overnight duty cycle is low and idle cost must approach zero.
- When local wake slots are temporarily exhausted during spikes.
- When regional GPU capacity is constrained and fallback is required.
- For lower-priority requests that can tolerate cold starts.

### Q4. How can OOM be prevented?
- Separate active KV, sleeping weights, and fragmentation guard budgets.
- Set strict limits for `max_num_seqs` and GPU memory utilization.
- Pre-warm hot models before peak windows.
- Apply per-tenant queue fairness and admission controls.

### Q5. How should spillover thresholds be defined?
- Use combined signals from queue wait P95, free wake slots, and error rate.
- Apply time-of-day threshold profiles instead of static global numbers.
- Keep premium traffic local while spilling lower-priority classes first.
- Reserve control-plane headroom for governance calls.

### Q6. How should cost attribution be modeled?
- Split GPU minutes by model, tenant, and request class.
- Account for wake failure retries and delayed queue penalties.
- Translate spillover cold starts into opportunity cost when relevant.
- Combine infra spend with SLO breach cost to quantify ROI.

### Q7. What telemetry closes the loop?
- Track wake latency, HBM hit ratio, spillover ratio, and OOM event rate.
- Record train-infer contention events in a common event bus.
- Generate daily and weekly replay dashboards for tuning decisions.
- Automate downgrade recommendations for unstable models.

### Q8. How should phased rollout be executed?
- Start with low-risk tenant pools and a two-week observation window.
- Expand gradually using feature flags and ring-based rollout.
- Maintain one-click rollback to static allocation mode.
- Review model heat maps weekly and retune allocation classes.

## 8. vLLM sleep-mode and controller snippets

```bash
vllm serve /models/base-model-a \
  --tensor-parallel-size 4 \
  --enable-sleep-mode \
  --gpu-memory-utilization 0.88 \
  --max-num-seqs 256 \
  --swap-space 16
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep-wake-controller
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: controller
        image: ghcr.io/example/sleep-wake-controller:2026.11
        env:
        - name: WAKE_P95_TARGET_MS
          value: "1500"
        - name: SPILLOVER_QUEUE_TARGET_MS
          value: "2200"
```

## 9. Envoy queue and routing snippet

```yaml
route_config:
  virtual_hosts:
  - name: llm-host
    routes:
    - match: { prefix: "/v1/chat/completions" }
      route:
        cluster: aks-sleep-pool
        timeout: 30s
        retry_policy:
          retry_on: 5xx,reset
          num_retries: 2
```

## 10. Enterprise scenario A: vertical SaaS with 8 to 20 models

- Scenario A point 01: apply model heat-tiering, keep hot models in Level-1 sleep, and wake long-tail models on demand with P95 guardrails.

## 11. Enterprise scenario B: RLHF train-plus-infer same-GPU platform

- Scenario B point 01: switch inference sleep levels by training phase and enforce queue priorities so online traffic remains protected.

## 12. Three-tier three-year TCO table (USD)

| Tier | Model count | Route A annual | Route B annual | Route C annual | Anti-pattern loss | Three-year total (recommended mix) |
|---|---:|---:|---:|---:|---:|---:|
| Small | 8 | 480,000 | 520,000 | 210,000 | 380,000 | 3,150,000 |
| Medium | 16 | 1,920,000 | 2,140,000 | 680,000 | 1,260,000 | 12,120,000 |
| Large | 30 | 5,760,000 | 6,440,000 | 1,980,000 | 3,800,000 | 35,940,000 |

## 13. Day-0 / Day-1 / Day-2 readiness checklist

### Day-0 design phase
- [ ] Day-0 design phase check 01: verify thresholds, rollback hooks, capacity budgets, owner, alerting path, and evidence retention.

### Day-1 rollout phase
- [ ] Day-1 rollout phase check 01: verify thresholds, rollback hooks, capacity budgets, owner, alerting path, and evidence retention.

### Day-2 operations phase
- [ ] Day-2 operations phase check 01: verify thresholds, rollback hooks, capacity budgets, owner, alerting path, and evidence retention.

## 14. Anti-pattern breakdown and remediation

- Remediation 01: prohibit static low-duty dedicated GPU pools and require sleep-wake or serverless spillover governance.

## 15. Appendix: parameter and governance control catalog

- capture weekly trends for `wake_p95_ms`, `hbm_hit_ratio`, `spillover_ratio`, `oom_event_rate`, `gpu_idle_cost`, and `tenant_fairness`.
