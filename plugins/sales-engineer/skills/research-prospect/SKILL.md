---
name: research-prospect
description: "Research a prospect company for demo preparation. Gathers industry, products, audience from web. Activates on: research prospect, company research, demo preparation."
tools: WebSearch, WebFetch, Read, Write
---

# Research Prospect

Research prospect company to build a demo profile.

## Demo Context

- **Read**: `meta.prospect_name` | **Write**: `research` (all fields)

## Disambiguation (before searching)

If the prospect name is a **common word or shared by multiple companies** (e.g., "Zest", "Arc", "Bloom"), do NOT start searching blindly. First ask:

> "Could you share the prospect's website URL? This avoids targeting the wrong company if the name is shared by multiple businesses."

If a URL is provided, use `WebFetch` on that URL as search #1 — skip the generic `"{company}"` query.

## Search Strategy

**Max 10 targeted web searches** — no full-site scraping. Focus queries to extract essentials:

| # | Query pattern | Goal |
|---|--------------|-------|
| 1 | `"{company}"` (or WebFetch on URL if provided) | Official site, about page |
| 2 | `"{company}" products OR services pricing` | Catalog + price ranges |
| 3 | `"{company}" customers OR clients target` | Audience + personas |
| 4 | `"{company}" industry sector B2B OR B2C` | Business model |
| 5 | `"{company}" size employees revenue` | Company scale |
| 6-10 | Adaptive | Fill gaps: specific product lines, markets, competitors, news |

Stop early if all post-conditions are met before 10 searches. Use `WebFetch` only on high-value pages (official site, product page) — skip generic directories.

## Workflow

1. Read `demo-context.json` for prospect name and known info
2. If name is ambiguous, ask for URL before searching (see Disambiguation above)
3. Run targeted searches (max 10, see strategy above)
3. Extract: industry, business model (B2B/B2C), products/services with prices, target audience personas, company size, markets
5. Update `demo-context.json → research`:
   ```json
   { "industry": "", "business_model": "", "company_size": "", "markets": [],
     "products_services": [{"name": "", "description": "", "price_range": ""}],
     "target_audience": [{"persona": "", "description": ""}],
     "image_keywords": [], "custom_object_recommendations": [] }
   ```
6. Present findings to user

## Post-conditions

- Industry + business model identified
- 5+ products/services with price ranges
- 2+ audience personas
- Image keywords for product photos
- Context updated
