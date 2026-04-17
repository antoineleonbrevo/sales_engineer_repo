---
name: create-email-templates
description: "Create 3 branded email templates in Brevo for demos, using prospect brand colors and products. Activates on: create email templates, email templates, demo templates, branded emails."
tools: Bash, Read, Write, WebFetch, WebSearch
---

# Create Email Templates

Create 3 branded email templates via the Brevo API, personalized with the prospect's colors, products, and messaging.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `meta.prospect_name`, `research`, `created.products`, `created.contact_list` | **Write**: `created.email_templates`

## Templates to Create

Always create exactly **3 templates** in this order:

| # | Internal Name | Purpose | Target Segment |
|---|--------------|---------|---------------|
| 1 | `demo_{prospect}_welcome` | Welcome / onboarding new users | New contacts |
| 2 | `demo_{prospect}_promo` | Promotional — current offers & featured products | Active + VIP |
| 3 | `demo_{prospect}_reengagement` | Re-engagement — win back inactive contacts | At-risk |

Where `{prospect}` = `meta.prospect_name` lowercased, spaces replaced with `_` (e.g., `demo_zest_welcome`).

## Workflow

### Step 1 — Read context

Read `demo-context.json`:
- `meta.prospect_name` — company name
- `research.industry`, `research.products_services` — for content
- `research.markets` — for locale/currency
- `created.products` — top products to feature (use first 3 by price desc)
- `created.contact_list.name` — for subject line personalization reference

Also check if `meta.prospect_website` exists in the context. If not, ask the user:

> "Could you share the prospect's website URL? I need it to extract brand colors for the email templates."

### Step 2 — Extract brand colors

Fetch the prospect's website and extract primary brand colors:

```python
# Write to /tmp/extract_colors.py then run with python3
import urllib.request, re

url = "https://www.example.com"  # prospect website
req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
try:
    with urllib.request.urlopen(req, timeout=10) as r:
        html = r.read().decode("utf-8", errors="ignore")
except Exception as e:
    print("FETCH_FAILED:" + str(e))
    html = ""

# Extract hex colors from inline styles and CSS
colors = re.findall(r'#([0-9a-fA-F]{6})\b', html)
# Frequency count
from collections import Counter
freq = Counter(colors)
# Filter out pure black/white/gray
filtered = [(c, n) for c, n in freq.most_common(20)
            if c.lower() not in ("ffffff", "000000", "333333", "666666", "999999", "cccccc", "eeeeee", "f5f5f5", "e5e5e5")]
print("TOP_COLORS:" + str(filtered[:5]))
```

From the results, define:
- `PRIMARY_COLOR` — most dominant brand color (hex, e.g. `#1A73E8`)
- `SECONDARY_COLOR` — second color or a darker/lighter shade of primary
- `TEXT_COLOR` — dark color for body text (default `#333333` if none found)
- `BG_COLOR` — light background (default `#F8F9FA` if none found)

**Fallback**: If fetch fails or no colors found, use neutral corporate palette:
`PRIMARY=#2563EB`, `SECONDARY=#1E40AF`, `TEXT=#1F2937`, `BG=#F9FAFB`

### Step 3 — Retrieve a valid sender

```bash
curl -s "https://api.brevo.com/v3/senders" \
  -H "api-key: $(cat /tmp/.brevo_key)" | python3 -c "
import json, sys
data = json.load(sys.stdin)
senders = data.get('senders', [])
active = [s for s in senders if s.get('active', False)]
if active:
    s = active[0]
    print(s['email'] + '|' + s.get('name', s['email']) + '|' + str(s['id']))
elif senders:
    s = senders[0]
    print(s['email'] + '|' + s.get('name', s['email']) + '|' + str(s['id']))
else:
    print('NO_SENDER')
"
```

Use the first active sender found. Extract `sender_email`, `sender_name`, `sender_id`.
If `NO_SENDER` is returned, inform the user they need to configure a sender in Brevo first.

### Step 4 — Find and verify images

Each template requires **at least 1 image** (hero banner). Use Pexels — no API key needed.

#### Image assignment per template

| Template | Image type | Search keywords |
|----------|-----------|-----------------|
| Welcome | Hero: charger / charging station exterior | `pexels.com electric vehicle charging station` |
| Promo | Hero: charging hub / multiple EVs / infrastructure | `pexels.com EV charging hub multiple cars` |
| Re-engagement | Hero: driver + car / charging cable close-up / lifestyle | `pexels.com electric car charging cable close` |

#### Search & verify workflow

1. Use `WebSearch site:pexels.com {keywords}` to find candidate URLs
2. Extract the numeric photo ID from any Pexels URL (e.g. `pexels.com/photo/12345678` → ID `12345678`)
3. Verify each candidate with curl before using it:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" \
     "https://images.pexels.com/photos/{ID}/pexels-photo-{ID}.jpeg?auto=compress&cs=tinysrgb&w=600"
   ```
4. Only use IDs that return `200`. If a candidate returns 404, try the next one.

#### Image URL format

```
https://images.pexels.com/photos/{ID}/pexels-photo-{ID}.jpeg?auto=compress&cs=tinysrgb&w=600
```

Use `w=600` for email — keeps file size under 150 KB. The `?auto=compress` suffix is required (no bare extension-only URL).

#### Fallback

If 3 searches yield no valid ID → use a generic EV/tech image from the fallback list below and continue. Never block template creation on image search.

| Fallback | ID | Use for |
|----------|----|---------|
| EV charger close-up | `9800004` | Welcome |
| Multiple EVs charging | `9800029` | Promo |
| Charging cable plug-in | `10800215` | Re-engagement |

### Step 5 — Generate HTML templates

Build the HTML for each template using the extracted colors and prospect data.

#### Image block (insert at top of each template, after header)

```html
<!-- Hero image — required for every template -->
<tr>
  <td style="padding:0;">
    <img src="{{IMAGE_URL}}" alt="{{IMAGE_ALT}}" width="600"
         style="width:100%;max-width:600px;height:220px;object-fit:cover;display:block;" />
  </td>
</tr>
```

Use `height:220px; object-fit:cover` to ensure consistent rendering across email clients regardless of the image's original dimensions.

#### Standard multi-section structure (mandatory for all 3 templates)

All templates share the same shell. Only the content sections between header and footer change per template type. This ensures visual consistency and a premium feel across the demo.

```
[View online bar]           ← navy bg, centered link
[Meta row]                  ← newsletter label (left) + date (right), italic Georgia serif
[Logo header]               ← company name in Georgia serif + tagline
[Hero image]                ← full-width, height:260px, object-fit:cover
[Main block]                ← white bg, centered — headline (serif italic), body, primary CTA
[Section A]                 ← light bg — title uppercase + primary content (tiles or product cards)
[Section B]                 ← white bg — alternating 2-col layout (image/text or text/image)
[Section C]                 ← light bg — 3 numbered steps OR 2 info cards side by side
[Footer nav]                ← navy bg, 3 italic serif links (Solutions / Support / About)
[Footer legal]              ← copyright + unsubscribe
```

**Typography rules** (from Galimard template inspiration):
- Headings: Georgia serif, italic for main h1
- Labels/tags: Arial, uppercase, letter-spacing: 2px, small size (11px)
- Body: Arial, 13–14px, line-height 1.6–1.7
- Buttons: flat (no border-radius), bold Arial 12–14px

**Section backgrounds alternate**: white → light brand tint → white → light brand tint

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1.0">
</head>
<body style="margin:0;padding:0;background-color:#F0F4F8;font-family:Arial,Helvetica,sans-serif;">
<table width="100%" cellpadding="0" cellspacing="0" style="background-color:#F0F4F8;">
<tr><td align="center" style="padding:24px 16px;">
<table width="590" cellpadding="0" cellspacing="0" style="max-width:590px;width:100%;">

  <!-- VIEW IN BROWSER -->
  <tr>
    <td style="background-color:{{SECONDARY}};padding:8px 24px;text-align:center;">
      <a href="{{ mirror }}" style="color:#8B9BB4;font-family:Arial,sans-serif;font-size:11px;text-decoration:underline;">View online version</a>
    </td>
  </tr>

  <!-- META ROW -->
  <tr>
    <td style="background-color:{{SECONDARY}};padding:6px 24px 0;">
      <table width="100%" cellpadding="0" cellspacing="0"><tr>
        <td><span style="color:#8B9BB4;font-family:Georgia,serif;font-size:12px;font-style:italic;">{{META_LABEL}}</span></td>
        <td style="text-align:right;"><span style="color:#8B9BB4;font-family:Georgia,serif;font-size:12px;font-style:italic;">{{META_DATE}}</span></td>
      </tr></table>
    </td>
  </tr>

  <!-- LOGO HEADER -->
  <tr>
    <td style="background-color:{{SECONDARY}};padding:20px 24px;text-align:center;">
      <span style="color:#ffffff;font-family:Georgia,serif;font-size:28px;font-weight:700;">{{COMPANY_NAME}}</span>
      <span style="color:{{PRIMARY}};font-family:Georgia,serif;font-size:28px;font-weight:700;">.</span><br>
      <span style="color:#8B9BB4;font-family:Arial,sans-serif;font-size:11px;letter-spacing:2px;text-transform:uppercase;">{{TAGLINE}}</span>
    </td>
  </tr>

  <!-- HERO IMAGE -->
  <tr>
    <td><img src="{{IMAGE_URL}}" alt="{{IMAGE_ALT}}" width="590"
         style="width:100%;max-width:590px;height:260px;object-fit:cover;display:block;" /></td>
  </tr>

  <!-- CONTENT SECTIONS (swap per template type) -->
  {{CONTENT_SECTIONS}}

  <!-- FOOTER NAV -->
  <tr>
    <td style="background-color:{{SECONDARY}};padding:20px 40px 8px;text-align:center;">
      <table width="100%" cellpadding="0" cellspacing="0"><tr>
        <td style="text-align:center;"><a href="{{WEBSITE}}" style="color:#8B9BB4;font-family:Georgia,serif;font-size:11px;font-style:italic;text-decoration:none;">Our Solutions</a></td>
        <td style="text-align:center;"><a href="{{WEBSITE}}/support" style="color:#8B9BB4;font-family:Georgia,serif;font-size:11px;font-style:italic;text-decoration:none;">Support</a></td>
        <td style="text-align:center;"><a href="{{WEBSITE}}/about" style="color:#8B9BB4;font-family:Georgia,serif;font-size:11px;font-style:italic;text-decoration:none;">About</a></td>
      </tr></table>
    </td>
  </tr>
  <tr>
    <td style="background-color:{{SECONDARY}};padding:12px 40px 24px;text-align:center;">
      <p style="margin:0 0 6px;color:#8B9BB4;font-family:Arial,sans-serif;font-size:11px;">&copy; {{YEAR}} {{COMPANY_NAME}}</p>
      <p style="margin:0;"><a href="{{unsubscribe}}" style="color:#8B9BB4;font-family:Arial,sans-serif;font-size:11px;text-decoration:underline;">Unsubscribe</a></p>
    </td>
  </tr>

</table>
</td></tr>
</table>
</body>
</html>
```

#### Content blocks per template

**Template 1 — Welcome**
```html
<tr>
  <td style="padding:40px 40px 24px;">
    <h2 style="margin:0 0 16px;color:{{PRIMARY_COLOR}};font-size:22px;">Welcome to {{COMPANY_NAME}}!</h2>
    <p style="margin:0 0 16px;color:{{TEXT_COLOR}};font-size:15px;line-height:1.6;">
      Hi {{contact.FIRSTNAME}},<br><br>
      We're thrilled to have you on board. {{WELCOME_BODY}}
    </p>
    <table cellpadding="0" cellspacing="0"><tr>
      <td style="background-color:{{PRIMARY_COLOR}};border-radius:6px;padding:14px 28px;">
        <a href="{{PROSPECT_WEBSITE}}" style="color:#ffffff;font-size:15px;font-weight:600;text-decoration:none;">Get Started →</a>
      </td>
    </tr></table>
  </td>
</tr>
<tr>
  <td style="padding:0 40px 40px;">
    <hr style="border:none;border-top:1px solid #eeeeee;margin:0 0 24px;">
    <h3 style="margin:0 0 8px;color:{{TEXT_COLOR}};font-size:14px;font-weight:600;text-transform:uppercase;letter-spacing:1px;">What's included</h3>
    <ul style="margin:8px 0;padding-left:20px;color:{{TEXT_COLOR}};font-size:14px;line-height:1.8;">
      {{WELCOME_FEATURES}}
    </ul>
  </td>
</tr>
```

**Template 2 — Promotional**
```html
<tr>
  <td style="padding:40px 40px 24px;">
    <p style="margin:0 0 4px;color:{{PRIMARY_COLOR}};font-size:12px;font-weight:700;text-transform:uppercase;letter-spacing:1.5px;">{{PROMO_TAG}}</p>
    <h2 style="margin:0 0 16px;color:{{TEXT_COLOR}};font-size:22px;">{{PROMO_HEADLINE}}</h2>
    <p style="margin:0 0 24px;color:{{TEXT_COLOR}};font-size:15px;line-height:1.6;">{{PROMO_BODY}}</p>
  </td>
</tr>
<tr>
  <td style="padding:0 40px 32px;">
    {{PRODUCT_CARDS}}
  </td>
</tr>
<tr>
  <td style="padding:0 40px 40px;text-align:center;">
    <table cellpadding="0" cellspacing="0" style="margin:0 auto;"><tr>
      <td style="background-color:{{PRIMARY_COLOR}};border-radius:6px;padding:14px 32px;">
        <a href="{{PROSPECT_WEBSITE}}" style="color:#ffffff;font-size:15px;font-weight:600;text-decoration:none;">Shop Now →</a>
      </td>
    </tr></table>
  </td>
</tr>
```

**Template 3 — Re-engagement**
```html
<tr>
  <td style="padding:40px 40px 24px;text-align:center;">
    <p style="margin:0 0 8px;font-size:40px;">👋</p>
    <h2 style="margin:0 0 16px;color:{{PRIMARY_COLOR}};font-size:22px;">We miss you, {{contact.FIRSTNAME}}!</h2>
    <p style="margin:0 0 24px;color:{{TEXT_COLOR}};font-size:15px;line-height:1.6;">
      {{REENGAGEMENT_BODY}}
    </p>
    <table cellpadding="0" cellspacing="0" style="margin:0 auto 24px;"><tr>
      <td style="background-color:{{PRIMARY_COLOR}};border-radius:6px;padding:14px 32px;">
        <a href="{{PROSPECT_WEBSITE}}" style="color:#ffffff;font-size:15px;font-weight:600;text-decoration:none;">{{REENGAGEMENT_CTA}}</a>
      </td>
    </tr></table>
    <p style="margin:0;color:#888888;font-size:13px;">{{REENGAGEMENT_URGENCY}}</p>
  </td>
</tr>
```

#### Populating template variables from research

| Variable | Source |
|----------|--------|
| `{{COMPANY_NAME}}` | `meta.prospect_name` |
| `{{PROSPECT_WEBSITE}}` | prospect website URL |
| `{{YEAR}}` | current year |
| `{{WELCOME_BODY}}` | 1–2 sentences about the company's value proposition from `research` |
| `{{WELCOME_FEATURES}}` | 3–4 `<li>` bullets from top `research.products_services` names |
| `{{PROMO_TAG}}` | e.g. "Limited Offer", "New Arrivals", "Exclusive Deal" — based on industry |
| `{{PROMO_HEADLINE}}` | Catchy headline using top product name or current offer |
| `{{PROMO_BODY}}` | 1–2 sentences referencing a real product or promotion |
| `{{PRODUCT_CARDS}}` | HTML cards for top 3 products (see Product Cards format below) |
| `{{REENGAGEMENT_BODY}}` | 2 sentences referencing what the contact might be missing |
| `{{REENGAGEMENT_CTA}}` | Action button text, e.g. "Come Back & Save", "Resume Charging" |
| `{{REENGAGEMENT_URGENCY}}` | Urgency line, e.g. "Offer valid for 7 days only." |

#### Product cards format (Template 2)

For each of the top 3 products (by price desc from `created.products`):
```html
<table width="100%" cellpadding="0" cellspacing="0" style="margin-bottom:16px;border:1px solid #eeeeee;border-radius:6px;overflow:hidden;">
  <tr>
    <td style="padding:16px;">
      <strong style="color:{{TEXT_COLOR}};font-size:14px;">{{PRODUCT_NAME}}</strong><br>
      <span style="color:{{PRIMARY_COLOR}};font-size:18px;font-weight:700;">{{PRODUCT_PRICE}}</span>
    </td>
  </tr>
</table>
```

### Step 5 — CHECKPOINT: Preview & user validation

**Before any API call**, present a preview of all 3 templates to the user for approval.

Extract the preview data from the generated HTML using a Python script:

```python
# Write to /tmp/preview_templates.py then run with python3
import re

def strip_html(html):
    """Remove all HTML tags and collapse whitespace."""
    text = re.sub(r'<[^>]+>', ' ', html)
    text = re.sub(r'&nbsp;', ' ', text)
    text = re.sub(r'\s+', ' ', text).strip()
    return text

def extract_urls(html):
    """Extract all href URLs from the HTML."""
    return list(dict.fromkeys(re.findall(r'href=["\']([^"\']+)["\']', html)))

# templates = list of dicts with keys: name, subject, html, sender_email, sender_name, tag
for t in templates:
    body_preview = strip_html(t["html"])[:220]
    urls = extract_urls(t["html"])
    print("---")
    print("NAME:    " + t["name"])
    print("SUBJECT: " + t["subject"])
    print("SENDER:  " + t["sender_name"] + " <" + t["sender_email"] + ">")
    print("PREVIEW: " + body_preview + "...")
    print("URLS:    " + str(urls))
```

Present the output as a formatted table/summary to the user:

```
┌─────────────────────────────────────────────────────────────────┐
│  TEMPLATE PREVIEW — please review before creation               │
├─────────────────────────────────────────────────────────────────┤
│ 🎨 Brand colors: PRIMARY=#1A73E8  SECONDARY=#1558B0             │
│    Sender: Zest <hello@zest.co.uk>                              │
├────┬──────────────────────────────┬───────────────────────────┤
│ #  │ Template                     │ Details                   │
├────┼──────────────────────────────┼───────────────────────────┤
│ 1  │ demo_zest_welcome            │                           │
│    │ Subject: Welcome to Zest...  │                           │
│    │ Preview: Hi {{contact.FIRST- │                           │
│    │ NAME}}, We're thrilled to... │                           │
│    │ CTA URL: https://zest.co.uk  │                           │
├────┼──────────────────────────────┼───────────────────────────┤
│ 2  │ demo_zest_promo              │                           │
│    │ Subject: Charge smarter...   │                           │
│    │ Preview: Exclusive deals...  │                           │
│    │ Products: Zest Ultra 300kW,  │                           │
│    │   Zest Destination 50kW,     │                           │
│    │   Fleet Depot 22kW           │                           │
│    │ CTA URL: https://zest.co.uk  │                           │
├────┼──────────────────────────────┼───────────────────────────┤
│ 3  │ demo_zest_reengagement       │                           │
│    │ Subject: We miss you 👋      │                           │
│    │ Preview: We miss you,        │                           │
│    │   {{contact.FIRSTNAME}}!...  │                           │
│    │ CTA URL: https://zest.co.uk  │                           │
│    │ Urgency: Offer valid 7 days  │                           │
└────┴──────────────────────────────┴───────────────────────────┘
```

Then ask:

> "Do these templates look good? You can ask me to change the subject lines, body text, colors, CTAs, or featured products before I create them in Brevo."

**Wait for explicit approval** before proceeding to Step 6. Accept changes freely — update the HTML in memory (no API call yet) and re-display the preview after each revision.

Only proceed when the user confirms (e.g. "go ahead", "looks good", "create them").

### Step 6 — Create templates via API (POST)

Write a Python script to `/tmp/create_email_templates.py` and execute it:

```python
import urllib.request, urllib.error, json

api_key = open("/tmp/.brevo_key").read().strip()

# Fill these from Steps 1-4
prospect = "PROSPECT_NAME"
sender_email = "sender@example.com"
sender_name = "Sender Name"

templates = [
    {
        "templateName": "demo_{}_welcome".format(prospect.lower().replace(" ", "_")),
        "subject": "Welcome to {} — Let's get started!".format(prospect),
        "htmlContent": "WELCOME_HTML",  # full HTML string
        "isActive": True,
        "tag": "demo",
        "sender": {"email": sender_email, "name": sender_name},
        "replyTo": sender_email,
        "toField": "{{contact.FIRSTNAME}} {{contact.LASTNAME}}"
    },
    {
        "templateName": "demo_{}_promo".format(prospect.lower().replace(" ", "_")),
        "subject": "PROMO_SUBJECT",
        "htmlContent": "PROMO_HTML",
        "isActive": True,
        "tag": "demo",
        "sender": {"email": sender_email, "name": sender_name},
        "replyTo": sender_email,
        "toField": "{{contact.FIRSTNAME}} {{contact.LASTNAME}}"
    },
    {
        "templateName": "demo_{}_reengagement".format(prospect.lower().replace(" ", "_")),
        "subject": "We miss you, {{contact.FIRSTNAME}} 👋",
        "htmlContent": "REENGAGEMENT_HTML",
        "isActive": True,
        "tag": "demo",
        "sender": {"email": sender_email, "name": sender_name},
        "replyTo": sender_email,
        "toField": "{{contact.FIRSTNAME}} {{contact.LASTNAME}}"
    }
]

created = []
for t in templates:
    data = json.dumps(t).encode()
    req = urllib.request.Request(
        "https://api.brevo.com/v3/smtp/templates",
        data=data,
        headers={
            "api-key": api_key,
            "content-type": "application/json",
            "accept": "application/json"
        }
    )
    try:
        with urllib.request.urlopen(req) as resp:
            body = json.loads(resp.read().decode())
            template_id = body.get("id")
            print("CREATED {} -> id={}".format(t["templateName"], template_id))
            created.append({"name": t["templateName"], "subject": t["subject"], "id": template_id, "isActive": True})
    except urllib.error.HTTPError as e:
        err = e.read().decode()
        print("ERROR {}: {} {}".format(t["templateName"], e.code, err))
        created.append({"name": t["templateName"], "subject": t["subject"], "id": None, "error": err})

print("RESULT:" + json.dumps(created))
```

### Step 7 — Update context

Write to `demo-context.json → created.email_templates`:

```json
{
  "email_templates": [
    {"name": "demo_zest_welcome",       "id": 12, "subject": "Welcome to Zest — Let's get started!", "isActive": true},
    {"name": "demo_zest_promo",         "id": 13, "subject": "Charge smarter — Exclusive deals inside", "isActive": true},
    {"name": "demo_zest_reengagement",  "id": 14, "subject": "We miss you, {{contact.FIRSTNAME}} 👋",   "isActive": true}
  ]
}
```

## Post-creation Modifications

If the user requests changes **after** templates have been created (IDs exist in `created.email_templates`), use `PUT` instead of recreating:

```python
# Write to /tmp/update_email_template.py then run with python3
import urllib.request, urllib.error, json

api_key = open("/tmp/.brevo_key").read().strip()

template_id = 12  # from created.email_templates
updates = {
    # Only include fields that changed — all fields are optional on PUT
    "subject": "NEW SUBJECT",
    "htmlContent": "NEW HTML",
    # "templateName": "new_name",
    # "isActive": True,
    # "sender": {"email": "sender@example.com", "name": "Sender Name"},
    # "tag": "demo",
    # "toField": "{{contact.FIRSTNAME}} {{contact.LASTNAME}}"
}

data = json.dumps(updates).encode()
req = urllib.request.Request(
    "https://api.brevo.com/v3/smtp/templates/{}".format(template_id),
    data=data,
    headers={
        "api-key": api_key,
        "content-type": "application/json",
        "accept": "application/json"
    },
    method="PUT"
)
try:
    with urllib.request.urlopen(req) as resp:
        print("UPDATED template {} -> {}".format(template_id, resp.status))
except urllib.error.HTTPError as e:
    print("ERROR {}: {}".format(e.code, e.read().decode()))
```

### Update endpoint reference

| Detail | Value |
|--------|-------|
| Method | `PUT` |
| URL | `https://api.brevo.com/v3/smtp/templates/{templateId}` |
| `templateId` | Numeric ID from `created.email_templates[].id` |
| Response `200` | Success — no body returned |
| Response `400` | Validation error |
| Response `404` | Template not found — check the ID |

**All fields are optional on PUT** — only send what changed. The existing template content is preserved for fields not included.

## Pre-call Validation

Before `POST /v3/smtp/templates`:

| Check | Rule |
|-------|------|
| `templateName` | **Required** — unique string, no special characters other than `_` and `-`. Use `demo_{prospect}_{type}` format |
| `subject` | **Required** — non-empty string |
| `sender` | **Required** — object with `email` (string) OR `id` (int). **Never both**. Use `email` from `GET /v3/senders` |
| `htmlContent` | **Required** if `htmlUrl` is empty — minimum **10 characters**. Must be valid HTML with at least 1 `<img>` hero per template |
| Image URLs | Every `<img src="...">` in `htmlContent` must return HTTP 200 — verify with curl before generating HTML |
| `htmlUrl` | **Required** if `htmlContent` is empty — must be a valid absolute URL |
| `isActive` | Set to `true` — templates are inactive by default and won't appear in campaign builder otherwise |
| `toField` | Optional — use `{{contact.FIRSTNAME}} {{contact.LASTNAME}}` for personalization |
| `replyTo` | Optional — must be a valid email address if provided |
| `tag` | Optional — use `"demo"` for easy filtering in Brevo UI |

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `400 templateName already exists` | Template name taken | Append `_v2` to the name |
| `400 sender not found` | Sender email not in Brevo account | Use `GET /v3/senders` to get a valid sender |
| `400 htmlContent too short` | HTML under 10 chars | Ensure full HTML template is passed |
| `400 invalid_parameter` | Malformed JSON or missing required field | Check `templateName`, `subject`, `sender`, `htmlContent` are all present |
| `401` | Invalid or missing API key | Check `/tmp/.brevo_key` |

## Response Codes

| Code | Description |
|------|-------------|
| `201` | Template created — response body contains `{"id": int}` |
| `400` | Validation error — check `code` and `message` fields in response |
| `401` | Unauthorized — invalid API key |

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues with HTML content |
| Escape HTML in JSON payload using `json.dumps()` | Never hand-build JSON containing raw HTML — double-quote escaping will break |
| Never inline HTML strings directly in bash heredocs | Shell will corrupt quotes inside HTML — always use Python json.dumps() |

## Reference

- **Endpoint**: `POST /v3/smtp/templates`
- **Get senders**: `GET /v3/senders` — use first active sender
- `isActive: true` — mandatory for templates to appear in the campaign builder
- `templateName` — must be unique per account; use `demo_{prospect}_{type}` convention
- `sender` — use `email` **or** `id`, never both
- `htmlContent` — min 10 chars; pass full HTML via Python `json.dumps()` to avoid escaping issues
- Template IDs are written to `created.email_templates` in `demo-context.json`
