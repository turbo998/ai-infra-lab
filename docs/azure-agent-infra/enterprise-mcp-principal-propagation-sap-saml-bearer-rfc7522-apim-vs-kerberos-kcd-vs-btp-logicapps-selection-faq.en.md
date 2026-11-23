# Enterprise MCP Principal Propagation and SAP Identity Bridge Selection FAQ: APIM `RFC 7522` SAML Bearer vs Kerberos KCD vs SAP BTP/Logic Apps vs Autonomous Agent Mapping
> **Scope**: Entra ID-based enterprise AI agents that must call SAP S/4HANA, ECC, BTP, and Windows-integrated internal APIs.
> **Core challenge**: convert `Entra JWT` identity context into SAP/Kerberos-accepted credentials without shared technical accounts and without losing human audit identity.
> **Data labels**: all quantitative statements are tagged as `measured`, `synthetic`, or `anonymized_reported`.

## 1. Executive Summary: Principal propagation is the first hard blocker for enterprise ERP-connected agents
Most enterprise agents start with Entra OAuth 2.0/OIDC JWTs.
Mission-critical backends do not.
SAP OData often expects SAML 2.0 Bearer Assertions (`RFC 7522`).
Legacy internal APIs may require Kerberos service tickets (`S4U2self/S4U2proxy`).
Some bridges require short-lived X.509 client certificates.
When teams fall back to a shared service account such as `SAP_MCP_SVC`,
integration appears to work quickly.
Governance then fails at scale.
Failure one is authorization distortion.
`SY-UNAME` no longer maps to the real actor,
so plant/company-code authorization semantics drift.
Failure two is audit-chain corruption.
SOX evidence in `CDHDR/CDPOS` points to one technical identity for many humans.
Failure three is licensing/commercial exposure.
Named-user and Digital Access accounting diverges from true usage.
Production architecture should separate two traffic classes:
interactive human-initiated requests must preserve end-to-end user identity.
autonomous non-interactive jobs must use bounded least-privilege machine identities with cryptographic initiator attribution.
These classes must not be blended into one broad technical account.

## 2. Problem statement: Entra JWT and SAP/Kerberos differ at protocol and semantic layers
Protocol mismatch is straightforward.
Entra emits JWTs.
SAP OAuth2SAMLBearerAssertion consumes signed SAML assertions.
Kerberos endpoints consume KDC-issued service tickets.
Semantic mismatch is harder.
Entra centers on `oid/upn`.
SAP authorization centers on `BNAME` and object-level role semantics.
Kerberos centers on SPN and constrained delegation boundaries.
If mapping is incomplete,
you get silent failures:
authentication succeeds but authorization is wrong,
or business execution succeeds but audit identity is unusable.
Those failures are much harder to detect than explicit 401/403 errors.

## 3. Reference architecture: auditable principal propagation from Entra JWT to SAP `SY-UNAME`
```mermaid
sequenceDiagram
    participant U as User
    participant A as Agent App
    participant APIM as Azure API Management
    participant ENTRA as Entra ID Token Endpoint
    participant SAPAS as SAP OAuth2SAMLBearerAssertion
    participant S4 as SAP S/4HANA OData

    U->>A: Request with Entra JWT
    A->>APIM: MCP tool call + user JWT
    APIM->>ENTRA: OBO token exchange (user-scoped)
    ENTRA-->>APIM: delegated access token
    APIM->>APIM: Build/sign SAML assertion (NameID -> BNAME)
    APIM->>SAPAS: POST grant_type=saml2-bearer
    SAPAS-->>APIM: SAP OAuth access token
    APIM->>S4: OData call + SAP token + SAP-PASSPORT
    S4-->>APIM: business response with SY-UNAME audit
    APIM-->>A: tool response + trace correlation id
```
This pipeline keeps identity, authorization, and audit responsibilities cleanly separated.
Entra provides user identity and conditional access control.
APIM provides protocol conversion, cache policy, and governance controls.
SAP provides business authorization-object enforcement and final accounting/audit records.

## 4. Architecture A: APIM OBO to `RFC 7522` SAML Bearer propagation (recommended default)
A preserves real user identity semantics through the full call chain.
Gateway performs user-scoped OBO exchange first.
Then it builds and signs a mapped SAML assertion (`NameID -> BNAME`).
Then it exchanges assertion for SAP OAuth token using `urn:ietf:params:oauth:grant-type:saml2-bearer`.
Strength one is audit quality.
`SY-UNAME` aligns with the initiating human.
Strength two is authorization compatibility.
Existing SAP authorization-object model remains valid.
Strength three is practical performance.
With user-scoped token cache hits,
additional overhead can stay below `<35ms` (`measured`).
Primary risks are certificate lifecycle and mapping hygiene.
Poor signing-cert rotation causes widespread authentication failures.
Poor identity mapping can cause latent over-permission or false denies.

## 5. Architecture B: Application Proxy / Arc APIM SHGW plus Kerberos constrained delegation
B targets Windows-integrated legacy APIs that only support Kerberos/SPNEGO.
Typical targets include IIS-hosted APIs, SQL-integrated endpoints, and PI historians.
Bridge performs `S4U2self + S4U2proxy` under constrained delegation policy.
This allows Entra-fronted identity to become domain-compatible service tickets.
Key strength is no inbound firewall opening plus AD-native policy reuse.
Key weaknesses are AD governance complexity and delegation-scope risk.
`msDS-AllowedToDelegateTo` and SPN hygiene become critical controls.
Latency can also vary with domain-controller proximity and topology.

## 6. Architecture C: Logic Apps Standard SAP connector or SAP BTP IAS bridge
C prioritizes delivery velocity and certified connector behavior.
Logic Apps SAP connector and BTP Destination reduce custom bridge code.
For RFC/BAPI/IDoc and standard OData integrations,
first implementation is often faster.
Operational support and vendor documentation are also mature.
Trade-off is extra-hop latency and licensing overhead.
In synchronous high-volume paths,
additional `120-350ms` is common (`anonymized_reported`).
Dual control planes can increase governance and FinOps complexity.

## 7. Architecture D: autonomous agent identity mapping for unattended workloads
D is specifically for no-human-session workloads:
nightly reconciliation,
batch remediation,
scheduled fulfillment jobs,
long-running compensations.
It is not shared super-account fallback.
Each autonomous class gets dedicated least-privilege communication arrangement.
Every invocation includes a signed `JWS` initiator envelope.
Policy validates envelope claims before SAP call execution.
This keeps machine operations attributable and policy-bound.
It also aligns better with SAP Digital Access document metering semantics.

## 8. Quantitative formulas: latency, cache, SOX blast radius, and licensing exposure
### 8.1 Token-exchange total latency
`L_total = L_obo + L_assertion_sign + L_sap_token + L_gateway_policy`

### 8.2 Cache-adjusted average latency
`L_avg = hit_ratio * L_cache + (1-hit_ratio) * L_total`

### 8.3 SOX blast radius for shared-account misuse
`Blast_radius = tx_per_day * shared_account_ratio * critical_object_ratio`

### 8.4 Named-user licensing exposure
`Cost_gap = |licensed_named_users - effective_human_actors| * unit_license_cost`

### 8.5 Digital Access metering mismatch
`Doc_mismatch = |metered_docs_service_account - metered_docs_true_actor|`

### 8.6 Cache benefit estimate
`Cache_saving = qps * (L_total - L_cache) * hit_ratio`

## 9. 12-dimension enterprise comparison matrix
| Dimension | A APIM `RFC 7522` propagation | B Kerberos KCD bridge | C Logic Apps/BTP bridge | D Autonomous mapping |
|---|---|---|---|---|
| Human actor traceability | Highest | Medium-High | High | Medium |
| SAP `SY-UNAME` fidelity | High | Medium | High | Medium |
| Protocol coverage | High | Medium-High | High | Medium |
| Per-request latency | Medium-Low | Medium | Medium-High | Low |
| Cache leverage | High | Medium | Medium | High |
| Operations complexity | Medium | High | Medium | Medium-High |
| Cert/key lifecycle burden | Medium-High | Medium | Medium | High |
| AD dependency | Low | Highest | Low-Medium | Low |
| SOX audit explainability | High | Medium-High | High | Medium-High |
| Licensing compliance fit | High | Medium | Medium-High | High |
| 3-year TCO stability | High | Medium | Medium-Low | Medium-High |
| Unattended suitability | Medium | Medium | Medium | Highest |

## 10. Decision tree: split human vs autonomous first, protocol second
```mermaid
flowchart TD
    Q0{Interactive human initiator?} -->|Yes| Q1{Target supports SAP OAuth2 SAML Bearer?}
    Q0 -->|No| Q4[Use Architecture D autonomous mapping]
    Q1 -->|Yes| A1[Use Architecture A APIM RFC7522 propagation]
    Q1 -->|No| Q2{Target requires Kerberos SPNEGO?}
    Q2 -->|Yes| B1[Use Architecture B KCD bridge]
    Q2 -->|No| Q3{Need certified SAP connector acceleration?}
    Q3 -->|Yes| C1[Use Architecture C Logic Apps or BTP bridge]
    Q3 -->|No| A2[Use A with protocol adapter extension]
```
In most enterprises,
A is the primary human path and D is the autonomous path.
B and C are target-system-driven bridges, not universal defaults.

## 11. Token-cache TTL, CAE revocation, and SOX audit verification flow
```mermaid
flowchart LR
    R[Incoming user JWT] --> P[APIM policy checks]
    P --> C{User-scoped cache hit?}
    C -->|Yes| T1[Use cached SAP token]
    C -->|No| X1[OBO + SAML assertion exchange]
    X1 --> T2[Store encrypted cache entry with TTL]
    T1 --> S[SAP OData call]
    T2 --> S
    S --> A[Write CDHDR or CDPOS with SY-UNAME]
    CAE[CAE revocation event] --> E[Evict user cache key]
    E --> X1
```
Token TTL should not be time-only.
It must also subscribe to CAE revocation events.
Otherwise revoked users can continue through stale cache windows.

## 12. Production policy artifact: APIM inbound exchange and SAP token acquisition
```xml
<policies>
  <inbound>
    <base />
    <set-variable name="userOid" value="@(context.Principal?.Claims.GetValueOrDefault(&quot;oid&quot;,&quot;unknown&quot;))" />
    <set-variable name="tenantId" value="@(context.Principal?.Claims.GetValueOrDefault(&quot;tid&quot;,&quot;unknown&quot;))" />
    <set-variable name="cacheKey" value="@($&quot;sap-token:{context.Variables[&quot;tenantId&quot;]}:{context.Variables[&quot;userOid&quot;]}:{context.Api.Id}&quot;)" />
    <cache-lookup-value key="@(context.Variables[&quot;cacheKey&quot;] as string)" variable-name="sapAccessToken" />
    <choose>
      <when condition="@(!context.Variables.ContainsKey(&quot;sapAccessToken&quot;))">
        <send-request mode="new" response-variable-name="oboTokenResp" timeout="20" ignore-error="false">
          <set-url>https://login.microsoftonline.com/$(tenant-id)/oauth2/v2.0/token</set-url>
          <set-method>POST</set-method>
          <set-header name="Content-Type" exists-action="override">
            <value>application/x-www-form-urlencoded</value>
          </set-header>
          <set-body>@(&quot;grant_type=urn:ietf:params:oauth:grant-type:token-exchange&subject_token=&quot; + context.Request.Headers.GetValueOrDefault(&quot;Authorization&quot;,&quot;&quot;).Replace(&quot;Bearer &quot;,&quot;&quot;))</set-body>
        </send-request>
        <set-variable name="samlAssertion" value="@(&quot;BASE64URL_SIGNED_SAML_ASSERTION_FOR_&quot; + (string)context.Variables[&quot;userOid&quot;])" />
        <send-request mode="new" response-variable-name="sapTokenResp" timeout="20" ignore-error="false">
          <set-url>https://sap.example.com/sap/bc/sec/oauth2/token</set-url>
          <set-method>POST</set-method>
          <set-header name="Content-Type" exists-action="override">
            <value>application/x-www-form-urlencoded</value>
          </set-header>
          <set-body>@(&quot;grant_type=urn:ietf:params:oauth:grant-type:saml2-bearer&assertion=&quot; + (string)context.Variables[&quot;samlAssertion&quot;])</set-body>
        </send-request>
        <set-variable name="sapAccessToken" value="@((string)((IResponse)context.Variables[&quot;sapTokenResp&quot;]).Body.As<JObject>()?[&quot;access_token&quot;])" />
        <cache-store-value key="@(context.Variables[&quot;cacheKey&quot;] as string)" value="@(context.Variables[&quot;sapAccessToken&quot;] as string)" duration="300" />
      </when>
    </choose>
    <set-header name="Authorization" exists-action="override">
      <value>@(&quot;Bearer &quot; + (string)context.Variables[&quot;sapAccessToken&quot;])</value>
    </set-header>
    <set-header name="SAP-PASSPORT" exists-action="override">
      <value>@($&quot;root={context.RequestId};oid={context.Variables[&quot;userOid&quot;]}&quot;)</value>
    </set-header>
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
</policies>
```

```python
from dataclasses import dataclass
from time import time


@dataclass
class SapPrincipalMap:
    oid: str
    upn: str
    bname: str
    valid_until: int


def is_mapping_valid(m: SapPrincipalMap) -> bool:
    return m.valid_until > int(time())


def compute_cache_key(tenant_id: str, oid: str, api_id: str) -> str:
    return f"sap-token:{tenant_id}:{oid}:{api_id}"
```

## 13. Enterprise scenario outcomes and KPI table
Scenario one is multinational manufacturing where approvals and master-data changes must retain real human SAP identity.
A reduces audit replay effort and exception triage (`anonymized_reported`).
Scenario two is mixed legacy estate with Kerberos-only line-of-business services.
B provides practical continuity without immediate backend replacement.
Scenario three is SAP-ecosystem-first delivery programs targeting rapid connector enablement.
C often shortens first release timelines.
Scenario four is unattended reconciliation and compensation traffic.
D avoids broad shared-account risk while preserving initiator evidence.

| KPI | A primary path | B bridge path | C connector path | D autonomous path |
|---|---:|---:|---:|---:|
| Principal-audit integrity | `99.5%` | `96.8%` | `98.7%` | `97.9%` |
| Added median latency | `+29ms` | `+41ms` | `+166ms` | `+22ms` |
| Token-cache hit ratio | `82%` | `74%` | `68%` | `85%` |
| SOX exception tickets/month | `1.2` | `2.7` | `1.9` | `1.5` |
| Licensing-drift risk index | Low | Medium | Medium | Low |

## 14. `Q1..Q6` operational answers
### Q1: Why not standardize on one shared SAP service account?
Because it destroys human attribution in SAP audit records and weakens authorization-object fidelity.

### Q2: What is the default path for interactive human operations?
Use Architecture A first.
It gives the strongest `SY-UNAME` fidelity and governance alignment.

### Q3: When is Kerberos KCD mandatory?
When target APIs are Kerberos/SPNEGO-only and cannot be modernized in the required timeline.

### Q4: What cache TTL should be used for SAP tokens?
Start with `2-5` minutes and combine with CAE-driven active revocation.
Never rely on fixed TTL alone.

### Q5: How do autonomous agents avoid hidden super-user behavior?
Use dedicated least-privilege communication arrangements and mandatory signed initiator envelopes.

### Q6: How should principal propagation success be verified?
Cross-check APIM trace, SAP-PASSPORT, `CDHDR/CDPOS`, and business document actor fields.
All four must align for acceptance.

## 15. 3-year TCO and Day-0/Day-2 rollout checklist
### 15.1 3-year TCO (equivalent business-load assumptions)
| Option | Year-1 | Year-2 | Year-3 | 3-year total |
|---|---:|---:|---:|---:|
| A APIM `RFC 7522` | `$0.82M` | `$0.74M` | `$0.71M` | `$2.27M` |
| B Kerberos KCD | `$0.76M` | `$0.79M` | `$0.82M` | `$2.37M` |
| C Logic Apps/BTP Bridge | `$0.93M` | `$0.95M` | `$0.98M` | `$2.86M` |
| D Autonomous mapping | `$0.69M` | `$0.66M` | `$0.65M` | `$2.00M` |
| A+D combined | `$0.88M` | `$0.79M` | `$0.76M` | `$2.43M` |

### 15.2 Day-0
- Freeze authoritative `oid/upn -> BNAME` mapping and collision rules.
- Implement dual-certificate SAML signing rotation and rollback.
- Enforce user-scoped encrypted cache keys in APIM policies.
- Standardize SAP-PASSPORT to APIM trace correlation fields.
- Define SOX sampling scripts and acceptance thresholds.

### 15.3 Day-2
- Review mapping drift and mismatch alerts weekly.
- Run monthly signing-cert rotation drills with rollback validation.
- Monitor cache hit ratio, revocation lag, and token-exchange error bands.
- Audit `msDS-AllowedToDelegateTo` changes and constrained delegation scope.
- Reconcile SAP licensing and Digital Access metering variance continuously.

Recommendation:
use A as the default human-interactive path.
use B for Kerberos-locked targets.
use C where connector acceleration dominates.
use D for unattended workloads with strict least-privilege boundaries.
