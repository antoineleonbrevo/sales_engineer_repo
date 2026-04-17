# Create Loyalty Transactions — Reference

Companion reference for the `create-loyalty-transactions` skill. Contains API parameter tables and response schemas.

## Direct Transaction API

- **Create**: `POST /v3/loyalty/balance/programs/{pid}/transactions`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `contactId` | integer | Yes* | Brevo contact ID (*required unless `loyaltySubscriptionId` provided) |
| `loyaltySubscriptionId` | string | Yes* | External member ID (*required unless `contactId` provided) |
| `balanceDefinitionId` | string (UUID) | Yes | Which balance to credit or debit |
| `amount` | double | Yes | Positive = credit, negative = debit. Must not be zero |
| `autoComplete` | boolean | No | `true` = complete immediately. `false` (default) = stays `pending` |
| `eventTime` | string | No | ISO 8601 timestamp of when the triggering event occurred |
| `expiryBalanceMinutes` | integer | No | Minutes before credited balance expires (must be > 0) |
| `ttl` | integer | No | Time-to-live for pending transaction in minutes (must be > 0) |
| `meta` | object | No | Arbitrary key-value metadata (e.g., `reason`, `campaign_id`) |

### Response (200)

```json
{
  "id": "txn_aaa111-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "amount": 50,
  "balanceDefinitionId": "a74cxx1d-...",
  "contactId": 12345,
  "loyaltyProgramId": "27xxdd7a-...",
  "status": "pending",
  "createdAt": "2025-03-01T10:00:00.000Z",
  "updatedAt": "2025-03-01T10:00:00.000Z",
  "cancelledAt": null,
  "completedAt": null,
  "rejectReason": null,
  "rejectedAt": null
}
```

## Balance Order API (purchase-linked)

- **Create**: `POST /v3/loyalty/balance/programs/{pid}/create-order`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `contactId` | integer | Yes | Brevo contact ID (must be >= 1) |
| `balanceDefinitionId` | string (UUID) | Yes | Which balance to credit |
| `amount` | double | Yes | Points or cashback amount. Must not be zero |
| `source` | string | Yes | `"engine"` (system-triggered) or `"user"` (customer action) |
| `dueAt` | string | Yes | RFC 3339 timestamp — when the order should be processed |
| `expiresAt` | string | No | Optional expiration for the order itself (RFC 3339) |
| `meta` | object | No | Store order ID, cart total, or custom metadata |

### Response (200)

```json
{
  "id": "ord_xyz789-...",
  "transactionid": "txn_bbb222-...",
  "contactId": 12345,
  "loyaltyProgramId": "27xxdd7a-...",
  "balanceDefinitionId": "a74cxx1d-...",
  "amount": 100,
  "source": "user",
  "dueAt": "2025-03-02T10:00:00.000Z",
  "processedAt": null,
  "createdAt": "2025-03-01T10:00:00.000Z",
  "updatedAt": "2025-03-01T10:00:00.000Z"
}
```

## Transaction Lifecycle Endpoints

- **Complete**: `POST /v3/loyalty/balance/programs/{pid}/transactions/{tid}/complete` — finalizes credit/debit, updates balance, re-evaluates tiers, checks reward rules
- **Cancel**: `POST /v3/loyalty/balance/programs/{pid}/transactions/{tid}/cancel` — cancels the transaction, balance unchanged

## Error Codes

| Code | Meaning | Action |
|------|---------|--------|
| `401` | Invalid or missing API key | Check `api-key` header |
| `403` | Insufficient permissions | Ensure Loyalty module is active |
| `404` | Program or transaction not found | Check `{pid}` and `{tid}` |
| `422` | Unprocessable entity | Common causes: `amount` is 0, `contactId` doesn't exist, transaction already completed/cancelled, program not published |
| `500` | Internal server error | Retry with exponential backoff |
