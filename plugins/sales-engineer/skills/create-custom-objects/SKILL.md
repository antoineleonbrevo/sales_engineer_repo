---
name: create-custom-objects
description: "Create custom objects and records in Brevo (Marketing scope — no CRM Sales). Provides schema for manual object creation, then upserts records with contact associations. Max 5 object types, 50 records each. Activates on: custom objects, demo objects."
tools: Bash, Read, Write
---

# Create Custom Objects

Guide user through object schema creation, then upsert records.

> **Reference**: See `REFERENCE.md` for full upsert examples, association patterns, attribute types, and industry suggestions.

## Demo Context

- **Read**: `plan.custom_objects`, `created.contacts` (brevo_ids, segments), `meta.volumes` | **Write**: `created.custom_objects`

## Workflow

### Step A — Present object schemas for manual creation

For each planned custom object, present the schema using the following format:

```
╔══════════════════════════════════════════╗
  OBJET À CRÉER : {object_name}
╚══════════════════════════════════════════╝

| Attribut      | Type   | Notes                                        |
|---------------|--------|----------------------------------------------|
| {obj}_id ★ ID | text   | Identifiant objet + ajouter en attribut visible |
| {attr_1}      | {type} | {desc}                                       |
| {attr_2}      | {type} | {desc}                                       |

Association : Contact
Chemin : Settings → Data Model → Custom Objects → Create
```

**ID attribute rule**: Every object MUST have a `{object_name_singular}_id` attribute that serves as both the **object identifier** (for deduplication) and a **regular visible attribute** (for Brevo UI display). Mark it with `★ ID` in the table.

### Step B — Wait for user confirmation

Ask: *"Please create these custom objects in Brevo following the schemas above. Confirm when done and I will populate them with records."*

**This is the ONLY pause during Phase 2.** Required because object schemas cannot be created via API.

### Step C — Upsert records

Once confirmed, upsert records via `POST /objects/{object_type}/batch/upsert`. No need to GET objects first.

See REFERENCE.md for full payload examples with associations.

## Identifiers

Each record uses `identifiers` to enable upsert (create or update):

- `"ext_id": "MAG-001"` — External ID (recommended for demo — you control the value, idempotent)
- `"id": 400` — Brevo-generated internal ID (for existing records)

**Never provide both `id` and `ext_id` in the same request.**

## Association Pre-flight Checklist

| Check | Rule |
|-------|------|
| Contact IDs loaded | `created.contacts[].brevo_id` must be populated |
| Contact mapping | Each record has at least 1 contact association |
| Segment coherence | VIP contacts -> premium objects, At-risk -> expiring/expired |
| Association defined in schema | Relation must be configured in Brevo UI |
| Referenced records exist | All contacts and custom objects referenced must exist |
| Inter-object order | Independent objects first, then dependents |
| `ext_id` consistency | Same value in `identifiers` and in other objects' `associations` |
| Max limits | Max 10 association types per record, max 10 linked records per type |

## Consistency Rules

- `{object}_id` values must be unique and follow a pattern (e.g., `MAG-001`, `VEN-001`)
- `{object}_id` must appear BOTH as the record identifier AND in `attributes` for UI visibility
- VIP contacts -> premium records; At-risk -> expiring/expired

## Pre-call Validation

| Check | Rule |
|-------|------|
| `{type}` in URL | Must match the object name created in Brevo UI |
| `records` | **Required** array of record objects |
| Batch limit | Max **1000 records** per request, max **1 MB** body size |
| `attributes` per record | Max **500 attributes**. Undefined attributes silently ignored |
| `identifiers` per record | Use `ext_id` OR `id`. **Never both** |
| `associations[].object_type` | `"contact"` or custom object name — no CRM Sales objects |
| `associations[].records[].identifiers` | Referenced records must exist |
| Object schema | Must exist in Brevo UI **before** upserting |

## Response

- Async: returns `{"processId": int, "message": "Batch object records are being processed"}`
- Attributes not defined in schema are silently ignored (no error)

## Reference

- Endpoint: `POST /v3/objects/{object_type}/batch/upsert`
- Batch: 1000 records/req, 1MB max, 500 attrs/record, 10 associations/type
- Use `ext_id` for demo (idempotent, you control the value)
- Object schema must be created manually in Brevo UI before upserting
