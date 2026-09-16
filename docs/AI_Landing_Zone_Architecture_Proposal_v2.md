# AI Landing Zone Architecture Proposal

## 1. Executive Summary

### Recommendation

> **Adopt a Lightweight AI Landing Zone for the initial 1–2 AI projects, leveraging the existing Azure Landing Zone (vWAN Hub + Azure Firewall), while establishing the security, networking, identity, and governance foundations — and the specific forward-compatible design decisions — required to evolve into a centralized AI Governance Hub (Citadel) when scale justifies it.**

### Why?

ปัจจุบันองค์กรมี:

- AI workloads จำนวน 1–2 projects ในระยะแรก
- Existing Azure Landing Zone (hub/spoke topology with Azure vWAN Hub)
- Centralized Azure Firewall
- ยังไม่มีความจำเป็นต้องมี centralized AI gateway สำหรับหลายทีม
- แต่ AI workloads เป็น production workloads จึงต้องมี security/governance baseline ตั้งแต่เริ่มต้น

ดังนั้นเราจะ **ไม่ deploy full AI Governance Hub ตั้งแต่วันแรก** เนื่องจาก:

1. **Scale ไม่สอดคล้อง** — Governance Hub ออกแบบมาสำหรับ multi-BU / multi-use-case governance
2. **Standing cost สูง** — APIM (Premium/StandardV2) + Event Hub + Cosmos DB + Logic App มี fixed monthly cost ที่มีนัยสำคัญ ในขณะที่ Lightweight path มี incremental platform cost ใกล้ศูนย์ (จ่ายเฉพาะ AI consumption + private endpoints)
3. **Operational overhead** — ต้องมี platform team ดูแล gateway, contracts, usage pipeline

แต่ architecture จะถูกออกแบบให้ evolve ไปสู่ **Central AI Governance Hub** ได้ โดยการ migration เป็นเพียง **endpoint + credential swap** ในระดับ application (ดู Section 9)

### Reference Architecture

Proposal นี้อ้างอิงจาก Microsoft reference architecture:

- **Citadel Governance Hub** (AI Hub Gateway Solution Accelerator) — [github.com/Azure-Samples/ai-hub-gateway-solution-accelerator](https://github.com/Azure-Samples/ai-hub-gateway-solution-accelerator) — reference implementation ของ Layer 1 ใน **AI Citadel Blueprint** (aka.ms/foundry-citadel)
- **AI Landing Zones for Microsoft Foundry** — [github.com/Azure/AI-Landing-Zones](https://github.com/Azure/AI-Landing-Zones) — spoke-side landing zone (Agent Execution Plane) รองรับ **brownfield deployment** ที่ reuse existing VNet และ centralized monitoring
- Terraform variant: [github.com/Azure/terraform-ai-gateway-landing-zone](https://github.com/Azure/terraform-ai-gateway-landing-zone)

**AI Citadel Blueprint** ประกอบด้วย 4 layers:

| Layer | Name | Responsibility | Phase ที่เรา adopt |
|---|---|---|---|
| Layer 1 | Governance Hub | Unified AI gateway, policy-as-code, token limits, cost attribution | **Phase 2** |
| Layer 2 | Agent Operations | Foundry control plane, agent traces, AI evaluations | Phase 1 (project-level) |
| Layer 3 | Agent Management | Agent identity & lifecycle (Agent 365) | Phase 2+ |
| Layer 4 | Security Foundation | Defender for AI, Purview, Entra | **Phase 1** (subscription-level, low cost) |

> Key insight: Lightweight ≠ less secure — เรา adopt Layer 4 (Security Foundation) และ Layer 2 (project-level observability) ตั้งแต่ Phase 1 และ defer เฉพาะ Layer 1 (Gateway) ซึ่งเป็น scale-driven component

---

# 2. Current Situation

```text
                    Azure vWAN
                        │
                   ┌────▼─────┐
                   │   Hub    │
                   │ Firewall │
                   └────┬─────┘
                        │
              Existing Workload Spokes
```

### AI Requirement

ต้องการรองรับ:

- AI application / Agent (1–2 projects)
- Microsoft Foundry / Azure OpenAI
- AI Search / RAG
- Enterprise data
- Private connectivity
- Enterprise identity
- Monitoring & audit

### Key Architectural Question

> **Should we introduce a dedicated centralized AI Governance Hub now, or build the AI foundation incrementally?**

---

# 3. Architecture Principles

### Principle 1 — Security by Design

AI workloads ต้องอยู่ภายใต้ enterprise security controls ตั้งแต่วันแรก

### Principle 2 — Reuse Existing Enterprise Platform

ใช้ infrastructure ที่มีอยู่: Azure vWAN, Azure Firewall, Entra ID, Azure Monitor / central Log Analytics — ไม่สร้าง platform ซ้ำซ้อน

### Principle 3 — Native Controls First

ใช้ native capability ของ service ก่อน (เช่น Azure OpenAI content filtering, per-deployment quota) ก่อนที่จะสร้าง gateway-level control

### Principle 4 — Least Privilege

ใช้ Managed Identity + RBAC แทน static credentials และ disable local auth (API keys) ทุกที่ที่รองรับ

### Principle 5 — Private by Default

Private Endpoint / Private DNS สำหรับ AI services และ data services; disable public network access

### Principle 6 — Centralize When Scale Requires It

ไม่สร้าง centralized AI gateway ก่อนที่ workload volume และ governance requirements จะ justify standing cost และ operational overhead

### Principle 7 — Design for Evolution

ทุก application ใน Phase 1 ต้อง externalize AI endpoint + credentials เพื่อให้ migration ไปสู่ gateway เป็น configuration change ไม่ใช่ code change

---

# 4. Phase 1 — Lightweight AI Landing Zone

## Target Architecture

Phase 1 spoke สามารถ deploy ด้วย **AI Landing Zones (brownfield pattern)** ซึ่งออกแบบมาเพื่อ integrate กับ existing enterprise landing zone โดย reuse existing networking และ centralized monitoring — ไม่ต้องออกแบบ spoke เองทั้งหมด

```text
                              Azure Enterprise
                                   Network
                                      │
                              ┌───────▼───────┐
                              │ Azure vWAN Hub │
                              │               │
                              │ Azure Firewall│
                              └───────┬───────┘
                                      │
                              Hub Connection
                                      │
                       ┌──────────────▼──────────────┐
                       │       AI Workload Spoke     │
                       │   (per project / use case)  │
                       │                             │
                       │  ┌───────────────────────┐  │
                       │  │ AI Application/Agent  │  │
                       │  │                       │  │
                       │  │ endpoint + credential │  │
                       │  │ from Key Vault/config │◄─┼── forward-compatible
                       │  └───────────┬───────────┘  │   design (Section 9)
                       │              │              │
                       │       Managed Identity      │
                       │              │              │
                       │  ┌───────────▼───────────┐  │
                       │  │   AI Services         │  │
                       │  │ Foundry / Azure OpenAI│  │
                       │  │  + content filtering  │  │
                       │  │ AI Search (RAG)       │  │
                       │  └───────────────────────┘  │
                       │                             │
                       │  ┌───────────────────────┐  │
                       │  │ Enterprise Data       │  │
                       │  │ Storage / DB          │  │
                       │  └───────────────────────┘  │
                       │                             │
                       │  ┌───────────────────────┐  │
                       │  │ Key Vault             │  │
                       │  └───────────────────────┘  │
                       │                             │
                       │  All PaaS via Private       │
                       │  Endpoint + Private DNS     │
                       └─────────────────────────────┘

        Subscription-level: Defender for AI · Purview · Azure Policy
```

---

# 5. Phase 1 — Components

| Area | Component | Design |
|---|---|---|
| Network | Azure vWAN | Reuse existing enterprise network |
| Network Security | Azure Firewall | Central inspection + FQDN-based AI egress rules |
| Workload Isolation | AI Spoke VNet (per project) | Deploy via AI Landing Zones brownfield pattern |
| Connectivity | VNet ↔ vWAN Hub | Existing enterprise connectivity pattern |
| AI | Microsoft Foundry / Azure OpenAI | Private endpoint, Entra-only auth, built-in content filtering |
| RAG | Azure AI Search | Private endpoint |
| Data | Storage / DB | Private endpoint |
| Secrets & Config | Azure Key Vault | Private endpoint; holds AI endpoint reference + any credentials |
| Identity | Microsoft Entra ID + Managed Identity | Central enterprise identity; disable local auth where supported |
| Access | Azure RBAC | Least privilege |
| Monitoring | Azure Monitor / Log Analytics | Diagnostic settings to central workspace |
| AI Observability | Foundry tracing / evaluations (project-level) | Citadel Layer 2 at project scope |
| Threat Protection | **Microsoft Defender for Cloud (AI workloads)** | Subscription-level AI security posture (Citadel Layer 4) |
| Data Governance | **Microsoft Purview** | Classification / sensitivity labeling (Citadel Layer 4) |
| Governance | Azure Policy | Region, SKU, private-endpoint-required, deny public network access |
| Cost Attribution | Resource group per project + mandatory tags | Cost Management views per project/BU |
| Deployment | IaC (Bicep/Terraform) + CI/CD | Same tooling family as Citadel accelerator |

---

# 6. Network Design

หนึ่งในเหตุผลหลักที่ไม่ต้องสร้าง Governance Hub ตอนนี้คือ **องค์กรมี network security foundation อยู่แล้ว**

### Network Flow

```text
AI Application ──► AI Spoke ──► vWAN Hub ──► Azure Firewall ──► Enterprise / Internet
```

สำหรับ Azure AI services ที่รองรับ Private Link:

```text
AI Application ──► Private Endpoint ──► Azure AI Service
```

### vWAN-Specific Considerations

| Topic | Design |
|---|---|
| Private DNS resolution | Private DNS zones (`privatelink.openai.azure.com`, `privatelink.search.windows.net`, `privatelink.vaultcore.azure.net`, ฯลฯ) linked ตาม existing enterprise DNS pattern — ผ่าน Azure Firewall DNS proxy หรือ Private DNS Resolver ตามที่องค์กรใช้อยู่ |
| Traffic inspection | ใช้ vWAN **Routing Intent** (secured hub) เพื่อบังคับ spoke-to-spoke และ egress traffic ผ่าน Azure Firewall |
| AI egress control | Firewall application rules อนุญาตเฉพาะ required FQDNs (Azure AI endpoints, Entra, monitoring) — deny direct egress ไปยัง external LLM providers ที่ไม่ได้ approve |

### Key Architectural Point

**Azure Firewall และ AI Governance Gateway มีหน้าที่ต่างกัน**

| Layer | Responsibility |
|---|---|
| Azure vWAN | Enterprise network connectivity |
| Azure Firewall | Network-level security (L3/L4/FQDN) |
| APIM AI Gateway (Phase 2) | API-level / AI-level governance (token limits, model routing, content policy, usage attribution) |

การมี Azure Firewall อยู่แล้ว **ไม่ได้ eliminate the future need for an AI Gateway** — แต่ทำให้เราไม่ต้อง deploy AI Gateway เพื่อแก้ปัญหา network security ที่มี solution อยู่แล้ว

---

# 7. Phase 1 — Security Baseline

## Identity

- Microsoft Entra ID + Managed Identity + Azure RBAC
- **Disable local authentication (API keys)** บน Azure OpenAI / AI Search ที่รองรับ
- No embedded credentials

## Network

- Dedicated AI spoke (per project)
- Central Azure Firewall + routing intent
- Private Endpoint + Private DNS
- Public network access disabled

## AI Safety (native, Phase 1)

- Azure OpenAI / Foundry **built-in content filtering** (default severity + custom configuration ตาม use case)
- Per-deployment **TPM quota** เป็น capacity guardrail
- Foundry evaluations / tracing สำหรับ agent workloads

## Secrets

- Azure Key Vault (private endpoint, RBAC-based access, rotation)
- **AI endpoint URL + credential references เก็บใน Key Vault / app configuration** — ไม่ hard-code ใน application

## Monitoring

- Diagnostic settings → central Log Analytics
- Azure Monitor alerts, Activity Logs
- Defender for Cloud — AI workload threat protection

## Governance

- Azure Policy: allowed regions/SKUs, require private endpoint, deny public access, mandatory tagging (project, BU, environment, cost center)
- Purview data classification สำหรับ data sources ที่ AI ใช้

---

# 8. Deferred Capabilities — and Their Phase 1 Substitutes

เราไม่ได้ "ไม่มี governance" ใน Phase 1 — แต่ใช้ **native substitutes** สำหรับทุก capability ที่ defer:

| Deferred (Phase 2 Gateway capability) | Phase 1 Native Substitute | Adequate for 1–2 projects? |
|---|---|---|
| Central AI Gateway (APIM) | Direct private access + Managed Identity + RBAC | ✅ |
| Rate limiting / token quota per use case | Per-deployment TPM quota บน Azure OpenAI | ✅ (coarse-grained แต่เพียงพอ) |
| Gateway-level Content Safety / Prompt Shield / PII masking | Azure OpenAI built-in content filtering + app-level handling | ✅ พอสำหรับ workloads ปัจจุบัน |
| Central usage & cost analytics (Event Hub → Cosmos → Power BI) | Azure Cost Management per resource group/tags + Azure OpenAI diagnostic logs (token usage) ใน Log Analytics | ✅ ตอบ who/how much ระดับ project ได้ |
| Model access control / approved model pool | Azure Policy + RBAC + IaC review (approved models ผ่าน code review) | ✅ |
| Access Contracts | IaC + RBAC + project-level configuration | ✅ |
| AI Asset Registry (API Center) | Wiki/repo documentation | ✅ assets มีจำนวนน้อย |
| Multi-region routing / failover | Single-region deployment (+ optional per-app retry to secondary) | ✅ ยังไม่มี requirement |
| Semantic caching (Redis) | ไม่จำเป็นที่ volume ปัจจุบัน | ✅ |

**ข้อจำกัดที่ยอมรับ (accepted trade-offs):**

- Usage attribution ละเอียดได้แค่ระดับ project/resource group — ไม่ถึงระดับ per-user/per-feature
- Content policy บังคับที่ service level — application สามารถ config ได้เอง (mitigate ด้วย Azure Policy + code review)
- ไม่มี central choke point สำหรับ audit ทุก AI request (mitigate ด้วย diagnostic logs ของแต่ละ service)

---

# 9. Phase 1 Design Decisions that Enable Phase 2 (Forward Compatibility)

นี่คือหัวใจของ "evolve without redesign" — Citadel onboarding สำหรับ existing agents คือการ **เปลี่ยน endpoint + credentials ให้ชี้ไปที่ unified gateway** ดังนั้น Phase 1 ต้อง commit ต่อ design rules ต่อไปนี้:

| # | Design Rule | ทำไม |
|---|---|---|
| 1 | **Externalize AI endpoint + credentials** — เก็บใน Key Vault / app configuration เท่านั้น ห้าม hard-code | Phase 2 migration = config change (ชี้ endpoint ไป APIM gateway + ใช้ subscription key จาก Access Contract) |
| 2 | **ใช้ standard OpenAI-compatible API surface** (Azure OpenAI / Foundry APIs) ไม่สร้าง custom protocol | APIM gateway insert ได้แบบ transparent |
| 3 | **หนึ่ง spoke / resource group ต่อ project (use case)** | ตรงกับ Citadel recommendation: one Access Contract per BU/use-case/environment |
| 4 | **Mandatory tagging** (project, BU, environment, cost center) | กลายเป็น dimension ของ central cost attribution ใน Phase 2 |
| 5 | **Abstraction layer บาง ๆ สำหรับ LLM client ใน application code** (single module ที่อ่าน endpoint/auth จาก config) | จำกัด blast radius ของ auth model change |
| 6 | **Log token usage ตั้งแต่วันแรก** (diagnostic settings) | มี baseline data สำหรับ sizing Phase 2 (Citadel Sizing Guide / PTU Estimation) |

### ⚠️ Known Migration Change: Authentication Model

| | Phase 1 | Phase 2 (Citadel) |
|---|---|---|
| Auth | Managed Identity → Azure OpenAI โดยตรง | APIM subscription key (+ optional Entra JWT) จาก Access Contract |
| Endpoint | Service private endpoint | Unified gateway endpoint |

การเปลี่ยนแปลงนี้เป็น **configuration change** ถ้าปฏิบัติตาม Design Rules ข้อ 1 และ 5 — Citadel แนะนำเก็บ gateway credentials ใน Key Vault ซึ่งตรงกับ pattern ที่เราใช้อยู่แล้ว

---

# 10. Why Lightweight Architecture?

| Criteria | Lightweight | Full AI Governance Hub |
|---|---:|---:|
| Fit for 1–2 projects | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Initial + standing cost | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Operational simplicity | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Time to deploy | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| Central AI governance | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Multi-BU | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Multi-model | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Usage attribution | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Agent governance | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Future scalability | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### Cost Perspective

- **Governance Hub standing cost**: APIM (Premium หรือ StandardV2 สำหรับ VNet integration) + Event Hub + Cosmos DB + Logic App + optional Redis — fixed monthly platform cost ที่มีนัยสำคัญ ไม่ขึ้นกับ usage
- **Lightweight incremental cost**: private endpoints + Log Analytics ingestion เท่านั้น — จ่ายหลักคือ AI consumption ซึ่งจ่ายเท่ากันทั้งสอง architecture
- **ตัวเลขประมาณการ**: การเริ่มด้วย Phase 1 หลีกเลี่ยง fixed platform cost ประมาณ **~$3,100/เดือน (~$37,000/ปี)** สำหรับ production-grade private hub — ดูรายละเอียดใน **Appendix A**

### Conclusion

> **The Lightweight architecture provides sufficient governance for the current scale without introducing the standing cost and operational overhead of a centralized AI platform prematurely — while Layer 4 security (Defender, Purview) and forward-compatible design rules preserve a low-cost migration path.**

---

# 11. Phase 2 — Central AI Governance Hub (Citadel)

เมื่อ AI usage เติบโตถึง triggers (Section 14) เราจะ deploy **Citadel Governance Hub** ตาม solution accelerator

### Deployment Topology

Citadel รองรับ 2 แบบ: (1) dedicated spoke peered to hub, (2) ใน enterprise hub VNet — เนื่องจากเราใช้ **vWAN (managed hub)** ทางเลือกที่เหมาะสมคือ **dedicated Governance Hub spoke** เชื่อมกับ vWAN hub ซึ่งเป็นแบบที่ Citadel แนะนำสำหรับองค์กรที่ต้องการ network segmentation ชัดเจน

```text
                         Azure vWAN Hub
                              │
                          Firewall
                              │
                 ┌────────────▼────────────┐
                 │ Citadel Governance Hub  │
                 │ (dedicated spoke)       │
                 │                         │
                 │ APIM — Unified Gateway  │
                 │ API Center — Registry   │
                 │ Foundry — Control Plane │
                 │ Content Safety / PII    │
                 │ Event Hub → Logic App   │
                 │        → Cosmos DB      │
                 │ App Insights / LA       │
                 └────────────┬────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
          AI Spoke A      AI Spoke B      AI Spoke C
          (existing        (existing      (new project)
           Phase 1)         Phase 1)
```

---

# 12. Phase 2 — What We Add

## 12.1 Azure API Management — Unified AI Gateway

- Authentication / Authorization (subscription key + JWT)
- Model access control / routing / model aliases
- Token rate limiting & quota per use case
- Load balancing, failover, circuit breaker (AI resiliency)
- Gateway-level Content Safety, Prompt Shield, PII detection & masking
- Central logging

## 12.2 Contract-Driven Governance

Citadel ดำเนินงานด้วย 3 contract types (IaC-declared, version-controlled):

| Contract | Side | Governs |
|---|---|---|
| **Backend Contracts** | Supply | LLM backends & models ที่ gateway route ได้, load balancing, model aliases |
| **Access Contracts** | Demand | ใครใช้ model อะไรได้บ้าง — APIM product/subscription/policy per BU/use-case/environment, quotas, credential delivery ผ่าน Key Vault |
| **Publish Contracts** (Preview) | Publishing | Tools (MCP) และ Agents (A2A) ที่ publish ผ่าน gateway + API Center registration |

จากเดิม (Phase 1):

```text
Project → RBAC → AI Service
```

เป็น:

```text
Business Unit → Use Case → Access Contract → AI Gateway → Approved Model Pool
```

## 12.3 Central Usage & Cost Analytics

```text
APIM → Event Hub → Logic App → Cosmos DB → Power BI
```

ตอบคำถาม: Who / which model / which BU / which use case / how many tokens / what cost — พร้อม chargeback/showback

## 12.4 Onboarding Existing Phase 1 Workloads

ตาม Citadel guidance สำหรับ existing agents:

1. สร้าง Access Contract ต่อ use case (ตรงกับ per-project structure ของเราอยู่แล้ว)
2. Contract provisioning ส่ง gateway endpoint + credentials เข้า Key Vault
3. Application อ่าน config ใหม่ — **ไม่ต้องแก้ application logic** (เพราะ Design Rules ใน Section 9)

---

# 13. Phase 2 — Governance Maturity

```text
                    AI Governance Maturity

Phase 1                         Phase 2
────────                        ────────
Project Governance              Enterprise Governance

RBAC                            Access Contracts
Azure Policy                    APIM Policies
Native content filtering        Gateway Content Safety + PII
Project monitoring              Central AI Observability
Key Vault                       Central Credential Governance
Firewall                        Firewall + AI Gateway
IaC                             Contract-driven IaC
Defender / Purview (L4)         Defender / Purview (L4) — continues
                                   │
                                   ▼
                         Multi-BU / Multi-Model
                         Agent Governance (Agent 365 — L3)
                         Cost Attribution / Chargeback
                         Resiliency / Multi-region
```

---

# 14. Triggers for Moving to Phase 2

ไม่ควรกำหนดเพียงว่า "หลัง 6 เดือนให้ทำ Phase 2" — ควรใช้ **business / technical triggers** โดย review ทุก quarter:

## Scale

- AI projects ≥ 3–4 use cases หรือมีมากกว่า 1 Business Unit ที่ใช้ AI ใน production
- หลาย environments ที่ต้องการ consistent policy

## Governance

- ต้องการ centralized model authorization ที่ application bypass ไม่ได้
- ต้องการ central quota / token rate limiting per use case
- ต้องการ consistent AI safety policies (Prompt Shield, PII masking) ที่ gateway

## Cost

- ต้องการ chargeback / showback ระดับ use case
- AI spend ต่อเดือนเริ่มมีนัยสำคัญเมื่อเทียบกับ Governance Hub standing cost (break-even indicator — ดู Appendix A: ~$3,100/เดือน)

## AI Ecosystem

- ใช้หลาย LLM providers (Foundry + Bedrock + external)
- มี MCP tools / A2A agents ที่ต้อง publish และ govern กลาง
- ต้องการ model routing / failover / semantic caching

## Compliance

- Regulator / audit ต้องการ centralized audit trail ของ AI requests
- ต้องการ policy enforcement ที่ enforce at runtime ทุก request

---

# 15. Evolution Path

```text
                 TODAY
                   │
                   ▼
        ┌──────────────────────┐
        │ Lightweight AI LZ    │
        │                      │
        │ Existing vWAN + FW   │
        │ AI Spoke (brownfield │
        │  AI Landing Zones)   │
        │ Private AI Services  │
        │ Entra / KV / Monitor │
        │ Defender / Purview   │
        │ Forward-compatible   │
        │  design rules (§9)   │
        └──────────┬───────────┘
                   │
                   │ Scale / Governance Trigger (§14)
                   ▼
        ┌──────────────────────┐
        │ Citadel Governance   │
        │ Hub (dedicated spoke)│
        │                      │
        │ APIM AI Gateway      │
        │ API Center           │
        │ Backend Contracts    │
        │ Access Contracts     │
        │ Central Analytics    │
        │                      │
        │ Existing spokes      │
        │ onboard via endpoint │
        │ + credential swap    │
        └──────────┬───────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │ Enterprise AI        │
        │ Platform             │
        │                      │
        │ Multi-BU/Multi-Model │
        │ Publish Contracts    │
        │  (MCP / A2A)         │
        │ Agent 365 (L3)       │
        │ Cost Governance      │
        │ DR / Multi-region    │
        └──────────────────────┘
```

---

# 16. Key Recommendation to Design Committee

> **Recommendation**
>
> We recommend adopting a **Lightweight AI Landing Zone** for the initial 1–2 AI workloads, deployed as dedicated spokes on the existing **Azure Landing Zone (vWAN Hub + Azure Firewall)**, following the brownfield pattern of Microsoft's **AI Landing Zones** guidance.
>
> Phase 1 implements mandatory AI security controls: private connectivity, identity-based access (Managed Identity, no local auth), native content filtering, secrets management, centralized monitoring, Defender for AI, Purview, and policy-based governance — together with **forward-compatible design rules** (externalized endpoints/credentials, standard API surface, per-use-case isolation, mandatory tagging).
>
> A centralized **AI Governance Hub based on Microsoft's Citadel Governance Hub accelerator** (Azure API Management unified AI gateway with contract-driven governance) will be introduced as a dedicated spoke when defined scale, governance, cost, or compliance triggers are reached. Because of the Phase 1 design rules, onboarding existing workloads to the hub will be an endpoint-and-credential configuration change rather than an application redesign.

---

# 17. One-Slide Architecture Decision

## AI Landing Zone — Progressive Architecture

### Current State

> 1–2 AI projects + Existing Azure Landing Zone (vWAN + Firewall)

↓

### Decision

> **Start Lightweight — defer the centralized AI Governance Hub (Citadel) until scale triggers are met**

↓

### Phase 1

```text
vWAN + Firewall
       │
   AI Spoke (per project)
       │
 ┌─────┼──────┐
 AI   Search  Data
       │
   Key Vault (endpoint/credential abstraction)
       │
 Entra + Monitor + Defender + Purview
```

↓

### Future State

> Deploy Citadel Governance Hub as a dedicated spoke; existing workloads onboard via endpoint + credential swap

```text
vWAN + Firewall
       │
Citadel Governance Hub
 (APIM AI Gateway · API Center · Contracts · Analytics)
       │
 ├── AI Spoke A
 ├── AI Spoke B
 └── AI Spoke C
```

### Key Rationale

> **Avoid premature platform cost and complexity while preserving a configuration-level (not redesign-level) path to enterprise-scale AI governance.**

---

# Appendix A — Phase 2 Fixed Platform Cost Estimate

> **Headline: การเริ่มด้วย Phase 1 (Lightweight) หลีกเลี่ยง fixed platform cost ประมาณ ~$3,100/เดือน (~$37,000/ปี) สำหรับ production-grade private Governance Hub — หรืออย่างน้อย ~$1,000/เดือน แม้ใน configuration ที่ lean ที่สุด**

Fixed cost ของ Governance Hub ถูกกำหนดโดย APIM tier เป็นหลัก ซึ่งขึ้นกับระดับ private connectivity ที่ต้องการ จึงประมาณการเป็น 2 scenarios:

## A.1 Scenarios

| Scenario | APIM Tier | Inbound Connectivity | เหมาะกับ |
|---|---|---|---|
| **A — Lean hub** | Standard v2 (~$700/เดือน, รวม 50M requests) | Public endpoint (ป้องกันด้วย Entra/key) + outbound VNet integration | องค์กรที่ยอมรับ public inbound ได้ |
| **B — Enterprise private hub** | Premium / Premium v2 (~$2,795–2,801/เดือน) | Full VNet injection / inbound Private Link — no public IP | องค์กรที่ต้องการ network segmentation เต็มรูปแบบ (แนวทางที่คาดว่าจะใช้จริง) |

## A.2 Monthly Fixed Cost Breakdown (USD, list price)

| Component | Lean (A) | Enterprise (B) | หมายเหตุ |
|---|---:|---:|---|
| APIM (Unified AI Gateway) | $700 | $2,800 | 1 unit; Premium 99.99% SLA ต้องการ ≥1 unit ใน ≥2 availability zones — ถ้าต้อง 2 units จะเป็น ~$5,600 |
| Logic App Standard (usage ingestion) | $175 | $175 | WS1 plan ~$143–185/เดือน |
| Cosmos DB (usage analytics store) | $50 | $75 | Autoscale เริ่มบิลที่ 400 RU/s หรือ 10% ของ max; สมมติ max 1,000–4,000 RU/s |
| Event Hub Standard (1–2 TU) | $22 | $44 | ~$0.03/TU/ชั่วโมง ≈ $22/TU/เดือน |
| App Insights + Log Analytics | $40 | $60 | ~10–25 GB/เดือน gateway telemetry ที่ ~$2.30/GB |
| Private endpoints + storage (~5–6) | $40 | $45 | ~$7.30/endpoint/เดือน + Logic App storage |
| **รวม fixed / เดือน** | **~$1,030** | **~$3,200** | |
| **รวม fixed / ปี** | **~$12,400** | **~$38,400** | |

## A.3 Phase 1 Incremental Cost (เปรียบเทียบ)

Phase 1 มี incremental platform cost เพียง **~$70–100/เดือน** (private endpoints 4–6 จุด + Log Analytics ingestion) — **AI token consumption จ่ายเท่ากันทั้งสอง architecture จึงตัดออกจากการเปรียบเทียบ**

## A.4 Caveats

1. ตัวเลขเป็น **US East list prices** — Southeast Asia สูงกว่าประมาณ 5–10%; ตรวจสอบด้วย Azure Pricing Calculator สำหรับ region จริงก่อน budget approval
2. **ไม่รวม**: optional Azure Managed Redis (semantic cache), APIM unit ที่ 2 สำหรับ 99.99% SLA (ดัน APIM เป็น ~$5,600/เดือน), และ multi-region deployment
3. **ไม่รวม operational cost**: Phase 2 ต้องมี platform team ดูแล gateway policies, contracts, usage pipeline — เป็นต้นทุนคนที่มีนัยสำคัญแต่วัดเป็นตัวเลขได้ยาก
4. ตัวเลขนี้ยังใช้เป็น **cost trigger สำหรับ Section 14**: เมื่อ AI spend ต่อเดือนเข้าใกล้ fixed cost ของ hub (~$3,100) การลงทุน centralized governance + chargeback เริ่มคุ้มค่า
5. ราคา ณ กันยายน 2026 — review ใหม่เมื่อถึงเวลาตัดสินใจ Phase 2

## A.5 Suggested Statement

> "By starting with the Lightweight Landing Zone, we avoid approximately **$3,100/month (~$37,000/year)** in fixed platform costs for a production-grade private Governance Hub (APIM Premium + usage pipeline) — or at minimum ~$1,000/month even in the leanest configuration — until AI adoption reaches the scale that justifies it. This also defines our Phase 2 cost trigger: when monthly AI spend approaches the hub's fixed cost, centralized governance and chargeback become worth the investment."

---

# Appendix B — Management Group Design for AI

Diagram: [`docs/diagrams/ai-management-groups.drawio`](diagrams/ai-management-groups.drawio) — extends the current tenant management group hierarchy (`docs/diagrams/current-management-group.png`, "Soft Isolation" model).

## B.1 Target Hierarchy

```text
Tenant Root
├── Platform                              (unchanged in Phase 1)
│   ├── Management
│   ├── Connectivity
│   ├── Identity
│   ├── Security
│   └── AI-Platform                       ★ PHASE 2 ONLY — created when triggers (§14) are met
│       └── 🔑 AI-Governance-Hub           (Citadel dedicated spoke: APIM, API Center,
│                                          usage pipeline — shared platform service)
├── NonFinancial
│   └── Subsidiary
│       ├── Production                    (existing workloads — untouched)
│       ├── NonProduction                 (existing workloads — untouched)
│       └── AI                            ★ NEW — AI archetype MG
│           ├── Production
│           │   └── 🔑 AI-Production       (one RG + spoke VNet per project)
│           └── NonProduction
│               └── 🔑 AI-NonProduction    (dev/staging per project)
├── Sandbox
│   ├── 🔑 Subsidiary-A / Subsidiary-B     (existing)
│   └── 🔑 AI-Sandbox                      ★ OPTIONAL — experimentation, audit-mode policies
└── Decommissioned                        (unchanged)
```

## B.2 Design Rationale

1. **`AI` เป็น workload archetype MG แยกจาก `Production`/`NonProduction` เดิม** — เพราะ `ai-lz-guardrails` initiative (deny public network access, disable local auth, mandatory tags, allowed locations — ดู `governance/`) เข้มงวดกว่า policy ที่ workloads เดิมใช้อยู่ การ assign ที่ scope `AI` MG ทำให้ inherit ลงทุก resource ได้สะอาด โดยไม่ต้องแก้หรือขอ exemption ให้ workloads เดิม
2. **`AI` MG scope ใช้เป็น boundary เดียวสำหรับ**: RBAC delegation ให้ AI program team, การเปิด Defender for AI plans เฉพาะจุดที่ต้องการ, และ Cost Management view รวมของ AI spend ทั้งหมด — ซึ่งป้อนตรงเข้า cost trigger ใน Appendix A
3. **`AI-Platform` (Phase 2) อยู่ใต้ `Platform` ไม่ใช่ใต้ `AI`** — เพราะ Citadel Governance Hub เป็น shared platform service ที่ให้บริการหลาย BU เหมือน Connectivity และ Identity หลักการ ALZ คือ platform team ดูแล platform MG, workload team ดูแล workload MG จึงไม่สร้างอะไรใต้ `AI-Platform` จนกว่า trigger ใน Section 14 จะถึง
4. **Subscription strategy**: Phase 1 ใช้ 2 subscriptions (`AI-Production`, `AI-NonProduction`) โดยหนึ่ง resource group + spoke VNet ต่อหนึ่ง project (Design Rule 3, Section 9) เมื่อ scale ขึ้นสู่ Phase 2 เปลี่ยนเป็น subscription-vending หนึ่ง subscription ต่อหนึ่ง project ใต้ `AI` MG ซึ่ง map 1:1 กับ Citadel Access Contract
5. **Multi-subsidiary**: ถ้า subsidiary อื่นเริ่มใช้ AI ในอนาคต pattern นี้ทำซ้ำได้ (`Subsidiary-N > AI`) โดย custom policy definitions ยังคง define ที่ tenant root ครั้งเดียว แล้ว assign initiative ที่ `AI` MG ของแต่ละ subsidiary

## B.3 Policy Assignment Map

| Scope | Assignment | Effect |
|---|---|---|
| Tenant Root (หรือ NonFinancial) | *Define* custom policy definitions + `ai-lz-guardrails` initiative ที่นี่ | — (definitions only, reusable) |
| `AI` MG | `ai-lz-guardrails` initiative + inherit-tags (×5 tags) | Deny (Production), Audit→Deny (NonProduction ระหว่าง brownfield rollout) |
| `AI-Sandbox` subscription | Same initiative | Audit only + budget cap |
| `AI-Platform` MG (Phase 2) | Guardrails variant สำหรับ hub (private APIM enforced) | Deny |

ดูขั้นตอน deploy ที่ `governance/README.md` — `management-group` scope ใน `az policy definition/set-definition/assignment create` ให้ชี้ไปที่ `AI` MG ตาม hierarchy นี้แทน placeholder เดิม
