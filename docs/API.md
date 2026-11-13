# API reference

The interactive version is served by the running API at `/docs` (Swagger UI), with the raw OpenAPI spec at `/docs/docs.json`. This page walks through the main calls.

All routes are under `/v1`. Conventions:

- Every response is an envelope: `{ "status": "success" | "error", "message": string, "data": ... }`. Paginated responses add `meta` (`next_cursor`, `limit`) next to `data`. Errors carry `data: { code, details? }` with a stable machine-readable `code`.
- **Amounts are integer strings in minor units.** cNGN, cGHS, USDx and EURx have 2 decimals (`"150025"` = 1,500.25); cXAF has 0. Wallet balances include each currency's `scale`; transaction amounts use the same per-currency scale.
- Field names are snake_case. IDs are ULIDs.
- Every response carries an `X-Request-Id` header. Send your own (a ULID) to correlate.

The values below are illustrative; live rates move.

## Create a wallet

```http
POST /v1/wallets
{ "email": "ada@example.com", "phone": "+2348031234567" }
```

```json
201
{
  "status": "success",
  "message": "Wallet created",
  "data": {
    "id": "01JBWALLETALICE0000000000A",
    "email": "ada@example.com",
    "phone": "+2348031234567",
    "status": "ACTIVE",
    "created_at": "2026-10-30T12:00:00.000Z",
    "updated_at": "2026-10-30T12:00:00.000Z"
  }
}
```

At least one of `email` and `phone` is required. Emails are lower-cased; phones must be international and are stored as E.164. Reusing either returns `409 WALLET_ALREADY_EXISTS`.

## Deposit

All money movement goes through `POST /v1/transactions`, selected by `type`.

```http
POST /v1/transactions
{
  "type": "DEPOSIT",
  "unique_reference": "dep-0001",
  "destination_wallet_id": "01JBWALLETALICE0000000000A",
  "currency": "cNGN",
  "amount": "1500000"
}
```

```json
201
{
  "status": "success",
  "message": "Transaction completed",
  "data": {
    "id": "01JBTXN0000000000000000001",
    "type": "DEPOSIT",
    "status": "COMPLETED",
    "unique_reference": "dep-0001",
    "source": { "wallet_id": null, "amount": "1500000", "currency": "cNGN" },
    "destination": { "wallet_id": "01JBWALLETALICE0000000000A", "amount": "1500000", "currency": "cNGN" },
    "fx": null,
    "failure": null,
    "created_at": "2026-10-30T12:00:00.000Z",
    "completed_at": "2026-10-30T12:00:00.000Z"
  }
}
```

## Swap (same wallet, two currencies)

```http
POST /v1/transactions
{
  "type": "SWAP",
  "unique_reference": "swap-0001",
  "source_wallet_id": "01JBWALLETALICE0000000000A",
  "source_currency": "cNGN",
  "destination_currency": "USDx",
  "amount": "750000"
}
```

```json
201
{
  "status": "success",
  "message": "Transaction completed",
  "data": {
    "type": "SWAP",
    "source": { "wallet_id": "01JBWALLETALICE0000000000A", "amount": "750000", "currency": "cNGN" },
    "destination": { "wallet_id": "01JBWALLETALICE0000000000A", "amount": "498", "currency": "USDx" },
    "fx": {
      "mid_rate": "0.00066666666666666666667",
      "effective_rate": "0.00066333333333333333334",
      "spread_bps": 50,
      "fee_amount": "2",
      "rate_source": "live",
      "rate_refreshed_at": "2026-10-30T00:00:01.000Z"
    },
    "...": "..."
  }
}
```

At 1 USD = 1,500 NGN, 7,500.00 cNGN is exactly 5.00 USDx (500). The 50 bps spread is 2.5, floored to 2, so the wallet receives 4.98 USDx (498).

## Transfer (another wallet, same or different currency)

```http
POST /v1/transactions
{
  "type": "TRANSFER",
  "unique_reference": "trf-0001",
  "source_wallet_id": "01JBWALLETALICE0000000000A",
  "destination_wallet_id": "01JBWALLETBOB000000000000B",
  "source_currency": "USDx",
  "destination_currency": "cGHS",
  "amount": "400"
}
```

If `destination_currency` is omitted, it defaults to `source_currency` (no conversion). The record holds the sender, receiver, source amount and currency, destination amount and currency, the applied rate and fee, and timestamps.

**Failures** are returned in the error envelope and recorded:

```json
422
{
  "status": "error",
  "message": "Insufficient USDx balance",
  "data": { "code": "INSUFFICIENT_FUNDS", "details": { "transaction_id": "01JBTXN..." } }
}
```

Other codes you are likely to see: `VALIDATION_ERROR` (400), `WALLET_NOT_FOUND` (404), `TRANSACTION_IN_PROGRESS` (409), `UNIQUE_REFERENCE_REUSED`, `AMOUNT_TOO_SMALL`, `SAME_CURRENCY_SWAP`, `SAME_WALLET_TRANSFER` (422), and `FX_RATE_UNAVAILABLE` / `TRANSACTION_PENDING` (503).

## Balances

```http
GET /v1/wallets/01JBWALLETALICE0000000000A
```

```json
200
{
  "status": "success",
  "message": "Wallet retrieved",
  "data": {
    "id": "01JBWALLETALICE0000000000A",
    "status": "ACTIVE",
    "balances": [
      { "currency": "cNGN", "amount": "750000", "scale": 2 },
      { "currency": "cGHS", "amount": "0", "scale": 2 },
      { "currency": "cXAF", "amount": "0", "scale": 0 },
      { "currency": "USDx", "amount": "98", "scale": 2 },
      { "currency": "EURx", "amount": "0", "scale": 2 }
    ],
    "total_usd": { "currency": "USDx", "amount": "598", "scale": 2 },
    "...": "..."
  }
}
```

Balances are read live from TigerBeetle. `total_usd` values each balance at the **mid** rate (no spread), rounds down, and is `null` if no rate is available.

## History and single transactions

```http
GET /v1/transactions?wallet_id=01JB...&type=TRANSFER&currency=USDx&limit=20&cursor=01JB...
GET /v1/transactions/{id}
```

`wallet_id` is required. History includes deposits, swaps and transfers the wallet sent and received, newest first. Each item has a `direction` (`IN`, `OUT`, `SELF`). `currency` matches either side. Pagination is cursor-based: pass `meta.next_cursor` back as `cursor`.

```json
{
  "status": "success",
  "message": "Transactions retrieved",
  "data": [{ "...transaction...": "", "direction": "OUT" }],
  "meta": { "next_cursor": "01JB...", "limit": 20 }
}
```

## Rates

```http
GET /v1/fx-rates
```

Returns all 20 directed pairs:

```json
{
  "base_currency": "USDx",
  "quote_currency": "cNGN",
  "mid_rate": "1500",
  "spread_bps": 50,
  "effective_rate": "1492.5",
  "source": "live",
  "refreshed_at": "2026-10-30T00:00:01.000Z"
}
```

## Admin (header `X-Admin-Key`)

| Endpoint                                  | Purpose                                                                        |
| ----------------------------------------- | ------------------------------------------------------------------------------ |
| `GET /v1/admin/fx-rates`                  | Stored (static / fallback) rate and configuration per pair                     |
| `PATCH /v1/admin/fx-rates/{base}/{quote}` | Set `rate`, `spread_bps` or `prefer_static` for a pair                         |
| `GET /v1/admin/jobs`                      | Registered background workers and schedules                                    |
| `POST /v1/admin/jobs/{name}/trigger`      | Run a worker now: `fx-refresh`, `reconciler`, … (optional `{ "payload": {} }`) |
