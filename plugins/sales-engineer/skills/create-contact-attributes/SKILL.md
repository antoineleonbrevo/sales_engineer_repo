---
name: create-contact-attributes
description: "Create contact attributes in Brevo, avoiding duplicates and following existing naming conventions. Activates on: contact attributes, custom attributes, demo attributes."
tools: Bash, Read, Write
---

# Create Contact Attributes

Create new attributes, skip existing ones, respect account naming conventions. **Goal: ensure every attribute will be filled with data** for maximum demo impact.

> **Language**: Communicate with the user in the same language they use — French or English.

## Demo Context

- **Read**: `plan.attributes`, `existing_attributes` | **Write**: `created.attributes`

## Workflow

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
     -d '{"type": "{text|date|float|boolean|category}"}'
   ```
   For category: add `"enumeration": [{"value": 1, "label": "..."}]`
6. Update context: store complete list of **all attributes to fill** (existing + created) with their types, so the contacts skill knows exactly what to populate

## Pre-call Validation

Before each `POST /contacts/attributes/normal/{NAME}`:

| Check | Rule |
|-------|------|
| `attributeCategory` | Must be `normal`, `category`, or `transactional` |
| `attributeName` | UPPERCASE only, no spaces, immutable once created |
| `type` | `normal` → `text`, `date`, `float`, `boolean`, `id` or `category`. `transactional` → `id` only. `category` → `category` only |
| `enumeration` | Required if type=`category`. Array of `{"value": int, "label": "..."}`. Each label max **200 chars** |
| `value` | Only for `calculated` or `global` category attributes |
| Duplicate check | `GET /v3/contacts/attributes` first — POST on existing attribute returns error |

## Reference

- Names must be UPPERCASE, immutable once created
- Max 200 chars per category label
- POST on existing attribute returns error — always check first
