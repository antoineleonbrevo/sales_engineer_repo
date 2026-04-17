---
name: create-tier-group
description: "Create a tier group and individual tiers for a Brevo Loyalty program. Activates on: create tiers, tier group, loyalty tiers, niveaux fidélité."
tools: Bash, Read, Write
---

# Create Tier Group

Create a tier group, then add individual tiers one by one, each linked to a balance definition.

> **Language**: Communicate with the user in the same language they use — French, English or German.

## Demo Context

- **Read**: `program.id`, `balanceDefinitions[0].id`, `tierGroups` (planned) | **Write**: `tierGroups` (with `id`, `tiers[].id`)

## Important: Tiers Are Created Individually

The `tiers` field is **NOT supported inline** in the tier group creation payload — passing `"tiers": [...]` is silently ignored or rejected. Tiers must be created one by one via a separate endpoint after the tier group exists.

## Workflow

1. Read `loyalty-demo-context.json` for `program.id`, `balanceDefinitionId`, and planned tier structure
2. **The program must already be published once** before creating tiers (publish happens in `create-loyalty-program` skill)
3. Create the tier group:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   PID=$(cat /tmp/.brevo_loyalty_pid)
   curl -s -X POST "https://api.brevo.com/v3/loyalty/tier/programs/$PID/tier-groups" \
     -H "api-key: $(cat /tmp/.brevo_key)" \
     -H "content-type: application/json" \
     -d '{
       "name": "<tier_group_name>",
       "upgradeEvaluationTrigger": "real_time",
       "downgradeEvaluationTrigger": "membership_anniversary"
     }'
   SCRIPT
   bash /tmp/script.sh
   ```
4. Store `gid` (tier group `id`) in `/tmp/.brevo_loyalty_gid`
5. Create each tier individually (one call per tier, lowest threshold first):
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   PID=$(cat /tmp/.brevo_loyalty_pid)
   GID=$(cat /tmp/.brevo_loyalty_gid)
   BALANCE_ID=$(cat /tmp/.brevo_loyalty_balance_id)
   curl -s -X POST "https://api.brevo.com/v3/loyalty/tier/programs/$PID/tier-groups/$GID/tiers" \
     -H "api-key: $(cat /tmp/.brevo_key)" \
     -H "content-type: application/json" \
     -d '{
       "name": "<tier_name>",
       "accessConditions": [
         {
           "balanceDefinitionId": "'$BALANCE_ID'",
           "minimumValue": <threshold>
         }
       ]
     }'
   SCRIPT
   bash /tmp/script.sh
   ```
   Repeat for each tier (e.g., Bronze=0, Silver=300, Gold=500). Store each tier's returned `id`.
6. Optionally update a tier (add image, rewards) — note: **update path does NOT include `{gid}`**:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   PID=$(cat /tmp/.brevo_loyalty_pid)
   TID="<tier_id>"
   curl -s -X PUT "https://api.brevo.com/v3/loyalty/tier/programs/$PID/tiers/$TID" \
     -H "api-key: $(cat /tmp/.brevo_key)" \
     -H "content-type: application/json" \
     -d '{
       "name": "<tier_name>",
       "imageRef": "<brevo_image_url>",
       "accessConditions": [
         {
           "balanceDefinitionId": "'$(cat /tmp/.brevo_loyalty_balance_id)'",
           "minimumValue": <threshold>
         }
       ]
     }'
   SCRIPT
   bash /tmp/script.sh
   ```
7. Re-publish the program to activate tier changes:
   ```bash
   cat > /tmp/script.sh << 'SCRIPT'
   PID=$(cat /tmp/.brevo_loyalty_pid)
   curl -s -X POST "https://api.brevo.com/v3/loyalty/config/programs/$PID/publish" \
     -H "api-key: $(cat /tmp/.brevo_key)"
   SCRIPT
   bash /tmp/script.sh
   ```
8. Update `loyalty-demo-context.json` with tier group and tier IDs

## Pre-call Validation

### Tier Group

| Check | Rule |
|-------|------|
| `name` | **Required** — internal name for the tier group |
| `upgradeEvaluationTrigger` | **Required** — `"real_time"`, `"membership_anniversary"`, or `"tier_anniversary"` |
| `downgradeEvaluationTrigger` | **Required** — same options |
| `pid` | Must exist in `/tmp/.brevo_loyalty_pid` |
| Program state | Must be published at least once before creating tiers |

### Individual Tier

| Check | Rule |
|-------|------|
| `name` | **Required** — tier name (e.g., "Bronze") |
| `accessConditions` | **Required** — array with at least one condition |
| `accessConditions[].balanceDefinitionId` | **Required** — UUID from balance definition |
| `accessConditions[].minimumValue` | **Required** — threshold (0 for base tier). Must be unique across tiers |
| `gid` | Must exist in `/tmp/.brevo_loyalty_gid` |
| `balanceDefinitionId` | Must exist in `/tmp/.brevo_loyalty_balance_id` |

## Evaluation Trigger Options

| Value | Description | Recommended use |
|-------|-------------|-----------------|
| `real_time` | Tier re-evaluated on every completed transaction | **Upgrades** — reward immediately |
| `membership_anniversary` | Re-evaluated once a year on enrollment anniversary | **Downgrades** — avoid mid-year status loss |
| `tier_anniversary` | Re-evaluated once a year on tier entry anniversary | When tier-year tracking matters |

**Best practice for demos**: `upgradeEvaluationTrigger: "real_time"` + `downgradeEvaluationTrigger: "membership_anniversary"` — instant upgrades are impactful during presentations while protecting members from losing status mid-year.

## API Reference

- **Create Group**: `POST /v3/loyalty/tier/programs/{pid}/tier-groups`
- **Create Tier**: `POST /v3/loyalty/tier/programs/{pid}/tier-groups/{gid}/tiers` (one call per tier)
- **Update Tier**: `PUT /v3/loyalty/tier/programs/{pid}/tiers/{tid}` (note: NO `{gid}` in update path)
- **Re-Publish**: `POST /v3/loyalty/config/programs/{pid}/publish` — MANDATORY after creating/modifying tiers
- Base path is `/tier/` — tier operations have their own domain prefix
- Tiers are created **individually**, NOT inline in the tier group payload
- `balanceDefinitionId` is passed inside each tier's `accessConditions`, not at the tier group level
- Threshold uniqueness: no two tiers can share the same `minimumValue`
- Any threshold change requires re-publishing the program

### Path Confusion Warning

| Operation | Path | Note |
|-----------|------|------|
| Create tier group | `POST .../tier/programs/{pid}/tier-groups` | — |
| Create tier | `POST .../tier/programs/{pid}/tier-groups/{gid}/tiers` | `{gid}` required |
| **Update tier** | `PUT .../tier/programs/{pid}/tiers/{tid}` | **NO `{gid}`** — common mistake |

### Expected Responses

- `200` — Tier group or tier created/updated
- `400` — Invalid request (duplicate thresholds, missing fields)
- `409` — Conflict (resource already exists)

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Never use `False`/`True` in inline `-c` heredocs | Shell may lowercase them — define a helper function instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues |
| Use `python3 << 'PYEOF' ... PYEOF` only for simple scripts | Use a unique delimiter (`PYEOF` not `EOF`) to avoid conflicts |
| Always write curl commands to `/tmp/script.sh` and execute with `bash /tmp/script.sh` | Avoids shell interpolation issues |
