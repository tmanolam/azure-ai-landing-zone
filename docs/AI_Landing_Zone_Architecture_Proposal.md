# AI Landing Zone Architecture Proposal

## 1. Executive Summary

### Recommendation

> **Adopt a Lightweight AI Landing Zone for the initial 1–2 AI projects, leveraging the existing Azure vWAN Hub and Azure Firewall, while establishing the security, networking, identity, and governance foundations required for future expansion to a centralized AI Governance Hub.**

### Why?

ปัจจุบันองค์กรมี:

- AI workloads จำนวน 1–2 projects ในระยะแรก
- Existing Azure enterprise network
- Azure vWAN Hub
- Centralized Azure Firewall
- ยังไม่มีความจำเป็นต้องมี centralized AI gateway สำหรับหลายทีม
- แต่ AI workloads เป็น production workloads จึงต้องมี security/governance baseline ตั้งแต่เริ่มต้น

ดังนั้นเราจะ **ไม่ deploy full AI Governance Hub ตั้งแต่วันแรก** เนื่องจาก infrastructure และ operational complexity ยังไม่สอดคล้องกับ scale ปัจจุบัน

แต่ architecture จะถูกออกแบบให้สามารถ evolve ไปสู่ **Central AI Governance Hub** ในอนาคต โดยไม่ต้อง redesign application architecture ใหม่

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

- AI application / Agent
- Azure AI / Microsoft Foundry / Azure OpenAI
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

ใช้ infrastructure ที่องค์กรมีอยู่แล้ว เช่น:

- Azure vWAN
- Azure Firewall
- Entra ID
- Azure Monitor

ไม่สร้าง platform ซ้ำซ้อน

### Principle 3 — Least Privilege

ใช้ Managed Identity + RBAC แทน static credentials

### Principle 4 — Private by Default

ใช้ Private Endpoint / Private DNS สำหรับ AI services และ data services ที่รองรับ

### Principle 5 — Centralize When Scale Requires It

ไม่สร้าง centralized AI gateway ก่อนที่จำนวน workload และ governance requirements จะ justify

### Principle 6 — Design for Evolution

Architecture วันนี้ต้องสามารถ evolve ไปสู่ centralized AI Governance Hub ได้

---

# 4. Phase 1 — Lightweight AI Landing Zone

## Target Architecture

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
                              VNet Peering
                                      │
                       ┌──────────────▼──────────────┐
                       │       AI Workload Spoke     │
                       │                             │
                       │  ┌───────────────────────┐  │
                       │  │ AI Application/Agent  │  │
                       │  └───────────┬───────────┘  │
                       │              │              │
                       │       Managed Identity      │
                       │              │              │
                       │  ┌───────────▼───────────┐  │
                       │  │   AI Services         │  │
                       │  │                       │  │
                       │  │ Foundry / Azure OpenAI│  │
                       │  │ AI Search             │  │
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
                       └─────────────────────────────┘
```

---

# 5. Phase 1 — Components

| Area | Component | Design |
|---|---|---|
| Network | Azure vWAN | Reuse existing enterprise network |
| Network Security | Azure Firewall | Central network inspection |
| Workload Isolation | AI Spoke VNet | Dedicated AI workload network |
| Connectivity | VNet ↔ vWAN Hub | Existing enterprise connectivity pattern |
| AI | Microsoft Foundry / Azure OpenAI | Private connectivity where supported |
| RAG | Azure AI Search | Private endpoint |
| Data | Storage / DB | Private endpoint |
| Secrets | Azure Key Vault | Private endpoint + Managed Identity |
| Identity | Microsoft Entra ID | Central enterprise identity |
| Access | Azure RBAC | Least privilege |
| Monitoring | Azure Monitor / Log Analytics | Centralized telemetry |
| Governance | Azure Policy | Prevent non-compliant deployment |
| Deployment | IaC | Bicep/Terraform + CI/CD |

---

# 6. Network Design

หนึ่งในเหตุผลหลักที่ไม่ต้องสร้าง Governance Hub ตอนนี้คือ **องค์กรมี network security foundation อยู่แล้ว**

### Network Flow

```text
AI Application
      │
      ▼
AI Spoke
      │
      ▼
Azure vWAN Hub
      │
      ▼
Azure Firewall
      │
      ▼
Enterprise / Internet / Azure Services
```

สำหรับ Azure AI services ที่รองรับ Private Link:

```text
AI Application
      │
      ▼
Private Endpoint
      │
      ▼
Azure AI Service
```

### Key Architectural Point

**Azure Firewall และ AI Governance Gateway มีหน้าที่ต่างกัน**

| Layer | Responsibility |
|---|---|
| Azure vWAN | Enterprise network connectivity |
| Azure Firewall | Network-level security / traffic inspection |
| APIM AI Gateway | API-level / AI-level governance |

ดังนั้นการมี Azure Firewall อยู่แล้ว **ไม่ได้ eliminate the future need for an AI Gateway** แต่ทำให้เราไม่จำเป็นต้อง deploy AI Gateway เพื่อแก้ปัญหา network security ที่มี solution อยู่แล้ว

---

# 7. Phase 1 — Security Baseline

## Identity

- Microsoft Entra ID
- Managed Identity
- Azure RBAC
- No embedded credentials

## Network

- Dedicated AI spoke
- Central Azure Firewall
- Private Endpoint
- Private DNS
- Restrict public network access where supported

## Secrets

- Azure Key Vault
- RBAC-based access
- Secret rotation

## Monitoring

- Diagnostic Settings
- Log Analytics
- Azure Monitor
- Activity Logs

## Governance

- Azure Policy
- Resource tagging
- Allowed regions
- Allowed SKUs
- Network restrictions
- Private endpoint requirements

---

# 8. What We Deliberately Do NOT Build in Phase 1

## Central AI Gateway / APIM

ยังไม่จำเป็นเพราะมีเพียง 1–2 AI projects

## Central AI Usage Platform

เช่น:

```text
Event Hub → Logic App → Cosmos DB
```

ยังไม่มี usage volume และ use-case diversity มากพอ

## AI Access Contracts

ยังสามารถจัดการผ่าน IaC + RBAC + project-level configuration ได้

## Central AI Asset Registry

ยังมี AI assets จำนวนไม่มาก

## Multi-region AI Gateway

ยังไม่มี requirement ที่ justify complexity

---

# 9. Why Lightweight Architecture?

| Criteria | Lightweight | Full AI Governance Hub |
|---|---:|---:|
| 1–2 projects | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Initial cost | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Operational simplicity | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Time to deploy | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| Central AI governance | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Multi-BU | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Multi-model | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Usage attribution | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Agent governance | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Future scalability | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### Conclusion

> **The Lightweight architecture provides sufficient governance for the current scale without introducing the operational overhead of a centralized AI platform prematurely.**

---

# 10. Phase 2 — Central AI Governance Hub

เมื่อ AI usage เติบโต เราค่อย introduce centralized AI Governance Hub

```text
                         Azure vWAN
                             │
                        Firewall
                             │
                 ┌───────────▼───────────┐
                 │ AI Governance Hub      │
                 │                        │
                 │ Azure APIM             │
                 │ AI Gateway             │
                 │ API Center             │
                 │ Governance Policies    │
                 │ Usage Analytics        │
                 └───────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
          AI Spoke A     AI Spoke B      AI Spoke C
          Project A      Project B       Project C
```

แนวทางนี้สอดคล้องกับ Citadel Governance Hub solution accelerator ของ Microsoft ซึ่งใช้ APIM เป็น centralized AI gateway และใช้ contract-driven governance สำหรับ backend และ access control

---

# 11. Phase 2 — What We Add

## 11.1 Azure API Management

ทำหน้าที่เป็น:

> **Central AI Gateway**

เพิ่มความสามารถ เช่น:

- Authentication
- Authorization
- Model access control
- Rate limiting
- Token quota
- API policy
- Routing
- Resiliency
- Central logging

## 11.2 AI Access Governance

จากเดิม:

```text
Project
  ↓
RBAC
  ↓
AI Service
```

เป็น:

```text
Business Unit
      ↓
Use Case
      ↓
Access Contract
      ↓
AI Gateway
      ↓
Approved AI Services
```

Access Contract สามารถกำหนด product, subscription, API attachment, policy และ credential management ต่อ use case

## 11.3 Backend Governance

จากเดิม:

```text
App → OpenAI
```

เป็น:

```text
App
 ↓
AI Gateway
 ↓
Approved Model Pool
 ├── Model A
 ├── Model B
 └── Model C
```

Platform Team สามารถควบคุมว่า model ใดเป็น approved model

## 11.4 Central Usage & Cost Analytics

เพิ่ม:

```text
APIM
 │
 ▼
Event Hub
 │
 ▼
Processing
 │
 ▼
Cosmos DB / Analytics
```

เพื่อให้สามารถตอบคำถามได้ว่า:

- Who is using AI?
- Which model?
- How much?
- Which business unit?
- Which use case?
- How many tokens?
- What is the cost?

---

# 12. Phase 2 — Governance Maturity

```text
                    AI Governance Maturity

Phase 1                         Phase 2
────────                        ────────
Project Governance              Enterprise Governance

RBAC                            Access Contracts
Azure Policy                    APIM Policies
Project Monitoring              Central AI Observability
Key Vault                       Central Credential Governance
Firewall                        Firewall + AI Gateway
IaC                             Contract-driven IaC
                                   │
                                   ▼
                         Multi-BU / Multi-Model
                         Agent Governance
                         Cost Attribution
                         Resiliency
```

---

# 13. Triggers for Moving to Phase 2

ไม่ควรกำหนดเพียงว่า “หลัง 6 เดือนให้ทำ Phase 2”

ควรใช้ **business / technical triggers**

## Scale

- AI projects เพิ่มขึ้นอย่างมีนัยสำคัญ
- มีหลาย Business Units
- มีหลาย environments

## Governance

- ต้องการ centralized model authorization
- ต้องการ central quota / rate limiting
- ต้องการ consistent AI safety policies

## Cost

- ต้องการ chargeback / showback
- AI cost เริ่มมีนัยสำคัญ

## AI Ecosystem

- มีหลาย LLM providers
- มี MCP tools
- มี A2A agents
- ต้องการ model routing / failover

## Compliance

- ต้องการ centralized audit
- ต้องการ policy enforcement ที่ application ไม่สามารถ bypass ได้

---

# 14. Evolution Path

```text
                 TODAY
                   │
                   ▼
        ┌──────────────────────┐
        │ Lightweight AI LZ    │
        │                      │
        │ Existing vWAN        │
        │ Existing Firewall    │
        │ AI Spoke             │
        │ Private AI Services  │
        │ Entra / KV / Monitor │
        └──────────┬───────────┘
                   │
                   │ Scale / Governance Trigger
                   ▼
        ┌──────────────────────┐
        │ AI Governance Hub    │
        │                      │
        │ APIM AI Gateway      │
        │ API Center           │
        │ Access Contracts     │
        │ Backend Contracts    │
        │ Central Analytics    │
        └──────────┬───────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │ Enterprise AI        │
        │ Platform             │
        │                      │
        │ Multi-BU             │
        │ Multi-Model          │
        │ Agent Governance     │
        │ Cost Governance      │
        │ DR / Resiliency      │
        └──────────────────────┘
```

---

# 15. Key Recommendation to Design Committee

> **Recommendation**
>
> We recommend adopting a **Lightweight AI Landing Zone** for the initial AI workloads.
>
> The solution will leverage the organization's existing **Azure vWAN and Azure Firewall** as the network and security foundation, while implementing mandatory AI security controls including private connectivity, identity-based access, secrets management, monitoring, and policy-based governance.
>
> A centralized **AI Governance Hub using Azure API Management** will be introduced as a subsequent phase when AI adoption reaches a level where centralized API governance, model access control, usage attribution, and multi-workload governance provide sufficient business value to justify the additional platform complexity.

---

# 16. One-Slide Architecture Decision

## AI Landing Zone — Progressive Architecture

### Current State

> 1–2 AI projects + Existing vWAN + Existing Firewall

↓

### Decision

> **Start Lightweight — Do not introduce a centralized AI Governance Hub yet**

↓

### Phase 1

```text
vWAN + Firewall
       │
   AI Spoke
       │
 ┌─────┼──────┐
 AI   Search  Data
       │
   Key Vault
       │
 Entra + Monitor
```

↓

### Future State

> Introduce centralized AI Governance Hub when scale/governance triggers are reached

```text
vWAN
 │
Firewall
 │
AI Governance Hub
 │
APIM AI Gateway
 │
 ├── AI Spoke A
 ├── AI Spoke B
 └── AI Spoke C
```

### Key Rationale

> **Avoid premature platform complexity while preserving a clear path to enterprise-scale AI governance.**
