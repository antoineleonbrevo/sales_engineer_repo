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

Present each object as a complete step-by-step guide. Use the format below — one block per object, in dependency order (independent objects first).

```
╔══════════════════════════════════════════════════════════╗
  OBJET À CRÉER : {object_name}
  Chemin : Settings → Data Model → Custom Objects → + Create
╚══════════════════════════════════════════════════════════╝

① Nom de l'objet : {object_name}

② Identifiant (Object ID) : {obj}_id
   → C'est le slug utilisé dans tous les appels API — choisis-le avec soin.
   → Ajoute aussi {obj}_id comme attribut classique (pour l'affichage dans l'UI Brevo).

③ Attributs à créer :

| #  | Attribute name   | Attribute type | Required | Notes / Valeurs possibles         |
|----|------------------|---------------|----------|-----------------------------------|
| 1  | {obj}_id         | Text          | OUI ★ ID | Aussi Object ID — ex: "{OBJ}-001" |
| 2  | {attr_1}         | {Type}        | {Oui/Non}| {description — valeurs possibles} |
| 3  | {attr_2}         | {Type}        | {Oui/Non}| {description — valeurs possibles} |
| …  | …                | …             | …        | …                                 |

   Types disponibles dans Brevo UI :
   Text · Number · Boolean · Date · URL

④ Associations :
   → Contact (obligatoire pour lier les records aux contacts Brevo)
   → {other_object} (si dépendance inter-objets)

⑤ Ordre de création : {object_name} avant {dependent_object} si applicable
   (l'objet dépendant nécessite que l'objet parent existe pour l'association)
```

**ID attribute rule**: Every object MUST have a `{obj}_id` attribute set as both the **Object ID** (identifier/deduplication key) AND added as a **regular attribute** (for UI visibility). Mark it `OUI ★ ID` in the table.

**Type mapping reference**:

| Brevo UI label | Use for |
|----------------|---------|
| Text | Names, codes, statuses, free text |
| Number | Integers and decimals (kWh, prices, counts) |
| Boolean | true/false flags |
| Date | Timestamps, registration dates, expiry dates |
| URL | Links |

### Step B — Wait for user confirmation + slug

After presenting all schemas, ask:

> *"Crée ces objets dans Brevo en suivant les schémas ci-dessus (Settings → Data Model → Custom Objects → + Create). Une fois terminé, confirme-moi en indiquant le nom exact de l'identifiant que tu as utilisé pour chaque objet — c'est ce slug qui sera utilisé dans les appels API."*
>
> *Exemple : "J'ai créé charging_stations avec l'identifiant station_id et vehicles avec vehicle_id."*

**This is the ONLY pause during Phase 2.** Required because object schemas cannot be created via API.

> **Critical**: The API slug (`{type}` in `POST /v3/objects/{type}/batch/upsert`) is the **identifier name** chosen at object creation — NOT the object display name. For example, if you named the identifier `station_id`, the URL must use `station_id` as the type. Never assume — always confirm with the user.

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
| `{type}` in URL | Must match the **identifier name** chosen at object creation (e.g., `station_id` not `charging_stations`). Always confirm with user after manual creation |
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
