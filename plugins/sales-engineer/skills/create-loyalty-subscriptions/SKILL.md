---
name: create-loyalty-subscriptions
description: "Subscribe contacts to a Brevo Loyalty program. Activates on: loyalty subscriptions, subscribe contacts, enroll members, inscrire membres."
tools: Bash, Read, Write
---

# Create Loyalty Subscriptions

Subscribe contacts to an active loyalty program.

> **Language**: Communicate with the user in the same language they use — French or English.

## Demo Context

- **Read**: `program.id`, `contacts` (with `contactId`) | **Write**: `contacts` (with subscription status, `loyaltySubscriptionId`)

## Prerequisites

Contacts **must already exist in Brevo** before subscribing them. Use the Contacts API (`POST /v3/contacts`) to create or sync contacts first.

## Workflow

1. Read `loyalty-demo-context.json` for `program.id` and contact list
2. If existing env with contacts: use those contacts
3. If new env: create demo contacts via Contacts API with realistic names matching the prospect's market
4. Retrieve each contact's `contactId` (Brevo internal integer ID) — needed for subscription
5. Subscribe each contact to the program:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   PID=$(cat /tmp/.brevo_loyalty_pid)
   curl -s -X POST "https://api.brevo.com/v3/loyalty/config/programs/$PID/subscriptions" \
     -H "api-key: $(cat /tmp/.brevo_key)" \
     -H "content-type: application/json" \
     -d '{
       "contactId": <contact_id>,
       "loyaltySubscriptionId": "<external_customer_id>",
       "creationDate": "<ISO_8601_date>"
     }'
   SCRIPT
   bash /tmp/script.sh
   ```
   - `contactId`: Brevo internal contact ID (integer, must be > 0). Get it from contact creation or `GET /v3/contacts/{email}`
   - `loyaltySubscriptionId`: your external customer ID (max 64 chars), useful for linking with your commerce system. Used as the identifier in subsequent transaction/balance API calls
   - `creationDate`: optional ISO 8601 date for historical imports. Omit for current date
   - On subscription, all balances are initialized to 0 and the member is placed on the default entry tier
   - Repeat for each contact
6. Update `loyalty-demo-context.json` with subscription results (including `contactId` and `loyaltySubscriptionId` for each contact)

## Pre-call Validation

| Check | Rule |
|-------|------|
| Program state | Must be `active` (published) before subscribing members |
| `pid` | Must exist in `/tmp/.brevo_loyalty_pid` |
| `contactId` | **Required** — Brevo internal contact ID (integer > 0). Contact must already exist |
| `loyaltySubscriptionId` | Optional — external customer ID, max 64 characters |
| `creationDate` | Optional — ISO 8601 format (e.g., `"2025-03-01T10:00:00.000Z"`). For historical data imports |

## Default Volumes

| Scenario | Count |
|----------|-------|
| New env | 20 demo contacts |
| Existing env | Reuse all fetched contacts (up to 50) |

## API Reference

- **Subscribe**: `POST /v3/loyalty/config/programs/{pid}/subscriptions`
- Base path is `/config/` — subscriptions are program configuration operations
- Program MUST be published (`state: active`) before members can be enrolled
- The `loyaltySubscriptionId` from the subscription request becomes the identifier used in subsequent member-level API calls (transactions, balances)

### Request Body

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `contactId` | integer | Yes | Brevo internal contact ID (must be > 0) |
| `loyaltySubscriptionId` | string | No | External customer ID, max 64 characters. Links with your commerce system |
| `creationDate` | string | No | Custom enrollment date (ISO 8601). For historical data imports |

### Response (200)

```json
{
  "contactId": 12345,
  "loyaltyProgramId": "27xxdd7a-af67-0020-ba65-19d60000a26e",
  "loyaltySubscriptionId": "cust_abc123",
  "organizationId": 1,
  "createdAt": "2025-03-01T10:00:00.000Z",
  "updatedAt": "2025-03-01T10:00:00.000Z",
  "versionId": 1
}
```

### Enrollment Behavior

- All balances are initialized to **0**
- Member is placed on the **default entry-level tier**

### Expected Responses

- `200` — Subscription created successfully
- `400` — Invalid request (contact does not exist, contactId <= 0)
- `409` — Conflict (member already subscribed)

## Shell Safety

Always write curl commands to `/tmp/script.sh` and execute with `bash /tmp/script.sh`.
