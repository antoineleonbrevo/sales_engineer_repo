---
name: create-crm-tasks
description: "Create CRM tasks in Brevo CRM Sales, linked to companies and deals. Activates on: create tasks, CRM tasks, demo tasks, tâches CRM."
tools: Bash, Read, Write
---

# Create CRM Tasks

Create tasks in Brevo CRM Sales, linked to companies and deals created in previous steps.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `plan.tasks`, `plan.contacts`, `created.companies` (brevo_id, name), `created.deals` (brevo_id, name), `created.contacts` (brevo_id, email), `meta.volumes.tasks` | **Write**: `created.tasks`

## Workflow

### Step 0 — API Key

Ask the user for the Brevo API key for this session. Write to `/tmp/.brevo_key` (always overwrite — never reuse a prior session's key). Validate:

```bash
curl -s "https://api.brevo.com/v3/account" -H "api-key: $(cat /tmp/.brevo_key)"
```

### Step 1 — Fetch task types

Always fetch live task types before creating tasks. The `taskTypeId` is required and must come from this call — never hardcode it.

```bash
curl -s "https://api.brevo.com/v3/crm/tasktypes" \
  -H "api-key: $(cat /tmp/.brevo_key)"
```

**Response shape:**
```json
[
  { "id": "tt_001", "title": "Call" },
  { "id": "tt_002", "title": "Email" },
  { "id": "tt_003", "title": "Meeting" },
  { "id": "tt_004", "title": "Todo" },
  { "id": "tt_005", "title": "Lunch" },
  { "id": "tt_006", "title": "Deadline" },
  { "id": "tt_007", "title": "LinkedIn" }
]
```

If the account has no custom types, Brevo returns the default set automatically. Build a title→ID map for use in Step 3.

### Step 2 — Read context

Read `/tmp/crm-sales-demo-<prospect_slug>.json`:
- `plan.tasks` — array of planned tasks
- `plan.contacts` — array of planned contacts, each with a `company` field
- `created.companies` — array with `brevo_id` and `name`
- `created.deals` — array with `brevo_id` and `name`
- `created.contacts` — array with `brevo_id` and `email`

Build lookup maps:
```python
company_map = {c["name"]: c["brevo_id"] for c in context["created"]["companies"]}
deal_map    = {d["name"]: d["brevo_id"]  for d in context["created"]["deals"]}

# Map company name → list of contact brevo_ids
plan_contact_company = {pc["email"]: pc["company"] for pc in context["plan"]["contacts"]}
contacts_by_company = {}
for c in context["created"]["contacts"]:
    company = plan_contact_company.get(c["email"])
    if company:
        contacts_by_company.setdefault(company, []).append(c["brevo_id"])
```

### Step 3 — Create tasks

Create each task via `POST /v3/crm/tasks`. Required fields: `name`, `taskTypeId`, `date`.

```bash
curl -s -X POST "https://api.brevo.com/v3/crm/tasks" \
  -H "api-key: $(cat /tmp/.brevo_key)" \
  -H "content-type: application/json" \
  -d '{
    "name": "Appel de suivi — Acme Corp",
    "taskTypeId": "tt_001",
    "date": "2026-06-10T10:00:00.000Z",
    "duration": 1800000,
    "notes": "Discuter du renouvellement du contrat Enterprise",
    "done": false,
    "assignToId": "marie.dupont@prospect.com",
    "companiesIds": ["abc123def456"],
    "dealsIds": ["deal_xyz789"],
    "contactsIds": [123, 456]
  }'
```

Resolve `contactsIds` from `contacts_by_company[company_name]` for **Call** and **Meeting** type tasks only. For Email, Todo, LinkedIn — omit `contactsIds`. If no contacts are mapped to the company, omit the field.

**Response 201:**
```json
{ "id": "task_aaa111" }
```

Capture the `id`. Log any non-201 response to `errors[]`.

#### Field rules — MANDATORY

- `taskTypeId` — must be a live ID from Step 1 (never hardcode)
- `date` — ISO 8601 with time component: `YYYY-MM-DDTHH:MM:SS.000Z`
- `duration` — integer in **milliseconds**: 30 min = `1800000`, 1 h = `3600000`, 1.5 h = `5400000`
- `assignToId` — email of the sales rep assigned to the linked deal/company; use account email if unknown
- `done` — native JSON boolean (`true`/`false`); past-dated tasks should be `true`, future tasks `false`
- `companiesIds` — array of strings (Brevo company IDs)
- `dealsIds` — array of strings (Brevo deal IDs)
- `contactsIds` — array of integers (marketing contact IDs, optional)
- `notes` — free text; make it realistic and relevant to the deal or company context

#### Type distribution (default)

Spread tasks across types for a realistic activity log:

| Type | % of tasks |
|------|-----------|
| Call | 35% |
| Email | 25% |
| Meeting | 20% |
| Todo | 10% |
| LinkedIn | 5% |
| Other (Lunch, Deadline…) | 5% |

#### Done vs undone distribution

Mix past completed tasks with upcoming tasks:
- ~50% `done: true` with past dates (last 30 days)
- ~50% `done: false` with future dates (next 30 days)

This makes the CRM timeline look active rather than empty or all-pending.

#### Linking rules

Each task must link to at least one entity. Preferred patterns:

| Task type | Link to |
|-----------|---------|
| Call | deal + company |
| Email | deal or company |
| Meeting | deal + company |
| Todo | deal |
| LinkedIn | company |

Distribute tasks across companies and deals — avoid concentrating all tasks on one record.

#### Date generation

- Past tasks (`done: true`): spread over the last 30 days, business hours (09:00–18:00)
- Future tasks (`done: false`): spread over the next 30 days, business hours

### Step 4 — Update context

Write results to `/tmp/crm-sales-demo-<prospect_slug>.json`:

```json
{
  "created": {
    "tasks": [
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
    ]
  }
}
```

Log any errors to `errors[]`:

```json
{
  "step": "create-crm-tasks",
  "task_name": "Appel de suivi — Acme Corp",
  "message": "API error: 400 — taskTypeId not found",
  "timestamp": "2026-05-15T10:00:00Z"
}
```

---

## Pre-call Validation

| Check | Rule |
|-------|------|
| `name` | Required — must not be empty |
| `taskTypeId` | Required — must be a live ID from `GET /crm/tasktypes` — never hardcode |
| `date` | Required — ISO 8601 with time: `YYYY-MM-DDTHH:MM:SS.000Z` |
| `duration` | Integer in milliseconds — not seconds, not minutes |
| `done` | Native JSON boolean (`true`/`false`) — not a string |
| `companiesIds` | Array of strings |
| `dealsIds` | Array of strings |
| `contactsIds` | Array of integers |

## Response Codes

| Code | Meaning |
|------|---------|
| `201` | Task created — body contains `{"id": "..."}` |
| `400` | Validation error — check `taskTypeId`, `date` format, `duration` type |
| `403` | Insufficient permissions — verify API key has CRM Sales write access |
| `404` | Task type or linked entity not found |

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues |
| Use `python3 << 'PYEOF' ... PYEOF` only for simple scripts | Use a unique delimiter (`PYEOF` not `EOF`) to avoid conflicts |

## Reference

| Action | Method | Endpoint |
|--------|--------|----------|
| Fetch task types | GET | `/v3/crm/tasktypes` |
| Create a task | POST | `/v3/crm/tasks` |
| Update a task | PATCH | `/v3/crm/tasks/{id}` |
| Get a task | GET | `/v3/crm/tasks/{id}` |
| Delete a task | DELETE | `/v3/crm/tasks/{id}` |
| List tasks | GET | `/v3/crm/tasks` |

### Key constraints

- No batch endpoint for tasks — create one at a time via `POST /v3/crm/tasks`
- `taskTypeId` must be fetched live from `GET /crm/tasktypes` — never hardcode
- `duration` is in **milliseconds** (not seconds or minutes)
- No separate `link-unlink` endpoint for tasks — associations are set at creation via `companiesIds`, `dealsIds`, `contactsIds`
- Task creation returns `201` (not `200`)
- Past-dated tasks with `done: true` are valid and useful for demo history
