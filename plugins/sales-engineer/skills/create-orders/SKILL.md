---
name: create-orders
description: "Create orders in Brevo linking contacts to products. Activates on: create orders, demo orders, order history."
tools: Bash, Read, Write
---

# Create Orders

Create orders referencing created contacts and products from context.

> **Language**: Communicate with the user in the same language they use — French or English.

## Demo Context

- **Read**: `plan.orders`, `created.contacts` (emails, segments), `created.products` (IDs, prices), `meta.volumes.orders` | **Write**: `created.orders`

## Consistency Rules

- `email` must exist in `created.contacts[].email`
- `productId` must exist in `created.products[].id`
- `price` must match `created.products[].price`
- VIP contacts → more orders, higher amounts
- At-risk contacts → older dates, fewer orders

## Workflow

1. Read context for contacts, products, volumes
2. Create orders (use batch, `historical: false` for live data). **Contact is linked via `email` field directly** (not `identifiers`):
   ```bash
   curl -s -X POST "https://api.brevo.com/v3/orders/status/batch" \
     -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
     -d '{"orders": [{"id": "ORD-2025-001", "createdAt": "2025-11-15T10:30:45.123Z", "updatedAt": "2025-11-15T11:15:22.456Z", "status": "completed", "amount": 1847.99, "email": "client@example.com", "products": [{"productId": "P001", "quantity": 1, "variantId": "P001-BLUE", "price": 1329.00}, {"productId": "P006", "quantity": 2, "price": 29.99}], "billing": {"address": "123 Rue de la Paix", "city": "Paris", "country": "France", "countryCode": "FR", "phone": "+33123456789", "postCode": "75001", "paymentMethod": "PayPal", "region": "Île-de-France"}, "coupons": ["WELCOME20"], "metaInfo": {"order_source": "Website", "sales_rep": "Marie Dupont"}}], "historical": false}'
   ```
3. Vary: statuses (completed/pending/shipped/cancelled), dates over 6 months, billing from `research.markets`
4. Use `metaInfo` for demo-relevant metadata (order_source, sales_rep, campaign_source, etc.)
5. Update context

### Distribution by segment

VIP: 4-6 orders | Active: 2-3 | New: 1 | At-risk: 1 (old date)

## Pre-call Validation

Before `POST /orders/status/batch`:

| Check | Rule |
|-------|------|
| **Batch level:** | |
| `orders` | **Required** array of order objects |
| `historical` | Optional boolean — `false` for live data (default), `true` for historical import |
| `notifyUrl` | Optional — webhook URL for batch status notification |
| Batch limit | Max **1000 orders** or **5 MB** per request |
| **Per order (required):** | |
| `id` | String — unique order ID |
| `createdAt` | String — UTC `YYYY-MM-DDTHH:mm:ss.SSSZ` |
| `updatedAt` | String — UTC `YYYY-MM-DDTHH:mm:ss.SSSZ` |
| `status` | String — `completed`, `pending`, `shipped`, `cancelled`, etc. |
| `amount` | Number — total incl. tax + shipping |
| `products` | Array of product objects (see below) |
| `email` | **Required** — contact email, must match `created.contacts[].email` |
| **Per order (optional):** | |
| `storeId` | String — store identifier |
| `billing` | Object: `address`, `city`, `country`, `countryCode` (ISO 2), `phone`, `postCode`, `paymentMethod`, `region` |
| `coupons` | Array of coupon code strings (case insensitive) |
| `metaInfo` | JSON object — custom key/value metadata (order_source, sales_rep, campaign_source, etc.) |
| **Per product:** | |
| `productId` | **Required** — must exist in `created.products[].id` |
| `price` | **Required** — must match `created.products[].price` |
| `quantity` | Integer (use `quantity` OR `quantityFloat`, not both) |
| `quantityFloat` | Number — for decimal quantities |
| `variantId` | Optional — product variant ID |
| Ecommerce | Must be activated first |

## Response

- `202` — batch accepted, returns `{"batchId": int, "count": int}`
- `400` — request error

## Reference

- Endpoint: `POST /v3/orders/status/batch`
- Batch limit: 1000 orders or 5MB
- `createdAt`/`updatedAt`: UTC `YYYY-MM-DDTHH:mm:ss.SSSZ`
- Use `email` field directly to link orders to contacts (not `identifiers`)
