# Sales Engineer

Prepare custom Brevo demos for prospects by generating realistic data (contacts, products, orders, events) via the Brevo API.

## Agent

| Agent | Description |
|-------|-------------|
| `sales-engineer-crm-demo` | Build complete custom Brevo demo environments for prospects. Researches the prospect, proposes a dataset for validation, then creates all demo data via the Brevo API. |

## Skills

| Skill | Description |
|-------|-------------|
| `research-prospect` | Research a prospect company to build a contextual demo profile. |
| `create-contact-attributes` | Create custom contact attributes in Brevo via API. |
| `create-contacts` | Create demo contacts with custom attributes via API. |
| `create-product-categories` | Activate ecommerce and create product categories via API. |
| `create-products` | Create products with royalty-free images via API. |
| `create-orders` | Create orders linking contacts to products via API. |
| `create-events` | Create tracking events associated with demo contacts via API. |
| `create-custom-objects` | Create custom CRM objects and records via API. |
| `create-email-templates` | Create 3 branded email templates using prospect colors and products via API. |

## Workflow

```
Phase 1 — Research & Validate (interactive)
  1. Collect inputs (prospect name, API key, volume overrides)
  2. Research prospect (max 10 targeted web searches)
  3. CHECKPOINT 1: Validate research findings
  4. Fetch existing attributes + propose dataset
  5. CHECKPOINT 2: Validate dataset proposal

Phase 2 — Execute (autonomous, no pauses except custom object schemas)
  6. Create attributes → 7. Import contacts → 8. Collect Brevo IDs
  9. Create categories → 10. Create products
  11. Create orders → 12. Create events
  13. Create custom objects (manual schema creation + upsert)
  Each step verifies record counts vs expected volumes.

Phase 3 — Summary
  14. Report with counts, errors, talking points, automation suggestions
```

## Prerequisites

- A Brevo account with API access
- API key provided interactively at runtime (stored securely in `/tmp/.brevo_key`, never logged)
- Enterprise/Pro plan for custom objects (optional)

## Installation

This plugin is available via the collective-intelligence repository. Clone or pull the repo, then start the agent:

```bash
git clone <repo-url>  # or git pull if already cloned
# Then in Claude Code, say: "sales demo" or "prepare a demo for [prospect]"
```
