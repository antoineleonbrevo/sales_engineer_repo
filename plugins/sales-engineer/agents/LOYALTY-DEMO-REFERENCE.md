# Loyalty Demo — Reference

Companion reference for the `sales-engineer-loyalty-demo` agent. Contains the demo context JSON schema, API quick reference, proposal templates, anti-patterns, and checklist.

## Demo Context Schema (`loyalty-demo-<prospect_slug>.json`)

```json
{
  "meta": {
    "prospect_name": "",
    "prospect_slug": "",
    "created_at": "",
    "updated_at": "",
    "current_phase": "init",
    "brevo_api_key_set": false,
    "existing_env": false,
    "language": "fr",
    "volumes": {
      "tiers": 3,
      "rewards": 6,
      "subscribed_contacts": 20,
      "transactions": 50
    },
    "use_cases": []
  },
  "research": {},
  "program": {
    "id": null,
    "name": "",
    "status": null
  },
  "balanceDefinitions": [],
  "tierGroups": [],
  "contacts": [],
  "rewards": [],
  "transactions": [],
  "events": [],
  "progress": {
    "program_created": false,
    "balance_created": false,
    "tier_group_created": false,
    "first_publish_done": false,
    "tiers_created": false,
    "rewards_created": false,
    "republish_done": false,
    "subscriptions_created": false,
    "transactions_created": false,
    "events_created": false
  },
  "errors": []
}
```

### Update protocol

After each skill: read context -> update relevant section + `meta.updated_at` + `meta.current_phase` + `progress.<step>: true` -> write back. Errors: append `{ "step", "message", "timestamp" }`.

## Session Resume Dialog Template

```
J'ai trouvé un contexte existant pour ce prospect :
  Fichier : /tmp/loyalty-demo-<slug>.json
  Créé le : {created_at}
  Phase actuelle : {current_phase}
  Étapes complétées : {list completed progress steps}

Que veux-tu faire ?
  1. Reprendre depuis {next_step}
  2. Repartir de zéro (supprime le contexte existant)
  3. Changer de prospect
```

## Loyalty Module Activation Steps

Before starting, the Loyalty module must be activated on the Brevo sub-account:

1. Log into Brevo as **admin**
2. Go to **Settings → Subscriptions**
3. Enable the **Loyalty** module on the target sub-account
4. Confirm activation — the `/v3/loyalty/` endpoints will return `403` until this is done

> If the prospect's account already has Loyalty active, skip this step.

## Progress Tracking Table

Update `progress` in context after each execution step:

| Step | Field | Set when |
|------|-------|----------|
| 5 | `program_created` | Program POST returns `id` |
| 6 | `balance_created` | Balance POST returns `id` |
| 7 | `tier_group_created` | Tier group POST returns `id` |
| 7 | `first_publish_done` | First publish returns `200` |
| 8 | `tiers_created` | All individual tiers created |
| 9 | `rewards_created` | All rewards created with images |
| 10 | `republish_done` | Re-publish returns `200` |
| 11 | `subscriptions_created` | All contacts subscribed |
| 12 | `transactions_created` | All transactions completed |
| 13 | `events_created` | All social events sent |

## API Quick Reference

| Operation | Method | Path |
|-----------|--------|------|
| Create program | POST | `/v3/loyalty/config/programs` |
| Publish program | POST | `/v3/loyalty/config/programs/{pid}/publish` |
| Create balance | POST | `/v3/loyalty/balance/programs/{pid}/balance-definitions` |
| Create tier group | POST | `/v3/loyalty/tier/programs/{pid}/tier-groups` |
| Create tier | POST | `/v3/loyalty/tier/programs/{pid}/tier-groups/{gid}/tiers` |
| Update tier | PUT | `/v3/loyalty/tier/programs/{pid}/tiers/{tid}` |
| Create reward | POST | `/v3/loyalty/offer/programs/{pid}/offers` |
| Subscribe contact | POST | `/v3/loyalty/config/programs/{pid}/members` |
| Create transaction | POST | `/v3/loyalty/balance/programs/{pid}/transactions` |
| Create balance order | POST | `/v3/loyalty/balance/programs/{pid}/create-order` |
| Complete transaction | POST | `/v3/loyalty/balance/programs/{pid}/transactions/{tid}/complete` |
| Upload image | POST | `/v3/emailCampaigns/images` |

## Temp Files

| File | Content |
|------|---------|
| `/tmp/.brevo_key` | API key |
| `/tmp/.brevo_loyalty_pid` | Program ID (UUID) |
| `/tmp/.brevo_loyalty_balance_id` | Balance definition ID (UUID) |
| `/tmp/.brevo_loyalty_gid` | Tier group ID (UUID) |

## Anti-patterns

| Mistake | Effect | Fix |
|---------|--------|-----|
| Creating tiers before first publish | `400` or tiers silently ignored | Always publish once before creating tiers |
| Passing `"tiers": [...]` inline in tier group payload | Silently ignored | Create tiers individually via separate endpoint |
| Subscribing contacts before publish | `422` — program not active | Publish before subscribing |
| Using UPDATE path with `{gid}` for tier update | `404` | Update tier path has NO `{gid}`: `PUT .../tiers/{tid}` |
| Direct external image URL in `publicImage` | `422` — image format invalid | Upload to Brevo first via `/emailCampaigns/images`, use returned URL |
| Bash curl loop with UUID variables | JSON parsing errors | Always use Python for transaction/event loops |
| `amount: 0` in transaction | `422` | Amount must be non-zero |
| Transaction on unsubscribed contact | `422` | Subscribe contacts before creating transactions |

## Pre-flight Checklist

Before starting Phase 3:

- [ ] API key validated (`GET /v3/account` returns 200)
- [ ] Loyalty module active on sub-account
- [ ] `prospect_slug` computed and context file initialized
- [ ] Use cases confirmed and program design validated by user
- [ ] Contacts available (existing or to be created via CRM demo)
- [ ] `/tmp/.brevo_key` written
