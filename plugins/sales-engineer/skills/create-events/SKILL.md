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
| Mandatory | `page_view` (page_url, page_title), `cart_updated` (cart_value, product_id, quantity) |
| Optional | Up to 3, industry-specific (see below) |
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
2. **Preferred: use the batch endpoint** (`POST /v3/events/batch`). Send a JSON **array** of event objects — **never wrapped in an object**. Use a Python script for reliable JSON serialization:
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
       {
           "event_name": "cart_updated",
           "event_date": "2025-12-10T14:35:00.000Z",
           "identifiers": {"email_id": "client@example.com"},
           "event_properties": {
               "cart_value": 2499.99,
               "product_id": "P001",
               "quantity": 1,
               "currency": "EUR"
           }
       },
       # ... more events
   ]

   # Send in batches of 200
   BATCH_SIZE = 200
   for i in range(0, len(events), BATCH_SIZE):
       batch = events[i:i+BATCH_SIZE]
       data = json.dumps(batch).encode()
       req = urllib.request.Request(
           "https://api.brevo.com/v3/events/batch",
           data=data,
           headers={
               "api-key": api_key,
               "content-type": "application/json",
               "accept": "application/json"
           }
       )
       try:
           with urllib.request.urlopen(req) as resp:
               status = resp.status
               body = resp.read().decode()
               if status == 202:
                   print(f"  Batch {i//BATCH_SIZE+1}: all {len(batch)} events accepted")
               elif status == 207:
                   print(f"  Batch {i//BATCH_SIZE+1}: partial success — check details: {body}")
       except urllib.error.HTTPError as e:
           print(f"  Batch {i//BATCH_SIZE+1} ERROR: {e.code} {e.read().decode()}")
           # Fallback: send events one by one via single endpoint
           print("  Falling back to single event endpoint...")
           for evt in batch:
               single_data = json.dumps(evt).encode()
               single_req = urllib.request.Request(
                   "https://api.brevo.com/v3/events",
                   data=single_data,
                   headers={"api-key": api_key, "content-type": "application/json"}
               )
               try:
                   with urllib.request.urlopen(single_req) as r:
                       print(f"    {evt['event_name']} for {evt['identifiers'].get('email_id', 'N/A')}: {r.status}")
               except urllib.error.HTTPError as se:
                   print(f"    ERROR {evt['event_name']}: {se.code}")
               time.sleep(0.1)
       time.sleep(0.5)
   ```

   **Why Python, not bash curl?** Bash curl loops with interpolated JSON cause `"Parsing error: unmarshal json"` due to line endings and quote escaping. Python's `json.dumps()` guarantees valid JSON serialization.

3. Distribution per segment:
   VIP: 8-15 events, all types, last 30 days
   Active: 5-8 events, 3-4 types, last 60 days
   New: 2-4 events, page_view + cart_updated, last 7 days
   At-risk: 1-2 events, page_view only, 90+ days ago
4. Follow funnels: `page_view → cart_updated → {event_3} → ...`
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

## Fallback Strategy

If the batch endpoint returns `400`, first verify the body is a raw JSON array (not wrapped). If the format is correct and the error persists, fall back to the **single event endpoint**:

- **Single endpoint**: `POST /v3/events`
- Sends one event per call
- Returns `204 No Content` on success
- Slower but isolates which individual events fail

The Python script above includes automatic fallback: if a batch call returns an error, it retries each event individually via the single endpoint.

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Never use `False`/`True` in inline `-c` heredocs | Shell may lowercase them — define a helper function instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues |
| Use `python3 << 'PYEOF' ... PYEOF` only for simple scripts | Use a unique delimiter (`PYEOF` not `EOF`) to avoid conflicts |

## Reference

- **Preferred**: `POST /v3/events/batch` — up to 200 events per call, 512 KB max
- **Fallback**: `POST /v3/events` — single event per call
- `event_name`: alphanumeric + `-` + `_` only, max 255 chars
- `identifiers`: use `email_id` (preferred) or `contact_id` — must match an existing contact
- `event_properties`: 50 KB max per event
- **Always write Python to a file** (`/tmp/events.py`) and run with `python3` — never inline heredoc for complex scripts
