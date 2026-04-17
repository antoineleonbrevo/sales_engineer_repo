---
name: create-loyalty-program
description: "Create a Brevo Loyalty program and perform first publish. Activates on: create loyalty program, loyalty program, programme fidélité."
tools: Bash, Read, Write
---

# Create Loyalty Program

Create a loyalty program via Brevo API, then publish it to activate.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `program.name`, `program.description` | **Write**: `program.id`, `program.state`

## Workflow

1. Read `loyalty-demo-context.json` for program name and description
2. Check if a program already exists:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   curl -s -X GET "https://api.brevo.com/v3/loyalty/config/programs" \
     -H "api-key: $(cat /tmp/.brevo_key)"
   SCRIPT
   bash /tmp/script.sh
   ```
   If a program exists, store its `id` and skip creation.
3. Create the program:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   curl -s -X POST "https://api.brevo.com/v3/loyalty/config/programs" \
     -H "api-key: $(cat /tmp/.brevo_key)" \
     -H "Content-Type: application/json" \
     -d '{
       "name": "<program_name>",
       "description": "<program_description>"
     }'
   SCRIPT
   bash /tmp/script.sh
   ```
4. Store `pid` in `/tmp/.brevo_loyalty_pid`
5. First publish to activate:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   PID=$(cat /tmp/.brevo_loyalty_pid)
   curl -s -X POST "https://api.brevo.com/v3/loyalty/config/programs/$PID/publish" \
     -H "api-key: $(cat /tmp/.brevo_key)"
   SCRIPT
   bash /tmp/script.sh
   ```
6. Update `loyalty-demo-context.json` with `program.id` and `program.state`

## Pre-call Validation

| Check | Rule |
|-------|------|
| `name` | **Required** — max 128 characters |
| `description` |**Required** — customer-facing, max 256 characters |
| `documentId` | Optional — external document reference |
| `meta` | Optional — arbitrary key-value metadata object |
| `api-key` | Must exist in `/tmp/.brevo_key` |

## API Reference

- **List**: `GET /v3/loyalty/config/programs` — returns `{ "programs": [...] }` with `id`, `name`, `state`, `description`
- **Create**: `POST /v3/loyalty/config/programs`
- **Get**: `GET /v3/loyalty/config/programs/{pid}` — verify a specific program
- **Publish**: `POST /v3/loyalty/config/programs/{pid}/publish` — returns program with `state: "active"` and `publishedAt`
- Program starts `inactive` — members cannot be enrolled until the program is published
- Save returned `id` as `{pid}` — used in ALL subsequent loyalty API calls

### Create Request Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Program name, max 128 characters |
| `description` | string | No | Customer-facing description, max 256 characters |
| `documentId` | string | No | Optional external document reference |
| `meta` | object | No | Arbitrary key-value metadata |

### Create Response (200)

```json
{
  "id": "27xxdd7a-af67-0020-ba65-19d60000a26e",
  "name": "VIP Club",
  "description": "Earn points on every purchase and unlock exclusive rewards.",
  "state": "inactive",
  "createdAt": "2025-03-01T10:00:00.000Z",
  "updatedAt": "2025-03-01T10:00:00.000Z"
}
```

### Expected Responses

| Endpoint | Success | Error codes |
|----------|---------|-------------|
| `GET /programs` | `200` — list of programs | `401` — invalid API key |
| `POST /programs` | `200` — created (returns `id`, `name`, `description`, `state: "inactive"`, `createdAt`, `updatedAt`) | `400` — invalid request, `409` — already exists |
| `GET /programs/{pid}` | `200` — program details | `404` — not found |
| `POST /programs/{pid}/publish` | `200` — returns program with `state: "active"`, `publishedAt` | `404` — program not found |

## Shell Safety

Always write curl commands to `/tmp/script.sh` and execute with `bash /tmp/script.sh`. Never interpolate UUIDs directly in multi-line Bash commands.
