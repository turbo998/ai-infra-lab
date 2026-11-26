# 企业级低抖动实时解码调度选型 FAQ：Chunked Prefill、Piecewise CUDA Graph 与 Overlap 运行时（Azure）

> 主题编号：#452 · 架构域：3.2 模型托管、路由与 GPU 编排
> 视角：Cloud & AI Solution Architect，聚焦实时交互 SLA、调度抖动、运维可持续性与成本。

## 1. 执行摘要
- Pattern A 在中等规模机群下兼顾延迟、灵活性与迭代速度，是默认优先方案。
- Pattern B 追求极致确定性延迟，适用于超严格语音与 IDE 场景，但编译和配置复杂。
- Pattern C 通过 PTU 托管能力降低内核工程负担，需通过 APIM 做长度分流隔离。
- Pattern D 会导致明显 HoL 卡顿与 p99/p99.9 抖动扩散，应明确禁止。

## 2. 边界与假设
- 目标：交互式流式输出 p99 ITL 控制在 20ms 以内，避免音频断续与界面卡顿。
- 负载：短 FIM 与长上下文 RAG 混合，50/250/1000 并发流。
- 资源：ND H100/H200 或 NCads_H100_v5，含 CPU/NUMA 与网络中断治理。

## 3. 四种架构模式
- Pattern A：AKS vLLM V1/SGLang overlap + chunked prefill + piecewise CUDA graph + NUMA pinning。
- Pattern B：TensorRT-LLM + Triton C++ in-flight batching，针对 SM90/SM100 FP8 优化。
- Pattern C：Foundry/Azure OpenAI PTU + APIM 按 prompt 长度分道。
- Pattern D：未分块 prefill、eager 回退、未 pin 的突发 CPU（反模式）。

## 4. 12 维比较矩阵
| 维度 | Pattern A | Pattern B | Pattern C | Pattern D |
|---|---|---|---|---|
| p99 ITL 稳定性 | 高 | 很高 | 中高 | 低 |
| p99.9 长尾 | 中高 | 高 | 中 | 低 |
| 模型切换灵活性 | 高 | 中 | 中高 | 低 |
| 工程复杂度 | 中 | 高 | 低 | 低（但风险高） |
| 资源利用率 | 高 | 高 | 中高 | 低 |
| 调参难度 | 中 | 高 | 低 | 低 |
| 故障可观测性 | 高 | 中高 | 中 | 低 |
| 上线速度 | 中高 | 中 | 高 | 高（不可接受） |
| 多模型共池 | 高 | 中 | 中 | 低 |
| 运维人力 | 中 | 高 | 低 | 低 |
| 成本可控性 | 高 | 中高 | 中 | 低 |
| 推荐结论 | 推荐 | 强推荐（极低抖动） | 推荐（托管优先） | 禁止 |

## 5. GPU 时间线与 N+1 overlap 架构图
```mermaid
flowchart LR
  Req[Streaming Requests] --> Adm[Admission by APIM lane]
  Adm --> Prefill[Chunked Prefill Queue]
  Adm --> Decode[Decode Queue]
  Prefill --> Overlap[N+1 CPU-GPU Overlap Scheduler]
  Decode --> Overlap
  Overlap --> Graph[Piecewise CUDA Graph Runner]
  Graph --> Out[Token Stream Output]
  Graph --> Metric[Prometheus ITL Metrics]
  Metric --> Alert[SLO Alerting and Circuit Controls]
```

## 6. 决策与 ITL 预算流图
```mermaid
stateDiagram-v2
  [*] --> ClassifyPrompt
  ClassifyPrompt --> RealtimeLane: <= 8k tokens
  ClassifyPrompt --> LongLane: > 8k tokens
  RealtimeLane --> DecodeBudgetCheck
  LongLane --> ChunkBudgetCheck
  DecodeBudgetCheck --> Serve
  ChunkBudgetCheck --> Serve
  Serve --> ObserveITL
  ObserveITL --> TuneChunk: p99 ITL breach
  ObserveITL --> [*]: healthy
```

## 7. Q1-Q6 定量问答
- Q1：Chunked Prefill token budget 如何平衡 TTFT 与 p99 ITL？
 - 可用预算方程：B_chunk = min(B_gpu_headroom, B_itl_guardrail)。当音频帧预算 120ms 且解码窗口 20ms 时，通常 1024-2048 token 分块更稳。
- Q2：Piecewise CUDA graph 的显存开销怎么算？
 - 按 bucket 数量与 padding 比例估算，graph 实例越多越稳但占用越高，需在 p99 抖动与 VRAM 之间取平衡。
- Q3：NUMA/IRQ pinning 为何能消除 5-9ms 抖动？
 - 将调度线程和关键中断绑定在本地 NUMA，减少跨 NUMA 抢占和 cache 抖动，可明显降低 host 调度尾延迟。
- Q4：单池 Chunked Prefill 与 P/D 双池何时分界？
 - 当长上下文比例持续高且 p99 超阈值时，双池收益明显；在中等规模负载下单池优化通常更经济。
- Q5：PTU 方案如何避免长 prompt 污染实时池？
 - 通过 APIM 按 token 长度分流，把 >8k 请求导向长文本 lane，实时 lane 保持稳定。
- Q6：反模式导致 300-850ms HoL 的主因？
 - 未分块 prefill 与 eager 回退叠加 CPU 抢占，阻塞 decode 步骤，形成可感知音频与 UI 卡顿。

## 8. 生产配置片段（AKS vLLM 部署）
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-realtime-serving
spec:
  replicas: 3
  template:
    spec:
      nodeSelector:
        accelerator: h100
      containers:
      - name: vllm
        image: ghcr.io/org/vllm-v1:stable
        args:
        - "--max-num-batched-tokens=2048"
        - "--enable-chunked-prefill"
        - "--cuda-graph-mode=piecewise"
        - "--scheduler-policy=overlap"
        env:
        - name: CPUSET_REALTIME
          value: "0-15"
        - name: CPUSET_PREFILL
          value: "16-31"
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: itl-slo-alert
spec:
  groups:
  - name: realtime-itl
    rules:
    - alert: ITLP99Breach
      expr: histogram_quantile(0.99, sum(rate(inter_token_latency_seconds_bucket[5m])) by (le)) > 0.020
      for: 10m
```

## 9. 场景 1：零售银行联络中心语音助手
- 需对接 Azure Speech TTS，音频 jitter buffer 上限 120ms。
- 采用 Pattern A，并在高峰期对长 prompt 分流至 long lane。
- 结果：交互流稳定性显著提升，用户感知卡顿下降。

## 10. 场景 2：15000 开发者企业编码助手
- 混合 512-token FIM 与 32k repo-chat，峰值并发高。
- 核心策略为 chunked prefill + overlap + CPU pinning。
- 在成本可控前提下维持 IDE 交互流畅。

## 11. 三年 TCO（50 / 250 / 1000 并发流）
| 规模 | Pattern A | Pattern B | Pattern C | 抖动事故暴露参考 |
|---|---:|---:|---:|---:|
| 50 streams | $0.88M | $1.32M | $1.05M | $0.26M |
| 250 streams | $3.95M | $5.84M | $5.10M | $1.42M |
| 1000 streams | $15.4M | $22.8M | $21.7M | $6.90M |

## 12. Day-0 / Day-1 / Day-2 清单
### Day-0
1. 定义 ITL/TTFT/SLA 指标与阈值
2. 建立 APIM 长短 prompt 分道策略
3. 建立 GPU/CPU 观测基线
4. 定义回滚与熔断策略
5. 完成成本标签与预算阈值
### Day-1
1. 启用 chunked prefill 与 overlap
2. 部署 piecewise CUDA graph
3. 实施 NUMA/IRQ pinning
4. 上线 ITL SLO 告警
5. 完成压测与基线对比
### Day-2
1. 按周优化分块预算
2. 评估是否升级 P/D 双池
3. 复盘长尾抖动来源
4. 持续优化模型切换流程
5. 更新 runbook 与容量计划

## 13. Good fit vs Not for
- Good fit：对实时交互抖动敏感、并发稳定增长、具备平台治理能力的团队。
- Not for：极低流量、无 SLA 要求、无法投入性能治理资源的短期项目。

## 14. 风险与缓解
- 风险：长 prompt 突发导致 lane 污染
 - 缓解：分流策略、灰度参数发布、线程 pinning、阈值分级与自动回退。
- 风险：图捕获桶配置失衡
- 风险：CPU 抢占引发 host 抖动
- 风险：告警阈值过紧导致误触发
- 风险：模型热切换导致瞬时退化

## 15. 参考来源与时间
- 来源：vLLM 项目 · https://github.com/vllm-project/vllm · 抓取时间：2026-11-26（UTC+8）
- 来源：SGLang 项目 · https://github.com/sgl-project/sglang · 抓取时间：2026-11-26（UTC+8）
- 来源：TensorRT-LLM 文档 · https://github.com/NVIDIA/TensorRT-LLM · 抓取时间：2026-11-26（UTC+8）
- 来源：Triton Inference Server · https://github.com/triton-inference-server/server · 抓取时间：2026-11-26（UTC+8）
- 来源：Azure OpenAI / Foundry 文档 · https://learn.microsoft.com/azure/ai-foundry/ · 抓取时间：2026-11-26（UTC+8）

---

### 附录 A：实时指标字典
- `inter_token_latency_p99`：记录阈值、告警、动作、回滚策略与负责人。
- `ttft_p95`：记录阈值、告警、动作、回滚策略与负责人。
- `chunked_prefill_queue_depth`：记录阈值、告警、动作、回滚策略与负责人。
- `decode_queue_depth`：记录阈值、告警、动作、回滚策略与负责人。
- `cpu_scheduler_stall_ms`：记录阈值、告警、动作、回滚策略与负责人。
- `cuda_graph_fallback_rate`：记录阈值、告警、动作、回滚策略与负责人。

- 审计项 01：保留压测配置、SLO 曲线、告警记录、变更审批与回滚证据。

- 验证 lane 分流、分块预算、CPU pinning、图捕获配置、告警门槛与成本标签。

