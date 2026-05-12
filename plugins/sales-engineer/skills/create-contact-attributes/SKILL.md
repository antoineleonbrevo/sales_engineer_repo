---
name: create-contact-attributes
description: "Create contact attributes in Brevo, avoiding duplicates and following existing naming conventions. Activates on: contact attributes, custom attributes, demo attributes."
tools: Bash, Read, Write
---

# Create Contact Attributes

Create new attributes, skip existing ones, respect account naming conventions. **Goal: ensure every attribute will be filled with data** for maximum demo impact.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `plan.attributes`, `existing_attributes` | **Write**: `created.attributes`

## Workflow

0. **API Key** — Ask the user for the Brevo API key for this session. Write it to `/tmp/.brevo_key` (always overwrite — never assume a prior session's key is still valid). Validate: `curl -s -H "api-key: $(cat /tmp/.brevo_key)" https://api.brevo.com/v3/account | python3 -c "import json,sys; d=json.load(sys.stdin); print('OK:', d.get('email','?'))"`
1. Read `demo-context.json` → planned + existing attributes
2. **Inventory ALL existing attributes** — the goal is to generate data for every single attribute (existing + new) so contacts have maximum field coverage during the demo
3. Compare planned vs existing:
   - Same name + type → skip, mark `reused`, add to fill list
   - Same name + different type → skip, reuse existing, warn, add to fill list
   - Similar name exists → reuse existing name, add to fill list
   - New → create, add to fill list
4. Detect naming convention (e.g., `FIRSTNAME` vs `FIRST_NAME`)
5. Create new attributes only:
   ```bash
   curl -s -X POST "https://api.brevo.com/v3/contacts/attributes/normal/{NAME}" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
     -d '{"type": "{text|date|float|boolean|category|multiple-choice}"}'
   ```
   For `category` (single-select with numeric values): add `"enumeration": [{"value": 1, "label": "..."}]`

   For `multiple-choice` (multi-select with string labels): add `"multiCategoryOptions": ["Option A", "Option B", "Option C"]`
   ```bash
   curl -s -X POST "https://api.brevo.com/v3/contacts/attributes/normal/{NAME}" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
     -d '{"type": "multiple-choice", "multiCategoryOptions": ["Newsletter", "Promotions", "Nouveautés"]}'
   ```
   When populating a `multiple-choice` attribute in contacts, send an **array of strings**: `"PREFERENCES": ["Newsletter", "Promotions"]`
6. Update context: store complete list of **all attributes to fill** (existing + created) with their types, so the contacts skill knows exactly what to populate

## Pre-call Validation

Before each `POST /contacts/attributes/normal/{NAME}`:

| Check | Rule |
|-------|------|
| `attributeCategory` | Must be `normal`, `category`, or `transactional` |
| `attributeName` | UPPERCASE only, no spaces, immutable once created |
| `type` | `normal` → `text`, `date`, `float`, `boolean`, `id`, `category`, or `multiple-choice`. `transactional` → `id` only. `category` → `category` only |
| `enumeration` | Required if type=`category`. Array of `{"value": int, "label": "..."}`. Each label max **200 chars** |
| `multiCategoryOptions` | Required if type=`multiple-choice`. Array of strings (e.g. `["Option A", "Option B"]`). Each label max **200 chars** |
| `value` | Only for `calculated` or `global` category attributes |
| Duplicate check | `GET /v3/contacts/attributes` first — POST on existing attribute returns error |

## Reference

- Names must be UPPERCASE, immutable once created
- Max 200 chars per category/multiple-choice label
- POST on existing attribute returns error — always check first
- `category` → single-select, values are integers, use `enumeration`
- `multiple-choice` → multi-select, values are strings, use `multiCategoryOptions`; contacts receive an array of strings
