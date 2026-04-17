# Create Custom Objects — Reference

Companion reference for the `create-custom-objects` skill. Contains association examples, attribute types, and industry suggestions.

## Full Upsert Example

```bash
curl -s -X POST "https://api.brevo.com/v3/objects/{object_type}/batch/upsert" \
  -H "api-key: $(cat /tmp/.brevo_key)" -H "content-type: application/json" \
  -d '{
    "records": [
      {
        "attributes": {
          "magasin_id": "MAG-001",
          "name": "Ixina Paris 15",
          "city": "Paris",
          "region": "Ile-de-France",
          "status": "active",
          "opening_date": "2020-03-15T10:00:00Z",
          "surface_m2": 450,
          "is_flagship": true,
          "services": ["Design 3D", "Installation", "SAV"],
          "contact_info": {
            "phone": "+33140123456",
            "email": "paris15@ixina.fr"
          }
        },
        "identifiers": {
          "ext_id": "MAG-001"
        },
        "associations": [
          {
            "object_type": "contact",
            "records": [
              {"identifiers": {"id": 12345}},
              {"identifiers": {"id": 12346}}
            ]
          }
        ]
      }
    ]
  }'
```

## Association Reference

**Associations are set inline during the upsert call** — not in a separate API call.

### Identifier rules by object type

| Object type | `object_type` value | Identifier to use |
|-------------|--------------------|--------------------|
| **Contact** | `"contact"` | `"id"` = `contact_id` (Brevo internal integer) |
| **Custom object** | `"{object_name}"` | `"id"` (Brevo internal) or `"ext_id"` (external) |

> **Scope**: This demo covers Marketing only (contacts, custom objects). CRM Sales objects (company, deal, task, note, file) are out of scope.

### 1. Associate with contacts

```json
{
  "records": [
    {
      "attributes": { "magasin_id": "MAG-001", "name": "Ixina Paris 15" },
      "identifiers": { "ext_id": "MAG-001" },
      "associations": [
        {
          "object_type": "contact",
          "records": [
            { "identifiers": { "id": 12345 } },
            { "identifiers": { "id": 67890 } }
          ]
        }
      ]
    }
  ]
}
```

- Use `"object_type": "contact"` — **never** `"contacts"`
- Use `contact_id` as `"id"` — from `created.contacts[].brevo_id`
- Contact **must already exist** in Brevo

### 2. Associate custom objects with each other

```json
{
  "records": [
    {
      "attributes": { "vendeur_id": "VEN-001", "name": "Marie Dupont" },
      "identifiers": { "ext_id": "VEN-001" },
      "associations": [
        {
          "object_type": "contact",
          "records": [
            { "identifiers": { "id": 12345 } }
          ]
        },
        {
          "object_type": "magasins",
          "records": [
            { "identifiers": { "id": 435435 } },
            { "identifiers": { "ext_id": "MAG-001" } }
          ]
        }
      ]
    }
  ]
}
```

- For custom objects: use either `"id"` (Brevo internal) or `"ext_id"` (external)
- **Both objects must already exist** before associating them
- The **association type must be defined** in the Brevo schema

### 3. Full example with multiple custom object associations

```json
{
  "records": [
    {
      "attributes": {
        "vendeur_id": "VEN-001",
        "name": "Marie Dupont",
        "performance": "A",
        "hire_date": "2022-01-15T00:00:00Z"
      },
      "identifiers": { "ext_id": "VEN-001" },
      "associations": [
        {
          "object_type": "contact",
          "records": [{ "identifiers": { "id": 12345 } }]
        },
        {
          "object_type": "magasins",
          "records": [
            { "identifiers": { "ext_id": "MAG-001" } },
            { "identifiers": { "ext_id": "MAG-002" } }
          ]
        }
      ]
    }
  ]
}
```

### Execution order for inter-object associations

When objects reference each other, **create them in dependency order**:
1. Upsert independent objects first (e.g. `magasins`) — store their `ext_id` in context
2. Upsert dependent objects with associations to contacts + previously created objects (e.g. `vendeurs` -> `contact` + `magasins`)

### Association error handling

- API returns error if the **referenced record doesn't exist**
- API returns error if the **association type is not defined** in the schema
- Other associations in the same request **can still succeed** (partial success)

## Supported Attribute Types

| Type | Example |
|------|---------|
| String | `"name": "Ixina Paris"` |
| Number (int) | `"surface_m2": 450` |
| Number (float) | `"price": 25000.50` |
| Boolean | `"is_flagship": true` |
| Date | `"opening_date": "2021-03-15T10:00:00Z"` |
| Array | `"services": ["Design 3D", "Installation"]` |
| Nested object | `"contact_info": {"phone": "...", "email": "..."}` |

### Date format (CRITICAL)

The time component is **mandatory** for custom object date attributes. `YYYY-MM-DD` alone is rejected.

| Format | Example | Valid |
|--------|---------|-------|
| `YYYY-MM-DDTHH:MM:SSZ` | `"2025-03-01T00:00:00Z"` | YES |
| `YYYY-MM-DDTHH:MM:SS.mmmZ` | `"2025-03-01T00:00:00.000Z"` | YES |
| `YYYY-MM-DD` (date only) | `"2025-03-01"` | **NO** |
| Unix timestamp | `1551398400000` | **NO** |

Always use full ISO 8601 with time: `YYYY-MM-DDTHH:MM:SSZ`

## Industry Suggestions

| Industry | Object | ID | Attributes |
|----------|--------|----|-----------|
| Franchise | magasins | magasin_id | name, city, region, status |
| Franchise | vendeurs | vendeur_id | name, store, hire_date, performance |
| SaaS | subscriptions | subscription_id | plan, mrr, renewal_date, status |
| Retail | loyalty_programs | loyalty_id | tier, points, member_since |
| Real Estate | properties | property_id | address, price, type, status |
