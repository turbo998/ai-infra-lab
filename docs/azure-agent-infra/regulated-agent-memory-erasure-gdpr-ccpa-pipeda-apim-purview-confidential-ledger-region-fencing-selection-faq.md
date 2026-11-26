# 受监管智能体记忆删除与跨境数据驻留选型 FAQ（GDPR/CCPA/PIPEDA：APIM + Purview + Confidential Ledger + Region Fencing）

> 适用对象：金融、医疗、公共部门、跨境电商等高合规行业的 Cloud & AI Solution Architect。

## 1. 一页结论

当企业 Agent 同时处理欧盟、北美与本地监管数据时，必须把“可用性优先”升级为“合规正确性优先”的架构原则。推荐组合：

- **Route A（控制平面）**：`APIM` 入口策略 + 租户/地域声明校验 + 操作级策略分流。
- **Route B（治理平面）**：`Purview` 数据目录、保留与删除策略、审计检索。
- **Route C（证据平面）**：`Confidential Ledger` 写入删除证据与审批链哈希。
- **Route D（反模式）**：只删主库，不触发缓存/向量索引/备份副本清理。

核心建议：**删除请求是法律事件，不是普通 CRUD。**

## 2. 业务与合规挑战

- 同一租户可能在多个主权区域有数据副本。
- 一次“被遗忘权”请求需要传播到对话历史、向量检索索引、分析湖仓、缓存与离线快照。
- 不同法域删除时限不同，SLA 与监管罚则联动。
- 企业需要证明“何时删除、删除了什么、谁批准、是否完成全链路传播”。

## 3. 路线定义

### Route A：APIM 地域围栏 + 操作门禁
- 按 `tenant_id`、`region_claim`、`data_classification` 做请求策略分流。
- 对删除请求强制走审计增强路径与异步回执通道。

### Route B：Purview 生命周期与删除策略编排
- 用 Purview 标签、分类器、保留策略识别受监管数据。
- 将删除工单与数据资产目录绑定，实现可追溯删除计划。

### Route C：Confidential Ledger 证据账本
- 写入删除请求哈希、审批节点、执行节点、完成状态摘要。
- 保障审计时证据不可抵赖。

### Route D：反模式
- 应用层返回“删除成功”，但缓存/索引/副本仍可命中。
- 缺乏跨域证据链，监管问询时无法证明完成度。

## 4. 定量模型

### 4.1 删除传播窗口
`W_erase = max(T_cache, T_vector, T_backup, T_analytics) - T_request`

### 4.2 法域违规暴露
`Risk_reg = Σ_j (P_miss_j * Fine_j * Exposure_j)`

### 4.3 跨境传输违规风险
`Risk_xborder = P(route_misclassification) * Impact_data_sovereignty`

### 4.4 删除证据完备率
`Evidence_coverage = N_verified_deletions / N_total_deletions`

## 5. 12 维对比矩阵

| 维度 | Route A APIM 门禁 | Route B Purview 治理 | Route C 证据账本 | Route D 反模式 |
|---|---|---|---|---|
| 地域路由准确性 | 高 | 中 | 中 | 低 |
| 删除策略表达力 | 中 | 高 | 中 | 低 |
| 证据不可抵赖 | 中 | 中 | 高 | 低 |
| 合规审计可读性 | 高 | 高 | 高 | 低 |
| 跨团队协同效率 | 中高 | 中高 | 中 | 低 |
| 初始建设复杂度 | 中 | 中 | 中 | 低 |
| 运行稳定性 | 高 | 高 | 高 | 低 |
| 违规罚则暴露降低 | 高 | 高 | 高 | 低 |
| 删除回执可追踪性 | 高 | 高 | 高 | 低 |
| 数据主权边界清晰度 | 高 | 高 | 中 | 低 |
| 多区域扩展性 | 高 | 中高 | 中高 | 低 |
| 推荐结论 | ✅ 必选 | ✅ 必选 | ✅ 强烈建议 | ⛔ 禁止 |

## 6. 深度问答（Q1-Q8）
### Q1：删除请求为何不能同步立即返回完成？
因为删除传播涉及多副本系统，必须异步确认全链路完成。

### Q2：为何需要 APIM 做法域门禁？
它是统一入口，最适合先做地域/租户声明校验并阻断违规路由。

### Q3：Purview 主要解决什么问题？
资产发现、分类、保留与删除策略治理，避免“遗漏资产”导致删不干净。

### Q4：Confidential Ledger 会增加时延吗？
会有小幅开销，但可换来审计不可抵赖证据。

### Q5：如何证明向量索引已删除？
对删除实体做抽样检索与全量索引扫描，输出可复验报告。

### Q6：跨境复制副本怎么清理？
通过区域账本与删除工单关联，逐区域签收完成。

### Q7：缓存与会话上下文如何处理？
引入版本租约与强制刷新策略，阻断旧上下文复活。

### Q8：最关键 KPI 是什么？
`erase_window_p95` 与 `evidence_coverage`。

## 7. Mermaid 架构图

```mermaid
flowchart LR
    Req[Erase Request] --> Gate[APIM Policy Gate]
    Gate --> Plan[Purview Deletion Plan]
    Plan --> Exec[Deletion Orchestrator]
    Exec --> Store[Primary Store]
    Exec --> Vector[Vector Index]
    Exec --> Cache[Cache Layer]
    Exec --> Lake[Analytics Lake]
    Exec --> Ledger[Confidential Ledger]
```

```mermaid
flowchart TD
    A[Regional Deletion Ticket] --> B[EU Region Execution]
    A --> C[US Region Execution]
    A --> D[APAC Region Execution]
    B --> E[Regional Evidence Hash]
    C --> E
    D --> E
    E --> F[Global Completion Certificate]
```

## 8. 生产片段：APIM 删除门禁策略（示意）

```xml
<inbound>
  <base />
  <choose>
    <when condition="@(context.Request.Headers.GetValueOrDefault("x-region","") != context.Request.Headers.GetValueOrDefault("x-tenant-region",""))">
      <return-response>
        <set-status code="403" reason="Region claim mismatch" />
      </return-response>
    </when>
  </choose>
</inbound>
```

## 9. 生产片段：删除执行伪代码

```python
def execute_erasure(ticket):
    for target in ticket.targets:
        erase(target)
        verify(target)
    ledger.append({
        "ticket_id": ticket.id,
        "status": "completed",
        "hash": sha256(ticket.summary()).hexdigest(),
    })
```

## 10. 企业场景（2+）

### 场景 A：跨境财富管理智能投顾
- 客户跨境迁移后请求删除历史对话与画像。
- 必须按法域分区执行并输出区域签收证明。

### 场景 B：医疗健康随访 Agent
- 患者撤回授权后，诊疗摘要与检索向量需同步删除。
- 需要可追溯证据链供审计团队复核。

### 场景 C：跨国零售客服平台
- 多区域会话缓存和推荐索引容易保留过期个人数据。
- 通过 APIM 门禁 + Purview 计划 + Ledger 证据闭环治理。

## 11. 三档三年 TCO（USD）

| 规模档位 | Route A+B | Route A+B+C | 反模式（隐性风险） |
|---|---:|---:|---:|
| 50 万数据主体 | 2,300,000 | 2,680,000 | 1,650,000（不含罚责） |
| 5,000 万数据主体 | 12,600,000 | 14,300,000 | 9,800,000（不含罚责） |
| 5 亿数据主体 | 67,000,000 | 73,500,000 | 54,200,000（不含罚责） |

## 12. Day-0 / Day-1 / Day-2 清单

### Day-0（治理基线）
- 定义删除 SLA、法域边界、证据字段与审批责任。

### Day-1（工程上线）
- 打通 APIM 门禁、Purview 策略、删除编排与证据账本。

### Day-2（运营稽核）
- 抽检删除窗口、跨区签收率、索引残留率与审计回放通过率。

## 13. Good fit vs Not for

### ✅ Good fit
- 强监管行业与跨境数据业务。
- 对“可证明删除”有明确法律责任的企业。

### ⛔ Not for
- 临时 PoC、无个人数据处理场景。
- 不具备数据治理与审计能力的轻量团队。

## 14. 五步落地指南
1. 建立租户+法域声明与 APIM 门禁规则。
2. 用 Purview 建模数据资产与删除策略。
3. 落地异步删除编排与跨区签收机制。
4. 记录 Confidential Ledger 证据哈希。
5. 以演练持续优化删除窗口与完备率。

## 15. 核心 KPI
- `erase_window_p95_minutes`
- `xborder_route_violation_rate`
- `evidence_coverage`
- `vector_residual_hit_rate`
- `audit_replay_pass_rate`

## 16. 最终建议
对受监管企业，Route A+B 是最低基线，Route C 是监管问询下的高价值增强；Route D 应明确禁用。
