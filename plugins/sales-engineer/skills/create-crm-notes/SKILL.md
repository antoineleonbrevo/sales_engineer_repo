---
name: create-crm-notes
description: "Create CRM notes in Brevo CRM Sales, linked to companies and deals. Activates on: create notes, CRM notes, demo notes, notes CRM, activity log."
tools: Bash, Read, Write
---

# Create CRM Notes

Create notes in Brevo CRM Sales to simulate a realistic activity log, linked to companies and deals created in previous steps.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `plan.notes`, `created.companies` (brevo_id, name), `created.deals` (brevo_id, name), `meta.volumes.notes` | **Write**: `created.notes`

## Workflow

### Step 0 — API Key

Ask the user for the Brevo API key for this session. Write to `/tmp/.brevo_key` (always overwrite — never reuse a prior session's key). Validate:

```bash
curl -s "https://api.brevo.com/v3/account" -H "api-key: $(cat /tmp/.brevo_key)"
```

### Step 1 — Read context

Read `/tmp/crm-sales-demo-<prospect_slug>.json`:
- `plan.notes` — array of planned notes
- `created.companies` — array with `brevo_id` and `name`
- `created.deals` — array with `brevo_id` and `name`

Build lookup maps:
```python
company_map = {c["name"]: c["brevo_id"] for c in context["created"]["companies"]}
deal_map    = {d["name"]: d["brevo_id"]  for d in context["created"]["deals"]}
```

### Step 2 — Create notes

Create each note via `POST /v3/crm/notes`. Only `text` is required; always include at least one entity link for demo value.

```bash
curl -s -X POST "https://api.brevo.com/v3/crm/notes" \
  -H "api-key: $(cat /tmp/.brevo_key)" \
  -H "content-type: application/json" \
  -d '{
    "text": "<p>Appel de découverte effectué avec le DG.</p><p>Le client est <b>très intéressé</b> par l'\''offre Enterprise — budget confirmé pour Q3.</p><p>Prochain step : envoi de la proposition commerciale.</p>",
    "companyIds": ["abc123def456"],
    "dealIds": ["deal_xyz789"]
  }'
```

**Response 200:**
```json
{ "id": "note_bbb222" }
```

Capture the `id`. Log any non-200 response to `errors[]`.

#### Text formatting rules — HTML supported

The `text` field supports these HTML tags — use them to make notes visually rich in the CRM UI:

| Tag | Use for |
|-----|---------|
| `<p>...</p>` | Paragraphs — wrap every block of text |
| `<b>` or `<strong>` | Bold — highlight key facts (amounts, decisions, next steps) |
| `<i>` or `<em>` | Italic — client quotes or context |
| `<u>` | Underline — action items |
| `<br>` | Line break within a paragraph |
| `<a href="...">` | Links — calendar invites, documents, etc. |

**Minimum structure for every note:**
```html
<p>{Context sentence describing what happened.}</p>
<p>{Key finding or outcome with <b>bold highlight</b>.}</p>
<p>{Next step or action item.}</p>
```

Avoid plain text — structured HTML notes look far better in the CRM timeline.

#### Content generation rules — MANDATORY

- Generate content that is **coherent with the linked company and deal** (industry, stage, amount)
- Each note must read like a real sales activity log entry — not generic filler
- Vary note styles across the dataset:

| Style | Example opening |
|-------|----------------|
| Call summary | `<p>Appel de suivi effectué avec {contact} chez {company}.</p>` |
| Meeting notes | `<p>Réunion de démonstration produit — {n} participants présents.</p>` |
| Email follow-up | `<p>Email de relance envoyé suite au silence de 2 semaines.</p>` |
| Internal memo | `<p>Note interne : deal bloqué en attente de validation budget.</p>` |
| Contract update | `<p>Contrat reçu et relu — négociation sur la clause de résiliation.</p>` |
| Win note | `<p>✅ Deal <b>Closed Won</b> — contrat signé ce jour.</p>` |
| Loss note | `<p>Deal perdu — client a choisi un concurrent pour des raisons de prix.</p>` |

Match note style to the deal stage of the linked deal:
- Prospecting/Qualification → discovery calls, first contact notes
- Proposal/Negotiation → proposal review, objection handling
- Closed Won → contract notes, onboarding kick-off
- Closed Lost → loss analysis, competitive intel

#### Linking rules

Each note must link to at least one entity. Preferred:
- Link to both `companyIds` and `dealIds` when the note is deal-specific
- Link to `companyIds` only for account-level notes (not tied to a specific deal)
- Spread notes across all companies and deals — avoid clustering on one record

#### Distribution

Aim for ~1–2 notes per deal, ~1 note per company without a deal:
- 70% linked to both company + deal
- 20% linked to company only
- 10% linked to deal only

### Step 3 — Update context

Write results to `/tmp/crm-sales-demo-<prospect_slug>.json`:

```json
{
  "created": {
    "notes": [
      {
        "brevo_id": "note_bbb222",
        "text_preview": "Appel de découverte effectué avec le DG...",
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
  "step": "create-crm-notes",
  "text_preview": "Appel de découverte...",
  "message": "API error: 400 — invalid companyId",
  "timestamp": "2026-05-15T10:00:00Z"
}
```

---

## Pre-call Validation

| Check | Rule |
|-------|------|
| `text` | Required — must not be empty; wrap content in `<p>` tags |
| HTML escaping | Escape single quotes in JSON strings: `\'` → use `"` inside HTML or escape properly |
| `companyIds` | Array of strings (Brevo company IDs) |
| `dealIds` | Array of strings (Brevo deal IDs) |
| `contactIds` | Array of integers (marketing contact IDs, optional) |
| At least one link | Every note should link to at least one entity for demo value |

## Response Codes

| Code | Meaning |
|------|---------|
| `200` | Note created — body contains `{"id": "..."}` |
| `400` | Validation error — check `text` is non-empty, entity IDs are valid |
| `403` | Insufficient permissions — verify API key has CRM Sales write access |
| `404` | Linked entity not found |
| `415` | Unsupported media type — ensure `content-type: application/json` header is set |

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues |
| Use `python3 << 'PYEOF' ... PYEOF` only for simple scripts | Use a unique delimiter (`PYEOF` not `EOF`) to avoid conflicts |
| Escape HTML single quotes in JSON | Use `"` inside HTML attributes or `&apos;` — never raw `'` inside a JSON string value |

## Reference

| Action | Method | Endpoint |
|--------|--------|----------|
| Create a note | POST | `/v3/crm/notes` |
| Update a note | PATCH | `/v3/crm/notes/{id}` |
| Get a note | GET | `/v3/crm/notes/{id}` |
| Delete a note | DELETE | `/v3/crm/notes/{id}` |
| List notes | GET | `/v3/crm/notes` |

### Key constraints

- No batch endpoint for notes — create one at a time via `POST /v3/crm/notes`
- Only `text` is required — all entity links are optional but mandatory for demo value
- `text` supports HTML: `<p>`, `<b>`, `<strong>`, `<i>`, `<em>`, `<u>`, `<br>`, `<a href="...">`
- No separate `link-unlink` endpoint — associations are set at creation or via `PATCH`
- Note creation returns `200` (not `201` unlike tasks and deals)
- Single quotes inside `text` HTML must be escaped when building JSON in bash
