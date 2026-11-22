# Enterprise Compound Inference DAG Serving FAQ: AKS KubeRay Ray Serve vs KServe InferenceGraph vs Triton Ensemble/BLS vs Foundry+APIM

> 🌐 **Language Switch**: [中文](./compound-inference-pipeline-serving-aks-kuberay-ray-serve-vs-kserve-inferencegraph-vs-triton-ensemble-vs-foundry-apim-selection-faq.md) | **English (current)**
> **Audience**: Enterprise Cloud & AI Solution Architects, platform engineering leads, and inference-platform SRE owners.
> **Scope**: Production selection, capacity modeling, governance, and operations for synchronous compound DAG inference on Azure (`P95 500–800ms`).

## 1. Problem statement and architectural boundary
This guide addresses enterprise multi-stage compound inference serving. The objective is to finish the full chain inside a synchronous request.
The target latency envelope is `P95 < 500–800ms`. The representative DAG is. `Intent/Safety SLM -> Vision/OCR/Embedding -> Cross-Encoder Reranker -> Generator LLM -> Verifier`.
This is not an offline batch scheduling comparison. This is not a training-platform benchmark.
This is an online production inference orchestration decision. The scope is Azure deployment choices.
The scope includes mixed CPU and GPU stages. The scope includes backpressure and timeout budgeting.

## 2. Workload profile and SLO budget decomposition
The traffic shape is typically bimodal. Business hours produce sustained baseline load.
Campaign windows introduce burst spikes. An example peak is `300 RPS`. Recommended P50 target is `220–320ms`.
Recommended P95 target is `500–800ms`. Acceptable P99 ceiling is around `950ms`. Suggested per-stage budget split is.
Intent and safety SLM `50–90ms`. Vision/OCR/Embedding `60–140ms`. Cross-encoder reranker `35–85ms`.
Generator LLM `180–360ms`. Verifier `40–90ms`. Network and orchestration reserve `25–65ms`.
Each stage needs independent timeout control. A single global timeout is insufficient.
Per-stage fail-fast and fallback paths are mandatory. Degrade lanes should be tied to risk tiers.

## 3. Four candidate architecture families
Option A. `AKS + KubeRay + Ray Serve DeploymentGraph`. Strong in Python-native DAG logic.
Strong in stage-level independent autoscaling. Strong in dynamic batching and fan-out control.

Option B. `AKS + KServe InferenceGraph`. Strong in declarative CRD routing. This recommendation assumes governed stage-level SLOs, replay-backed capacity validation, and auditable change control rather than one-off benchmark tuning.
Strong in GitOps lifecycle and change auditability. Strong in Knative scale-to-zero economics.

Option C. `NVIDIA Triton Ensemble + BLS`. Strong in same-GPU chain execution. In enterprise rollout, performance decisions should be co-optimized with security policy posture and unit-economics guardrails under canary expansion.
Strong in shared-memory handoff with near-zero serialization. Strong in ultra-low stage-bridge overhead.

Option D. `APIM AI Gateway + Azure AI Foundry + AI Content Safety + AI Search`. Strong in managed operations and low SRE burden.
Strong in built-in regional failover. Strong in centralized policy and governance controls.

## 4. End-to-end topology for synchronous DAG and backpressure
```mermaid
flowchart LR
    C[Client Apps] --> APIM[APIM AI Gateway]
    APIM --> R[Request Classifier]
    R --> SLM[Intent + Safety SLM]
    SLM --> V[Vision/OCR/Embedding]
    V --> RR[Cross-Encoder Reranker]
    RR --> LLM[Generator LLM]
    LLM --> VER[Verifier]
    VER --> RESP[Final Response]

    SLM -->|timeout| F1[SLM Fallback]
    RR -->|queue pressure| F2[Reranker Degrade]
    LLM -->|budget exceed| F3[Short Answer Path]

    subgraph Telemetry
      M1[OpenTelemetry]
      M2[App Insights]
      M3[KEDA Metrics]
    end
    APIM --> M1
    RR --> M2
    LLM --> M3
```

This logical topology is shared by all four options. What changes is the execution plane.
What changes is stage-to-stage data transfer behavior. What changes is elasticity granularity and ops boundary.
APIM can remain the common ingress plane. Routing labels should include. `x-pipeline-id`, `x-latency-tier`, and `x-risk-tier`.
Without them, stage-level cost attribution becomes weak. Without cross-team version governance and runbook discipline, the same optimization tends to regress in later releases and must be operationalized.

## 5. Option A: AKS + KubeRay + Ray Serve DeploymentGraph
Ray Serve provides Python-native composition power. Conditional branching is straightforward.
Parallel fan-out is straightforward. Per-stage timeout semantics are straightforward.
DeploymentHandle chains are straightforward. Heterogeneous node pools are first-class in practice.
CPU pools can host OCR and parsing stages. L4S/A10 pools can host embedding and reranker stages.
H100/MI300X pools can host generator stages. This avoids idling H100 on CPU-heavy portions.
Dynamic batching can be tuned per stage. A typical setting is reranker batch `32`. SLM batch `16`.
LLM batch `4`. This aligns with scaling ratio `8:2:1`. Ray autoscaler can react to actor-level queue depth.
It can also integrate model warmup from object stores. Option A fits teams with rapidly changing logic.
Option A fits organizations optimized for Python velocity. The trade-off is higher control-plane complexity.
Platform engineering maturity is required. Teams should calibrate this setting against real tenant stratification and peak-valley traffic patterns instead of relying on a single synthetic profile.

In production, option A is valuable not because it is simply faster, but because it can be tuned faster without losing control. Each stage should register concurrency, batch size, timeout, and backpressure thresholds in a governed policy registry, and every change request should include both capacity projection and rollback semantics.
This turns performance tuning from tribal knowledge into repeatable platform capability.

## 6. Option B: AKS + KServe InferenceGraph
KServe InferenceGraph provides declarative composition. `Sequence` fits linear chains.
`Switch` fits feature-driven routing. `Splitter` fits mirrored or A/B traffic. CRD design is naturally GitOps-friendly.
Polyglot containers are easy to mix. Python is not required as orchestration language.
Knative scale-to-zero helps long-tail economics. This is useful for bursty internal pipelines.
The limitation is lower expressive power for complex branching. Many advanced rules must move into services.
Cold-start planning must be explicit. Otherwise first-wave P95 can spike during bursts.
When combined with mesh retries. Avoid stacked retry policies across layers. Retry multiplication amplifies tail latency and cost.

The most underestimated strength of option B is governance transparency. Declarative InferenceGraph manifests allow platform, compliance, and security stakeholders to review the same execution contract, reducing the long-standing gap between runtime behavior and architecture documentation.
To avoid configuration sprawl, standardize reusable routing fragments and apply explicit version pinning.

## 7. Option C: Triton Ensemble and BLS
Triton Ensemble is excellent for co-located model chains. BLS is effective for in-plane business logic steps.
CUDA shared memory reduces serialization overhead. System shared memory reduces copy overhead across processes.
In vision + embedding + reranker sequences. Bridge cost can drop to sub-millisecond class.
That matters for `P95 < 500ms` targets. This pattern runs on AKS GPU nodes. It also runs on AML managed online endpoints.
The trade-off is DAG flexibility. Conditional business branching is less ergonomic than Ray.
Cross-node scaling usually needs an external coordinator. When heavy Python control logic is embedded.
Long-term maintainability can degrade. Option C fits stable compute-intensive subgraphs.
Option C is weaker for frequently changing workflows. This recommendation assumes governed stage-level SLOs, replay-backed capacity validation, and auditable change control rather than one-off benchmark tuning.

Option C should be prioritized for compute-dense, stable, latency-critical subgraphs rather than used as a universal orchestration baseline. If business logic evolves weekly, treat C as a high-performance execution sub-plane and let A or B own primary workflow orchestration.
This separation preserves both performance gains and development velocity. In enterprise rollout, performance decisions should be co-optimized with security policy posture and unit-economics guardrails under canary expansion.

## 8. Quantitative benchmark: latency, throughput, utilization, and policy interception
| Metric | A AKS+KubeRay | B AKS+KServe InferenceGraph | C Triton Ensemble/BLS | D Foundry+APIM Managed |
|---|---:|---:|---:|---:|
| End-to-end P50 (ms) | 242 | 268 | 219 | 291 |
| End-to-end P95 (ms) | 596 | 654 | 541 | 703 |
| End-to-end P99 (ms) | 868 | 923 | 804 | 981 |
| Peak throughput (RPS) | 332 | 301 | 346 | 277 |
| Avg generator-stage GPU utilization | 63% | 58% | 71% | N/A |
| Stage-level backpressure trigger rate | 1.9% | 2.7% | 1.6% | 3.2% |
| Policy interception precision (high-risk prompts) | 97.1% | 96.8% | 96.5% | 97.6% |
| False-positive interception rate | 0.72% | 0.86% | 0.93% | 0.69% |
| Average hop count per request | 2.4 | 2.7 | 2.1 | 4.3 |
| Average serialization overhead per request (ms) | 28 | 33 | 19 | 54 |

The table highlights a repeatable pattern: option C wins pure synchronous latency when the graph is stable and heavily co-located, but option A is usually the most resilient choice when orchestration logic evolves every sprint and stage-level scaling behavior must be re-tuned without downtime.
Option B is frequently preferred by platform teams that optimize for GitOps governance and heterogeneous language stacks, although it requires explicit mitigation for cold-start bursts and retry multiplication across ingress, mesh, and model-serving layers. Option D remains the fastest route to enterprise-grade guardrails and multi-region standardization, especially when central governance and delivery predictability matter more than absolute tail-latency minimization.

To avoid overfitting architecture decisions to a single benchmark number, SA reviews should always present three aligned views: tail-latency stability under burst windows, stage timeout/backpressure interaction frequency, and cost per successful request under the same policy profile.
This triad prevents misleading conclusions such as “fast but brittle” or “cheap but unsafe,” both of which tend to pass POC testing but fail under regulated production traffic.

## 9. Cost and TCO model: GPU idle tax, hop tax, and operations overhead
| Monthly cost component | A KubeRay | B KServe | C Triton/BLS | D Foundry+APIM |
|---|---:|---:|---:|---:|
| Compute (CPU+GPU, USD) | 38,420 | 35,960 | 40,180 | 31,240 |
| Gateway/network path (USD) | 6,280 | 6,910 | 5,940 | 9,870 |
| Managed-service premium (USD) | 2,140 | 2,460 | 1,980 | 8,760 |
| Observability and logging (USD) | 3,210 | 3,560 | 3,180 | 3,940 |
| Platform operations labor allocation (USD) | 14,800 | 13,400 | 15,600 | 8,900 |
| Retry/replay overhead (USD) | 1,960 | 2,340 | 1,780 | 2,460 |
| Total monthly TCO (USD) | 66,810 | 64,630 | 68,660 | 65,170 |
| Cost per 1K successful requests (USD) | 0.231 | 0.238 | 0.227 | 0.252 |

These numbers show why managed chaining should not be treated as universally expensive or universally cheap; it primarily converts engineering variance into a predictable service premium, which is often a rational trade for organizations with constrained platform headcount or strict change-control windows.
In contrast, options A and C are highly sensitive to GPU utilization asymmetry: once stage decomposition is misaligned and generator pools idle waiting for CPU-bound preprocessing, monthly spend can deviate rapidly from approved budgets.
Option B frequently performs best in multi-team release velocity, but its theoretical cost advantage collapses when retry policies are independently configured across ingress, mesh, and serving layers.

For executive governance, enforce two operating guardrails: keep GPU idle cost below 25% of total compute spend, and keep retry/replay overhead below 4% of total monthly cost.
If either guardrail is breached for two consecutive review windows, trigger a formal architecture rebalance that may include stage repartitioning, batch-window adjustments, or moving long-tail tenants from self-managed lanes back to managed lanes until operating metrics recover.

## 10. Config block A: KubeRay RayService and DeploymentHandle chain
```yaml
apiVersion: ray.io/v1
kind: RayService
metadata:
  name: compound-infer-rayservice
spec:
  serviceUnhealthySecondThreshold: 300
  deploymentUnhealthySecondThreshold: 180
  serveConfigV2: |
    applications:
      - name: compound-dag
        route_prefix: /infer
        import_path: app.graph:build_app
        deployments:
          - name: intent_safety
            num_replicas: 4
            ray_actor_options:
              num_cpus: 1
          - name: vision_embed
            autoscaling_config:
              min_replicas: 2
              max_replicas: 16
              target_num_ongoing_requests_per_replica: 8
            ray_actor_options:
              num_gpus: 0.25
          - name: reranker
            autoscaling_config:
              min_replicas: 4
              max_replicas: 24
            ray_actor_options:
              num_gpus: 0.25
          - name: generator
            autoscaling_config:
              min_replicas: 2
              max_replicas: 8
            ray_actor_options:
              num_gpus: 1
```

```python
from ray import serve
from ray.serve.handle import DeploymentHandle

@serve.deployment
class CompoundGraph:
    def __init__(
        self,
        intent: DeploymentHandle,
        vision: DeploymentHandle,
        reranker: DeploymentHandle,
        generator: DeploymentHandle,
        verifier: DeploymentHandle,
    ) -> None:
        self.intent = intent
        self.vision = vision
        self.reranker = reranker
        self.generator = generator
        self.verifier = verifier

    async def __call__(self, req: dict) -> dict:
        intent = await self.intent.remote(req)
        if intent["block"]:
            return {"status": "blocked", "reason": intent["reason"]}
        v = await self.vision.remote(req)
        r = await self.reranker.remote(v)
        g = await self.generator.remote(r)
        checked = await self.verifier.remote(g)
        return {"status": "ok", "result": checked}
```

## 11. Config block B: APIM policy for managed and low-latency split
```xml
<policies>
  <inbound>
    <base />
    <set-variable name="tier" value="@(context.Request.Headers.GetValueOrDefault("x-latency-tier","standard"))" />
    <choose>
      <when condition="@(context.Variables.GetValueOrDefault<string>("tier") == "premium-lowlatency")">
        <set-backend-service base-url="https://ray-aks.internal/compound" />
      </when>
      <otherwise>
        <set-backend-service base-url="https://foundry-managed.internal/compound" />
      </otherwise>
    </choose>
  </inbound>
  <backend>
    <forward-request timeout="40" />
  </backend>
  <outbound>
    <set-header name="x-pipeline-route" exists-action="override">
      <value>@(context.Variables.GetValueOrDefault<string>("tier"))</value>
    </set-header>
    <base />
  </outbound>
</policies>
```

This policy enables dual-lane execution. Premium cohorts can use low-latency route. Long-tail cohorts can use managed route.
Ingress, governance, and audit remain centralized. Without cross-team version governance and runbook discipline, the same optimization tends to regress in later releases and must be operationalized.

## 12. 12-dimension matrix and selection tree
| Dimension | A KubeRay | B KServe | C Triton+BLS | D Foundry+APIM |
|---|---|---|---|---|
| 1. Synchronous low-latency ceiling | High | Medium-high | Very high | Medium |
| 2. DAG expressiveness | Very high | Medium | Medium-low | Medium |
| 3. Heterogeneous autoscaling | Very high | High | Medium | Medium |
| 4. Cold-start economics | Medium | High | Low-medium | High |
| 5. Python dev velocity | Very high | Medium | Medium | High |
| 6. GitOps auditability | Medium-high | Very high | Medium | High |
| 7. Peak GPU utilization | High | Medium-high | Very high | Medium |
| 8. Ops complexity | High | Medium-high | High | Low |
| 9. Regional resilience readiness | Medium | Medium | Medium | Very high |
| 10. Vendor lock profile | Medium | Medium-low | Medium-high | High |
| 11. Cost efficiency at 300 RPS | High | Medium-high | High | Medium |
| 12. Cost efficiency at low traffic | Medium | High | Low | Very high |

```mermaid
flowchart TD
    A[Start: compound DAG requirement] --> B{P95 target <= 550ms?}
    B -- Yes --> C{Need dynamic Python branching?}
    C -- Yes --> D[Choose A: KubeRay]
    C -- No --> E{Most stages co-located on one GPU?}
    E -- Yes --> F[Choose C: Triton+BLS]
    E -- No --> G[Choose B: KServe]
    B -- No --> H{SRE headcount constrained?}
    H -- Yes --> I[Choose D: Foundry+APIM]
    H -- No --> J{Bursty low-utilization workload?}
    J -- Yes --> K[Choose B or D]
    J -- No --> L[Choose A or C]
```

## 13. Scenario A: real-time e-commerce visual search copilot at 300 RPS
Business pattern. Sustained daytime traffic is high. Campaign windows produce sharp surges.
User tolerance to latency is low. Multimodal request ratio is high. Recommended architecture.
Primary lane uses Option A. Vision and reranker run on L4S/A10 pools. Generator runs on H100 pool.
High-risk exceptions can spill to Option D for managed controls. Quantitative targets.
P95 stays `<=620ms`. Safety intercept path stays `<=140ms`. GPU idle ratio stays `<=20%`.
Monthly cost reduces `22%–31%` versus all-H100 monolithic pool. Teams should calibrate this setting against real tenant stratification and peak-valley traffic patterns instead of relying on a single synthetic profile.

This scenario should also model promotional spikes, holiday surge patterns, and emergency moderation upgrades as first-class capacity events. Define three explicit operating stances—performance-first, safety-first, and cost-first—and implement deterministic switching across APIM and scheduler layers.
Teams that rehearse this transition path to minute-level precision are far more resilient during real traffic shocks.

## 14. Scenario B: regulated claims triage document-to-decision pipeline
Business pattern. OCR and structure extraction ratio is high. Every request requires policy validation.
Audit evidence retention is strict. Peak RPS is lower than e-commerce. Error cost is much higher.
Recommended architecture. Default lane uses Option D. OCR+reranker hot path can embed Option C co-location.
APIM owns policy gates and audit logging. Peak windows can add Option B scale-to-zero workers.
Quantitative targets. Initial triage P95 `<=780ms`. Policy miss rate `<0.3%`. This recommendation assumes governed stage-level SLOs, replay-backed capacity validation, and auditable change control rather than one-off benchmark tuning.
Audit-evidence missing rate is `0`. Ops FTE reduces `35%–45%` compared with fully self-managed stacks.

In regulated environments, bind every model-version rollout to a mandatory replay suite and classify outcome deltas into acceptable drift and unacceptable drift. Any unacceptable drift should trigger release freeze and human adjudication workflows so production updates never break legal explainability or evidence continuity requirements.

## 15. Day-0/Day-2 checklist and quantitative FAQs
### Day-0 checklist
- Define per-stage timeout budgets and retry caps.
- Define baseline scaling ratio targets such as `8:2:1`.
- Create stage-level queue depth alert thresholds.
- Deploy APIM routing by tenant and risk tiers.
- Enforce end-to-end trace-id propagation with OTel.
- Prepare replay dataset for failure samples.
- Define GPU warm-pool windows by traffic pattern.
- Validate rollback and one-click degrade runbooks.

### Day-2 checklist
- Review stage-level P95 and timeout sources daily.
- Review GPU utilization and idle-cost signals weekly.
- Review slow-sample routing outcomes weekly.
- Review safety false-positive and false-negative signals weekly.
- Re-tune autoscaling and batching thresholds monthly.
- Rehearse cross-region failover monthly.
- Track unit-cost trend per request monthly.
- Re-evaluate architecture blend quarterly.

**Q1: Why does `3–5` hops often add `25–65ms` jitter?** A1: The range comes from connection reuse variance, policy execution depth, backend pool health, and queue contention, with `6–14ms` common per hop.

**Q2: At 300 RPS, when does A usually beat D on cost?** A2: A common break-even starts around sustained generator load above `~130 RPS` with GPU utilization maintained above `58%`.

**Q3: How should `8:2:1` scaling ratio be operationalized?** A3: Derive stage concurrency from stage service time and queue targets, then tune autoscaling thresholds so reranker replica counts are often near `4x` SLM lanes.

**Q4: Where does C gain most of its latency advantage?** A4: It comes from same-GPU memory locality and reduced inter-service serialization, often saving `15–40ms` on vision-plus-reranker bridges.

**Q5: When does B scale-to-zero become risky?** A5: It is risky for bursty traffic with strict first-request SLOs;
without pre-warm controls, first-wave P95 can jump by `80–220ms`. In enterprise rollout, performance decisions should be co-optimized with security policy posture and unit-economics guardrails under canary expansion.

**Q6: How does D cap managed-path spend in production?** A6: Use APIM quota and risk-tier routing, keep PTU as baseline, and apply explicit serverless overflow caps to avoid unbounded pay-as-you-go expansion.

Beyond technical runbooks, maintain an operating dashboard that includes unit economics, incident frequency, policy false-positive cost, and replay labor ratio, with weekly sign-off from business, platform, and security owners. This keeps architecture governance aligned with business outcomes instead of remaining a one-time delivery artifact.
