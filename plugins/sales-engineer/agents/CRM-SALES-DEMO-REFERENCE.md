# CRM Sales Demo — Reference

## Context File Schema

Path: `/tmp/crm-sales-demo-<prospect_slug>.json`

```json
{
  "meta": {
    "prospect_name": "",
    "prospect_slug": "",
    "prospect_website": "",
    "created_at": "",
    "updated_at": "",
    "current_phase": "init",
    "brevo_api_key_set": false,
    "volumes": {
      "contacts": 60,
      "companies": 20,
      "deals": 30,
      "tasks": 20,
      "notes": 30
    },
    "use_cases": []
  },
  "research": {
    "industry": "",
    "markets": [],
    "products_services": [],
    "company_size": "",
    "notes": ""
  },
  "plan": {
    "attributes": [],
    "contacts": [],
    "companies": [],
    "deals": [],
    "tasks": [],
    "notes": []
  },
  "created": {
    "attributes": [],
    "contact_list": { "name": "", "id": null },
    "contacts": [],
    "companies": [],
    "deals": [],
    "tasks": [],
    "notes": []
  },
  "errors": []
}
```

---

## Contact Object Schema

### Plan entry

```json
{
  "firstname": "Jean",
  "lastname": "Dupont",
  "email": "jean.dupont@acme.com",
  "job_title": "VP Sales",
  "company": "Acme Corp",
  "segment": "VIP"
}
```

### Created entry (after create-contacts skill)

```json
{
  "brevo_id": 123,
  "email": "jean.dupont@acme.com",
  "segment": "VIP",
  "status": "active"
}
```

> The `company` field is stored only in `plan.contacts`. Skills join on `email` to resolve the company association at runtime.

---

## Company Object Schema

### Plan entry

```json
{
  "name": "Acme Corp",
  "industry": "SaaS",
  "number_of_employees": "51-200",
  "revenue": 5000000,
  "country": "France",
  "owner": "Marie Dupont",
  "status": "active",
  "website": "https://acme.com",
  "phone": "+33123456789"
}
```

### Created entry (after API call)

```json
{
  "brevo_id": "abc123",
  "name": "Acme Corp",
  "industry": "SaaS",
  "owner": "Marie Dupont"
}
```

---

## Deal Object Schema

### Plan entry

```json
{
  "name": "Renouvellement Enterprise — Acme Corp",
  "company": "Acme Corp",
  "stage": "Negotiation",
  "amount": 45000,
  "close_date": "2026-06-30",
  "owner": "marie.dupont@prospect.com"
}
```

### Created entry (after API call)

```json
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
```

---

## CRM Attribute Internal Names

### Companies — `GET /v3/crm/attributes/companies`

| Internal Name | Label | Type |
|---|---|---|
| `name` | Company name | text |
| `industry` | Industry | select |
| `number_of_employees` | Employees | select |
| `phone_number` | Phone | phone |
| `website` | Website | text |
| `revenue` | Revenue | number |
| `owner` | Owner | user |

### Deals — `GET /v3/crm/attributes/deals`

| Internal Name | Label | Type | Notes |
|---|---|---|---|
| `deal_name` | Deal name | text | Display name |
| `deal_owner` | Owner | user | Email of the owner |
| `amount` | Amount | number | Numeric only |
| `close_date` | Close date | date | ISO 8601 with time: `YYYY-MM-DDTHH:MM:SS.000Z` |
| `pipeline` | Pipeline | text | Pipeline ID (from `GET /crm/pipeline/details/...`) |
| `deal_stage` | Stage | text | Stage ID within the pipeline |

> Always fetch live attributes before creating — the actual `internalName` values may differ per account.

---

## Task Object Schema

### Plan entry

```json
{
  "name": "Appel de suivi — Acme Corp",
  "type": "Call",
  "due_date": "2026-06-10",
  "done": false,
  "linked_to": "Acme Corp",
  "linked_deal": "Renouvellement Enterprise — Acme Corp",
  "owner": "marie.dupont@prospect.com",
  "notes": "Discuter du renouvellement du contrat Enterprise"
}
```

### Created entry (after API call)

```json
{
  "brevo_id": "task_aaa111",
  "name": "Appel de suivi — Acme Corp",
  "type": "Call",
  "type_id": "tt_001",
  "done": false,
  "date": "2026-06-10T10:00:00.000Z",
  "company_name": "Acme Corp",
  "company_brevo_id": "abc123def456",
  "deal_brevo_id": "deal_xyz789"
}
```

---

## Note Object Schema

### Plan entry

```json
{
  "content": "Appel de découverte effectué avec le DG. Client intéressé par l'offre Enterprise.",
  "linked_to": "Acme Corp",
  "linked_deal": "Renouvellement Enterprise — Acme Corp",
  "style": "call_summary"
}
```

### Created entry (after API call)

```json
{
  "brevo_id": "note_bbb222",
  "text_preview": "Appel de découverte effectué avec le DG...",
  "company_name": "Acme Corp",
  "company_brevo_id": "abc123def456",
  "deal_brevo_id": "deal_xyz789"
}
```

---

## Pipeline Stages (default Brevo)

| Stage Name | Position |
|---|---|
| Prospecting | 1 |
| Qualification | 2 |
| Proposal | 3 |
| Negotiation | 4 |
| Closed Won | 5 |
| Closed Lost | 5 |

> Fetch live pipeline stages with `GET /v3/crm/pipeline/details/{pipeline_id}` before creating deals.

---

## Dataset Proposal Template

```markdown
### Companies (20)

| Name | Industry | Size | Revenue | Country | Owner | Status |
|------|----------|------|---------|---------|-------|--------|
| Acme Corp | SaaS | 51-200 | 5M€ | France | Marie Dupont | active |
| ...  | ...      | ...  | ...     | ...     | ...   | ...    |

### Deals (30)

| Name | Company | Stage | Amount | Close Date | Owner |
|------|---------|-------|--------|------------|-------|
| Renouvellement Enterprise | Acme Corp | Negotiation | 45 000€ | 2026-06-30 | Marie Dupont |
| ...  | ...     | ...   | ...    | ...        | ...   |
```
