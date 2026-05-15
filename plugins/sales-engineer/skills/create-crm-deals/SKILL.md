---
name: create-crm-deals
description: "Create CRM deals in Brevo CRM Sales, assign to pipeline stages, and link to companies. Activates on: create deals, CRM deals, demo deals, pipeline deals, opportunités CRM."
tools: Bash, Read, Write
---

# Create CRM Deals

Create deals in Brevo CRM Sales, place them in the correct pipeline stage, and link to companies created in the previous step.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `plan.deals`, `created.companies` (brevo_id, name), `meta.volumes.deals` | **Write**: `created.deals`

## Workflow

### Step 0 — API Key

Ask the user for the Brevo API key for this session. Write to `/tmp/.brevo_key` (always overwrite — never reuse a prior session's key). Validate:

```bash
curl -s "https://api.brevo.com/v3/account" -H "api-key: $(cat /tmp/.brevo_key)"
```

### Step 1 — Fetch pipeline details

Brevo deals require a `pipeline` ID and a `deal_stage` ID — both are internal identifiers that must be fetched live. Never hardcode them.

**Step 1a — List pipelines** (to get the default pipeline ID):

```bash
curl -s "https://api.brevo.com/v3/crm/pipeline/details/all" \
  -H "api-key: $(cat /tmp/.brevo_key)"
```

If that endpoint returns 404, use the default pipeline ID `"default"` or try:

```bash
curl -s "https://api.brevo.com/v3/crm/pipeline/details/default" \
  -H "api-key: $(cat /tmp/.brevo_key)"
```

**Step 1b — Parse pipeline stages:**

From the response, extract the `id` and `name` of each stage. Map stage names to `deal_stage` IDs. Example response shape:

```json
{
  "pipeline": "pipeline_abc",
  "pipelineName": "Sales Pipeline",
  "stages": [
    { "id": "stage_001", "name": "Prospecting" },
    { "id": "stage_002", "name": "Qualification" },
    { "id": "stage_003", "name": "Proposal" },
    { "id": "stage_004", "name": "Negotiation" },
    { "id": "stage_005", "name": "Closed Won" },
    { "id": "stage_006", "name": "Closed Lost" }
  ]
}
```

Save `pipeline_id` and the stage name→ID mapping for Step 3.

### Step 2 — Fetch deal attributes

Always fetch live attributes to get the correct `internalName` values:

```bash
curl -s "https://api.brevo.com/v3/crm/attributes/deals" \
  -H "api-key: $(cat /tmp/.brevo_key)"
```

Parse each attribute's `internalName` and `attributeTypeName`. Common system attributes:

| Internal Name | Type | Notes |
|---|---|---|
| `deal_name` | text | Deal display name |
| `deal_owner` | user | Email of the owner |
| `amount` | number | Deal value |
| `close_date` | date | Expected close date |
| `pipeline` | text | Pipeline ID |
| `deal_stage` | text | Stage ID within the pipeline |

### Step 3 — Read context

Read `/tmp/crm-sales-demo-<prospect_slug>.json`:
- `plan.deals` — array of planned deals
- `created.companies` — array with `brevo_id` and `name` for each created company

Build a name→ID lookup map for companies:

```python
company_map = {c["name"]: c["brevo_id"] for c in context["created"]["companies"]}
```

### Step 4 — Create deals

Create each deal via `POST /v3/crm/deals`. Use pipeline and stage IDs from Step 1, and resolve company names to Brevo IDs from the map built in Step 3.

```bash
curl -s -X POST "https://api.brevo.com/v3/crm/deals" \
  -H "api-key: $(cat /tmp/.brevo_key)" \
  -H "content-type: application/json" \
  -d '{
    "name": "Renouvellement Enterprise — Acme Corp",
    "attributes": {
      "deal_owner": "marie.dupont@prospect.com",
      "pipeline": "pipeline_abc",
      "deal_stage": "stage_004",
      "amount": 45000,
      "close_date": "2026-06-30T00:00:00.000Z"
    },
    "linkedCompaniesIds": ["abc123def456"]
  }'
```

**Response 201:**
```json
{ "id": "deal_xyz789" }
```

Capture the `id`. Log any non-201 response to `errors[]`.

#### Attribute fill rules — MANDATORY

- **Fill every attribute** returned by Step 2 for every deal. Empty fields degrade the demo.
- `pipeline` — must be the pipeline `id` string from Step 1 (not the name)
- `deal_stage` — must be the stage `id` string from Step 1 (not the name)
- `deal_owner` — use a realistic email matching the sales rep assigned in `plan.deals`; if unknown, use the account email from `GET /v3/account`
- `amount` — numeric only (no currency symbol, no thousand separator)
- `close_date` — ISO 8601 with time component: `YYYY-MM-DDTHH:MM:SS.000Z`
- `linkedCompaniesIds` — array of Brevo company ID strings (from `created.companies`)

#### Stage distribution (default)

Spread deals across stages to show a realistic pipeline:

| Stage | % of deals |
|-------|-----------|
| Prospecting | 20% |
| Qualification | 25% |
| Proposal | 25% |
| Negotiation | 15% |
| Closed Won | 10% |
| Closed Lost | 5% |

Adjust distribution to match the prospect's use case (e.g. if showing "pipeline stuck" → more deals in Negotiation).

#### Deal amount guidelines

Match amounts to the company size from `created.companies`:
- SMB (1–50 employees): 1 000€ – 15 000€
- Mid-market (51–500): 15 000€ – 80 000€
- Enterprise (500+): 80 000€ – 500 000€

Each company should have 1–3 deals. Don't assign more than 5 deals to a single company.

### Step 5 — Link companies incrementally (if not linked at creation)

If a deal was created without `linkedCompaniesIds` (e.g. due to a missing ID), link afterwards using the incremental endpoint:

```bash
curl -s -X PATCH "https://api.brevo.com/v3/crm/deals/link-unlink/{deal_brevo_id}" \
  -H "api-key: $(cat /tmp/.brevo_key)" \
  -H "content-type: application/json" \
  -d '{
    "linkCompanyIds": ["abc123def456"]
  }'
```

> Always prefer `link-unlink` over `PATCH /crm/deals/{id}` for associations — it is incremental and won't overwrite existing links.

### Step 6 — Update context

Write results to `/tmp/crm-sales-demo-<prospect_slug>.json`:

```json
{
  "created": {
    "deals": [
      {
        "brevo_id": "deal_xyz789",
        "name": "Renouvellement Enterprise — Acme Corp",
        "stage": "Negotiation",
        "stage_id": "stage_004",
        "amount": 45000,
        "company_name": "Acme Corp",
        "company_brevo_id": "abc123def456",
        "owner": "marie.dupont@prospect.com"
      }
    ]
  }
}
```

Log any errors to `errors[]`:

```json
{
  "step": "create-crm-deals",
  "deal_name": "Renouvellement Enterprise — Acme Corp",
  "message": "API error: 400 — stage_id not found in pipeline",
  "timestamp": "2026-05-15T10:00:00Z"
}
```

---

## Pre-call Validation

| Check | Rule |
|-------|------|
| `name` | Required — must not be empty |
| `attributes.pipeline` | Must be a valid pipeline `id` fetched in Step 1 — never hardcode |
| `attributes.deal_stage` | Must be a valid stage `id` from that pipeline — never hardcode |
| `attributes.amount` | Number type — no string, no formatted currency |
| `attributes.close_date` | ISO 8601 with time: `YYYY-MM-DDTHH:MM:SS.000Z` |
| `linkedCompaniesIds` | Array of strings (company IDs from `created.companies`) |
| `linkedContactsIds` | Array of integers (marketing contact IDs, if applicable) |

## Response Codes

| Code | Meaning |
|------|---------|
| `201` | Deal created — body contains `{"id": "..."}` |
| `400` | Validation error — check attribute names, types, pipeline/stage IDs |
| `403` | Insufficient permissions — verify API key has CRM Sales write access |
| `404` | Pipeline or stage not found — re-fetch pipeline details |

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues |
| Use `python3 << 'PYEOF' ... PYEOF` only for simple scripts | Use a unique delimiter (`PYEOF` not `EOF`) to avoid conflicts |

## Reference

| Action | Method | Endpoint |
|--------|--------|----------|
| Fetch deal attributes | GET | `/v3/crm/attributes/deals` |
| Fetch pipeline details | GET | `/v3/crm/pipeline/details/{pipelineID}` |
| Create a deal | POST | `/v3/crm/deals` |
| Update a deal | PATCH | `/v3/crm/deals/{id}` |
| Link/unlink contacts & companies | PATCH | `/v3/crm/deals/link-unlink/{id}` |
| Get a deal | GET | `/v3/crm/deals/{id}` |
| List deals | GET | `/v3/crm/deals` |
| Delete a deal | DELETE | `/v3/crm/deals/{id}` |

### Key constraints

- No batch endpoint for deals — create one at a time via `POST /v3/crm/deals`
- `pipeline` and `deal_stage` in `attributes` require internal ID strings — fetch live, never hardcode
- `PATCH /crm/deals/{id}` with `linkedCompaniesIds` replaces the full list — always use `link-unlink` for incremental changes
- `close_date` must be ISO 8601 with time component (`T00:00:00.000Z` is valid)
- Deal creation returns `201` (not `200`)
