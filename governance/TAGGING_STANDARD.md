# AI Landing Zone — Tagging Standard

Tagging เป็น Design Rule ข้อ 4 ของ Phase 1 (Forward Compatibility) — tags ที่กำหนดวันนี้จะกลายเป็น dimensions ของ central cost attribution / chargeback ใน Phase 2 (Citadel Governance Hub) ดังนั้นต้อง enforce ตั้งแต่ resource แรก

## 1. Mandatory Tags

Tags ต่อไปนี้ **บังคับ** บนทุก resource group ใน AI Landing Zone (enforced ด้วย `require-tags-on-resource-groups` policy) และถูก inherit ลงทุก resource อัตโนมัติ (ด้วย `inherit-tags-from-resource-group` policy):

| Tag | ความหมาย | รูปแบบค่า | ตัวอย่าง |
|---|---|---|---|
| `Project` | ชื่อ AI project / use case | kebab-case, สั้น, คงที่ตลอดอายุ project | `claims-copilot` |
| `BusinessUnit` | หน่วยธุรกิจเจ้าของ workload | รหัส BU ตามทะเบียนองค์กร | `retail-banking` |
| `Environment` | สภาพแวดล้อม | ค่าใดค่าหนึ่ง: `prod` \| `staging` \| `dev` \| `sandbox` | `prod` |
| `CostCenter` | รหัสศูนย์ต้นทุนสำหรับ chargeback | รหัสตามระบบบัญชี | `CC-4512` |
| `Owner` | ผู้รับผิดชอบ (team alias หรือ email กลาง — ไม่ใช้ email ส่วนบุคคล) | distribution list | `ai-claims-team@company.com` |

## 2. Recommended Tags

แนะนำเพิ่มเติม (ไม่ deny แต่ควรใส่):

| Tag | ความหมาย | ตัวอย่าง |
|---|---|---|
| `DataClassification` | ระดับชั้นความลับของข้อมูลที่ workload ประมวลผล (สอดคล้อง Purview labels) | `confidential` |
| `Criticality` | ระดับความสำคัญต่อธุรกิจ | `mission-critical` \| `high` \| `medium` \| `low` |
| `ManagedBy` | เครื่องมือที่ deploy resource | `bicep` \| `terraform` |
| `Repo` | Git repository ของ IaC ที่ deploy resource | `scbx-azure-ai-landing-zone` |

## 3. Rules

1. **Tag ที่ resource group** — resources inherit อัตโนมัติผ่าน policy (`modify` effect + remediation task) ไม่ต้อง tag รายชิ้น ยกเว้นกรณี override ที่จำเป็น
2. **หนึ่ง resource group ต่อหนึ่ง project ต่อหนึ่ง environment** (สอดคล้อง Design Rule ข้อ 3) — เช่น `rg-claims-copilot-prod`
3. ค่า tag ต้องมาจาก **IaC เท่านั้น** (`.bicepparam` / `.tfvars`) — ห้ามแก้ผ่าน portal เพื่อให้ audit ได้จาก git history
4. Tag names ใช้ **PascalCase**, tag values ใช้ **lowercase/kebab-case** (ยกเว้นรหัสที่มีรูปแบบขององค์กรเอง เช่น `CostCenter`)
5. ห้ามใส่ข้อมูลลับหรือข้อมูลส่วนบุคคลในค่า tag (tags ไม่ถูก encrypt และปรากฏใน billing export)
6. การเพิ่ม/เปลี่ยน tag มาตรฐานต้องผ่าน design authority และอัปเดตเอกสารนี้ + policy parameter พร้อมกัน

## 4. Phase 2 Mapping

| Tag | บทบาทใน Phase 2 (Citadel) |
|---|---|
| `Project` | Map กับ Access Contract (one contract per use case) และ APIM product/subscription |
| `BusinessUnit` | Dimension หลักของ usage analytics / chargeback ใน Cosmos DB + Power BI |
| `Environment` | แยก contract ต่อ environment ตาม Citadel recommendation |
| `CostCenter` | Chargeback / showback report |

## 5. Cost Views (Phase 1)

ระหว่างที่ยังไม่มี central usage pipeline ให้สร้าง Azure Cost Management views โดย group by `Project` และ `BusinessUnit` tags และตั้ง budget + alert ต่อ resource group ของแต่ละ project — ใช้ร่วมกับ Azure OpenAI diagnostic logs (token usage) ใน Log Analytics เพื่อตอบคำถาม who/how much ระดับ project
