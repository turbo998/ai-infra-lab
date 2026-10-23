# AI Infra Lab (`turbo998/ai-infra-lab`) — English Edition

> 🌐 **Language / 语言版本切换**: **[🇬🇧 English Homepage (README.en.md)](README.en.md)** | **[🇨🇳 中文主页 (README.zh-CN.md)](README.zh-CN.md)** | **[📘 Azure Agent Infra Playbook (English)](docs/azure-agent-infra/README.en.md)** | **[📘 Azure Agent Infra Playbook (中文)](docs/azure-agent-infra/README.zh-CN.md)**

Full-stack AI Infrastructure architecture evaluations, runtime benchmarks, and visual atlases for **Principal AI Infrastructure Architects**, covering **AI Silicon (GPU / PPU / ASIC / CPU / Switch / SmartNIC)**, **Scale-Up & Scale-Out Cluster Networking (RDMA / RoCE / CXL / ICN)**, **Heterogeneous Scheduling**, and **Enterprise AI Agent Runtimes & Hardware-Virtualized Sandbox Infrastructure (`Azure AI Agent Infra`)**.

---

## 📚 Research & Architecture Evaluations (Bilingual EN / 中文)

| Research Deliverable | Core Scope | Visual Assets (`SVG` + `H5` + `7-Card`) | Evidence Ledger | Review Log | Updated |
|---|---|---|---|---|---|
| **[Azure AI Agent Infrastructure Playbook (English Edition)](docs/azure-agent-infra/README.en.md)** · **[中文完整版](docs/azure-agent-infra/README.zh-CN.md)** | **L1 Build Harness** (MAF · GitHub Copilot SDK · LangGraph · OpenClaw · Hermes · MCP/A2A) · **L2 Run 6-Tier Runtime Spectrum** (Foundry Hosted Agents · Durable Functions · ACA Dynamic Sessions · ACA Sandboxes Preview · AKS + Kata `AKSClaw` · E2B on Azure VMSS `Standard_D96as_v7` `108ms/132ms`) · **L3 Ground State & Memory** (Azure HorizonDB Preview + PgBouncer 5ms + Cosmos DB + Firecracker UFFD/NBD Snapshot) · **L4 Govern Zero-Trust Control Plane** (Entra Agent ID · Unified AI Gateway · `169.254.169.254` IMDS Block · 5 Hard Boundaries) · **183 Chapters, 178 IaC Blueprints & 177 Physics Simulators (`186/186` Unit Tests Passing)** | 8 Verified SVG Diagrams + [7-Card Benchmark Report (EN)](docs/azure-agent-infra/test-report.en.md) / [中文](docs/azure-agent-infra/test-report.md) + [4-Page Bilingual H5 Suite](docs/azure-agent-infra/h5/index.html) + [Live Demo](docs/azure-agent-infra/demo/run_live_demo.py) | [`evidence.json`](docs/azure-agent-infra/evidence.json) | [`review.md`](docs/azure-agent-infra/review.md) | 2026-10-24 |
| **[Alibaba Cloud AI Infra Full-Stack Evaluation](docs/alibaba-cloud-ai-infra/README.md)** | **L0 T-Head Silicon** (Hanguang 800, Zhenwu PPU, Yitian 710/750, ICN Switch) · **L1 Panjiu AL128 Supernode, CIPU & HPN 7.0 Dual-Plane RDMA** · **L2 PAI Platform** · **L3 Model Studio (Bailian)** | 6 SVG Diagrams + 2 Mermaid Decision Trees | [`evidence.json`](docs/alibaba-cloud-ai-infra/evidence.json) | [`review.md`](docs/alibaba-cloud-ai-infra/review.md) | 2026-10-14 |

---

## 📊 Quick Links — Azure AI Agent Infrastructure Playbook

- **Documentation (Bilingual)**: [English Playbook (`README.en.md`)](docs/azure-agent-infra/README.en.md) · [中文 Playbook (`README.zh-CN.md`)](docs/azure-agent-infra/README.zh-CN.md) · [English 7-Card Report (`test-report.en.md`)](docs/azure-agent-infra/test-report.en.md) · [中文 7-Card 报告 (`test-report.md`)](docs/azure-agent-infra/test-report.md) · [CSP-Locked 7-Card HTML (`test-report-cards.html`)](docs/azure-agent-infra/test-report-cards.html)
- **Interactive Bilingual H5 Workbench (`EN / 中文`)**: [Playbook Atlas (`h5/index.html`)](docs/azure-agent-infra/h5/index.html) · [Live Simulator (`h5/demo.html`)](docs/azure-agent-infra/h5/demo.html) · [Decision Engine, 100K Sizer & 63 Simulators (`h5/selector.html`)](docs/azure-agent-infra/h5/selector.html) · [64 Production Blueprints (`blueprints/`)](docs/azure-agent-infra/blueprints)
