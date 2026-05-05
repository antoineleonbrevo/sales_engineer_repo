---
name: create-object-events
description: "Create custom events linked to custom object records in Brevo, where the custom object is the primary entity (not the contact). Use this skill any time the user needs to create a custom event associated with a custom object — e.g. 'create events for my custom objects', 'add lifecycle events to my object records', 'track status changes on my custom object', 'create a custom event for a custom object'. Tracks object lifecycle activity (contract signed, dossier status changed, subscription renewed, intervention resolved). The key difference from create-events: the object is the primary entity and each event triggers one automation run per object record."
tools: Bash, Read, Write
---

# Create Object Events

Create events that track custom object lifecycle — the object is the primary entity, the contact is secondary.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `created.custom_objects` (ext_ids, object type), `created.contacts` (brevo_id, email, segment) | **Write**: `created.object_events`

## Core Concept

Unlike standard contact events (`create-events`), object events are attached to a custom object record. They're used primarily to **trigger automations scoped to one message per object** — for example, one renewal email per contract, one alert per dossier, one follow-up per subscription.

The contact link is still required by the API (`identifiers` at root level), but the meaningful entity is the object.

## Critical: Correct Payload Format

The `object` block uses `type` + `identifiers` — never use `entity`:

```json
// ✅ Correct
{
  "event_name": "dossier_status_changed",
  "event_date": "2026-05-05T10:18:00+02:00",
  "identifiers": { "email_id": "alice.guerin@bio.fr" },
  "event_properties": {
    "dossier_id": "BPI-DOS-001",
    "old_status": "en_cours",
    "new_status": "instruit"
  },
  "object": {
    "type": "dossier_bpi",
    "identifiers": {
      "ext_id": "BPI-DOS-001"
    }
  }
}

// ❌ Wrong — "entity" key with "type" + "id" directly
{
  "entity": {
    "type": "dossier_bpi",
    "id": "BPI-DOS-001"
  }
}
```

## Workflow

### Step A — Declare event types in Brevo (manual, required before API calls)

Custom events must be declared in Brevo before they can be used as automation triggers.

Present one block per event type in the user's language:

**French:**
```
╔══════════════════════════════════════════════════════════╗
  EVENT À DÉCLARER : {event_name}
  Chemin : Paramètres → Transactional → Events → + Create
╚══════════════════════════════════════════════════════════╝

① Nom de l'event : {event_name}
② Propriétés suggérées : {prop_1}, {prop_2}, ...

Usage automation : {description de l'usage — ex: "déclenche un email de renouvellement par contrat"}
```

**English:**
```
╔══════════════════════════════════════════════════════════╗
  EVENT TO DECLARE: {event_name}
  Path: Settings → Transactional → Events → + Create
╚══════════════════════════════════════════════════════════╝

① Event name: {event_name}
② Suggested properties: {prop_1}, {prop_2}, ...

Automation use: {description — e.g. "triggers one renewal email per contract record"}
```

**German:**
```
╔══════════════════════════════════════════════════════════╗
  EVENT ANLEGEN: {event_name}
  Pfad: Einstellungen → Transactional → Events → + Create
╚══════════════════════════════════════════════════════════╝

① Event-Name: {event_name}
② Vorgeschlagene Properties: {prop_1}, {prop_2}, ...

Automations-Zweck: {Beschreibung — z.B. „löst eine Verlängerungs-E-Mail pro Vertrag aus"}
```

After presenting all event types, ask the user to confirm creation before proceeding.

### Step B — Define event types from object nature

Infer event types from the custom object's domain. Each event should reflect a meaningful state change or lifecycle milestone of the object. Use 2–4 event types max.

| Domain | Object example | Event suggestions |
|--------|---------------|-------------------|
| Finance / public funding | dossier, application | `dossier_submitted`, `dossier_status_changed`, `dossier_approved` |
| SaaS / subscription | subscription, contract | `subscription_activated`, `subscription_renewed`, `subscription_churned` |
| Real estate | property, lease | `visit_scheduled`, `offer_submitted`, `lease_signed` |
| Retail / loyalty | loyalty_card, voucher | `voucher_issued`, `voucher_redeemed`, `tier_upgraded` |
| Franchise / store | store, intervention | `intervention_opened`, `intervention_resolved`, `audit_completed` |

Each event type should have 2–5 `event_properties` that reflect the object state (IDs, statuses, values, dates).

### Step C — Create events via batch API

Write a Python script to `/tmp/object_events.py` and run it with `python3`.

```python
import urllib.request, json, time

api_key = open("/tmp/.brevo_key").read().strip()

# Build all events — one per object record per event type
events = [
    {
        "event_name": "dossier_status_changed",
        "event_date": "2026-05-05T10:18:00+02:00",
        "identifiers": {"email_id": "alice.guerin@bio.fr"},
        "event_properties": {
            "dossier_id": "BPI-DOS-001",
            "old_status": "en_cours",
            "new_status": "instruit"
        },
        "object": {
            "type": "dossier_bpi",
            "identifiers": {"ext_id": "BPI-DOS-001"}
        }
    },
    # ... one entry per object record × event type
]

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
                print("Batch {}: {} events accepted".format(i//BATCH_SIZE+1, len(batch)))
            elif status == 207:
                print("Batch {}: partial success — {}".format(i//BATCH_SIZE+1, body))
    except urllib.error.HTTPError as e:
        print("Batch {} ERROR: {} {}".format(i//BATCH_SIZE+1, e.code, e.read().decode()))
        # Fallback: single endpoint
        for evt in batch:
            single_req = urllib.request.Request(
                "https://api.brevo.com/v3/events",
                data=json.dumps(evt).encode(),
                headers={"api-key": api_key, "content-type": "application/json"}
            )
            try:
                with urllib.request.urlopen(single_req) as r:
                    print("  {} for {}: {}".format(evt["event_name"], evt["identifiers"].get("email_id","?"), r.status))
            except urllib.error.HTTPError as se:
                print("  ERROR {}: {}".format(evt["event_name"], se.code))
            time.sleep(0.1)
    time.sleep(0.5)
```

### Step D — Update context

Write `created.object_events` with event types created, object type, and record count.

## Event Design Rules

- **One event per record per type**: if 20 object records × 3 event types = 60 events total
- **`event_date`**: use realistic past dates, spread over last 30–90 days. Include timezone offset (ISO 8601: `"2026-04-10T09:30:00+02:00"`)
- **`identifiers`** at root level: use `email_id` if available, otherwise `contact_id` (int). This is the contact linked to the object record
- **`object.type`**: exact name of the custom object (same slug used in `POST /v3/objects/{type}/batch/upsert`)
- **`object.identifiers`**: use `ext_id` if records were created with external IDs (recommended), otherwise `id`
- **`event_properties`**: include the object's own ID as a property (e.g. `"dossier_id": "BPI-DOS-001"`) for readability in the automation builder

## Pre-call Validation

| Check | Rule |
|-------|------|
| Object records exist | Custom object records must be created before events — check `created.custom_objects` |
| Event declared in Brevo | Confirm Step A is done before API calls |
| `object.type` | Must match the exact object slug (identifier name), not the display name |
| `object.identifiers` | Use `ext_id` OR `id` — never both |
| `identifiers` (root) | Required — `email_id` or `contact_id` of the contact linked to the object |
| Body format | Raw JSON array `[...]` — never wrapped in `{"events": [...]}` |
| Batch size | Max 200 events per request |
| `event_name` | Alphanumeric + `-` + `_` only, max 255 chars |

## Response Codes

| Code | Meaning |
|------|---------|
| `202` | All events accepted |
| `207` | Partial success — check response body for per-event details |
| `400` | Validation failed — check body format and required fields |

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Always write scripts to `/tmp/*.py` then run with `python3` | Avoids heredoc quoting and escaping issues |
| Never use `False`/`True` in inline `-c` heredocs | Shell may lowercase them — use a file instead |

## Automation Tip (mention in demo)

Once events are created, each event type linked to an object can trigger a **journey scoped to that object** — meaning one automation run per record, not per contact. This is the key differentiator vs contact-level events: a contact with 3 contracts triggers 3 separate automation runs.
