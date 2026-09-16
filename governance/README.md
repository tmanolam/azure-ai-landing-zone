# AI Landing Zone — Phase 1 Governance Guardrails

Azure Policy definitions และ tagging standard สำหรับ Phase 1 ของ Lightweight AI Landing Zone (ดู `docs/AI_Landing_Zone_Architecture_Proposal_v2.md` Section 7 และ 9)

## Contents

| Path | Purpose |
|---|---|
| `TAGGING_STANDARD.md` | มาตรฐาน tags บังคับ/แนะนำ + rules + Phase 2 mapping |
| `policies/require-tags-on-resource-groups.json` | Deny RG ที่ไม่มี mandatory tags (Project, BusinessUnit, Environment, CostCenter, Owner) |
| `policies/inherit-tags-from-resource-group.json` | Modify: inherit tag จาก RG ลง resources (assign หนึ่งครั้งต่อ tag) |
| `policies/allowed-locations.json` | Deny resources นอก approved regions |
| `policies/deny-public-network-access-ai-services.json` | Deny Cognitive Services / Azure OpenAI ที่เปิด public network access |
| `policies/disable-local-auth-ai-services.json` | Deny Cognitive Services / Azure OpenAI ที่ไม่ได้ปิด API key auth |
| `policies/deny-public-network-access-search.json` | Deny AI Search ที่เปิด public network access |
| `policies/disable-local-auth-search.json` | Deny AI Search ที่ไม่ได้บังคับ Entra-only auth |
| `policies/deny-public-network-access-keyvault.json` | Deny Key Vault ที่เปิด public network access |
| `policies/deny-public-network-access-storage.json` | Deny Storage ที่เปิด public network access |
| `ai-lz-guardrails-initiative.json` | Initiative รวมทุก policy พร้อม parameters (`networkEffect`, `authEffect`, `allowedLocations`) |

## Deployment

สร้าง definitions ที่ management group (แนะนำ) หรือ subscription scope แล้ว assign initiative:

```bash
MG="my-ai-mg"   # management group id

# 1) create policy definitions (repeat per file; name must match ids referenced in the initiative)
az policy definition create --name ai-lz-require-tags-on-rg \
  --management-group "$MG" --rules "$(python3 -c "import json;print(json.dumps(json.load(open('policies/require-tags-on-resource-groups.json'))['properties']['policyRule']))")" \
  --params "$(python3 -c "import json;print(json.dumps(json.load(open('policies/require-tags-on-resource-groups.json'))['properties']['parameters']))")" \
  --display-name "AI-LZ: Require mandatory tags on resource groups" --mode All

# ... (ai-lz-allowed-locations, ai-lz-deny-public-ai-services, ai-lz-disable-local-auth-ai-services,
#      ai-lz-deny-public-search, ai-lz-disable-local-auth-search,
#      ai-lz-deny-public-keyvault, ai-lz-deny-public-storage — same pattern)

# 2) replace <policy-definition-scope> in ai-lz-guardrails-initiative.json with
#    /providers/Microsoft.Management/managementGroups/$MG  then create the initiative
az policy set-definition create --name ai-lz-guardrails \
  --management-group "$MG" \
  --definitions "$(python3 -c "import json;print(json.dumps(json.load(open('ai-lz-guardrails-initiative.json'))['properties']['policyDefinitions']))")" \
  --params "$(python3 -c "import json;print(json.dumps(json.load(open('ai-lz-guardrails-initiative.json'))['properties']['parameters']))")" \
  --display-name "AI-LZ: AI Landing Zone Guardrails (Phase 1)"

# 3) assign to the AI landing zone scope (start with Audit in brownfield, then flip to Deny)
az policy assignment create --name ai-lz-guardrails \
  --scope "/providers/Microsoft.Management/managementGroups/$MG" \
  --policy-set-definition ai-lz-guardrails \
  --params '{ "networkEffect": { "value": "Audit" }, "authEffect": { "value": "Audit" } }'

# 4) inherit-tags policy needs a managed identity (Tag Contributor) — assign once per mandatory tag
az policy assignment create --name ai-lz-inherit-tag-project \
  --scope "/providers/Microsoft.Management/managementGroups/$MG" \
  --policy ai-lz-inherit-tags --params '{ "tagName": { "value": "Project" } }' \
  --mi-system-assigned --location southeastasia
```

## Rollout guidance

1. **Audit ก่อน Deny** — assign ด้วย `networkEffect`/`authEffect` = `Audit` ใน subscription ที่มี resources อยู่แล้ว review compliance report แล้วค่อยเปลี่ยนเป็น `Deny`
2. Policy exemptions ต้องมี expiry และผ่าน design authority เท่านั้น
3. รัน remediation tasks สำหรับ `inherit-tags` หลัง assign เพื่อ backfill resources เดิม

## Note on built-in policies

Azure มี built-in policies ครอบคลุม controls เหล่านี้บางส่วน (เช่น "Azure AI Services resources should restrict network access", "Cognitive Services accounts should have local authentication methods disabled") — องค์กรอาจเลือกใช้ built-in แทน custom definitions เพื่อลด maintenance โดย lookup definition IDs ล่าสุดจาก Azure Portal / `az policy definition list` แล้วแทนที่ references ใน initiative ได้; custom definitions ในโฟลเดอร์นี้มีไว้เพื่อให้ self-contained, version-controlled และปรับ logic ได้เอง
