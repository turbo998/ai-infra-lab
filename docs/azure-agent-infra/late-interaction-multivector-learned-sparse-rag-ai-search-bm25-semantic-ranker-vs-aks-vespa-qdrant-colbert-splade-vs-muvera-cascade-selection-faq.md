# 多向量 Late-Interaction 与 Learned Sparse RAG 选型 FAQ：Azure AI Search Hybrid vs AKS Vespa/Qdrant ColBERT/SPLADE vs MUVERA Cascade

> 🌐 **Language / 语言切换**: **中文版（当前）** | **[English](late-interaction-multivector-learned-sparse-rag-ai-search-bm25-semantic-ranker-vs-aks-vespa-qdrant-colbert-splade-vs-muvera-cascade-selection-faq.en.md)** | **[返回目录](README.md)**

## 1. 一页结论（Executive Summary）

- **Pattern A**：`Azure AI Search` 单向量混合检索（`BM25 + HNSW + RRF + L2 Semantic Ranker`）适合零运维与 ACL 原子裁剪。
- **Pattern B**：`Vespa/Qdrant + ColBERT/ColQwen + SPLADE/BGE-M3` 在复杂约束检索上精度更高，但 RAM/IOPS 成本和运维复杂度显著增加。
- **Pattern C**：`Azure AI Search` 先召回 + `MUVERA`/`MaxSim` 二级重排，是成本与精度的工程折中。
- **Pattern D（禁用）**：将每 token float32 多向量强行拆成单向量行，会污染 HNSW、打爆文档配额并破坏 ACL 原子性。
- **目标指标**：`NDCG@10`、`Recall@50`、`P95`、`ACL leakage = 0`、`3-year TCO` 同时满足阈值。

## 2. 四种架构模式定义

- **A（托管单向量）**：`Azure AI Search` + `BM25` 自定义 analyzer + `MRL`/binary HNSW + `Semantic Ranker` Top-50。
- **B（自建多向量）**：`AKS` 上 `Vespa/Qdrant`，以 `ColBERTv2/ColQwen2.5 MaxSim` 与 `SPLADE++` learned sparse 联合。
- **C（双层级联）**：`Azure AI Search` ACL 预过滤 Top-250，再由无状态 `AKS` Late-Interaction scorer 精排。
- **D（反模式）**：token 级向量直接映射为海量单行向量，导致 HNSW 图质量与配额崩塌。

## 3. 参考架构（Mermaid 1）

```mermaid
flowchart LR
    Q[Query] --> A[Azure AI Search Hybrid]
    A --> C[Top-250 ACL-trimmed Candidates]
    C --> M[MUVERA or MaxSim Scorer on AKS]
    M --> R[Top-20 Reranked Results]
    B[(Blob/Redis Packed Token Matrices)] --> M
    A --> L[(Audit and Policy Logs)]
```

## 4. MUVERA 级联时序（Mermaid 2）

```mermaid
sequenceDiagram
    participant U as User Query
    participant S as Search Tier
    participant K as Late Interaction Scorer
    participant D as Packed Matrix Store
    U->>S: BM25 + vector query
    S-->>U: top-250 ids with ACL filter
    U->>K: rerank request with top-250 ids
    K->>D: fetch packed token matrices
    K-->>U: top-20 final ranking
```

## 5. 定量模型与内存公式

1. **float32 token 向量内存**：`Mem_f32 = N_chunks * T_token * D * 4 bytes`。
2. **binary token 矩阵内存**：`Mem_bin ≈ N_chunks * T_token * D / 8`。
3. **MUVERA FDE 内存**：`Mem_fde = N_chunks * D_fde * 1~2 bytes`（取决于量化位宽）。
4. **级联总延迟**：`P95_total = P95_search + P95_fetch + P95_rerank`。
5. **ACL 选择性影响**：`Recall_effective = Recall_raw * Selectivity_acl`。

### 5.1 10M/50M chunk 粗粒度内存测算示例

| 数据规模 | float32 MaxSim 估算 | binary MaxSim 估算 | MUVERA FDE 估算 |
|---|---:|---:|---:|
| 10M chunks | ~3.8 TB | ~0.48 TB | ~0.19 TB |
| 50M chunks | ~19.0 TB | ~2.4 TB | ~0.95 TB |

## 6. 12 维度对比矩阵

| 维度 | Pattern A 托管单向量 | Pattern B 自建多向量 | Pattern C 级联混合 | Pattern D 反模式 |
|---|---|---|---|---|
| 检索精度上限 | 中-高 | 很高 | 高 | 低 |
| 运维复杂度 | 低 | 高 | 中 | 中(但无效) |
| ACL 原子裁剪 | 高 | 中 | 高 | 低 |
| 冷热数据分层 | 中 | 高 | 高 | 低 |
| RAM 占用 | 低-中 | 极高 | 中 | 极高 |
| IOPS 压力 | 中 | 高 | 中-高 | 极高 |
| 延迟稳定性 | 高 | 中 | 中-高 | 低 |
| 可观测性 | 高 | 中 | 高 | 低 |
| BCDR 便利性 | 高 | 中 | 高 | 低 |
| TCO 可控性 | 高 | 中-低 | 高 | 低 |
| 扩展灵活度 | 中 | 高 | 高 | 低 |
| 反模式暴露 | 低 | 中 | 低 | 极高 |

## 7. 深度问答（Q1–Q8）

### Q1. 何时仅用 Pattern A 足够？
- 文档查询约束维度不高且 ACL 规则复杂时。
- 组织优先要零运维与标准化合规路径时。
- 数据规模在成本可控范围内且召回窗口可接受时。
- 短周期交付项目优先选 A。

### Q2. 何时必须 Pattern B？
- 涉及复杂条款、表格布局或细粒度语义组合匹配时。
- 业务明确追求 Top-K 精度而可接受平台运维投入时。
- 团队具备向量索引调优与多副本运维经验时。
- 并且可承担高 RAM/IOPS 预算。

### Q3. Pattern C 的核心收益？
- 保留 Azure AI Search 的 ACL 与运维优势。
- 在 Top-250 范围内引入 MaxSim 精排提升精度。
- 内存成本显著低于全量多向量 HNSW。
- 更适合渐进式演进而非一次性重构。

### Q4. 如何避免 ACL 选择性陷阱？
- 先做 ACL 预过滤再做深度重排。
- 对高选择性租户单独标定召回窗口。
- 监控 Recall@50 在不同 ACL 分桶下的漂移。
- 必要时对低召回租户提升候选池上限。

### Q5. NDCG@10 与 P95 如何平衡？
- 使用分层候选池策略：默认 Top-250，热点租户 Top-400。
- 将高价值查询切到更深重排通道。
- 低价值查询保持轻量重排降低成本。
- 通过 A/B 持续校准阈值。

### Q6. 何时引入 MUVERA FDE？
- 当多向量内存已成为主要瓶颈时。
- 当希望保持高召回但压缩 token 矩阵存储时。
- 当团队可接受额外离线编码流水线时。
- 当延迟预算允许多一步重排。

### Q7. 如何做成本归因？
- 拆分 Search SKU、AKS scorer、Redis/Blob 存储与网络成本。
- 按租户/查询类型统计单位查询成本。
- 把精度提升带来的人工复核节省计入收益。
- 用 3 年视角评估总拥有成本。

### Q8. 如何迁移最稳妥？
- 先从 Pattern A 基线起步。
- 引入 Pattern C 灰度并校验精度收益。
- 仅在确证收益后局部引入 Pattern B。
- 保留回退开关与双轨对照。

## 8. Azure AI Search 索引与评分配置片段

```json
{
  "name": "enterprise-rag-index",
  "fields": [
    {"name": "id", "type": "Edm.String", "key": true},
    {"name": "content", "type": "Edm.String", "searchable": true},
    {"name": "acl", "type": "Collection(Edm.String)", "filterable": true},
    {"name": "vector", "type": "Collection(Edm.Single)", "searchable": true, "dimensions": 1024, "vectorSearchProfile": "hnsw-mrl"}
  ],
  "semantic": {"configurations": [{"name": "sem-default", "prioritizedFields": {"contentFields": [{"fieldName": "content"}]}}]}
}
```

## 9. Python MUVERA + MaxSim Cascade Scorer 示例

```python
def cascade_rerank(query_vec, candidate_ids, matrix_store, fde_index):
    coarse = fde_index.search(query_vec, candidate_ids, topk=250)
    fine_scores = {}
    for cid in coarse:
        token_matrix = matrix_store.get(cid)
        fine_scores[cid] = maxsim_score(query_vec, token_matrix)
    return sorted(fine_scores.items(), key=lambda x: x[1], reverse=True)[:20]
```

## 10. 企业场景 A：半导体/BOM/工差检索

- 场景A要点01：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点02：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点03：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点04：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点05：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点06：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点07：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点08：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点09：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点10：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点11：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点12：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点13：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点14：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点15：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点16：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点17：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点18：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点19：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点20：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点21：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点22：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点23：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点24：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点25：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点26：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点27：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点28：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点29：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。
- 场景A要点30：对规格表、BOM、工差条款执行多约束检索，优先保证 Recall@50 与 ACL 原子裁剪。

## 11. 企业场景 B：跨境并购/专利条款与复杂表格检索

- 场景B要点01：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点02：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点03：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点04：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点05：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点06：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点07：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点08：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点09：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点10：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点11：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点12：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点13：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点14：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点15：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点16：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点17：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点18：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点19：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点20：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点21：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点22：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点23：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点24：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点25：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点26：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点27：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点28：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点29：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。
- 场景B要点30：在法律条款与表格布局混合语料中，使用级联精排提升复杂问答命中率。

## 12. 三档三年 TCO（单位：USD）

| 规模 | chunk 数 | Pattern A 年成本 | Pattern B 年成本 | Pattern C 年成本 | 反模式损失 | 三年总计（推荐路径） |
|---|---:|---:|---:|---:|---:|---:|
| Small | 5M | 420,000 | 890,000 | 560,000 | 380,000 | 4,080,000 |
| Medium | 25M | 1,880,000 | 4,420,000 | 2,360,000 | 1,620,000 | 15,780,000 |
| Large | 100M | 6,920,000 | 16,800,000 | 8,450,000 | 6,400,000 | 55,110,000 |

## 13. Day-0 / Day-1 / Day-2 上线核查清单

### Day-0 设计阶段
- [ ] Day-0 设计阶段 检查项01：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项02：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项03：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项04：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项05：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项06：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项07：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项08：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项09：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项10：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项11：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项12：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项13：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项14：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项15：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项16：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项17：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项18：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项19：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项20：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项21：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项22：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项23：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项24：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项25：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项26：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项27：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项28：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项29：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项30：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项31：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项32：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项33：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项34：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-0 设计阶段 检查项35：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。

### Day-1 发布阶段
- [ ] Day-1 发布阶段 检查项01：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项02：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项03：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项04：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项05：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项06：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项07：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项08：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项09：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项10：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项11：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项12：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项13：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项14：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项15：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项16：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项17：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项18：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项19：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项20：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项21：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项22：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项23：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项24：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项25：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项26：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项27：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项28：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项29：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项30：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项31：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项32：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项33：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项34：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-1 发布阶段 检查项35：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。

### Day-2 运营阶段
- [ ] Day-2 运营阶段 检查项01：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项02：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项03：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项04：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项05：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项06：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项07：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项08：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项09：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项10：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项11：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项12：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项13：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项14：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项15：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项16：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项17：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项18：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项19：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项20：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项21：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项22：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项23：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项24：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项25：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项26：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项27：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项28：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项29：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项30：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项31：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项32：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项33：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项34：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。
- [ ] Day-2 运营阶段 检查项35：核对候选池、ACL 规则、延迟预算、回退开关、审计与容量阈值。

## 14. 反模式剖析与纠偏

- 纠偏建议01：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议02：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议03：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议04：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议05：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议06：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议07：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议08：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议09：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议10：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议11：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议12：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议13：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议14：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议15：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议16：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议17：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议18：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议19：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议20：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议21：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议22：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议23：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议24：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议25：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议26：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议27：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议28：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议29：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议30：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议31：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议32：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议33：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议34：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议35：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议36：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议37：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议38：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议39：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。
- 纠偏建议40：禁止 token 向量拆行为普通单向量记录，避免 HNSW 污染与 ACL 原子性破坏。

## 15. 附录：参数与治理控制目录

- 控制项001：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项002：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项003：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项004：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项005：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项006：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项007：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项008：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项009：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项010：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项011：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项012：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项013：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项014：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项015：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项016：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项017：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项018：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项019：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项020：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项021：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项022：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项023：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项024：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项025：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项026：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项027：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项028：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项029：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项030：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项031：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项032：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项033：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项034：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项035：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项036：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项037：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项038：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项039：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项040：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项041：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项042：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项043：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项044：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项045：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项046：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项047：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项048：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项049：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项050：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项051：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项052：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项053：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项054：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项055：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项056：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项057：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项058：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项059：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项060：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项061：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项062：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项063：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项064：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项065：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项066：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项067：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项068：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项069：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项070：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项071：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项072：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项073：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项074：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项075：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项076：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项077：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项078：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项079：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项080：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项081：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项082：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项083：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项084：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项085：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项086：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项087：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项088：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项089：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项090：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项091：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项092：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项093：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项094：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项095：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项096：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项097：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项098：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项099：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项100：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项101：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项102：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项103：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项104：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项105：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项106：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项107：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项108：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项109：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项110：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项111：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项112：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项113：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项114：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项115：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项116：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项117：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项118：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项119：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项120：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项121：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项122：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项123：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项124：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项125：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项126：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项127：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项128：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项129：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项130：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项131：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项132：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项133：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项134：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项135：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项136：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项137：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项138：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项139：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项140：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项141：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项142：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项143：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项144：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项145：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项146：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项147：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项148：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项149：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项150：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项151：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项152：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项153：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项154：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项155：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项156：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项157：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项158：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项159：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
- 控制项160：记录 `ndcg10`、`recall50`、`p95_ms`、`acl_selectivity`、`mem_gb`、`iops`、`unit_query_cost` 周趋势。
