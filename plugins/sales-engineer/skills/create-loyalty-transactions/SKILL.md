---
name: create-loyalty-transactions
description: "Create sample transactions for a Brevo Loyalty program demo. Activates on: loyalty transactions, demo transactions, add points, creer transactions."
tools: Bash, Read, Write
---

# Create Loyalty Transactions

Create sample transactions to populate a loyalty demo with realistic point balances and tier distribution.

> **Language**: Communicate with the user in the same language they use — French or English.
> **Reference**: See `REFERENCE.md` for full API parameter tables, response schemas, and error codes.

## Transaction Lifecycle

Every credit or debit follows a **two-phase lifecycle**:

```
Step 1: Create transaction / balance order  ->  status: PENDING   (balance unchanged)
Step 2a: Complete                           ->  status: COMPLETED (balance updated)
Step 2b: Cancel                             ->  status: CANCELLED (balance unchanged)
```

For demo purposes, use `autoComplete: true` on direct transactions to skip the pending phase.

## Demo Context

- **Read**: `program.id`, `contacts` (with `contactId` and `loyaltySubscriptionId`), `balanceDefinitions[0].id`, `tierGroups` | **Write**: `transactions`

## Workflow

1. Read context for program, contacts, balance definition ID, and tier structure
2. Design transaction distribution to showcase tier progression (see Tier Distribution below)
3. Create varied transaction types:
   - **Non-purchase** (direct transactions with `autoComplete: true`): welcome bonus, referral bonus, birthday bonus
   - **Purchase** (balance orders -> complete): linked to simulated orders with `meta.orderId`/`meta.orderAmount`
4. **IMPORTANT: Use Python for all transaction loops.** Bash curl causes JSON parsing errors with UUID variables
5. Write and execute a Python script (`/tmp/loyalty_transactions.py`):

```python
import urllib.request, json, time

pid = open("/tmp/.brevo_loyalty_pid").read().strip()
balance_id = open("/tmp/.brevo_loyalty_balance_id").read().strip()
api_key = open("/tmp/.brevo_key").read().strip()
base_url = f"https://api.brevo.com/v3/loyalty/balance/programs/{pid}"

def api_call(endpoint, payload):
    data = json.dumps(payload).encode()
    req = urllib.request.Request(
        f"{base_url}/{endpoint}", data=data,
        headers={"api-key": api_key, "content-type": "application/json"}, method="POST")
    try:
        with urllib.request.urlopen(req) as resp:
            return json.loads(resp.read()), resp.status
    except urllib.error.HTTPError as e:
        return json.loads(e.read()), e.code

# Direct transactions (bonuses): {"contactId": <id>, "balanceDefinitionId": balance_id, "amount": <amt>, "autoComplete": True}
# Balance orders (purchases): api_call("create-order", {...}) then complete via transactions/{tid}/complete
```

6. Update context with transaction results

## Pre-call Validation

| Check | Rule |
|-------|------|
| Program state | Must be `active` (published) |
| Contacts | Must be subscribed to the program |
| Member identifier | At least one of `contactId` (int) or `loyaltySubscriptionId` (string) |
| `pid` | Must exist in `/tmp/.brevo_loyalty_pid` |
| `balanceDefinitionId` | Must exist in `/tmp/.brevo_loyalty_balance_id` |
| `amount` | Must not be zero. Positive = credit, negative = debit |

## Default Volume

50 transactions distributed across contacts to fill all tiers.

## Tier Distribution Strategy

| Tier position | % of contacts | Transaction pattern |
|---------------|--------------|-------------------|
| Base (lowest) | ~40% | 1-3 small transactions |
| Middle | ~35% | 5-8 moderate transactions |
| Top (highest) | ~25% | 10+ transactions including bonuses |

## Which Endpoint to Use

| Scenario | Recommended endpoint |
|----------|---------------------|
| Signup bonus, birthday credit, referral reward | Direct transaction with `autoComplete: true` |
| Purchase — credit on confirmation | Balance order -> Complete |
| Purchase — cancelled before shipment | Balance order -> Cancel |
| Item returned after delivery | Direct transaction (negative amount) -> Complete |
| Manual admin adjustment | Direct transaction -> Complete immediately |
| Bulk import of historical points | Direct transaction with `eventTime` backdated -> Complete |

## Python Safety Rules

| Rule | Why |
|------|-----|
| Never use f-strings with nested braces | Fails on Python ≤ 3.11 — use `.format()` instead |
| Never use `False`/`True` in inline `-c` heredocs | Shell may lowercase them — define a helper function instead |
| Always write scripts to `/tmp/*.py` then run with `python3 /tmp/script.py` | Avoids heredoc quoting and escaping issues |
| Use `python3 << 'PYEOF' ... PYEOF` only for simple scripts | Use a unique delimiter (`PYEOF` not `EOF`) to avoid conflicts |
| **Single API calls**: write curl to `/tmp/script.sh` and execute with `bash /tmp/script.sh` | Avoids shell interpolation issues |
