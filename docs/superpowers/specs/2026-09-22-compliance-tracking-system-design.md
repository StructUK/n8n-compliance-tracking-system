# Compliance Tracking System — design

Reference/demo build for Struct Solutions' Agency Operations System offer, Module B. Not a live client deployment — seeded data throughout, built to be demoed on discovery calls alongside the Enquiry & Response System reference build.

## Scope

Every property under management has its Gas Safety, EICR and EPC renewal dates tracked automatically, with alerts firing at 60/30/7 days before expiry, and anything overdue visibly flagged. Driven by the Renters' Rights Act and Making Tax Digital pushing UK letting agencies to actively prove compliance.

## Decisions

- **Data store:** a separate Google Sheet ("Struct Solutions — Compliance Tracking Demo"), matching the Enquiry & Response build's column/formula conventions but kept as its own document.
- **Alert channels:** both email and Slack, same fan-out pattern as the Enquiry & Response build.
- **Schedule:** daily (more demoable — can show the workflow "catching" something).
- **Deploy:** imported live onto `n8n.struct.solutions` via browser automation, deactivated, credentials left as a documented go-live step (same as the Enquiry & Response build — that n8n account still has no Google Sheets/SMTP/Slack credentials configured).
- **Dedup:** each of the three thresholds (60/30/7 days) fires at most once per certificate, tracked via a `*_last_alert_days` field storing the tightest threshold already alerted.
- **Status persistence:** `overdue` / `due_soon` / `valid` status is recomputed every run regardless of alerting, so overdue certificates stay visibly flagged rather than alerted once and forgotten.

## Architecture

**1. Property record store** — Google Sheet, `Properties` tab:
`property_ref, address, landlord_name, landlord_email, gas_safety_expiry, gas_safety_status, gas_safety_last_alert_days, eicr_expiry, eicr_status, eicr_last_alert_days, epc_expiry, epc_status, epc_last_alert_days`

**2. n8n alert workflow**
- Schedule Trigger (daily) → Google Sheets read (`Properties`) → fans out to two branches:
  - **Status branch:** Code ("Compute Status Updates") → Google Sheets update — always runs, recomputes `status` and `last_alert_days` fields for all three certificates on every property.
  - **Alert branch:** Code ("Determine Alerts") — emits one item per certificate that just crossed a tighter threshold than previously recorded → Set ("Alert Config") → Send Email + Slack in parallel.

**3. Tracking view** — `Dashboard` tab, formula-only: certificates due within 30 days, due within 60 days (computed directly off expiry dates, so never stale), and currently overdue (off the status columns).

**4. Seed data** — 8 fake properties spanning: 2 overdue, 2 due within 7 days, at least 2 certificates due within 30–60 days, 2 properties fully valid (no cert due soon). Status fields pre-computed in the seed data itself, since no live n8n execution against real credentials will run this workflow during the build.

**5. Deliverables** — `workflow.json`, README (setup, acceptance test, "why now" framing around Renters' Rights Act / Making Tax Digital, what a real agency needs to provide to go live), 2–3 screenshots, seeded sheet.

## Out of scope

Real client/property data, integration with an actual certificate-issuing body, landlord reporting, maintenance triage, enquiry intake (Module A).

## Known dependency

Same as the Enquiry & Response build: Google Sheets OAuth (Client ID/Secret from a Google Cloud project) and real SMTP/Slack credentials are not configured on this n8n account. The workflow is imported and saved live, deactivated, with this flagged explicitly as a go-live step rather than worked around.
