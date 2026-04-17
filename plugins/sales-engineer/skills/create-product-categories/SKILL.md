---
name: create-product-categories
description: "Activate ecommerce and create product categories in Brevo. Activates on: product categories, create categories, ecommerce setup."
tools: Bash, Read, Write
---

# Create Product Categories

Activate ecommerce app, batch-create categories.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `plan.categories`, `meta.volumes.categories` | **Write**: `created.categories`

## Workflow

1. Activate ecommerce (ignore error if already active):
   ```bash
   curl -s -X POST "https://api.brevo.com/v3/ecommerce/activate" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json"
   ```
2. Batch create categories (`updateEnabled` at **batch level**, not per category):
   ```bash
   curl -s -X POST "https://api.brevo.com/v3/categories/batch" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
     -d '{"categories": [{"id": "CAT001", "name": "Cuisines équipées", "url": "https://shop.com/categories/cuisines", "isDeleted": false, "deletedAt": null}, {"id": "CAT002", "name": "Salles de bain", "url": "https://shop.com/categories/salles-de-bain", "isDeleted": false, "deletedAt": null}], "updateEnabled": true}'
   ```
3. Use hierarchical structure: main categories + sub-categories by brand/type
4. Update context with created category IDs

## Pre-call Validation

Before `POST /categories/batch`:

| Check | Rule |
|-------|------|
| `categories` | **Required** array of category objects |
| `updateEnabled` | Set `true` at **batch level** (outside `categories` array) for idempotent re-runs |
| Batch limit | Max **100 categories** per request |
| **Per category:** | |
| `id` | **Required**, string — unique category ID (e.g. `CAT001`) |
| `name` | **Mandatory on creation**, string — display name in shop |
| `url` | Optional, string — URL to the category page |
| `isDeleted` | Optional, boolean — `false` for active, `true` for deleted |
| `deletedAt` | Optional, string — UTC date-time `YYYY-MM-DDTHH:mm:ss.SSSZ` (only if `isDeleted: true`) |
| Ecommerce | Must be activated first (`POST /ecommerce/activate`) — ignore error if already active |

## Best Practices

- Use **hierarchical structure**: main categories + sub-categories (by brand, type, etc.)
- Add **category URLs** for each entry
- All attributes are strings except booleans (`isDeleted`)
- `updateEnabled: true` allows updating existing categories in same request

## Reference

- Endpoint: `POST /v3/categories/batch`
- Batch limit: 100 categories/request
- `updateEnabled` at batch level (not per category)
- Category IDs are strings
