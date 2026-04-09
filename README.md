# NOAVIA Client Onboarding Automation

**Candidate:** Hamza Zahid Butt  
**Position:** Full-Stack Automation Engineer — Intern / Working Student  
**Submitted:** April 2026

---

![n8n](https://img.shields.io/badge/n8n-automation-FF6D5A?style=flat-square)
![Status](https://img.shields.io/badge/status-working-success?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)

---

## Proof of Working System

- Workflow executed successfully end-to-end
- Enterprise email delivered successfully
- Professional email delivered successfully
- Google Sheets populated
- ERP import completed: 22 imported, 3 skipped

---

## What This Does

A complete end-to-end n8n workflow that automates B2B client onboarding:

> Webhook → Validate → Generate Ref → Enrich (Hunter.io) → ERP CSV Import → Merge → Generate Welcome Doc → Google Sheets → Route by Tier → Email (Enterprise/Professional)

**Proven working:** Enterprise email delivered ✓ · Google Sheets populated ✓ · ERP import: 22 imported, 3 skipped ✓

---

## Business Impact

This workflow reduces manual onboarding effort by automating:

- client intake and validation
- company enrichment
- legacy ERP migration
- tier-based communication routing
- centralized record keeping

Estimated manual effort reduction: ~70–80% for onboarding operations.

---

## Architecture Decisions

**n8n Code nodes for all business logic** — validation, ERP parsing, and document generation are in JavaScript Code nodes rather than n8n's built-in transform nodes. This makes the logic transparent, testable, and easy to version-control. n8n's native nodes are used only for I/O (Webhook, HTTP Request, Google Sheets, Gmail).

**Parallel ERP + Enrichment paths merged before document generation** — the workflow splits after validation: one branch calls Hunter.io for enrichment, another reads and parses the ERP CSV file from disk. A Merge node combines both outputs before the document generation step. This is more efficient than running them sequentially and demonstrates understanding of n8n's parallel execution model.

**Google Sheets for storage** — chosen over Supabase for zero infrastructure overhead in a demo context. Non-technical stakeholders can view and edit the client register directly. Supabase migration is listed in Sprint 2.

---

## Enrichment Approach: Hunter.io

I used **Hunter.io's `/v2/companies/find` API** instead of Clearbit's free autocomplete endpoint.

**Why:** Clearbit's free tier returns only company name and logo — it does not provide industry, company size, or location, which are explicitly required fields. Hunter.io's company endpoint returns structured firmographic data including industry classification, company size estimate, and full location (city, state, country).

**Trade-off acknowledged:** Hunter.io's free tier is limited to 25 requests/month. The Normalize Enrichment node handles API failures gracefully — if the call fails or returns null fields, it sets safe defaults (`"Unknown"`) rather than crashing the workflow. Sprint 2 would add a fallback to Clearbit's paid Enrichment API.

---

## ERP CSV Import: Encoding, Duplicates, Edge Cases

**File:** `Sample_ERP_Export_DATEV.csv` — 25 records, ISO-8859-1 encoding, semicolon-delimited.

**Encoding:** n8n's Read/Write Files from Disk node reads the raw file. The CSV is read as binary data and parsed inside an n8n Code node using Latin-1 decoding and semicolon-based parsing. German Umlauts (ä, ö, ü, ß) in company names like `Müller`, `Özkan`, `Weiß`, `Nüßler` are preserved correctly.

**Field mapping:** 14 German column headers mapped to internal English schema:

| German          | Internal        |
| --------------- | --------------- |
| Kundennr        | customer_number |
| Firma           | company_name    |
| Ansprechpartner | contact_person  |
| Branche         | industry        |
| Kundenseit      | customer_since  |
| Umsatz_2025     | revenue_2025    |

**Duplicate detection** using two JavaScript Sets (`seenNumbers`, `seenCompanies`) checked before each insert:

| Row                 | Issue                     | Action                                                       |
| ------------------- | ------------------------- | ------------------------------------------------------------ |
| 16 (Kundennr 10001) | Duplicate customer number | Skipped — `duplicate Kundennr: 10001`                        |
| 17 (Kundennr 10016) | Same Firma as row 1       | Skipped — `duplicate company name: Müller Maschinenbau GmbH` |
| 18 (Kundennr 10017) | Firma field empty         | Skipped — `missing company name (Firma leer)`                |

**Result: 22 records imported, 3 skipped.** All skipped records are logged with row index, Kundennr, and human-readable reason.

---

## Sprint 2 — What I Would Add

1. **Supabase migration** — PostgreSQL schema with proper column types, constraints, and a SQL view aggregating clients by tier and revenue
2. **Google Drive integration** — save the generated HTML welcome document to `/Clients/{CompanyName}/` as specified
3. **Full Clearbit Enrichment API** — as a fallback when Hunter.io returns null industry/size
4. **Respond to Webhook node** — return structured JSON `{success, client_ref, message}` to the calling form
5. **Error handling workflow** — catch unhandled errors, send Slack/email alert to ops team
6. **Retry logic** — exponential backoff on Hunter.io HTTP node (currently fails hard on timeout)
7. **Webhook HMAC verification** — prevent spoofed form submissions
8. **Scheduled ERP import** — watch a Google Drive folder for new CSV drops instead of reading from disk

---

## How to Run

```bash
# 1. Install and start n8n
npm install n8n -g
n8n start
# Opens at http://localhost:5678

# 2. Import workflow
# Menu → Import from JSON → select noavia_onboarding_workflow.json

# 3. Configure credentials
# - Google Sheets OAuth2
# - Gmail OAuth2
# - Hunter.io API key (free at hunter.io)

# 4. Activate and test
curl -X POST http://localhost:5678/webhook/onboarding \
  -H "Content-Type: application/json" \
  -d '{
    "company_name": "Acme GmbH",
    "first_name": "Max",
    "last_name": "Mustermann",
    "email": "max@acme.de",
    "website": "https://acme.de",
    "service_tier": "Enterprise",
    "notes": "Priority onboarding"
  }'
```

---

## Configuration

### Required API Keys

1. **Hunter.io API Key**
   - Sign up at https://hunter.io
   - Free tier: 25 requests/month
   - Navigate to API → API Keys
   - Copy your API key

2. **Google Sheets OAuth2**
   - Go to Google Cloud Console
   - Create new project or select existing
   - Enable Google Sheets API
   - Create OAuth 2.0 credentials
   - Download credentials.json

3. **Gmail OAuth2**
   - Same Google Cloud project as above
   - Enable Gmail API
   - Use the same OAuth credentials or create separate ones

### n8n Credentials Configuration

1. Open n8n → Credentials → Add Credential
2. For Hunter.io:
   - Type: HTTP Request
   - Authentication: Header Auth
   - Name: `Authorization`
   - Value: `Bearer YOUR_API_KEY`
3. For Google Sheets & Gmail:
   - Type: Google OAuth2 API
   - Follow OAuth flow to authorize

### Environment Variables (Optional)

For production deployment, create `.env`:

```env
HUNTER_API_KEY=your_hunter_io_api_key
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
```

---

## Testing the Workflow

### Sample Test Payload

Use this payload to test the webhook:

```bash
curl -X POST http://localhost:5678/webhook/onboarding \
  -H "Content-Type: application/json" \
  -d '{
    "company_name": "Test Corporation GmbH",
    "first_name": "Max",
    "last_name": "Mustermann",
    "email": "max.mustermann@testcorp.de",
    "website": "https://testcorp.de",
    "service_tier": "Enterprise",
    "phone": "+49 123 456789",
    "notes": "Priority onboarding - VIP client"
  }'
```

### Expected Results

1. **Validation Success**
   - All fields pass regex validation
   - Client reference generated: `NOA-2026-XXXX`

2. **Hunter.io Enrichment**
   - Company data retrieved (industry, size, location)
   - If domain not found, defaults to "Unknown"

3. **ERP CSV Import**
   - 25 total rows → 22 imported, 3 skipped
   - Skipped records logged with reason

4. **Google Sheets**
   - New row added with all data
   - Status: "completed"

5. **Email Sent**
   - For Enterprise tier: Full details to manager@noavia.de
   - For Professional tier: Brief summary to team@noavia.de
   - For Basic tier: No email (DB only)

### Verifying Results

1. Check n8n execution log for success
2. Open Google Sheet to verify data entry
3. Check email inbox for notification
4. Open `dashboard.html` in browser to see live dashboard

---

## Files

| File                              | Description                                 |
| --------------------------------- | ------------------------------------------- |
| `NOAVIA - Client Onboarding.json` | n8n workflow — import directly              |
| `dashboard.html`                  | Client pipeline dashboard — open in browser |
| `README.md`                       | This document                               |
| `Sample_ERP_Export_DATEV.csv`     | Sample ERP data (25 records)                |
| `screenshots/`                    | Workflow execution proof                    |
