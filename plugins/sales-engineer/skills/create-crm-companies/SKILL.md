---
name: create-crm-companies
description: "Create CRM companies in Brevo CRM Sales with attributes and optional contact links. Activates on: create companies, CRM companies, demo companies, entreprises CRM."
tools: Bash, Read, Write
---

# Create CRM Companies

Create companies in Brevo CRM Sales, fill all available attributes, and store Brevo IDs in context.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `plan.companies`, `plan.contacts`, `created.contacts` (brevo_id, email), `meta.volumes.companies`, `meta.prospect_name` | **Write**: `created.companies`

## Workflow

### Step 0 — API Key

Ask the user for the Brevo API key for this session. Write to `/tmp/.brevo_key` (always overwrite — never reuse a prior session's key). Validate:

```bash
curl -s "https://api.brevo.com/v3/account" -H "api-key: $(cat /tmp/.brevo_key)"
```

### Step 1 — Fetch available company attributes

Always fetch live attributes before creating companies. The attribute `internalName` is what Brevo accepts in the `attributes` object:

```bash
curl -s "https://api.brevo.com/v3/crm/attributes/companies" \
  -H "api-key: $(cat /tmp/.brevo_key)"
```

Parse the response to get the list of `internalName` + `attributeTypeName` for each attribute. Save to a local variable for use in Step 3.

### Step 2 — Read context

Read `/tmp/crm-sales-demo-<prospect_slug>.json`:
- `plan.companies` — array of planned companies with their attributes
- `plan.contacts` — array of planned contacts, each with a `company` field
- `created.contacts` — array of created contacts with `brevo_id` and `email`
- `meta.volumes.companies` — target count

Build the contacts-by-company lookup map (used in Step 4):

```python
plan_contact_company = {pc["email"]: pc["company"] for pc in context["plan"]["contacts"]}
contacts_by_company = {}
for c in context["created"]["contacts"]:
    company = plan_contact_company.get(c["email"])
    if company:
        contacts_by_company.setdefault(company, []).append(c["brevo_id"])
```

### Step 3 — Create companies

Create each company via `POST /v3/companies`. Map planned fields to the attribute `internalName` values retrieved in Step 1.

```bash
curl -s -X POST "https://api.brevo.com/v3/companies" \
  -H "api-key: $(cat /tmp/.brevo_key)" \
  -H "content-type: application/json" \
  -d '{
    "name": "Acme Corp",
    "attributes": {
      "industry": "SaaS",
      "number_of_employees": "51-200",
      "phone_number": "+33123456789",
      "website": "https://acme.com",
      "revenue": 5000000,
      "owner": "Marie Dupont"
    }
  }'
```

**Response 200:**
```json
{ "id": "abc123def456" }
```

Capture the `id` and store it alongside the company name for later steps (deal linking, task linking).

#### Attribute fill rules — MANDATORY

- **Fill every attribute** returned by Step 1 for every company. Empty fields degrade the demo.
- `industry` — use select option values exactly as returned by `attributeOptions` in the attributes list
- `number_of_employees` — use select option values exactly (e.g. `"1-10"`, `"11-50"`, `"51-200"`, `"201-500"`, `"500+"`)
- `phone_number` — always include country code (e.g. `+33`, `+34`, `+49`, `+1`)
- `revenue` — numeric (no currency symbol, no thousand separator)
- `owner` — full name of the sales rep (string)
- `website` — full URL with `https://`

#### Company size & owner distribution

Distribute companies across sizes:
- SMB (1–50 employees): ~40%
- Mid-market (51–500): ~40%
- Enterprise (500+): ~20%

Distribute across 3–4 fictional sales reps for variety.

### Step 4 — Link contacts to companies

For each created company, look up its contacts using the `contacts_by_company` map built in Step 2 and link them via the incremental endpoint. This step is **mandatory** when `created.contacts` is non-empty.

```bash
curl -s -X PATCH "https://api.brevo.com/v3/companies/link-unlink/{company_brevo_id}" \
  -H "api-key: $(cat /tmp/.brevo_key)" \
  -H "content-type: application/json" \
  -d '{
    "linkContactIds": [123, 456]
  }'
```

> Use `link-unlink` (not `PATCH /companies/{id}`) — it is incremental and never overwrites existing associations.

If a company has no matching contacts in the map (e.g. a company with no planned contacts), skip silently — do not fail.

### Step 5 — Update context

Write results to `/tmp/crm-sales-demo-<prospect_slug>.json`:

```json
{
  "created": {
    "companies": [
      {
        "brevo_id": "abc123def456",
        "name": "Acme Corp",
        "industry": "SaaS",
        "owner": "Marie Dupont",
        "size": "51-200"
      }
    ]
  }
}
```

Log any errors to `errors[]`:

```json
{
  "step": "create-crm-companies",
  "company_name": "Acme Corp",
  "message": "API error: 400 Bad Request",
  "timestamp": "2026-05-15T10:00:00Z"
}
```

---

## Pre-call Validation

| Check | Rule |
|-------|------|
| `name` | Required — must not be empty |
| `attributes` | Object — use `internalName` keys from Step 1 (not label strings) |
| Select attributes | Value must be an exact match of `attributeOptions[].value` |
| `phone_number` | Must include country code (`+33`, `+34`, etc.) |
| `revenue` | Must be a number (not a string, not formatted) |
| `linkedContactsIds` | Use integers — these are marketing contact IDs (from `created.contacts[].brevo_id`) |

## Response Codes

| Code | Meaning |
|------|---------|
| `200` | Company created — body contains `{"id": "..."}` |
| `400` | Validation error — check attribute names and value types |
| `403` | Insufficient permissions — verify API key has CRM Sales access |
| `404` | Endpoint not found — verify URL |

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues |
| Use `python3 << 'PYEOF' ... PYEOF` only for simple scripts | Use a unique delimiter (`PYEOF` not `EOF`) to avoid conflicts |

## Reference

| Action | Method | Endpoint |
|--------|--------|----------|
| Fetch attributes | GET | `/v3/crm/attributes/companies` |
| Create a company | POST | `/v3/companies` |
| Update a company | PATCH | `/v3/companies/{id}` |
| Link/unlink contacts & deals | PATCH | `/v3/companies/link-unlink/{id}` |
| Get a company | GET | `/v3/companies/{id}` |
| List companies | GET | `/v3/companies` |

### Key constraints

- `POST /v3/companies` creates one company at a time (no batch endpoint for companies)
- `PATCH /v3/companies/{id}` with `linkedContactsIds` **replaces** the full list — always use `link-unlink` for incremental association
- Attribute `internalName` values must be fetched live — do not hardcode them
- Select attribute values must match `attributeOptions` exactly (case-sensitive)
- All calls are synchronous — no `processId` polling needed
