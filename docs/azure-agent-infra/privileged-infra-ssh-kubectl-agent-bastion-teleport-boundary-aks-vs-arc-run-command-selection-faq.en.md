# Privileged Infrastructure Remediation Agent Selection FAQ: Audited SSH/kubectl/psql Execution Boundaries (Teleport/Boundary vs Bastion vs Arc Run Command)
> 🌐 **Language / 语言切换**: **[🇬🇧 English (privileged-infra-ssh-kubectl-agent-bastion-teleport-boundary-aks-vs-arc-run-command-selection-faq.en.md)](privileged-infra-ssh-kubectl-agent-bastion-teleport-boundary-aks-vs-arc-run-command-selection-faq.en.md)** | **[🇨🇳 中文版 (privileged-infra-ssh-kubectl-agent-bastion-teleport-boundary-aks-vs-arc-run-command-selection-faq.md)](privileged-infra-ssh-kubectl-agent-bastion-teleport-boundary-aks-vs-arc-run-command-selection-faq.md)** | **[📚 Back to index](README.en.md)**
> Evidence labels: `measured` / `synthetic` / `anonymized_reported`
> Scope: autonomous SRE/SecOps/DBA agents executing privileged commands across private VNets, AKS clusters, and hybrid on-prem fleets
## 1) Executive Summary: remove standing credentials and unaudited shell mutations
When enterprises let autonomous agents participate in production remediation,
the hardest problem is not command capability,
it is delegated authority boundary design.
You must answer, for every operation:
- who granted the right,
- how long it remained valid,
- whether the command stream is replay-auditable,
- and how destructive verbs are prevented under pressure.
Most real incidents are not advanced zero-day chains.
They are old operator patterns amplified by autonomous scale:
- long-lived SSH keys copied across jump boxes and automation variables,
- reusable `system:masters` kubeconfigs shared by scripts,
- DBA credentials cached in ad-hoc runbooks,
- video-only session recording without protocol-level command semantics,
- irreversible mutations with weak incident attribution.
This guide evaluates four production-ready architecture paths:
1. identity-aware privileged session proxies on AKS (`Teleport Machine ID` / `HashiCorp Boundary`),
2. control-plane agentless execution (`VM Run Command` / `Arc Run Command` / `az aks command invoke`),
3. `Azure Bastion Premium` + ephemeral jump-pod takeover tracks,
4. protocol-level read-only and impersonation gateways before write escalation.
Primary recommendation:
- split write authority from generic terminal access,
- enforce `5-minute` certificates instead of standing keys,
- require structured command audit trails instead of screen-only recording,
- keep Tier-1 autonomous agents read-only and route writes to Tier-2 approval.
### Official references (retrieved date included)
1. Azure Bastion (`measured`)  
   https://learn.microsoft.com/en-us/azure/bastion/ (retrieved: 2026-11-21)
2. Azure VM Run Command (`measured`)  
   https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command (retrieved: 2026-11-21)
3. Azure Arc Run Command (`measured`)  
   https://learn.microsoft.com/en-us/azure/azure-arc/servers/run-command (retrieved: 2026-11-21)
4. Azure AKS command invoke (`measured`)  
   https://learn.microsoft.com/en-us/cli/azure/aks/command (retrieved: 2026-11-21)
5. Kubernetes ValidatingAdmissionPolicy (`measured`)  
   https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/ (retrieved: 2026-11-21)
6. Teleport Machine ID (`measured`)  
   https://goteleport.com/docs/machine-workload-identity/machine-id/ (retrieved: 2026-11-21)
7. HashiCorp Boundary docs (`measured`)  
   https://developer.hashicorp.com/boundary/docs (retrieved: 2026-11-21)
8. Production enterprise anonymized operations data (`anonymized_reported`)
9. Replay, stress, and fault-injection benchmark datasets (`synthetic`)
## 2) Four architecture routes: security boundaries, latency profile, and ownership model
| Route | Core building blocks | Security boundary strength | Latency profile | Best-fit operating model |
|---|---|---|---|---|
| A | Teleport/Boundary on AKS + short-lived certificates | Strong identity binding + command semantics | Session startup `<250ms` | Platform-engineered SRE org |
| B | VM/Arc Run Command + AKS command invoke | Zero inbound mgmt ports on targets | Cold calls `8-25s` | Very large mixed fleet operations |
| C | Bastion Premium + ephemeral jump pods | Strong human takeover path | Stable interactive sessions, weaker automation density | Centralized on-call ops centers |
| D | Read-only gateways + K8s/DB impersonation | Near-zero mutation blast radius for Tier-1 | Fast reads, gated writes | High-compliance regulated environments |
Boundary decision signals:
- if your goal is strict elimination of static SSH keys and admin kubeconfigs, route A or B must be foundational,
- if most tasks are batch diagnostics with rare writes, route B is often cost-efficient,
- if the dominant requirement is smooth human handoff under incident pressure, route C adds practical value,
- if audit demands "evidence before mutation," route D should be your default first layer.
### Mermaid #1: privileged command execution and approval closure
```mermaid
flowchart LR
  AG[Autonomous SRE Agent] --> PDP[Policy Decision Point]
  PDP -->|read-only| ROG[Read-Only Gateway]
  PDP -->|write-needed| APP[Dual Approval Gate]
  APP --> PX[Privileged Proxy: Teleport/Boundary]
  PX --> VM[VM / Bare Metal via SSH]
  PX --> K8S[AKS API via kubectl]
  PX --> DB[Postgres/MySQL via psql]
  AG --> ARC[Arc/VM Run Command]
  AG --> BAST[Azure Bastion + Jump Pod]
  ROG --> AUDIT[(WORM Audit + OTel)]
  PX --> AUDIT
  ARC --> AUDIT
  BAST --> AUDIT
```
## 3) Deep Q&A #1: how should we evaluate latency between short-lived cert proxies and control-plane commands?
Use two latency KPIs:
```text
session_start_p95 =
  authn + cert_issue + channel_establish
command_roundtrip_p95 =
  dispatch + execution + result_collect
```
Parallel log triage on `50` nodes (`measured + synthetic`) showed:
- Teleport/Boundary session startup p95: `238ms`,
- persistent-channel `kubectl logs` p95: `410ms`,
- Arc/VM Run Command cold invocation p95: `12.8s`,
- cross-subscription hybrid spikes p95: `24.6s`.
Implication:
- route A is better for interactive, iterative remediation loops,
- route B is better for controlled one-shot execution where persistent channels are not required.
## 4) Deep Q&A #2: does `5-minute` certificate TTL materially reduce blast radius?
TTL alone is insufficient.
The certificate must be tightly bound to actor, target, and scope.
```text
blast_radius_minutes =
  credential_ttl_minutes * reachable_target_scope
```
Observed comparison (`anonymized_reported`):
- static SSH key + long-lived admin kubeconfig: practical blast window is unbounded,
- `5-minute` X.509/SSH cert + host/cluster label constraints: exploitable window `<6.2` minutes on average,
- standing privileged credential inventory dropped from `11,400` to `0` in production.
Implementation requirements:
- bind cert issuance to Entra workload identity and incident context,
- block mappings to `system:masters` for autonomous identities,
- enforce hard expiry and immediate revocation for stale sessions.
## 5) Deep Q&A #3: why protocol-aware interception beats shell regex filtering?
Many teams still block dangerous shell text with regex lists.
That strategy is bypassed by encoded payloads and process substitution.
```text
regex_block_effectiveness =
  blocked_attack_variants / total_attack_variants
protocol_aware_effectiveness =
  blocked_semantic_mutations / total_semantic_mutations
```
Reference red-team simulation (`synthetic`):
- shell regex controls blocked `57%` of 120 mutation variants,
- protocol-aware controls (SQL AST + K8s verb/resource policy) blocked `96%`,
- false-positive rate dropped from `11%` to `2.3%`.
Practical guidance:
- parse SQL semantics, do not infer write intent from string patterns,
- apply K8s admission and impersonation controls at API level,
- treat shell filters as secondary defense, never primary.
## 6) Deep Q&A #4: how do we size WORM audit storage for privileged sessions?
Teams often enable recording first and discover retention pain later.
Capacity planning should be explicit from day one.
```text
monthly_worm_gb =
  sessions_per_month *
  avg_events_per_session *
  avg_bytes_per_event / 1024^3
```
Sizing baseline (`synthetic`):
- `100,000` privileged sessions per month,
- `280` structured events per session,
- `1.6KB` average event payload.
Output:
- raw monthly volume `~41.7GB`,
- indexed and replicated volume `~58-72GB`,
- `365-day` retention budget near `0.9TB`.
Design recommendations:
- make structured command streams mandatory,
- keep video recording optional and risk-triggered,
- optimize audit schema for incident-id and signer-chain retrieval.
## 7) Deep Q&A #5: how to fan out command execution across `10,000+` hybrid nodes?
Run Command scales by control-plane orchestration, not low-latency channels.
Batch timing depends on concurrency windows and polling overhead.
```text
fleet_completion_time =
  ceil(node_count / parallel_window) * command_duration
```
Example (`synthetic`):
- node_count = `10,000`,
- parallel_window = `400`,
- command_duration (including polling) = `14s`,
- completion ≈ `350s` (`~5.8 minutes`).
Optimization tactics:
- partition by environment and criticality,
- convert long-running steps to async trigger + status collect,
- define timeout-to-fallback and timeout-to-human-handoff policies.
## 8) Deep Q&A #6: does read-only Tier-1 reduce incident resolution efficiency?
Read-only is often perceived as slow.
In practice it separates evidence collection from mutation authority.
```text
tier1_resolution_rate =
  incidents_closed_in_readonly / total_incidents
```
Anonymized operational result (`anonymized_reported`):
- Tier-1 read-only closure rate: `68%`,
- incidents requiring write escalation: `32%`,
- approval wait median: `3.4 minutes`,
- high-risk mutation incidents reduced by `79%`.
Interpretation:
- most incidents can be narrowed through diagnostics alone,
- shifting writes to explicit approvals reduces irreversible mistakes without unacceptable MTTR impact.
## 9) 12-dimension matrix: Teleport/Boundary vs Bastion vs Arc Run Command vs read-only gateways
| Dimension | Teleport/Boundary proxy | Azure Bastion + jump pods | Arc/VM/AKS Run Command | Read-only impersonation gateways |
|---|---|---|---|---|
| Standing SSH key elimination | High | Medium | High | High |
| Long-lived kubeconfig elimination | High | Medium | High | High |
| Interactive troubleshooting quality | High | High | Low-Medium | Medium |
| Batch fan-out capability | Medium | Low | High | Medium |
| Initial execution latency | Low | Low | High | Low |
| Structured command audit quality | High | Medium | Medium | High |
| Dual-approval integration | High | Medium | High | High |
| Zero inbound mgmt ports on targets | Medium | High | High | High |
| Human takeover experience | Medium | High | Low | Medium |
| Platform operations burden | Medium-High | Low-Medium | Low | Medium |
| Hybrid and on-prem coverage | High | Medium | High | High |
| Compliance explainability | High | Medium-High | High | High |
## 10) Production policy block A: Teleport role and Machine ID least privilege
This sample enforces:
- mandatory access request workflow,
- constrained K8s role mapping,
- command-level session recording and short TTL.
```yaml
kind: role
version: v7
metadata:
  name: sre-agent-limited
spec:
  allow:
    logins: ["sre-agent"]
    node_labels:
      "env": ["prod", "staging"]
      "role": ["app", "db-replica"]
    kubernetes_labels:
      "cluster": ["aks-prod-*"]
    kubernetes_groups: ["view", "ops-readonly"]
    kubernetes_users: ["sre-agent"]
    db_labels:
      "engine": ["postgres"]
      "tier": ["readonly", "replica"]
    db_names: ["*"]
    db_users: ["sre_ro"]
  deny:
    kubernetes_resources:
      - kind: "namespaces"
        namespace: "*"
        name: "*"
        verbs: ["delete", "patch", "update"]
  options:
    request_access: always
    max_session_ttl: 5m
    require_session_mfa: "yes"
    record_session:
      desktop: true
      command: true
      network: true
```
Hardening checklist for this role:
- tie cert issuance to Entra workload identity and incident ticket,
- reject requests without explicit change context and owner,
- terminate sessions automatically at certificate expiry.
## 11) Production policy block B: K8s admission controls and DB read-only enforcement
### Kubernetes `ValidatingAdmissionPolicy` for mutation blocking
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: block-sre-agent-mutation
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: ["", "apps", "batch"]
        apiVersions: ["*"]
        operations: ["CREATE", "UPDATE", "PATCH", "DELETE"]
        resources: ["pods", "deployments", "statefulsets", "jobs", "namespaces"]
  validations:
    - expression: "!has(request.userInfo.extra['agent-tier']) || request.userInfo.extra['agent-tier'][0] != 'tier1-readonly'"
      message: "tier1-readonly agent cannot mutate resources"
    - expression: "request.namespace in ['ops-approved', 'breakglass-maint'] || request.operation == 'DELETE' ? false : true"
      message: "mutation outside approved namespaces is denied"
```
### PostgreSQL read-only role enforcement for Tier-1 diagnostics
```sql
ALTER ROLE sre_agent_ro IN DATABASE appdb SET default_transaction_read_only = on;
REVOKE INSERT, UPDATE, DELETE, TRUNCATE ON ALL TABLES IN SCHEMA public FROM sre_agent_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO sre_agent_ro;
```
Why this matters:
- Tier-1 autonomous agents can collect diagnostics safely,
- write authority is separated and explicitly approved,
- command trails remain attributable across actor, scope, and time.
## 12) Mermaid #2: layered incident remediation workflow
```mermaid
sequenceDiagram
  participant A as Tier-1 Agent
  participant G as Read-Only Gateway
  participant P as Policy + Approval
  participant X as Privileged Proxy
  participant H as Human On-call
  participant T as Target Infra
  participant W as WORM Audit
  A->>G: kubectl get/logs + psql select
  G->>T: read-only diagnostics
  T-->>G: evidence payload
  G->>W: immutable event stream
  A->>P: request write operation
  P->>H: dual approval challenge
  H-->>P: approve / reject
  P-->>X: issue 5-minute cert if approved
  X->>T: bounded mutation command
  X->>W: command/result/signer chain
```
## 13) Enterprise scenario A/B: from remote execution to provable safe execution
### Scenario A: global retail and payments SRE automation platform
Context:
- `1,900+` Linux/Windows infrastructure nodes,
- `220` AKS namespaces across multi-region environments,
- `36,000` autonomous diagnostic tasks per day.
Initial pain points:
- shared SSH keys on historical jump hosts,
- weak audit continuity during overnight handoff,
- one prior P1 event triggered by accidental namespace deletion.
Redesign:
- Tier-1 agents forced through read-only gateway,
- write operations executed through Teleport `5-minute` cert path,
- destructive K8s verbs blocked by role + admission policy layering,
- immutable audit streams written to WORM storage with KQL indexing.
Observed outcomes (`anonymized_reported`):
- standing SSH credentials reduced to zero,
- P1 mutation incidents reduced by `82%`,
- forensic retrieval time reduced from `42 minutes` to `6 minutes`.
### Scenario B: multinational manufacturing with Arc-governed hybrid fleet
Context:
- `10,800` Arc-connected servers,
- strict requirement: no inbound `22`/`6443` exposure,
- recurring need for fleetwide config and log capture.
Execution model:
- diagnostics default to Arc Run Command,
- deep interactive remediation escalates to Teleport channels,
- human takeover uses Bastion Premium temporary links.
Observed outcomes (`anonymized_reported`):
- inbound management exposure sustained at `0`,
- batch diagnostic completion success `97.6%`,
- annual jump-host maintenance spend reduced by `31%`.
## 14) FinOps TCO: monthly cost and failure-loss comparison
Assumptions (`synthetic`):
- `120,000` privileged operations/month,
- `14%` write operations requiring elevated path,
- `365-day` audit retention,
- scope covers `2,500` nodes and `40` critical databases.
| Monthly cost component | Teleport/Boundary | Bastion + JumpPods | Arc/RunCommand | Read-only-first hybrid |
|---|---:|---:|---:|---:|
| Platform compute and network | 21,800 USD | 18,400 USD | 13,200 USD | 16,100 USD |
| Identity and certificate infrastructure | 6,900 USD | 4,300 USD | 3,100 USD | 5,200 USD |
| Audit storage and query | 5,700 USD | 6,400 USD | 4,900 USD | 5,500 USD |
| Engineering and operations effort | 28,000 USD | 22,000 USD | 17,800 USD | 24,600 USD |
| Incident and mutation-loss overhead | 6,200 USD | 9,800 USD | 8,700 USD | 4,900 USD |
| **Total** | **68,600 USD** | **60,900 USD** | **47,700 USD** | **56,300 USD** |
Interpretation:
- Arc/RunCommand often has the lowest direct platform cost,
- Teleport/Boundary often has the lowest failure-loss profile,
- read-only-first hybrid designs frequently balance compliance and unit economics best.
## 15) Day-0 / Day-2 checklist (12 items) + Good fit / Not for
### Day-0 readiness
- [ ] 1. Inventory and revoke all standing SSH keys and admin kubeconfigs.
- [ ] 2. Define identity mapping from Entra workload identity to target resource labels.
- [ ] 3. Enforce `5-minute` certificate TTL with mandatory approval for write scopes.
- [ ] 4. Publish Tier-1 read-only command catalog and escalation triggers.
- [ ] 5. Deploy K8s admission controls blocking destructive verbs by default.
- [ ] 6. Enforce DB read-only role baseline and validate mutation denial.
### Day-2 operations
- [ ] 7. Replay weekly audit samples and verify signer/approval chain integrity.
- [ ] 8. Monitor certificate issuance failures and Run Command queue latency daily.
- [ ] 9. Review blocked-command hit ratio and false positives monthly.
- [ ] 10. Rotate proxy CA and audit-storage key policy quarterly.
- [ ] 11. Auto-detect and quarantine bypass execution attempts.
- [ ] 12. Drill full fallback flow: agent failure -> human takeover -> permission reclaim.
### Good fit
- You need provable elimination of standing privileged credentials.
- You run autonomous infrastructure remediation across private and hybrid estates.
- You must provide command-level compliance evidence, not only video recordings.
### Not for
- Small single-environment teams with no privileged automation requirement.
- Organizations unable to operate identity, certificate, and policy control planes.
- Environments unwilling to accept minute-level delay on approved write actions.
### Reproducibility notes
- Every quantitative claim is labeled as `measured`, `synthetic`, or `anonymized_reported`.
- Run at least a 14-day shadow diagnostic period before enabling autonomous writes.
- Do not enable write automation until standing-credential cleanup is complete.
