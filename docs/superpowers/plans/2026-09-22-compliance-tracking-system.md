# Compliance Tracking System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans (inline — this plan needs a continuous logged-in browser session across n8n and Google Sheets, same rationale as the Enquiry & Response build).

**Goal:** Build and deploy a demoable n8n "Compliance Tracking System" reference workflow on the live `n8n.struct.solutions` instance, with a seeded Google Sheet property/certificate store + dashboard, and showcase-repo documentation, matching the Enquiry & Response System build's conventions.

**Architecture:** One n8n workflow, single daily Schedule Trigger, reading all property rows once and evaluating all three certificates (Gas Safety, EICR, EPC) per property in one Code node, whose output fans out to (a) a Google Sheets update that always refreshes status/last-alert fields, and (b) an alert-extraction Code node feeding Email + Slack for any certificate that just crossed a tighter 60/30/7-day threshold than previously recorded.

**Tech Stack:** n8n (scheduleTrigger, googleSheets, code, set, emailSend, slack), Google Sheets, plain HTML/CSS not needed this time (no intake form in this module), browser automation for live deployment + screenshots.

## Global Constraints

- Brand tokens (only needed if any custom view styling is built): bg `#08090A`/`#0F1010`, green `#2A9D2A`/`#3DBF3D`, text `#F0F2EF`, Syne headings, DM Sans body.
- All sample data uses obviously-fake placeholders — no real agency/property/landlord data.
- Repo layout matches `n8n-enquiry-response-system`: `workflow.json`, `README.md`, `workflow-diagram.svg`, `LICENSE`, `.gitignore`, `screenshots/`.
- Node type strings/versions: `n8n-nodes-base.scheduleTrigger@1.2`, `n8n-nodes-base.googleSheets@4.5`, `n8n-nodes-base.code@2`, `n8n-nodes-base.set@3.4`, `n8n-nodes-base.emailSend@2.1`, `n8n-nodes-base.slack@2.2`.
- Properties sheet columns, in order: `property_ref, address, landlord_name, landlord_email, gas_safety_expiry, gas_safety_status, gas_safety_last_alert_days, eicr_expiry, eicr_status, eicr_last_alert_days, epc_expiry, epc_status, epc_last_alert_days`.
- Threshold set: `[60, 30, 7]` days. Status values: `overdue | due_soon | valid`.

---

### Task 1: Repo scaffolding

**Files:** Already created: `.gitignore`, `LICENSE`, `docs/superpowers/specs/...`, `docs/superpowers/plans/...`, `screenshots/`.
- Create: `README.md` (stub)

- [ ] **Step 1:** Write README stub (title + "Built by Struct Solutions" byline + "in progress" note).
- [ ] **Step 2:** Commit: `git add README.md && git commit -m "Scaffold repo"`

---

### Task 2: Author the workflow — evaluation + status-update branch

**Files:** Create `workflow.json`

Node definitions:

```json
{
  "parameters": { "rule": { "interval": [{ "field": "days", "triggerAtHour": 8 }] } },
  "id": "schedule-check-compliance",
  "name": "Check Compliance",
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "position": [-1200, 0]
}
```

```json
{
  "parameters": {
    "operation": "read",
    "documentId": { "__rl": true, "mode": "list", "value": "" },
    "sheetName": { "__rl": true, "mode": "list", "value": "Properties" },
    "options": {}
  },
  "id": "sheets-read-properties",
  "name": "Read Properties",
  "type": "n8n-nodes-base.googleSheets",
  "typeVersion": 4.5,
  "position": [-960, 0]
}
```

```json
{
  "parameters": {
    "jsCode": "const today = new Date();\ntoday.setHours(0,0,0,0);\nconst CERTS = [\n  { key: 'gas_safety', label: 'Gas Safety' },\n  { key: 'eicr', label: 'EICR' },\n  { key: 'epc', label: 'EPC' }\n];\n\nfunction daysUntil(dateStr) {\n  const d = new Date(dateStr);\n  d.setHours(0,0,0,0);\n  return Math.round((d.getTime() - today.getTime()) / 86400000);\n}\n\nfunction applicableThreshold(daysLeft) {\n  if (daysLeft > 60) return null;\n  if (daysLeft <= 7) return 7;\n  if (daysLeft <= 30) return 30;\n  return 60;\n}\n\nconst out = [];\nfor (const item of $input.all()) {\n  const row = item.json;\n  const update = { row_number: row.row_number, property_ref: row.property_ref };\n  const alerts = [];\n\n  for (const cert of CERTS) {\n    const expiry = row[`${cert.key}_expiry`];\n    const daysLeft = daysUntil(expiry);\n    const status = daysLeft < 0 ? 'overdue' : (daysLeft <= 60 ? 'due_soon' : 'valid');\n    const threshold = applicableThreshold(daysLeft);\n    const lastAlertRaw = row[`${cert.key}_last_alert_days`];\n    const lastAlert = (lastAlertRaw === '' || lastAlertRaw === undefined || lastAlertRaw === null) ? null : Number(lastAlertRaw);\n\n    let newLastAlert = lastAlert;\n    if (threshold !== null && (lastAlert === null || threshold < lastAlert)) {\n      alerts.push({\n        property_ref: row.property_ref,\n        address: row.address,\n        landlord_name: row.landlord_name,\n        certificate: cert.label,\n        expiry_date: expiry,\n        days_left: daysLeft,\n        threshold,\n        status\n      });\n      newLastAlert = threshold;\n    }\n\n    update[`${cert.key}_status`] = status;\n    update[`${cert.key}_last_alert_days`] = newLastAlert === null ? '' : newLastAlert;\n  }\n\n  update._alerts = alerts;\n  out.push({ json: update });\n}\nreturn out;"
  },
  "id": "code-evaluate-compliance",
  "name": "Evaluate Compliance",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [-720, 0]
}
```

```json
{
  "parameters": {
    "operation": "update",
    "documentId": { "__rl": true, "mode": "list", "value": "" },
    "sheetName": { "__rl": true, "mode": "list", "value": "Properties" },
    "columns": {
      "mappingMode": "defineBelow",
      "matchingColumns": ["row_number"],
      "value": {
        "row_number": "={{ $json.row_number }}",
        "gas_safety_status": "={{ $json.gas_safety_status }}",
        "gas_safety_last_alert_days": "={{ $json.gas_safety_last_alert_days }}",
        "eicr_status": "={{ $json.eicr_status }}",
        "eicr_last_alert_days": "={{ $json.eicr_last_alert_days }}",
        "epc_status": "={{ $json.epc_status }}",
        "epc_last_alert_days": "={{ $json.epc_last_alert_days }}"
      }
    },
    "options": {}
  },
  "id": "sheets-update-statuses",
  "name": "Update Statuses",
  "type": "n8n-nodes-base.googleSheets",
  "typeVersion": 4.5,
  "position": [-480, -120]
}
```

- [ ] **Step 1:** Write `workflow.json` with `name: "Compliance Tracking System"`, the four nodes above, `connections` wiring `Check Compliance → Read Properties → Evaluate Compliance`, and `Evaluate Compliance → Update Statuses` (main output 0, one of two targets — the second target is added in Task 3). Add `pinData: {}`, `settings: { "executionOrder": "v1" }`.
- [ ] **Step 2:** Validate: `python3 -c "import json; json.load(open('workflow.json', encoding='utf-8')); print('valid')"` → expect `valid`.
- [ ] **Step 3:** Commit: `git add workflow.json && git commit -m "Add compliance evaluation and status-update branch"`

---

### Task 3: Add the alert-extraction and notification branch

**Files:** Modify `workflow.json`

```json
{
  "parameters": {
    "jsCode": "const out = [];\nfor (const item of $input.all()) {\n  for (const alert of item.json._alerts || []) {\n    out.push({ json: alert });\n  }\n}\nreturn out;"
  },
  "id": "code-extract-alerts",
  "name": "Extract Alerts",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [-480, 120]
}
```

```json
{
  "parameters": {
    "options": {},
    "assignments": {
      "assignments": [
        { "id": "compliance-owner-email", "name": "compliance_owner_email", "type": "string", "value": "compliance-demo@example.com" },
        { "id": "compliance-slack-channel", "name": "slack_channel", "type": "string", "value": "#compliance-demo" }
      ]
    }
  },
  "id": "set-alert-config",
  "name": "Alert Config",
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "position": [-240, 120]
}
```

```json
{
  "parameters": {
    "fromEmail": "notifications@struct.solutions",
    "toEmail": "={{ $json.compliance_owner_email }}",
    "subject": "=Compliance alert: {{ $json.certificate }} for {{ $json.property_ref }} ({{ $json.days_left < 0 ? 'OVERDUE' : $json.days_left + ' days left' }})",
    "emailFormat": "text",
    "text": "=Certificate: {{ $json.certificate }}\nProperty: {{ $json.property_ref }} — {{ $json.address }}\nLandlord: {{ $json.landlord_name }}\nExpiry date: {{ $json.expiry_date }}\nDays left: {{ $json.days_left }}\nStatus: {{ $json.status }}",
    "options": {}
  },
  "id": "email-notify-compliance-owner",
  "name": "Notify Compliance Owner (Email)",
  "type": "n8n-nodes-base.emailSend",
  "typeVersion": 2.1,
  "position": [0, 40]
}
```

```json
{
  "parameters": {
    "select": "channel",
    "channelId": { "__rl": true, "mode": "name", "value": "={{ $json.slack_channel }}" },
    "text": "=:warning: *Compliance alert* — {{ $json.certificate }} for `{{ $json.property_ref }}`\n{{ $json.address }} (landlord: {{ $json.landlord_name }})\nExpiry: {{ $json.expiry_date }} ({{ $json.days_left }} days) — status: *{{ $json.status }}*",
    "otherOptions": {}
  },
  "id": "slack-notify-compliance-owner",
  "name": "Notify Compliance Owner (Slack)",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [0, 200]
}
```

- [ ] **Step 1:** Add the four nodes above. Update `Evaluate Compliance`'s connections so main output 0 goes to BOTH `Update Statuses` and `Extract Alerts`. Wire `Extract Alerts → Alert Config → Notify Compliance Owner (Email)` and `Alert Config → Notify Compliance Owner (Slack)` (fan-out, same pattern as `Config Values` in the Enquiry & Response workflow).
- [ ] **Step 2:** Validate node count: expect `8 nodes`.
- [ ] **Step 3:** Commit: `git add workflow.json && git commit -m "Add alert extraction and email/Slack notification branch"`

---

### Task 4: Create and seed the Google Sheet

- [ ] **Step 1:** Create spreadsheet "Struct Solutions — Compliance Tracking Demo", tab `Properties`, header row matching the Global Constraints column list.
- [ ] **Step 2:** Seed 8 fake properties (today = 2026-09-22) with pre-computed `status` fields and blank `last_alert_days`:
  - DEMO-P01, 12 Example Street: gas overdue (2026-08-15, status=overdue), eicr valid (2027-03-01), epc valid (2028-01-10)
  - DEMO-P02, 4 Fictional Road: eicr due in 5 days (2026-09-27, status=due_soon), gas valid (2027-05-01), epc valid (2027-11-01)
  - DEMO-P03, 88 Sample Avenue: epc due in 45 days (2026-11-06, status=due_soon), gas valid (2027-02-15), eicr valid (2027-01-15)
  - DEMO-P04, 21 Mockingbird Lane: gas overdue (2026-09-01, status=overdue), eicr due in 18 days (2026-10-10, status=due_soon), epc valid (2028-06-01)
  - DEMO-P05, 7 Testville Close: all valid — gas (2027-04-01), eicr (2027-08-01), epc (2027-09-01)
  - DEMO-P06, 3 Placeholder Gardens: epc due in 3 days (2026-09-25, status=due_soon), gas valid (2027-01-01), eicr valid (2027-06-01)
  - DEMO-P07, 56 Dummy Terrace: all valid — gas (2028-01-01), eicr (2027-12-01), epc (2027-10-01)
  - DEMO-P08, 19 Sample Close: all valid — gas (2027-07-01), eicr (2027-05-01), epc (2027-06-01)
  - Landlord name/email per row: fake, e.g. "Jane Smith" / `jane.smith@example.com`, varied.
- [ ] **Step 3:** Add tab `Dashboard` with formulas (columns E/H/K = expiry dates, F/I/L = status, per Global Constraints ordering):
```
A1: Certificates due within 30 days
B1: =COUNTIFS(Properties!E2:E,">="&TODAY(),Properties!E2:E,"<="&(TODAY()+30))+COUNTIFS(Properties!H2:H,">="&TODAY(),Properties!H2:H,"<="&(TODAY()+30))+COUNTIFS(Properties!K2:K,">="&TODAY(),Properties!K2:K,"<="&(TODAY()+30))

A2: Certificates due within 60 days
B2: =COUNTIFS(Properties!E2:E,">="&TODAY(),Properties!E2:E,"<="&(TODAY()+60))+COUNTIFS(Properties!H2:H,">="&TODAY(),Properties!H2:H,"<="&(TODAY()+60))+COUNTIFS(Properties!K2:K,">="&TODAY(),Properties!K2:K,"<="&(TODAY()+60))

A3: Certificates overdue
B3: =COUNTIF(Properties!F2:F,"overdue")+COUNTIF(Properties!I2:I,"overdue")+COUNTIF(Properties!L2:L,"overdue")
```
- [ ] **Step 4:** Verify Dashboard shows non-zero values for all three (visual check).
- [ ] **Step 5:** Record the Sheet ID for Task 5.

---

### Task 5: Deploy to n8n.struct.solutions

- [ ] **Step 1:** Fill in `documentId.value` in both Google Sheets nodes in `workflow.json` with the Sheet ID from Task 4.
- [ ] **Step 2:** Validate JSON, commit: `git add workflow.json && git commit -m "Wire Google Sheet document ID for live deployment"`
- [ ] **Step 3:** Import `workflow.json` into n8n.struct.solutions as a new workflow (reuse existing browser session if still logged in).
- [ ] **Step 4:** Leave credentials unconfigured (documented dependency, same as Enquiry & Response build), save, leave deactivated.

---

### Task 6: Screenshots, diagram, README

**Files:** `screenshots/workflow-canvas.jpg`, `screenshots/properties-sheet.jpg`, `screenshots/dashboard.jpg`, `workflow-diagram.svg`, `README.md`

- [ ] **Step 1:** Capture three screenshots: n8n canvas (both branches), `Properties` tab, `Dashboard` tab.
- [ ] **Step 2:** Draw `workflow-diagram.svg` (same visual style as the Enquiry & Response diagram): `Schedule (daily) → Read Properties → Evaluate Compliance` branching to `Update Statuses` and `Extract Alerts → Config → Email + Slack`.
- [ ] **Step 3:** Write full README: pitch, Renters' Rights Act / Making Tax Digital "why now" framing, how-it-works walkthrough, what it connects to, setup, acceptance test, what's configured live vs. not, what a real agency needs to provide to go live.
- [ ] **Step 4:** Commit: `git add screenshots/ workflow-diagram.svg README.md && git commit -m "Add screenshots, diagram, and full README"`

---

### Task 7: Publish to GitHub

- [ ] **Step 1:** `gh repo create n8n-compliance-tracking-system --public --source=. --description "..." --push`
- [ ] **Step 2:** Add topics matching the Enquiry & Response repo's convention plus compliance-specific ones.

## Self-Review Notes

- **Spec coverage:** property store with all required fields ✓ (Task 4), daily schedule ✓ (Task 2), 60/30/7 thresholds firing once each ✓ (Task 2 Evaluate Compliance logic), overdue persistently flagged ✓ (status recomputed every run, Task 2), email+Slack alerts ✓ (Task 3), dashboard with 30/60/overdue counts ✓ (Task 4), 8 seed properties spanning the required spread ✓ (Task 4 Step 2), README with go-live + "why now" ✓ (Task 6).
- **Out-of-scope confirmed absent:** no certificate-provider API calls, no landlord reporting/maintenance triage logic, no enquiry intake in this workflow.
- **DRY check:** status computation and alert-threshold logic live in exactly one place (`Evaluate Compliance`), consumed by both downstream branches — no duplicated threshold logic.
