# CRM Demo — Reference

Companion reference for the `sales-engineer-crm-demo` agent. Contains the demo context JSON schema and dataset proposal template.

## Demo Context Schema (`demo-context.json`)

```json
{
  "meta": {
    "prospect_name": "",
    "created_at": "",
    "updated_at": "",
    "current_phase": "init",
    "brevo_api_key_set": false,
    "volumes": {
      "contacts": 100,
      "categories": 10,
      "products": 50,
      "orders": 200,
      "events_per_type_min": 20,
      "event_types_max": 5,
      "custom_objects_per_type": 50,
      "custom_object_types_max": 5
    }
  },
  "research": {},
  "existing_attributes": [],
  "plan": {
    "attributes": [], "contacts": [], "categories": [],
    "products": [], "orders": [],
    "events": { "event_types": [], "total_events_count": 0 },
    "event_funnels": {},
    "custom_objects": []
  },
  "created": {
    "attributes": [], "contact_list": { "name": "", "id": null },
    "contacts": [], "categories": [], "products": [],
    "orders": [], "events": [], "custom_objects": []
  },
  "errors": []
}
```

### Update protocol

After each skill: read context -> update section + `meta.updated_at` + `meta.current_phase` -> write back. Errors: append `{ "step", "message", "timestamp" }`.

### Cross-skill dependencies

`existing_attributes` -> `create-contact-attributes` | `created.contacts[].email` -> orders, events | `created.contacts[].brevo_id` -> custom objects | `created.categories[].id` -> products | `created.products[].id/.price` -> orders

## Dataset Proposal Template

```
DATASET PROPOSAL: {Company}

EXISTING ATTRIBUTES (reused): {table}
NEW ATTRIBUTES TO CREATE: {table} — Convention: {detected}
CONTACTS ({N}): {segment distribution %} — {sample 3-5 rows}
CATEGORIES ({N}): {table}
PRODUCTS ({N}): {sample 3-5 rows}
ORDERS ({N}): {status distribution %} — {sample 3-5 rows}
EVENTS ({N} types x min 20 contacts):
  Types:
  - page_view (required)
  - cart_updated (required)
  - {industry_event_3..5}
  Funnels per segment: {description}

  Payload example per event type:
  +-------------------------------------------------+
  | page_view                                       |
  | {                                               |
  |   "event_name": "page_view",                    |
  |   "event_date": "2025-...",                     |
  |   "identifiers": {"email_id": "..."},           |
  |   "event_properties": {                         |
  |     "page_url": "https://...",                  |
  |     "page_title": "...",                        |
  |     "device_type": "desktop"                    |
  |   }                                             |
  | }                                               |
  +-------------------------------------------------+
  | cart_updated                                    |
  | {                                               |
  |   "event_name": "cart_updated",                 |
  |   "identifiers": {"email_id": "..."},           |
  |   "event_properties": {                         |
  |     "cart_value": 299.99,                       |
  |     "product_id": "P001",                       |
  |     "quantity": 2,                              |
  |     "currency": "EUR"                           |
  |   }                                             |
  | }                                               |
  +-------------------------------------------------+

CUSTOM OBJECTS ({N} records x max 5 types — if applicable):
  Note: You will need to create these objects manually in Brevo UI

  For each object type, present full schema:
  +--------------------------------------------------+
  | OBJECT: {object_name}                             |
  |                                                   |
  | Attributes:                                       |
  | | Name     | Type   | Required | Notes     |      |
  | | {obj}_id | text   | YES      | Object ID |      |
  | | {attr_1} | {type} | {y/n}    | {desc}    |      |
  |                                                   |
  | Association: Contact                              |
  |                                                   |
  | Sample record:                                    |
  | {                                                 |
  |   "attributes": {                                 |
  |     "{obj}_id": "MAG-001",                        |
  |     "name": "...",                                |
  |     "status": "active"                            |
  |   },                                              |
  |   "associations": [{                              |
  |     "object_type": "contact",                     |
  |     "records": [{"identifiers": {"id": 12345}}]   |
  |   }]                                              |
  | }                                                 |
  +--------------------------------------------------+

TOTAL: {summary line}
```
