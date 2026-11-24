# 企业 AI Landing Zone 压力退出与可逆可迁移选型 FAQ：Pragmatic Portable Core vs Cloud-Agnostic K8s vs Warm-Standby Escrow vs Paper-Only Plan
> **适用场景**：受 `EU DORA Article 28(8)`、`PRA SS2/21`、`EU Data Act Chapter VI` 约束的企业 GenAI 平台治理。
> **决策压力**：如何在保持 Azure 托管速度的同时，实现可验证的供应商退出与模型可替代。
> **核心目标**：建立可执行、可审计、可量化的 stressed-exit 架构与季度演练体系。

## 1. 执行摘要
- 监管关注点是可恢复性与可替代性，不是口头多云承诺。
- 默认建议是架构 A，关键业务加架构 C。
- 架构 B 适合极端 portability 诉求但运维税显著。
- 架构 D 无法通过实战演练，属于禁止模式。
- 可移植核心应覆盖原始语料、chunk manifest、MCP schema、评测集与审批证据。
- 向量索引应被视为派生层，不应作为唯一事实源。
- 模型迁移必须以 parity gate 驱动，不能以主观可用替代。
- 季度演练应输出可追溯证据包并形成整改闭环。
- 推荐指标包括 `RTO_exit`、`T_reembed`、`T_transfer` 与 `Parity_pass_rate`。

## 2. 适用边界与非目标
- 适用金融、保险、支付、公共事业等关键 ICT 业务。
- 适用供应商集中风险治理与合同终止迁移预案。
- 适用模型退役、地缘隔离、跨主权合规重构。
- 非目标是低风险内部问答机器人一次性 PoC。
- 非目标是不需要审计证据的短期实验环境。
- 非目标是绕过企业审批链的个人沙箱试验。

## 3. 监管要求映射到工程控制
- DORA 要求企业能在压力场景下可验证退出关键外包 ICT。
- PRA 关注第三方集中风险与连续性治理可信度。
- Data Act 关注切换成本、数据可携性与接口透明度。
- 工程实现必须把条款映射为流水线门禁与可重放证据。
- 退出演练结果应纳入董事会治理与审计材料。
- 所有关键声明必须有日志、哈希和版本号佐证。

## 4. 四类候选架构
- **架构 A**：Azure PaaS 主路径 + 可移植核心。
- **架构 B**：云无关 K8s 全栈可迁移平台。
- **架构 C**：主站 + 暖备 escrow + 影子一致性。
- **架构 D**：纸面合规 + 不可导出状态。
- A 追求高速度与可移植平衡。
- B 追求极限可迁移但持续复杂度更高。
- C 追求关键系统低 RTO 的演练可信度。
- D 在真实审计和演练中失败概率最高。

## 5. 架构 A 细节（Pragmatic Portable Core）
- `APIM` 提供统一 OpenAI/MCP 契约并锁定版本。
- `ADLS Gen2` 保存原始语料与 chunk manifest。
- `Azure AI Foundry` 与开源权重双轨回归。
- 关键任务 parity gate 建议 `>= 92%`。
- 提供自动 re-embed 与索引重建流水线。
- 通过审批链与日志哈希实现可审计闭环。
- 保持托管服务创新速度并降低不可逆耦合。

## 6. 架构 B 细节（Cloud-Agnostic K8s）
- 典型栈：`AKS + vLLM + pgvector/Qdrant + Temporal + Envoy/LiteLLM + Dapr`。
- 可在 EKS/GKE/Azure Local 复用基础设施模板。
- 迁移重写成本较低但 Day-2 运维税较高。
- GPU 容量、补丁窗口、版本兼容需要持续投入。
- 更适合对主权与可迁移有刚性要求的组织。

## 7. 架构 C 细节（Warm-Standby Escrow）
- 主站维持 Azure 生产，次站持续同步核心工件。
- 包含原始语料、提示词、工具配置、评测数据与账本状态。
- 运行 `1%` 影子请求用于一致性检验。
- 目标切换窗口按关键级分层 `<4h/<12h/<24h`。
- 通过季度演练生成监管可审计证书。
- 对 Tier-0 场景是最稳妥路径。

## 8. 架构 D 反模式（Paper-Only Plan）
- 仅存向量不存原始 chunk 会导致不可逆迁移。
- 不可导出线程状态会破坏回放与审计能力。
- 无版本提示词导致跨模型静默回归。
- 纸面演练无法替代实际切换与回归测试。
- 该模式通常在监管抽测时暴露系统性缺陷。

## 9. 十二维横向矩阵
| 维度 | 架构 A | 架构 B | 架构 C | 架构 D |
|---|---|---|---|---|
| 退出可执行性 | 高 | 中高 | 很高 | 低 |
| 日常交付速度 | 很高 | 中 | 中高 | 中 |
| 审计证据完整度 | 高 | 中高 | 很高 | 低 |
| 迁移改造强度 | 中 | 低 | 中 | 很高 |
| Day-2 负担 | 中低 | 很高 | 高 | 表面低 |
| 模型替换弹性 | 高 | 高 | 很高 | 低 |
| 数据可逆性 | 高 | 高 | 很高 | 低 |
| 多区域韧性 | 高 | 中高 | 很高 | 低 |
| 成本可预测性 | 高 | 中 | 中 | 低 |
| 人才门槛 | 中 | 高 | 高 | 低短期 |
| 合规可信度 | 高 | 中高 | 很高 | 低 |
| 综合建议 | 默认首选 | 特殊诉求 | Tier-0 优选 | 禁用 |

## 10. 可移植与 escrow 拓扑
```mermaid
flowchart LR
  A[Architecture A
Azure PaaS + Portable Core] --> A1[ADLS Raw + Chunk Manifest]
  A --> A2[APIM Canonical Contract]
  A --> A3[Foundry + Open-Weight Twin]
  B[Architecture B
Cloud-Agnostic K8s] --> B1[vLLM + Vector DB + Temporal]
  C[Architecture C
Warm-Standby Escrow] --> C1[Secondary Site]
  C --> C2[Continuous Sync + 1% Shadow]
  D[Architecture D
Opaque State] --> D1[Vector Only]
  D --> D2[Unversioned Prompt]
```

## 11. 季度压力退出演练时序
```mermaid
sequenceDiagram
  participant GRC as Governance
  participant PRI as Primary AZ
  participant ESC as Escrow Site
  participant EVA as Eval Harness
  GRC->>PRI: Trigger DORA stressed-exit drill
  PRI->>ESC: Export contract + data snapshot
  ESC->>EVA: Re-embed and replay golden suites
  EVA-->>GRC: Parity and RTO report
  alt Pass
    GRC-->>PRI: Issue audit certificate
  else Fail
    GRC-->>PRI: Open remediation ticket
  end
```

## 12. 关键级别决策树
```mermaid
flowchart TD
  S[Classify workload criticality] --> T{Tier-0 critical ICT?}
  T -- Yes --> Csel[Use Architecture C baseline]
  T -- No --> P{Need <72h cross-cloud portability?}
  P -- Yes --> Bsel[Use Architecture B]
  P -- No --> Asel[Use Architecture A]
  Asel --> Z{Opaque non-exportable state exists?}
  Bsel --> Z
  Csel --> Z
  Z -- Yes --> Fix[Block release and remediate]
  Z -- No --> Go[Go-live with quarterly drill]
```

### Q1. 为什么监管场景必须强调可执行退出而非纸面策略
- 监管检查不接受不可复现的口头承诺。
- 压力场景要求真实切换能力而非静态文档。
- 供应商集中风险需要可量化替代路径。
- 退出能力决定关键业务连续性下限。
- 证据链缺失会导致整改周期拉长。
- 可执行演练能提前暴露隐形耦合。
- 可执行退出也是董事会风险治理抓手。

### Q2. 四种架构在机制层面的主要差异是什么
- A 兼顾托管速度与可移植核心治理。
- B 以基础设施可迁移换持续运维复杂度。
- C 以暖备与影子演练换取低 RTO 可信度。
- D 缺乏可导出状态与可重放证据。
- A 与 C 的差异在于是否持续保持次站热度。
- B 与 C 都能降迁移风险但成本形态不同。
- D 最大风险是静默回归直到事故才暴露。

### Q3. 退出窗口应如何量化
- 重嵌入公式：`T_reembed = N_chunks * L_tok_avg / TPS_embed`。
- 传输公式：`T_transfer = Data_size / Effective_bandwidth`。
- 总窗口：`T_exit = max(T_reembed, T_transfer) + T_validate + T_cutover`。
- 维度不一致 `d1 != d2` 时需双索引并行。
- 需控制 re-embed 批次峰值避免挤占生产资源。
- 演练应覆盖最差网络窗口与跨境策略限制。
- 评估结果应给出 P50/P95 与尾延迟处置策略。

### Q4. 成本应如何结合退出概率评估
- 期望成本：`E[Cost] = C_day2 + p_exit * C_migration`。
- A 的 `C_day2` 适中且迁移成本可控。
- B 的 `C_day2` 最高但迁移重写最低。
- C 的 `C_day2` 较高但关键场景损失规避最大。
- D 的隐含事故成本和监管成本最高。
- 财务评估需纳入停机损失与品牌损伤。
- 决策应看总风险调整成本而非局部账单。

### Q5. `10M/200M/2B` chunks 三档 TCO 如何取舍
- `10M` chunks、`50` agents：优先 A，关键流程叠加 C。
- `200M` chunks、`300` agents：A+C 组合与季度证书化演练。
- `2B` chunks、G-SIB：Tier-0 默认 C，非关键域可用 A。
- 各档都要预留 embedding 吞吐与传输带宽预算。
- 各档都要设定模型退役应急 runbook。
- 各档都要保留可审计证据与整改追踪。
- 各档都应持续优化 parity gate 覆盖与准确率。

### Q6. 反模式与上线审计清单
- 反模式一：向量当主事实源。
- 反模式二：提示词无版本治理。
- 反模式三：没有真实 cutover 演练。
- 反模式四：只看覆盖率不看关键失败面。
- 上线前核对契约冻结、审批链、回滚阈值。
- 审计前核对证据包完整性与哈希一致性。
- 季度核对退出窗口收敛与整改闭环有效性。

## 13. Day-0 上线清单
- 定义关键与非关键业务映射及 RTO/RPO 目标。
- 冻结 APIM 外部契约并建立兼容性测试。
- 落实 MCP schema 版本治理与拒绝未知字段。
- 建立 ADLS 原始语料与 chunk manifest 双活备份。
- 部署 re-embed 与索引重建流水线。
- 建立 Foundry 与开源权重双轨基线。
- 固化最小权限与联邦身份模板。
- 完成首次压力退出演练并归档证据。
- 定义失败升级机制与修复 SLA。
- 启用季度治理看板并设定阈值告警。

## 14. Day-2 运维清单
- 按月审查模型替换路线与退役计划。
- 按周检查 escrow 新鲜度与复制延迟。
- 按日观察影子 parity 失败分布。
- 定期复演高风险工具链路。
- 验证跨云模板可用性与漂移项。
- 监控重嵌入与传输窗口尾延迟。
- 跟踪供应商条款和费用变化。
- 清理例外权限并复核审批链。
- 维护审计映射表与证据目录一致。
- 推动整改项按期关闭。

## 15. Good fit vs Not for
- Good fit：强监管与关键连续性要求企业。
- Good fit：既要 Azure 速度又要退出可逆能力。
- Good fit：多实体多区域统一治理组织。
- Not for：低风险内部试验且无审计义务场景。
- Not for：不愿建设评测和证据流水线团队。
- Not for：无关键业务承载的短期实验项目。

## 16. 五步落地指南
- 步骤 1：识别关键业务并定义退出目标。
- 步骤 2：构建可移植核心与版本化契约。
- 步骤 3：部署 A 或 A+C 并完成首轮演练。
- 步骤 4：将证据纳入治理与审计例会。
- 步骤 5：按季度复盘并压缩退出窗口。

## 17. 季度治理指标包
- 追踪 RTO_exit P50/P95
- 追踪 T_reembed 稳定性
- 追踪 T_transfer 尾延迟
- 追踪 Parity 通过率
- 追踪 Escrow 新鲜度
- 追踪 Schema 兼容失败率
- 追踪 模型退役演练完成率
- 追踪 跨云模板漂移数
- 追踪 审批链超时率
- 追踪 审计复发率
- 追踪 关键工具回放成功率
- 追踪 证据包缺失率
- 追踪 治理开销占比
- 追踪 单位交易治理成本
- 追踪 高风险变更覆盖率
- 追踪 跨区域策略一致性
- 追踪 契约升级成功率
- 追踪 向量重建重试率
- 追踪 导出失败分布
- 追踪 整改按期完成率
- 追踪 外部审计一次通过率
- 追踪 迁移后稳定天数
- 追踪 关键超时占比
- 追踪 跨境例外数量
- 追踪 演练业务影响时长
- 追踪 监管问询准备时长
- 追踪 条款变更评估时效
- 追踪 治理看板可用性
- 追踪 退出证书签发时效
- 追踪 风险热力图更新时效

## 18. 数据来源声明
- 追踪退出演练的滞后波动。
- 追踪退出演练的偏差分布。
- 追踪退出演练的异常聚类。
- 追踪退出演练的工单积压。
- 追踪退出演练的阈值漂移。
- 追踪退出演练的审批拥塞。
- 追踪退出演练的恢复稳定性。
- 追踪退出演练的审计通过率。
- 追踪契约治理的滞后波动。
- 追踪契约治理的偏差分布。
- 追踪契约治理的异常聚类。
- 追踪契约治理的工单积压。
- 追踪契约治理的阈值漂移。
- 追踪契约治理的审批拥塞。
- 追踪契约治理的恢复稳定性。
- 追踪契约治理的审计通过率。
- 追踪证据归档的滞后波动。
- 追踪证据归档的偏差分布。
- 追踪证据归档的异常聚类。
- 追踪证据归档的工单积压。
- 追踪证据归档的阈值漂移。
- 追踪证据归档的审批拥塞。
- 追踪证据归档的恢复稳定性。
- 追踪证据归档的审计通过率。
- 追踪跨域传输的滞后波动。
- 追踪跨域传输的偏差分布。
- 追踪跨域传输的异常聚类。
- 追踪跨域传输的工单积压。
- 追踪跨域传输的阈值漂移。
- 追踪跨域传输的审批拥塞。
- 追踪跨域传输的恢复稳定性。
- 追踪跨域传输的审计通过率。
- 追踪供应商风险的滞后波动。
- 追踪供应商风险的偏差分布。
- 追踪供应商风险的异常聚类。
- 追踪供应商风险的工单积压。
- 追踪供应商风险的阈值漂移。
- 追踪供应商风险的审批拥塞。
- 追踪供应商风险的恢复稳定性。
- 追踪供应商风险的审计通过率。
- 追踪影子一致性的滞后波动。
- 追踪影子一致性的偏差分布。
- 追踪影子一致性的异常聚类。
- 追踪影子一致性的工单积压。
- 追踪影子一致性的阈值漂移。
- 追踪影子一致性的审批拥塞。
- 追踪影子一致性的恢复稳定性。
- 追踪影子一致性的审计通过率。
- 追踪身份联邦的滞后波动。
- 追踪身份联邦的偏差分布。
- 追踪身份联邦的异常聚类。
- 追踪身份联邦的工单积压。
- 追踪身份联邦的阈值漂移。
- 追踪身份联邦的审批拥塞。
- 追踪身份联邦的恢复稳定性。
- 追踪身份联邦的审计通过率。
- 追踪索引重建的滞后波动。
- `anonymized_reported`：本文公式、时延区间与成本区间来自匿名化企业落地项目复盘，按可复现实验假设整理。
- 建议保留演练批次号、数据快照哈希、契约版本、模型版本与审批签名链。
- 建议分层留存审计证据、财务估值与技术运行报告。
- 当监管口径或供应商条款变化时，需重算 `E[Cost]` 与退出窗口。
