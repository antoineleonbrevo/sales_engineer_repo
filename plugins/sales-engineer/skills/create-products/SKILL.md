---
name: create-products
description: "Create products in Brevo with royalty-free images. Activates on: create products, demo products, product catalog."
tools: Bash, Read, Write, WebSearch
---

# Create Products

Create products with verified image URLs.

## Demo Context

- **Read**: `plan.products`, `created.categories`, `research.image_keywords`, `meta.volumes.products` | **Write**: `created.products`

## Image Selection Strategy

### Goal
Each product must have an image that is **visually coherent** with what the product actually is. A generic lifestyle photo is not acceptable if a specific product photo exists.

### Search approach — strict by default

1. **Search specifically** for the product name or category using `WebSearch site:pexels.com {product_name}` or `site:pexels.com {product_category}`
2. Use the **most specific keyword** possible:
   - ✅ `pexels.com electric radiator wall` (specific)
   - ❌ `pexels.com interior home` (too generic)
3. If no specific result → broaden one level (e.g. `electric heater` → `home heating`)
4. If still no result → **reuse the best matching image from a sibling product** rather than using an unrelated photo
5. **Same image for multiple products is acceptable** if they share the same category and no better option exists

### Image URL format

Use **Pexels** direct URLs (free, no API key needed):
```
https://images.pexels.com/photos/{photo_id}/pexels-photo-{photo_id}.jpeg?auto=compress&cs=tinysrgb&w=600
```

**Before using any image URL**, verify it returns HTTP 200:
```bash
curl -s -o /dev/null -w "%{http_code}" "https://images.pexels.com/photos/{id}/pexels-photo-{id}.jpeg?w=600"
```

If 404 → try another photo ID. **Never send an unverified URL to the Brevo API.**

### Search keywords by product type

Build keywords from: `{material/technology} + {product shape} + {color/finish}` for the most relevant results.

| Product type | Primary keywords | Fallback keywords |
|---|---|---|
| Wall electric radiator | `electric panel heater wall white` | `electric radiator modern` |
| Vertical radiator | `vertical electric radiator tall` | `column radiator design` |
| Skirting heater / baseboard | `baseboard heater low wall` | `electric skirting heating` |
| Towel rail / sèche-serviettes | `heated towel rail bathroom chrome` | `towel warmer wall bathroom` |
| Smart thermostat | `smart thermostat wall white round` | `digital thermostat home` |
| Sensor / detector | `motion sensor white wall smart home` | `home automation sensor` |
| Installation pack | `technician installing heating home` | `home renovation heating` |
| Maintenance contract / SAV | `technician service repair home` | `customer service professional` |
| Solar panels | `solar panels roof house` | `photovoltaic installation` |
| Water heater / chauffe-eau | `heat pump water heater white` | `domestic hot water system` |
| Accessory / remote | `remote control white home device` | `smart home accessory` |

## Create vs Update — Upsert Model

Brevo uses a **create-or-update (upsert)** model. There is **no `PATCH` or `PUT`** on products — calling `PATCH /v3/products/{id}` or `PUT /v3/products/{id}` returns `404 not_found — Invalid route/method passed`.

To update existing products, use the same `POST` endpoints with `"updateEnabled": true`.

| Operation | Endpoint | `updateEnabled` |
|-----------|----------|----------------|
| Create new product | `POST /v3/products` | `false` (default) |
| Update existing product | `POST /v3/products` | **`true`** |
| Create/update up to 100 at once | `POST /v3/products/batch` | **`true`** at batch level |

> `updateEnabled` goes **outside** the `products` array in batch mode, at the root of the payload.

**Response codes**: `201` = created, `204` = updated, `400` = invalid request.

## Workflow

1. Read context for planned products + category IDs
2. Find and **verify** image URLs for each product
3. Create or update (max 100/batch):
   ```bash
   # Batch create or update
   curl -s -X POST "https://api.brevo.com/v3/products/batch" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
     -d '{"products": [{"id": "P001", "name": "Product Name", "url": "https://shop.com/product", "imageUrl": "https://images.pexels.com/...", "sku": "SKU001", "price": 29.99, "stock": 10, "categories": ["cat-001"], "brand": "Brand", "description": "...", "metaInfo": {"vendor": "...", "stock_level": "En stock"}}], "updateEnabled": true}'

   # Single product create or update
   curl -s -X POST "https://api.brevo.com/v3/products" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
     -d '{"id": "P001", "name": "Product Name", "imageUrl": "https://images.pexels.com/...", "price": 29.99, "updateEnabled": true}'
   ```
4. Update context with created product IDs, prices, image URLs

## Pre-call Validation

Before `POST /v3/products/batch`:

| Check | Rule |
|-------|------|
| `products` | **Required** array of product objects |
| `updateEnabled` | **`true`** to allow updates — at **batch level** (outside products array) |
| Batch limit | Max **100 products** per request |
| **Per product:** | |
| `id` | **Required**, string — unique product ID in shop |
| `name` | **Mandatory on creation**, string |
| `url` | Optional, string — absolute URL to product page |
| `imageUrl` | Optional, string — absolute URL to image. **Must return HTTP 200** (verify before sending) |
| `sku` | Optional, string — stock keeping unit |
| `price` | Optional, positive float |
| `stock` | Optional, integer — current stock level |
| `categories` | Optional, array of category ID strings — must match `created.categories[].id` |
| `parentId` | Optional, string — ID of parent product (for variants) |
| `brand` | Optional, string |
| `description` | Optional, string |
| `isDeleted` | Optional, boolean |
| `deletedAt` | Optional, string — UTC date-time `YYYY-MM-DDTHH:mm:ss.SSSZ` |
| `metaInfo` | Optional, JSON object — custom key/value metadata (vendor, producer, stock_level, warranty, color, etc.) |
| Ecommerce | Must be activated first |

## Reference

- `POST /v3/products` — single product upsert (with `updateEnabled: true` to update)
- `POST /v3/products/batch` — up to 100 products upsert (`updateEnabled` at root level)
- **No PATCH or PUT** — those routes do not exist and return 404
- `imageUrl` must be absolute URL returning valid image (verify with curl before sending)
- `metaInfo` accepts any key/value pairs for custom product metadata
