---
name: create-balance-definition
description: "Create a balance definition for a Brevo Loyalty program. Activates on: create balance, balance definition, loyalty balance, points definition."
tools: Bash, Read, Write
---

# Create Balance Definition

Create a balance definition (points/currency) for an existing loyalty program.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `program.id`, `balanceDefinitions` (planned) | **Write**: `balanceDefinitions` (with `id`)

## Workflow

1. Read `loyalty-demo-context.json` for `program.id` and planned balance config
2. Check for existing balance definitions:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   PID=$(cat /tmp/.brevo_loyalty_pid)
   curl -s -X GET "https://api.brevo.com/v3/loyalty/balance/programs/$PID/balance-definitions" \
     -H "api-key: $(cat /tmp/.brevo_key)" \
     -H "Content-Type: application/json"
   SCRIPT
   bash /tmp/script.sh
   ```
   If a matching balance exists, store its `id` and skip creation.
3. Create the balance definition:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   PID=$(cat /tmp/.brevo_loyalty_pid)
   curl -s -X POST "https://api.brevo.com/v3/loyalty/balance/programs/$PID/balance-definitions" \
     -H "api-key: $(cat /tmp/.brevo_key)" \
     -H "content-type: application/json" \
     -d '{
       "name": "<balance_name>",
       "unit": "POINTS",
       "expiryPolicy": {
         "type": "<never|rolling|fixed|inactivity>",
         "duration": <duration>,
         "unit": "<month|year|day>"
       },
       "roundingPolicy": "<none|round_half_up|floor|ceiling>",
       "maxBalance": <max>,
       "minBalance": <min>,
       "maxCreditPerOperation": <max_credit>,
       "maxDebitPerOperation": <max_debit>
     }'
   SCRIPT
   bash /tmp/script.sh
   ```
4. Store `balanceDefinitionId` in `/tmp/.brevo_loyalty_balance_id`
5. Update `loyalty-demo-context.json`

## Pre-call Validation

| Check | Rule |
|-------|------|
| `name` | **Required** — internal name. Carries the business label (e.g., "Graines Veja", "Miles", "Stars") |
| `unit` | **Required** — strict enum: `POINTS`, `EUR`, `USD`, `MXN`, `GBP`, `INR`, `CAD`, `SGD`, `RON`, `JPY`, `MYR`, `CLP`, `PEN`, `MAD`, `AUD`, `CHF`, `BRL`. Use `POINTS` for points/miles/stars/stamps — the `name` field carries the business label |
| `pid` | Must exist in `/tmp/.brevo_loyalty_pid` |

## Industry-specific Expiry Guidelines

| Industry | `expiryPolicy.type` | Duration | Unit | Rationale |
|----------|---------------------|----------|------|-----------|
| Retail / E-commerce | `rolling` | 12 | month | Encourages regular purchases |
| Airlines / Travel | `rolling` | 18-24 | month | Longer cycles between trips |
| Food & Beverage | `inactivity` | 6 | month | Drives frequent visits |
| SaaS / B2B | `rolling` | 36+ | month | Long-term relationship |
| Beauty / Cosmetics | `rolling` | 12 | month | Seasonal purchase cycles |
| Hospitality | `fixed` | — | — | Calendar-year reset encourages annual stays |

## API Reference

- **Create**: `POST /v3/loyalty/balance/programs/{pid}/balance-definitions`
- **List**: `GET /v3/loyalty/balance/programs/{pid}/balance-definitions`
- **Update**: `PUT /v3/loyalty/balance/programs/{pid}/balance-definitions/{bdid}`
- Base path is `/balance/` — balance definitions are balance domain operations
- Save returned `id` as `{balanceDefinitionId}` — needed for tier group creation and transactions

### Expected Responses

- `201` — Created successfully, returns `{ "id": "...", "name": "...", "unit": "...", "state": "active" }`
- `400` — Invalid request (check required fields)
- `409` — Conflict (balance definition already exists)

### Full parameter reference

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Internal name — carries the business label (e.g., "Graines Veja", "Miles", "Stars") |
| `unit` | string (enum) | Yes | Strict enum: `POINTS`, `EUR`, `USD`, `MXN`, `GBP`, `INR`, `CAD`, `SGD`, `RON`, `JPY`, `MYR`, `CLP`, `PEN`, `MAD`, `AUD`, `CHF`, `BRL`. Free-text values are rejected |
| `expiryPolicy.type` | string | No | `"never"`, `"rolling"` (from last activity), `"fixed"` (calendar date each year), or `"inactivity"` (after N months without purchase) |
| `expiryPolicy.duration` | int | For rolling/inactivity | Duration before expiry (e.g., `12`) |
| `expiryPolicy.unit` | string | For rolling/inactivity | Time unit: `"month"`, `"year"`, `"day"` |
| `roundingPolicy` | string | No | `"none"` (maintain decimals), `"round_half_up"` (nearest), `"floor"` (always down), `"ceiling"` (always up) |
| `maxBalance` | number | No | Maximum balance cap — credits refused once reached, triggers `balance_transaction_unauthorized` event |
| `minBalance` | number | No | Minimum balance threshold — falling below can trigger automations |
| `maxCreditPerOperation` | number | No | Maximum amount that can be credited in a single transaction |
| `maxDebitPerOperation` | number | No | Maximum amount that can be debited in a single transaction |
| `creditFrequencyLimit` | object | No | Maximum number of credit operations allowed per period |
| `debitFrequencyLimit` | object | No | Maximum number of debit operations allowed per period |
| `meta` | object | No | Additional metadata. Supports `isInternal` (boolean) to hide from member-facing balance reads unless `includeInternal=true` is passed |

## Shell Safety

Always write curl commands to `/tmp/script.sh` and execute with `bash /tmp/script.sh`.
