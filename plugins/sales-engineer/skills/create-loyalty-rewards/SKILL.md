---
name: create-loyalty-rewards
description: "Create rewards (offers) for a Brevo Loyalty program. Activates on: loyalty rewards, create rewards, create offers, récompenses fidélité."
tools: Bash, Read, Write, WebSearch
---

# Create Loyalty Rewards

Create rewards (offers) for an existing loyalty program.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `program.id`, `rewards` (planned) | **Write**: `rewards` (with `id`)

## Workflow

1. Read `loyalty-demo-context.json` for `program.id` and planned rewards
2. For each reward, find a royalty-free image and upload it to Brevo (see Image Upload section)
3. Create each reward:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   PID=$(cat /tmp/.brevo_loyalty_pid)
   curl -s -X POST "https://api.brevo.com/v3/loyalty/offer/programs/$PID/offers" \
     -H "api-key: $(cat /tmp/.brevo_key)" \
     -H "Content-Type: application/json" \
     -d '{
       "name": "<internal_name_no_spaces>",
       "publicName": "<client_facing_name>",
       "publicDescription": "<client_facing_description>",
       "publicImage": "<brevo_hosted_image_url>"
     }'
   SCRIPT
   bash /tmp/script.sh
   ```
   Repeat for each reward.
4. Update `loyalty-demo-context.json` with reward IDs

## Image Upload (mandatory before reward creation)

Images must be uploaded to Brevo first via `POST /emailCampaigns/images`. Direct external URLs (Unsplash, Pexels) are **NOT accepted** as `publicImage` — only Brevo-hosted URLs work.

### Upload Endpoint

```bash
cat > /tmp/script.sh << 'SCRIPT'
curl -s -X POST "https://api.brevo.com/v3/emailCampaigns/images" \
  -H "api-key: $(cat /tmp/.brevo_key)" \
  -H "content-type: application/json" \
  -d '{
    "imageUrl": "<public_url_with_extension>",
    "name": "<filename_with_extension>"
  }'
SCRIPT
bash /tmp/script.sh
```

### Critical Rules

| Rule | Detail |
|------|--------|
| `imageUrl` | Must be a public URL with a **visible file extension** (`.jpg`, `.png`). URLs with query params only (e.g., `?w=400&q=80`) are rejected: `"image format passed in the url is not valid"` |
| `name` | Must include the file extension (e.g., `reward-welcome-bonus.jpg`). Without extension → `"image format passed in the name is not valid"` |
| **Size limit** | **Brevo rejects images over 2MB.** Prefer Pexels URLs with `?w=400&h=400&fit=crop` for small file size. If upload fails, try a different image or append resize params |
| Response | Returns a Brevo-hosted URL (`https://img.mailinblue.com/...`) — use this as `publicImage` on rewards and `imageRef` on tiers |

### Image Upload Fallback Strategy

If an upload fails (size, format, or timeout):
1. **Try a smaller variant** — append `?w=400&h=400&fit=crop&auto=compress` to the Pexels URL and retry
2. **If still failing** — search for an alternative image with similar keywords
3. **If 2 retries fail** — create the reward without `publicImage` (field is optional), log the failure, and continue. Note it in the final summary so the user can add images manually in Brevo UI

### Image Selection Strategy

#### Goal
Each reward image must be **visually coherent** with what the reward actually is. A generic photo is not acceptable if a specific one exists.

#### Search approach — strict by default

1. **Search specifically** for the reward using `WebSearch site:pexels.com {reward_name}` or `site:pexels.com {reward_type}`
2. Use the **most specific keyword** possible:
   - ✅ `pexels.com gift card shopping red` (specific)
   - ❌ `pexels.com happy customer` (too generic)
3. If no specific result → broaden one level (e.g. `gift card` → `gift box`)
4. If still no result → **reuse the best matching image from a similar reward** rather than an unrelated photo
5. **Same image for multiple rewards is acceptable** if they share the same reward type and no better option exists
6. **Avoid** Unsplash URLs with only query params (`?w=400&q=80`) — they lack a visible file extension required by Brevo
7. Prefer Pexels direct URLs: `https://images.pexels.com/photos/<id>/pexels-photo-<id>.jpeg`

#### Keywords by reward type

Build keywords from: `{reward object} + {context/setting} + {visual style}` for the most relevant results.

| Reward type | Primary keywords | Fallback keywords |
|---|---|---|
| Discount / voucher | `gift card discount voucher` | `coupon shopping red` |
| Free shipping | `delivery cardboard box shipping` | `package delivery door` |
| Free product / sample | `product sample gift box` | `free gift wrapped` |
| Upgrade (room / seat / tier) | `luxury upgrade premium gold` | `vip exclusive access` |
| Cashback / points | `coins money cashback reward` | `gold coins pile` |
| Free drink / coffee | `coffee cup takeaway barista` | `hot drink cafe` |
| Free meal | `gourmet plate restaurant food` | `meal restaurant table` |
| Spa / wellness | `spa wellness relaxation towel` | `massage candle relaxation` |
| Priority access / lounge | `airport lounge vip comfortable` | `exclusive lounge seating` |
| Exclusive event / experience | `event concert exclusive vip` | `experience celebration` |
| Subscription / premium feature | `premium subscription digital` | `laptop screen subscription` |
| Birthday reward | `birthday cake candles celebration` | `birthday gift surprise` |

## Pre-call Validation

| Check | Rule |
|-------|------|
| `name` | **Required** — internal name, **no spaces** (e.g., `reward_tier1_remise`, not `Reward Tier 1`). The visible label goes in `publicName` |
| `publicName` | Optional — client-facing reward name |
| `publicDescription` | Optional — client-facing description |
| `publicImage` | Optional — URL to a royalty-free image (must be publicly accessible) |
| `pid` | Must exist in `/tmp/.brevo_loyalty_pid` |

## Industry-specific Reward Examples

| Industry | Reward Examples |
|----------|----------------|
| Retail / E-commerce | 10% discount, free shipping, exclusive product access |
| Food & Beverage | Free drink, meal upgrade, birthday dessert |
| Beauty / Cosmetics | Free sample, mini product, birthday gift set |
| Airlines / Travel | Seat upgrade, lounge access, priority boarding |
| Hospitality | Room upgrade, late checkout, spa credit |
| SaaS / B2B | Premium support, training credits, feature unlock |

## API Reference

- **Create**: `POST /v3/loyalty/offer/programs/{pid}/offers`
- Rewards are optional but recommended for impactful demos
- Once created, rewards can be attributed to members via automated rules or manually via the API

### Request Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Internal name, **no spaces** (e.g., `reward_welcome_bonus`). Use `publicName` for the client-facing label |
| `publicName` | string | No | Client-facing reward name |
| `publicDescription` | string | No | Client-facing description |
| `publicImage` | string (URI) | No | **Brevo-hosted URL** only (from `POST /emailCampaigns/images`). Direct external URLs are not accepted |

### Response (200)

```json
{
  "id": "uuid",
  "loyaltyProgramId": "uuid",
  "name": "Welcome Bonus",
  "publicName": "Bonus de bienvenue",
  "publicDescription": "10% de réduction sur votre prochaine commande",
  "publicImage": "https://images.unsplash.com/photo-...",
  "createdAt": "2024-01-01T00:00:00.000Z",
  "updatedAt": "2024-01-01T00:00:00.000Z"
}
```

### Expected Responses

- `200` — Reward created successfully
- `401` — Authentication failed
- `403` — Access denied
- `422` — Validation errors (check required fields)

## Shell Safety

Always write curl commands to `/tmp/script.sh` and execute with `bash /tmp/script.sh`.
