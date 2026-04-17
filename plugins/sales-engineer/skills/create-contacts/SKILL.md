---
name: create-contacts
description: "Create demo contacts in Brevo with custom attributes and retrieve Brevo IDs. Activates on: create contacts, demo contacts, populate contacts."
tools: Bash, Read, Write
---

# Create Contacts

Create contacts, add to demo list, retrieve Brevo internal IDs.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `plan.contacts`, `created.attributes`, `meta.volumes.contacts` | **Write**: `created.contact_list`, `created.contacts` (with `brevo_id`)

## Mode A — Reuse existing contacts (update + new list)

When the user chooses to reuse existing contacts:

1. Fetch existing contacts from the account:
   ```bash
   curl -s "https://api.brevo.com/v3/contacts?limit=100&offset=0" \
     -H "api-key: $(cat /tmp/.brevo_key)" | python3 -c "
   import json,sys
   data=json.load(sys.stdin)
   contacts=data.get('contacts',[])
   print(json.dumps([{'id':c['id'],'email':c['email'],'attributes':c.get('attributes',{})} for c in contacts]))"
   ```
2. Create a new demo list:
   ```bash
   curl -s -X POST "https://api.brevo.com/v3/contacts/lists" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
     -d '{"name": "Demo - {Prospect}", "folderId": 1}'
   ```
3. Re-import existing contacts into the new list **with all attributes filled/updated** via `POST /contacts/import` with `updateEnabled: true` — this updates their attributes AND adds them to the list in one call.
4. Retrieve Brevo IDs from the new list.

## Mode B — Create new contacts from scratch

1. Create demo list:
   ```bash
   curl -s -X POST "https://api.brevo.com/v3/contacts/lists" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
     -d '{"name": "Demo - {Prospect}", "folderId": 1}'
   ```
2. Batch import new contacts via `POST /contacts/import`:
   ```bash
   curl -s -X POST "https://api.brevo.com/v3/contacts/import" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
     -d '{"jsonBody": [...], "listIds": [12], "updateEnabled": true}'
   ```

## Workflow (both modes)

1. Read context: `plan.contacts`, `created.attributes`, `meta.volumes.contacts`, `meta.contact_mode`
2. Apply the selected mode (A or B) above
3. **Fill ALL attributes** for every contact (see rules below)
4. Retrieve all Brevo IDs: `GET /contacts/lists/{list_id}/contacts?limit=500`
5. Update context: each contact gets `brevo_id`, `email`, `segment`, `status`

### Segment distribution

VIP 15% | Active 40% | New 25% | At-risk 20%

### Attribute fill rules — MANDATORY

**Every contact must have every attribute filled.** Empty fields degrade the demo. This is non-negotiable.

- **Before importing**: fetch all existing attributes with `GET /v3/contacts/attributes` to get the full list of attribute names and types
- **Fill every attribute** in `existing_attributes` AND every newly created attribute — no field left empty
- Generate realistic, coherent values per contact (name, city, age, segment all consistent)
- SMS: include **country code** matching the demo market (`+33` France, `+34` Spain, `+49` Germany, `+1` US)
- WHATSAPP: same format as SMS
- Category attributes: use **numeric value** (not label string)
- Boolean attributes: assign `true`/`false` based on segment (VIP → IS_VIP: true, etc.)
- Date attributes: use realistic past dates (registration 6-24 months ago, last purchase 1-90 days ago) — format **MM/DD/YYYY** (e.g., `"06/15/2024"`). ISO 8601 (`"2024-06-15T10:00:00Z"`) is **not accepted** by this endpoint
- If a contact already has a value for an attribute (Mode A), keep it unless it's empty — then fill it

## Pre-call Validation

Before `POST /contacts/import`:

| Check | Rule |
|-------|------|
| **Batch level:** | |
| `jsonBody` | **Required** — array of contact objects (alternative: `fileBody` for CSV) |
| `listIds` or `newList` | **One required** — contacts must go into a list |
| `updateEnabled` | Set `true` for idempotent re-runs |
| Body size | Max **10 MB** (safe limit: 8 MB) |
| **Per contact (identification — one required):** | |
| `email` | String — valid email format (recommended for demo) |
| `sms` | String — phone number with country code |
| **Per contact (optional):** | |
| `attributes` | Object — attribute names UPPERCASE. Unknown attributes silently ignored |
| `SMS` attribute | Must include country code (`+33`, `+34`, etc.) matching demo market |
| `WHATSAPP` attribute | Same format: full international number with country code |

### Attribute types supported

| Type | Example |
|------|---------|
| String | `"FIRSTNAME": "Jean"` |
| Number | `"AGE": 35` |
| Float | `"SALARY": 45000.50` |
| Boolean | `"IS_VIP": true` |
| Date | `"REGISTRATION_DATE": "06/15/2024"` — format **MM/DD/YYYY** (not ISO 8601) |

### ID retrieval (`GET /contacts/lists/{id}/contacts`)

- `limit` max 500, use `offset` pagination if volume > 500

## Response

- `202` — async, returns `{"processId": int}`
- Unknown attributes are silently ignored

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Never use `False`/`True` in inline `-c` heredocs | Shell may lowercase them — define a helper function instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues |
| Use `python3 << 'PYEOF' ... PYEOF` only for simple scripts | Use a unique delimiter (`PYEOF` not `EOF`) to avoid conflicts |
| Check empty values with a helper function | `def is_empty(v): return v is None or (isinstance(v, str) and v.strip() == '')` |

```python
# Safe pattern for attribute completion analysis
def is_empty(v):
    return v is None or (isinstance(v, str) and v.strip() == '')

for key in all_keys:
    filled = sum(1 for c in contacts if not is_empty(c.get('attributes', {}).get(key)))
    pct = filled / total * 100
    line = "{:<35} {:>8} {:>8} {:>5.0f}%".format(key, filled, total - filled, pct)
    print(line)
```

## Reference

- Endpoint: `POST /v3/contacts/import`
- Wrapping field: `jsonBody` (not `contacts`)
- `listIds` and `updateEnabled` at root level (not per contact)
- Body limit: 10 MB (safe: 8 MB)
- Attribute names MUST be UPPERCASE
- SMS/WHATSAPP: always include country code prefix
- Fill ALL attributes for every contact — maximize demo value
- Async operation — returns `processId`
