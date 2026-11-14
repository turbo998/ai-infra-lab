# APIM MCP 重入一致性护栏延迟桶公平选型 FAQ（Cycle 2152）

## 1) 3 分钟结论
- **核心问题：** 重入高峰叠加配置不一致时，延迟桶恢复顺序会产生跨租户偏移。
- **推荐方案：** “一致性护栏车道 + 延迟桶配给 + 分级降档 + 回放审计”闭环治理。
- **目标：** 在租户公平前提下，将一致性窗口内恢复抖振和回退影响面保持在可控区间。

## 2) Q1 / Q2 / Q3
### Q1：为什么要引入 consistency guard lane？
1) 统一限流难以抑制配置不一致导致的恢复错位；
2) 一致性护栏可隔离高风险会话并先校验配置状态；
3) 与延迟桶联动可同时约束额度、时效、顺序与一致性边界。

### Q2：生产分层怎么设计？
- **一致性护栏层：** 按租户等级与一致性状态映射恢复车道；
- **配给层：** 车道内发放延迟桶额度与续租窗口；
- **回收层：** 对超额租约执行窗口化回收并抑制抖振；
- **降档层：** 峰值触发分级降档并进入人工复盘队列。

### Q3：上线判据是什么？
- 无护栏漂移率持续下降；
- 一致性车道方差率稳定且可追踪；
- 恢复收益（percentage points）跨窗口可复现。

## 3) 架构图
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

## 4) 选型矩阵
| 维度 | 统一延迟桶 | Consistency Guard Lane + Latency Bucket（推荐） |
|---|---:|---:|
| 一致性漂移控制 | 中 | 高 |
| 跨租户抖振抑制 | 中 | 高 |
| 一致性回放可解释性 | 中 | 高 |
| 峰值降档稳定性 | 中 | 高 |
| 实施复杂度 | 中 | 中高 |

## 5) 数据证据与可复算算例（含 n 分母来源）
- 证据文件：
  - `workshops/azure-agent-infra/evidence/cycle-2152-benchmark.json`
  - `workshops/azure-agent-infra/evidence/cycle-2152-latency.csv`
  - `workshops/azure-agent-infra/evidence/cycle-2152-throughput.csv`
  - `workshops/azure-agent-infra/evidence/cycle-2152-cost.csv`
- 合同键：
  - `percentage_contract.unfenced_reentry_consistencyguardlane_drift_rate`
  - `percentage_contract.reentry_consistencyguardlane_variance_rate`
  - `percentage_contract.reentry_consistencyguardlane_recovery_gain_pts`
- 标签：`measured / synthetic / anonymized_reported`。

## 6) 生产检查清单
- [ ] consistency 护栏映射规则可按租户等级重放。
- [ ] 延迟桶授予/续租/回收与一致性护栏一致。
- [ ] 峰值降档时一致性优先车道无饥饿。
- [ ] 审计链路可关联策略版本、一致性等级与恢复轨迹。
- [ ] FAQ/H5/SVG/7-card/evidence 的 cycle/topic 一致。
