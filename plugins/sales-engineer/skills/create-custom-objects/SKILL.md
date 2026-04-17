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

**Language rule**: Use the same language as the user (French, English or German) for all labels in the guide (step titles, column headers, notes, instructions).

**French template:**
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

| #  | Attribute name   | Attribute type | Obligatoire | Notes / Valeurs possibles          |
|----|------------------|---------------|-------------|------------------------------------|
| 1  | {obj}_id         | Text          | OUI ★ ID    | Aussi Object ID — ex: "{OBJ}-001"  |
| 2  | {attr_1}         | {Type}        | {Oui/Non}   | {description — valeurs possibles}  |
| …  | …                | …             | …           | …                                  |

   Types disponibles dans Brevo UI : Text · Number · Boolean · Date · URL

④ Associations :
   → Contact (obligatoire pour lier les records aux contacts Brevo)
   → {other_object} (si dépendance inter-objets)

⑤ Ordre de création : {object_name} avant {dependent_object} si applicable
   (l'objet dépendant nécessite que l'objet parent existe pour l'association)
```

**English template:**
```
╔══════════════════════════════════════════════════════════╗
  OBJECT TO CREATE: {object_name}
  Path: Settings → Data Model → Custom Objects → + Create
╚══════════════════════════════════════════════════════════╝

① Object name: {object_name}

② Identifier (Object ID): {obj}_id
   → This is the API slug used in all subsequent API calls — choose carefully.
   → Also add {obj}_id as a regular attribute (for display in the Brevo UI).

③ Attributes to create:

| #  | Attribute name   | Attribute type | Required    | Notes / Possible values            |
|----|------------------|---------------|-------------|------------------------------------|
| 1  | {obj}_id         | Text          | YES ★ ID    | Also Object ID — e.g. "{OBJ}-001"  |
| 2  | {attr_1}         | {Type}        | {Yes/No}    | {description — possible values}    |
| …  | …                | …             | …           | …                                  |

   Available types in Brevo UI: Text · Number · Boolean · Date · URL

④ Associations:
   → Contact (required to link records to Brevo contacts)
   → {other_object} (if inter-object dependency exists)

⑤ Creation order: {object_name} before {dependent_object} if applicable
   (the dependent object requires the parent object to exist for the association)
```

**German template:**
```
╔══════════════════════════════════════════════════════════╗
  OBJEKT ERSTELLEN: {object_name}
  Pfad: Settings → Data Model → Custom Objects → + Create
╚══════════════════════════════════════════════════════════╝

① Objektname: {object_name}

② Bezeichner (Object ID): {obj}_id
   → Dies ist der API-Slug für alle nachfolgenden API-Aufrufe — sorgfältig wählen.
   → {obj}_id zusätzlich als reguläres Attribut hinzufügen (für die Anzeige in der Brevo UI).

③ Zu erstellende Attribute:

| #  | Attribute name   | Attribute type | Pflichtfeld | Hinweise / Mögliche Werte          |
|----|------------------|---------------|-------------|------------------------------------|
| 1  | {obj}_id         | Text          | JA ★ ID     | Auch Object ID — z.B. "{OBJ}-001"  |
| 2  | {attr_1}         | {Type}        | {Ja/Nein}   | {Beschreibung — mögliche Werte}    |
| …  | …                | …             | …           | …                                  |

   Verfügbare Typen in der Brevo UI: Text · Number · Boolean · Date · URL

④ Verknüpfungen (Associations):
   → Contact (erforderlich, um Datensätze mit Brevo-Kontakten zu verknüpfen)
   → {other_object} (falls abhängiges Objekt)

⑤ Erstellungsreihenfolge: {object_name} vor {dependent_object} (falls zutreffend)
   (das abhängige Objekt benötigt das übergeordnete Objekt für die Verknüpfung)
```

**ID attribute rule**: Every object MUST have a `{obj}_id` attribute set as both the **Object ID** (identifier/deduplication key) AND added as a **regular attribute** (for UI visibility). Mark it `OUI ★ ID` (FR) / `YES ★ ID` (EN) / `JA ★ ID` (DE) in the Required column.

**Type mapping reference**:

| Brevo UI label | Use for | Format / Example |
|----------------|---------|-----------------|
| Text | Names, codes, statuses, free text | `"Paris"` |
| Number | Integers and decimals (kWh, prices, counts) | `450` or `25000.50` |
| Boolean | true/false flags | `true` or `false` — JSON native, **no quotes** |
| Date | Timestamps, registration dates, expiry dates | `"2024-06-15T10:00:00Z"` — ISO 8601 with time **mandatory**. `"2024-06-15"` alone is **rejected** |
| URL | Links | `"https://example.com/page"` |

> ⚠️ **Date vs contacts**: custom object dates use ISO 8601 (`"YYYY-MM-DDTHH:MM:SSZ"`), while contact import dates use `MM/DD/YYYY`. These are different endpoints with different formats.

### Step B — Wait for user confirmation + slug

After presenting all schemas, ask in the user's language:

**French:**
> *"Crée ces objets dans Brevo en suivant les schémas ci-dessus (Settings → Data Model → Custom Objects → + Create). Une fois terminé, confirme-moi en indiquant le nom exact de l'identifiant que tu as utilisé pour chaque objet — c'est ce slug qui sera utilisé dans les appels API."*
>
> *Exemple : "J'ai créé charging_stations avec l'identifiant station_id et vehicles avec vehicle_id."*

**English:**
> *"Please create these objects in Brevo following the schemas above (Settings → Data Model → Custom Objects → + Create). Once done, confirm and tell me the exact identifier name you used for each object — that slug is what I'll use in all API calls."*
>
> *Example: "I created charging_stations with identifier station_id and vehicles with vehicle_id."*

**German:**
> *"Bitte erstelle diese Objekte in Brevo gemäß den obigen Schemata (Settings → Data Model → Custom Objects → + Create). Bestätige mir anschließend den genauen Bezeichner (Identifier), den du für jedes Objekt verwendet hast — dieser Slug wird in allen API-Aufrufen verwendet."*
>
> *Beispiel: „Ich habe charging_stations mit dem Bezeichner station_id und vehicles mit vehicle_id erstellt."*

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
