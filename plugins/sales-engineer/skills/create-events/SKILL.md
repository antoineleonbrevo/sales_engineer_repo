---
name: create-events
description: "Create tracking events in Brevo for demo contacts. Max 5 types, page_view + cart_updated mandatory. Activates on: create events, demo events, tracking events."
tools: Bash, Read, Write
---

# Create Events

Create events: max 5 types, 2 mandatory, each attached to min 20 contacts.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `plan.events`, `plan.event_funnels`, `created.contacts`, `created.products`, `research` | **Write**: `created.events`

## Event Type Rules

| Constraint | Value |
|-----------|-------|
| Max types | 5 |
| Mandatory | `page_view` (page_url, page_title) |
| Optional | Up to 4, industry-specific (see below). `cart_updated` is optional and only relevant for e-commerce clients with an online shopping cart — omit for franchise, services, B2B or any client without a cart |
| Min contacts/type | 20 |

### Industry suggestions (pick up to 3)

E-commerce: `purchase_completed`, `product_viewed`, `review_submitted`
SaaS: `feature_used`, `plan_upgraded`, `trial_started`
Franchise/Retail: `store_visit_booked`, `quote_requested`, `project_created`
Hospitality: `booking_confirmed`, `check_in`, `feedback_submitted`
Real Estate: `property_viewed`, `visit_scheduled`, `offer_submitted`

## Batch Format — Critical

The `POST /v3/events/batch` body **must be a top-level JSON array** `[...]`. Wrapping it in an object causes a `400 — json: cannot unmarshal array into Go value of type main.unifiedEventsRequest` error.

**Correct** — raw array:
```bash
curl --request POST \
     --url https://api.brevo.com/v3/events/batch \
     --header 'accept: application/json' \
     --header 'api-key: YOUR_API_KEY' \
     --header 'content-type: application/json' \
     --data '[{"event_name":"order_created","identifiers":{"email_id":"jane@example.com"},"event_properties":{"order_id":"ORD-1234"}}]'
```

**Wrong** — wrapped in object (causes 400):
```json
{ "events": [...] }
```

### Common Mistakes

| Mistake | Effect |
|---------|--------|
| Wrapping array in `{"events": [...]}` | `400 cannot unmarshal array` |
| Sending a single object `{}` instead of array `[{}]` | `400` |
| Missing `content-type: application/json` header | `400` |
| Missing `event_name` or `identifiers` per event | `400` |

## Workflow

1. Read context for event types, contacts, products
2. **Use the single event endpoint** (`POST /v3/events`) via a Python script. The batch endpoint (`POST /v3/events/batch`) is documented but returns `400 — cannot unmarshal array` in practice even with a correctly formatted raw array body — do not use it. The single endpoint is reliable and returns `204` per event:
   ```python
   # Write to /tmp/loyalty_events.py then execute with python3
   import urllib.request, json, time

   api_key = open("/tmp/.brevo_key").read().strip()

   # Build all events
   events = [
       {
           "event_name": "page_view",
           "event_date": "2025-12-10T14:30:00.000Z",
           "identifiers": {"email_id": "client@example.com"},
           "contact_properties": {"FIRSTNAME": "Jane"},
           "event_properties": {
               "page_url": "https://shop.com/products/cuisine-premium",
               "page_title": "Cuisine Premium",
               "device_type": "desktop",
               "browser": "Chrome"
           }
       },
       # ... more events
   ]

   # Send one by one via single endpoint (batch endpoint is unreliable — returns 400)
   ok = 0
   for evt in events:
       data = json.dumps(evt).encode()
       req = urllib.request.Request(
           "https://api.brevo.com/v3/events",
           data=data,
           headers={"api-key": api_key, "content-type": "application/json"}
       )
       try:
           with urllib.request.urlopen(req) as r:
               ok += 1
       except urllib.error.HTTPError as e:
           print("  ERROR {} for {}: {}".format(evt["event_name"], evt["identifiers"].get("email_id","?"), e.code))
       time.sleep(0.05)
   print("Sent: {}/{}".format(ok, len(events)))
   ```

   **Why Python, not bash curl?** Bash curl loops with interpolated JSON cause `"Parsing error: unmarshal json"` due to line endings and quote escaping. Python's `json.dumps()` guarantees valid JSON serialization.

3. Distribution per segment:
   VIP: 8-15 events, all types, last 30 days
   Active: 5-8 events, 3-4 types, last 60 days
   New: 2-4 events, page_view + 1 optional type, last 7 days
   At-risk: 1-2 events, page_view only, 90+ days ago
4. Follow funnels coherent with the industry — e.g. `page_view → {lead_event} → {conversion_event}`
5. Optionally associate events with custom objects via `object` field
6. Update context

## Pre-call Validation

### Batch level

| Check | Rule |
|-------|------|
| Body format | JSON **array** of event objects (not wrapped in `{"events": [...]}`) |
| Batch limit | Max **200 events** per request — split into multiple batches if needed |
| Request size | Max **512 KB** per request |

### Per event

| Check | Rule |
|-------|------|
| `event_name` | **Required** — String, max **255 chars**, alphanumeric + `-` + `_` only |
| `identifiers` | **Required** — use `email_id` (string) or `contact_id` (int). `email_id` preferred. Must resolve to an existing contact |
| `event_date` | Optional — ISO 8601 (`YYYY-MM-DDTHH:mm:ss.SSSZ`). Defaults to now if omitted |
| `event_properties` | Optional — JSON object, max **50 KB**. Supports: string, number, boolean, date, nested objects, arrays |
| `contact_properties` | Optional — update contact attributes on event (attribute names UPPERCASE) |
| `object` | Optional — associate with custom object: `{"type": "object_name", "identifiers": {"ext_id": "..."}}` |

## Response Codes

| Code | Description |
|------|-------------|
| `202` | All events accepted and queued for processing |
| `207` | Partial success — some events failed (check per-event details in response body) |
| `400` | All events failed validation |
| `401` | Unauthorized — missing or invalid API key |

## Batch Endpoint — Do Not Use

`POST /v3/events/batch` is documented in the Brevo API but returns `400 — "cannot unmarshal array into Go value of type main.unifiedEventsRequest"` even when the body is a correctly formatted raw JSON array. This is a known Brevo API issue. Always use `POST /v3/events` (single endpoint) instead.

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Never use `False`/`True` in inline `-c` heredocs | Shell may lowercase them — define a helper function instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues |
| Use `python3 << 'PYEOF' ... PYEOF` only for simple scripts | Use a unique delimiter (`PYEOF` not `EOF`) to avoid conflicts |

## Reference

- **Use**: `POST /v3/events` — single event per call, returns `204 No Content`
- **Avoid**: `POST /v3/events/batch` — returns `400` in practice even with correct format
- `event_name`: alphanumeric + `-` + `_` only, max 255 chars
- `identifiers`: use `email_id` (preferred) or `contact_id` — must match an existing contact
- `event_properties`: 50 KB max per event
- **Always write Python to a file** (`/tmp/events.py`) and run with `python3` — never inline heredoc for complex scripts
