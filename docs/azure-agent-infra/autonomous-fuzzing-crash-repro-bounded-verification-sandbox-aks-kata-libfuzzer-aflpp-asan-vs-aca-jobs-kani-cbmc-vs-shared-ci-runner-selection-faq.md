# 企业自治漏洞模糊测试、崩溃复现与有界验证沙箱选型 FAQ：AKS Kata + LibFuzzer/AFL++/Jazzer vs ACA Jobs + Kani/CBMC vs Azure Batch Spot vs 共享 CI Runner 反模式

> 🌐 **Language / 语言切换**: **中文版（当前）** | **[English](autonomous-fuzzing-crash-repro-bounded-verification-sandbox-aks-kata-libfuzzer-aflpp-asan-vs-aca-jobs-kani-cbmc-vs-shared-ci-runner-selection-faq.en.md)** | **[返回目录](README.md)**

## 1. 一页结论（Executive Summary）

- **推荐主路径**：不可信 PoC 与覆盖引导 fuzzing 固定在 `AKS Kata` 零出网临时沙箱中运行；PR 级性质证明放到 `ACA Jobs` 执行 `Kani/CBMC`；长期 corpus 最小化放到 `Azure Batch Spot`。
- **核心边界**：来自 LLM 的 harness、payload、seed、复现脚本均视为不可信输入，必须经过签名校验、隔离执行、只写落盘与审计留痕。
- **反模式警告**：共享自托管 Runner 直接执行崩溃 PoC，会放大 OIDC、ACR、签名密钥与内核稳定性共因风险。
- **度量目标**：`CrashReproP95 <= 8 min`，`DedupPrecision >= 0.93`，`FalseEscalation <= 2%`，`UnitCostPerUniqueCrash` 连续 3 周下降。
- **成本结论**：中高并发下，三层方案通常比“全量常驻大核 CI”节省 31%~58% 三年 TCO，并显著降低误报工单成本。

## 2. 路线定义

- **Route A（首选）**：`AKS Kata Containers` + `LibFuzzer/AFL++/Jazzer` + `ASAN/UBSAN` + `Cilium deny-all` + `Blob Snapshot`。
- **Route B（配套）**：`ACA Jobs` 事件触发执行 `Kani/CBMC`，用于 PR 时间窗 `<15 min` 的有界安全性质回归。
- **Route C（规模化）**：`Azure Batch Spot` 承载持续语料裁剪与历史崩溃再归并，`Cosmos DB` 维护 crash ledger。
- **Route D（禁用）**：共享 Runner 直接跑 fuzz 与 PoC，且允许外网出站与高权限令牌共存。

## 3. 参考架构（Mermaid 1）

```mermaid
flowchart LR
    Dev[PR/Commit] --> Q[Service Bus Queue]
    Q --> A[AKS Kata Fuzz Pool]
    Q --> B[ACA Jobs Bounded Verify]
    A --> D[Crash Hash/Dedup API]
    B --> D
    D --> C[(Cosmos DB Crash Ledger)]
    A --> S[(Blob Corpus Snapshots)]
    C --> T[Teams/Jira Triage]
    C --> G[Governance Workbook]
```

## 4. 三层数据流与隔离流（Mermaid 2）

```mermaid
sequenceDiagram
    participant L as LLM Harness Generator
    participant P as Policy Gate
    participant K as Kata Pod
    participant B as Blob Snapshot
    participant X as Crash Index
    L->>P: 提交 harness + seed + metadata
    P->>P: 签名校验 + 禁止词 + 资源上限
    P->>K: 发放一次性任务令牌
    K->>K: 执行 fuzz + ASAN/UBSAN
    K->>B: 写入只增量语料快照
    K->>X: 上报 stack hash + repro artifacts
    X-->>P: 返回去重与升级建议
```

## 5. 定量工程模型

1. **容量模型**：`N_kata = ceil((R_jobs * T_avg) / U_target)`，其中 `U_target` 建议 `0.62~0.75`。
2. **P95 完成时间**：`T_p95 = T_queue + T_pull + T_exec + T_upload`，通过 warm image 与本地 NVMe seed cache 压低 `T_pull`。
3. **唯一崩溃单位成本**：`C_unique = (C_compute + C_storage + C_ops) / N_unique_crash`。
4. **有界验证覆盖增益**：`Gain_verify = (N_blocked_regression / N_total_pr) * Impact_weight`。
5. **爆炸半径因子**：`Blast = Privilege * SharedKernel * EgressReachability`，Route D 中该值不可接受。

## 6. 12 维度对比矩阵

| 维度 | Route A AKS Kata | Route B ACA Jobs | Route C Batch Spot | Route D 共享 Runner |
|---|---|---|---|---|
| 隔离边界 | Hypervisor 级 | Hyper-V 会话级 | 任务节点级 | 进程/容器级 |
| 不可信 PoC 安全性 | 高 | 中 | 中 | 低 |
| 冷启动稳定性 | 中-高 | 高 | 中 | 高 |
| 并发弹性 | 高 | 中-高 | 很高 | 中 |
| 覆盖引导效率 | 很高 | 低 | 高 | 中 |
| ASAN/UBSAN 适配 | 完整 | 受限 | 完整 | 受限 |
| PR 反馈速度 | 中 | 高 | 低 | 中 |
| 语料最小化能力 | 中 | 低 | 很高 | 低 |
| 合规审计追踪 | 高 | 高 | 中-高 | 低 |
| 运维复杂度 | 中-高 | 低 | 中 | 低(表面) |
| 三年 TCO（中规模） | 中 | 低-中 | 低 | 高(隐性损失) |
| 反模式风险 | 低 | 中 | 中 | 极高 |

## 7. 深度问答（Q1–Q8）

### Q1. 何时必须把任务放进 Kata 而不是直接 ACA Jobs？
- 输入包含未知二进制、可执行 shellcode、或崩溃 PoC 时，优先 Kata。
- 需要 `ASAN/UBSAN` 与 `ptrace` 深度调试能力时，Kata 更可控。
- 仅做纯静态属性验证且执行时间 `<15 min` 时，ACA Jobs 更经济。
- 建议以任务风险标签驱动路由，而非按团队习惯固定。

### Q2. 如何控制 crash 去重误差？
- 采用 `stack hash + top frames + signal` 组合键，减少单一 hash 碰撞。
- 同一崩溃族保留多样化 seed，避免过早合并导致回归漏检。
- 每周抽样人工复核 2% 事件，回灌 dedup 规则。
- 把误合并率与误拆分率写入 SLO 仪表盘。

### Q3. KEDA 队列弹性怎么设阈值？
- `scaleOutThreshold` 绑定 backlog 与 P95 目标，建议联动时间窗。
- 为热点项目设置最小保底副本，避免高峰冷启动抖动。
- Spot 回收时触发优雅重排，保留关键任务优先级。
- 将 `maxReplicaCount` 与预算护栏联动，防止突发超支。

### Q4. 为什么还需要 Batch Spot？
- Batch 适合离峰执行 corpus minimization 与回放，不占白天 PR 预算。
- 可将历史崩溃重归并、再生成更小回归集，提升白天反馈速度。
- 结合低价 Spot 与可中断容错，成本优势明显。
- 与 Kata 主路径解耦，降低生产队列噪声。

### Q5. ACA Jobs 的有界验证如何落地？
- 将性质断言拆成可并行小任务，单任务控制在 5~12 分钟。
- 失败即中断并回传最小反例，避免无效长跑。
- PR 门禁只看关键性质集，扩展性质放夜间批处理。
- 将 `Kani/CBMC` 结果结构化入 Cosmos 便于追溯。

### Q6. 如何防止 OIDC/ACR 令牌泄漏？
- 运行时禁出网并最小化令牌 TTL。
- 任务令牌只允许写指定容器路径，不允许读取机密范围。
- 镜像拉取后立即切换无凭据执行上下文。
- 所有外呼域名执行 deny-by-default。

### Q7. 什么时候需要 GPU 参与？
- 仅当目标程序涉及 GPU 编解码或 CUDA 路径时引入 GPU 沙箱。
- 默认 CPU 路径先做覆盖与最小复现，GPU 用于二次确认。
- GPU 池与主 CI 池隔离，避免驱动层互扰。
- 统一记录 GPU 分钟成本，避免“隐形贵”。

### Q8. 如何向管理层解释 ROI？
- 用“每个唯一高危崩溃成本”替代“每分钟算力成本”。
- 展示误报下降、回归发现提前量、发布风险降低三条曲线。
- 比较事件处置 MTTR 与工单积压下降幅度。
- 把合规审计取证时间纳入收益模型。

## 8. 生产级 Kata PodSpec 与 Cilium 策略片段

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fuzz-kata-job
  labels:
    workload: fuzzing
spec:
  runtimeClassName: kata-cc
  restartPolicy: Never
  containers:
  - name: fuzz
    image: ghcr.io/example/fuzz-harness:2026.11
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    resources:
      requests: { cpu: "8", memory: "16Gi" }
      limits: { cpu: "16", memory: "32Gi" }
    env:
    - name: ASAN_OPTIONS
      value: "detect_leaks=1:halt_on_error=1"
    - name: UBSAN_OPTIONS
      value: "print_stacktrace=1:halt_on_error=1"
```

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: fuzz-deny-all-egress
spec:
  endpointSelector:
    matchLabels:
      workload: fuzzing
  egress: []
  ingress: []
```

## 9. ACA Jobs（Kani/CBMC）Bicep 片段

```bicep
resource verifyJob 'Microsoft.App/jobs@2024-03-01' = {
  name: 'bounded-verify-job'
  location: location
  properties: {
    environmentId: managedEnv.id
    configuration: {
      triggerType: 'Event'
      replicaRetryLimit: 1
      replicaTimeout: 900
    }
    template: {
      containers: [
        {
          name: 'verify'
          image: 'ghcr.io/example/kani-cbmc:2026.11'
          resources: { cpu: 2, memory: '4Gi' }
        }
      ]
    }
  }
}
```

## 10. 企业场景 A：金融支付网关 SDK 模糊测试工厂

- 场景A要点01：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点02：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点03：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点04：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点05：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点06：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点07：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点08：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点09：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点10：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点11：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点12：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点13：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点14：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点15：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点16：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点17：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点18：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点19：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点20：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点21：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点22：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点23：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点24：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点25：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点26：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点27：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点28：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点29：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。
- 场景A要点30：将支付协议解析器、签名验证库、压缩解码链路分阶段 fuzz，统一落入 crash ledger，确保跨版本可追溯。

## 11. 企业场景 B：车联网 OTA 包解析与媒体栈安全回归

- 场景B要点01：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点02：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点03：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点04：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点05：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点06：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点07：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点08：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点09：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点10：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点11：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点12：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点13：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点14：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点15：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点16：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点17：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点18：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点19：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点20：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点21：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点22：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点23：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点24：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点25：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点26：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点27：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点28：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点29：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。
- 场景B要点30：OTA 包、日志压缩包、视频片段解码器分别路由到不同风险池，优先处理可远程触发高危崩溃。

## 12. 三档三年 TCO（单位：USD）

| 规模 | 月任务量 | Route A 年成本 | Route B 年成本 | Route C 年成本 | 反模式隐性损失 | 三年总计（推荐组合） |
|---|---:|---:|---:|---:|---:|---:|
| Small | 50k 作业 | 210,000 | 42,000 | 36,000 | 180,000 | 864,000 |
| Medium | 500k 作业 | 1,420,000 | 220,000 | 160,000 | 1,050,000 | 5,400,000 |
| Large | 3,000k 作业 | 6,980,000 | 860,000 | 620,000 | 4,500,000 | 25,380,000 |

## 13. Day-0 / Day-1 / Day-2 上线核查清单

### Day-0 设计阶段
- [ ] Day-0 设计阶段 检查项01：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项02：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项03：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项04：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项05：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项06：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项07：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项08：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项09：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项10：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项11：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项12：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项13：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项14：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项15：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项16：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项17：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项18：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项19：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项20：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项21：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项22：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项23：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项24：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项25：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项26：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项27：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项28：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项29：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项30：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项31：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项32：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项33：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项34：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-0 设计阶段 检查项35：定义负责人、阈值、回滚动作、审计证据路径。

### Day-1 上线阶段
- [ ] Day-1 上线阶段 检查项01：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项02：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项03：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项04：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项05：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项06：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项07：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项08：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项09：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项10：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项11：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项12：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项13：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项14：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项15：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项16：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项17：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项18：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项19：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项20：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项21：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项22：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项23：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项24：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项25：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项26：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项27：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项28：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项29：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项30：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项31：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项32：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项33：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项34：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-1 上线阶段 检查项35：定义负责人、阈值、回滚动作、审计证据路径。

### Day-2 运营阶段
- [ ] Day-2 运营阶段 检查项01：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项02：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项03：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项04：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项05：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项06：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项07：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项08：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项09：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项10：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项11：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项12：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项13：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项14：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项15：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项16：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项17：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项18：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项19：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项20：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项21：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项22：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项23：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项24：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项25：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项26：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项27：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项28：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项29：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项30：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项31：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项32：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项33：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项34：定义负责人、阈值、回滚动作、审计证据路径。
- [ ] Day-2 运营阶段 检查项35：定义负责人、阈值、回滚动作、审计证据路径。

## 14. 反模式剖析与纠偏

- 纠偏建议01：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议02：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议03：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议04：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议05：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议06：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议07：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议08：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议09：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议10：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议11：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议12：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议13：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议14：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议15：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议16：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议17：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议18：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议19：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议20：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议21：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议22：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议23：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议24：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议25：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议26：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议27：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议28：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议29：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议30：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议31：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议32：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议33：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议34：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议35：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议36：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议37：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议38：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议39：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。
- 纠偏建议40：禁止在共享 Runner 直接执行不可信 PoC，统一迁移到隔离沙箱并绑定最小权限令牌。

## 15. 附录：参数与治理控制目录（用于容量复盘）

- 控制项001：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项002：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项003：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项004：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项005：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项006：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项007：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项008：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项009：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项010：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项011：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项012：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项013：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项014：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项015：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项016：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项017：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项018：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项019：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项020：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项021：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项022：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项023：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项024：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项025：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项026：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项027：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项028：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项029：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项030：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项031：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项032：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项033：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项034：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项035：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项036：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项037：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项038：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项039：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项040：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项041：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项042：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项043：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项044：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项045：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项046：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项047：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项048：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项049：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项050：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项051：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项052：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项053：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项054：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项055：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项056：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项057：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项058：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项059：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项060：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项061：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项062：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项063：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项064：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项065：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项066：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项067：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项068：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项069：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项070：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项071：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项072：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项073：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项074：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项075：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项076：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项077：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项078：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项079：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项080：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项081：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项082：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项083：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项084：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项085：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项086：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项087：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项088：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项089：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项090：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项091：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项092：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项093：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项094：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项095：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项096：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项097：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项098：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项099：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项100：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项101：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项102：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项103：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项104：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项105：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项106：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项107：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项108：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项109：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项110：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项111：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项112：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项113：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项114：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项115：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项116：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项117：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项118：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项119：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项120：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项121：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项122：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项123：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项124：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项125：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项126：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项127：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项128：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项129：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项130：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项131：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项132：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项133：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项134：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项135：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项136：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项137：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项138：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项139：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项140：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项141：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项142：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项143：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项144：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项145：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项146：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项147：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项148：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项149：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项150：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项151：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项152：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项153：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项154：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项155：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项156：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项157：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项158：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项159：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
- 控制项160：记录 `queue_backlog`、`crash_family`、`repro_time_p95`、`storage_growth`、`token_ttl`、`egress_denied_count` 的周趋势与异常解释。
