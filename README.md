# AI Infra Lab (`turbo998/ai-infra-lab`)

面向 **AI Infra 架构师** 的云厂商与智算基础设施全栈技术深度测评与可视化图谱仓库，覆盖 **AI 算力芯片（GPU / PPU / ASIC / CPU / Switch / SmartNIC / SSD 主控）**、**超节点与 Scale-Up/Scale-Out 智算网络（RDMA / RoCE / CXL / ICN）**、**异构算力调度平台**、**云原生分布式训推平台** 与 **企业级 MaaS / Agent 平台**。

---

## 📚 深度测评报告目录（Research & Architecture Evaluations）

| 专题报告 | 核心覆盖范围 | 可视化图表 (`SVG` + `Mermaid`) | 证据台账 | 审校记录 | 更新日期 |
|---|---|---|---|---|---|
| **[阿里云 AI Infra 全栈架构深度测评报告](docs/alibaba-cloud-ai-infra/README.md)** | **L0 平头哥芯片**（含光 800、真武 810E / M890 / V900 / J900、倚天 710/750、ICN Switch 1.0、磐脉 920、镇岳 510）· **L1 智算底座与网络**（磐久 AL128 超节点、CIPU、HPN 7.0 双上联×多轨×双平面、Solar-RDMA、eRDMA、ACCL、CPFS）· **L1.5 灵骏智算**（`hz` 亲和性调度、双平面网卡）· **L2 人工智能平台 PAI（“Pi平台”）**（DSW / DLC AIMaster+EasyCKPT / EAS Prefix-Cache 路由+EP/PD 分离 / 父子配额潮汐抢占）· **L3 百炼平台（Model Studio）**（Token / PTU / DTU / MU 四大算力阶梯与后训练流水线） | 6 张高清矢量架构图 + 2 幅交互式 Mermaid 决策图 | [`evidence.json`](docs/alibaba-cloud-ai-infra/evidence.json)（21 篇一手信源） | [`review.md`](docs/alibaba-cloud-ai-infra/review.md) | 2026-10-14 |

---

## 📊 阿里云 AI Infra 核心可视化图谱速览（Visual Atlas）

### 1. “通-云-哥” L0–L3 全栈垂直整合架构与三大跨层协同闭环
![阿里云 AI Infra 全栈垂直整合架构图](docs/alibaba-cloud-ai-infra/assets/01-tong-yun-ge-architecture.svg)

### 2. 平头哥真武 PPU 四代演进（810E → M890 → V900 → J900）与主流 GPU 关键指标对比
![真武 PPU 四代演进与主流 GPU 对比](docs/alibaba-cloud-ai-infra/assets/02-zhenwu-ppu-vs-gpu-comparison.svg)

### 3. 磐久 AL128 高密超节点“算-存-网”平头哥五芯全自研数据通路图
![磐久 AL128 超节点五芯数据通路图](docs/alibaba-cloud-ai-infra/assets/03-panjiu-al128-five-chip-datapath.svg)

### 4. 阿里云 HPN 7.0（SIGCOMM 2024）双上联 × 多轨 × 双平面网络架构与实测收益
![阿里云 HPN 7.0 双平面网络拓扑与性能收益](docs/alibaba-cloud-ai-infra/assets/04-hpn7-dual-plane-topology.svg)

### 5. PAI 训推一体潮汐配额树与 PAI-EAS Prefix-Cache 智能路由官方实测加速比
![PAI 潮汐抢占与 EAS Prefix-Cache 实测对比](docs/alibaba-cloud-ai-infra/assets/05-pai-tidal-preemption-and-eas-benchmark.svg)

### 6. 百炼平台四大算力阶梯对比与 AI Infra 架构师 6 大核心场景选型矩阵
![百炼四大算力模式与架构师选型矩阵](docs/alibaba-cloud-ai-infra/assets/06-bailian-resource-modes-and-architect-matrix.svg)

---

## 🗂️ 仓库目录结构

```text
ai-infra-lab/
├── README.md                                                     # 仓库总览与可视化图谱导览
└── docs/
    └── alibaba-cloud-ai-infra/
        ├── README.md                                             # 阿里云 AI Infra 全栈架构深度测评主报告
        ├── evidence.json                                         # 21 篇一手技术信源与证据链台账
        ├── review.md                                             # 完整性、证据边界与可视化核验记录
        └── assets/
            ├── 01-tong-yun-ge-architecture.svg                   # L0-L3 全栈架构与跨层协同闭环图
            ├── 02-zhenwu-ppu-vs-gpu-comparison.svg               # 真武 PPU 四代 vs NVIDIA GPU 指标对比图
            ├── 03-panjiu-al128-five-chip-datapath.svg            # 磐久 AL128 超节点五芯协同数据通路图
            ├── 04-hpn7-dual-plane-topology.svg                   # HPN 7.0 双平面网络拓扑与实测对比图
            ├── 05-pai-tidal-preemption-and-eas-benchmark.svg     # PAI 训推潮汐抢占与 EAS Prefix-Cache 加速图
            └── 06-bailian-resource-modes-and-architect-matrix.svg# 百炼四大算力模式与 6 大场景选型矩阵图
```