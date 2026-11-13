# Design notes

How YAPA stores money, keeps requests idempotent across two databases, handles concurrency and sources FX rates. The [README](../README.md) has the short version.

## Contents

- [Data model](#data-model)
- [Money precision](#money-precision)
- [Idempotency](#idempotency)
- [Concurrency](#concurrency)
- [FX rates and provider failures](#fx-rates-and-provider-failures)
- [Background jobs, audit and request logs](#background-jobs-audit-and-request-logs)

## Data model

### TigerBeetle (balances and money movement)

- **Ledgers:** one per currency: cNGN=1, cGHS=2, cXAF=3, USDx=4, EURx=5. Transfers never cross ledgers.
- **Accounts** (`code`):
  - `100` USER: one per wallet per currency, with no overdraft allowed;
  - `200` TREASURY: one per currency, funds simulated deposits;
  - `300` FX_POOL: one per currency, the counterparty for every conversion;
  - `400` FEE: one per currency, collects the spread.
- **Transfers** (`code` 1 DEPOSIT, 2 SWAP, 3 TRANSFER, 4 FX_FEE). `user_data_128` holds the business transaction id.

| Operation                            | Legs (one linked chain)                                                               |
| ------------------------------------ | ------------------------------------------------------------------------------------- |
| Deposit                              | TREASURY[c] → USER[c]                                                                 |
| Transfer, same currency              | USER_src[c] → USER_dst[c]                                                             |
| Swap / cross-currency transfer A → B | USER_src[A] → POOL[A] (amount) · POOL[B] → USER_dst[B] (net) · POOL[B] → FEE[B] (fee) |

Each ledger always sums to zero. Pool balances show the platform's net FX position per currency.

### MongoDB (snake_case fields, ULID `_id`, `created_at`/`updated_at`)

| Collection     | Purpose and key fields                                                                                                                                                                                                                                                                                                                                                                | Indexes                                                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `wallets`      | `email?`, `phone?` (E.164), `status` (`PROVISIONING` → `ACTIVE`)                                                                                                                                                                                                                                                                                                                      | unique sparse `email`, unique sparse `phone`                                                                                     |
| `accounts`     | Registry of ledger accounts. `_id` **is** the TigerBeetle account id. `wallet_id` (null for operator accounts), `type`, `currency`, `ledger`                                                                                                                                                                                                                                          | unique `(wallet_id, currency)` for USER; unique `(type, currency)` for operator accounts                                         |
| `transactions` | `type`, `status` (`PENDING`/`COMPLETED`/`FAILED`), `unique_reference`, `request_hash`, `source {wallet_id, amount, currency}`, `destination {…}`, `fx {mid_rate, effective_rate, spread_bps, fee_amount, rate_source, rate_refreshed_at}`, `ledger_transfers[]` (the exact legs, with ids), `failure`, `response` (status and body replayed to retries), `request_id`, `completed_at` | unique `unique_reference`; `(source.wallet_id, _id)`, `(destination.wallet_id, _id)`; partial `(status, updated_at)` for PENDING |
| `fx_rates`     | Per directed pair: static/fallback `rate`, `spread_bps`, `prefer_static`, `source` (`provider`/`admin`)                                                                                                                                                                                                                                                                               | unique `(base_currency, quote_currency)`                                                                                         |
| `audit_logs`   | Wallet event trail: `wallet_id`, `event`, `transaction_id`, `request_id`, `metadata`, `occurred_at`                                                                                                                                                                                                                                                                                   | `(wallet_id, _id)`; unique `(wallet_id, event, transaction_id)`                                                                  |
| `request_logs` | `request_id`, method, path, query, redacted headers, body, response status and body, `duration_ms`                                                                                                                                                                                                                                                                                    | unique `request_id`; TTL on `occurred_at` (`REQUEST_LOG_TTL_DAYS`)                                                               |

`amount` on a transaction is always the **source** amount. Amounts are stored as integer strings, which stay exact beyond 2^53.

## Money precision

- **Integer minor units everywhere.** The API accepts and returns amounts as integer strings (`^[1-9]\d*$`, below 2^63). They are `bigint` in TypeScript and u128 in TigerBeetle. No floating point ever touches an amount.
- **Per-currency scale.** cXAF has 0 decimals (like the CFA franc); the others have 2. Conversions rescale between them.
- **Rates** are decimal strings, rounded **half-up to 20 significant digits**. The rounded rate is the one recorded and applied, so anyone can recompute a conversion from the stored transaction.
  - An early version truncated rates to 15 digits. That turned 1/1500 into 0.000666…6 and valued 1,500.00 NGN at 0.99 USD. A test caught it, and it's fixed in its own commit.
- **Conversion:** `gross = floor(amount × mid_rate × 10^(scale_dst − scale_src))`, `fee = floor(gross × spread_bps / 10000)`, `net = gross − fee`.
  - Every step rounds **down**, so a client never receives more than the exact value. Sub-unit residue stays in the FX pool.
  - If `net` would be 0, the request is rejected with `AMOUNT_TOO_SMALL` rather than taking money for nothing.
- **Balances can't go negative**, because TigerBeetle enforces it on every user account (`exceeds_credits` → `422 INSUFFICIENT_FUNDS`).

## Idempotency

`unique_reference` (required, globally unique) is the idempotency key for `POST /v1/transactions`.

1. The request is validated and normalised, then fingerprinted (`request_hash`).
2. If the reference already exists:
   - with a different fingerprint → `422 UNIQUE_REFERENCE_REUSED`;
   - `COMPLETED` / `FAILED` → the **stored status code and body are replayed** verbatim;
   - `PENDING` and recent → `409 TRANSACTION_IN_PROGRESS`;
   - `PENDING` and stale (the original request died) → it is **resumed** (step 4).
3. Otherwise the FX quote, amounts and ledger legs, **including pre-generated TigerBeetle transfer ids**, are computed and stored as a `PENDING` transaction. The unique index on `unique_reference` settles concurrent duplicates: only one insert wins, and the others become replays.
4. The stored legs are submitted to TigerBeetle as one linked chain, then the transaction is finalized with its response.

Because the ledger ids are fixed before submission, any re-execution (client retry or reconciler) resubmits the same ids. TigerBeetle answers `exists` for legs it already applied, so **money can't move twice**, even after a crash between the ledger write and the Mongo update. Failed transactions are final: TigerBeetle rejects reused failed ids (`id_already_failed`), and the client should retry with a new reference.

**If TigerBeetle is unreachable**, every ledger request is bounded by `TB_REQUEST_TIMEOUT_MS` (the client would otherwise retry forever), so the transaction stays `PENDING` and the API answers `TRANSACTION_PENDING` (503). The client only accepts IPs, so `TB_ADDRESS` is resolved at boot; after any ledger failure the hostname is resolved again and, if the replica moved (e.g. its container was recreated), the API reconnects without a restart.

Validation errors, unknown wallets and FX unavailability are rejected **before** anything is stored, so the same reference can be reused after fixing the request.

## Concurrency

- **Balances:** TigerBeetle executes transfers serially and enforces no-overdraft atomically. N concurrent transfers draining one balance can't overdraw it, with no locks or retry loops in the API. This is tested with 20–25 parallel transfers and swaps against one balance.
- **Duplicates:** the unique index on `unique_reference` plus conditional finalization (`status: PENDING` → final, at most once) means concurrent identical requests produce one movement.
- **Wallet creation:** unique sparse indexes on email and phone; racing creations get exactly one `201`.
- **Multi-instance safety:** all state is in MongoDB, Redis and TigerBeetle. Repeatable jobs are scheduled through BullMQ job schedulers, so they run once per interval across instances.

## FX rates and provider failures

Live rates come from [open.er-api.com](https://open.er-api.com) (keyless, USD-based, covers NGN/GHS/XAF/EUR) with a 3-second timeout. Each stablecoin is pegged 1:1 to the fiat currency it tracks, and cross rates are derived from the USD base.

Resolution for a pair (`FxRateService.quote`):

1. The pair must exist in `fx_rates`. If it doesn't, rates are fetched from the provider and **all missing pairs are inserted** (default spread `FX_DEFAULT_SPREAD_BPS`).
2. If the pair is `prefer_static` → its stored rate (`source: "static"`).
3. Otherwise the live snapshot from Redis (`source: "live"`). When the cache is cold, one fetch is shared by all concurrent callers.
4. If the provider fails → the pair's stored rate as **fallback** (`source: "fallback"`, `refreshed_at` = when it was stored).
5. If the pair has never been stored and the provider is down → `503 FX_RATE_UNAVAILABLE`, with no money moved.

The `fx-refresh` worker refreshes the Redis snapshot and the stored rates on boot and every `FX_REFRESH_EVERY_MS`, so the fallback is always the last known-good rate. Admin-pinned (`prefer_static`) rows are never overwritten. Spreads are configured **per directed pair**. Every transaction records the mid rate, effective rate, spread, fee, rate source and refresh time it used.

## Background jobs, audit and request logs

Workers extend `BaseWorker` and are marked with `@JobWorker('name')`. The `JobsRegistry` discovers them at boot, creates their BullMQ queue and worker, and registers any schedule. Adding a worker is one class plus one line in a module's providers. Any worker can be triggered on demand with `POST /v1/admin/jobs/{name}/trigger`.

| Worker        | Trigger                                   | Does                                                                                                         |
| ------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `fx-refresh`  | every `FX_REFRESH_EVERY_MS` (and on boot) | Fetch live rates into Redis and `fx_rates`                                                                   |
| `reconciler`  | every `RECONCILER_EVERY_MS`               | Re-execute stale `PENDING` transactions with their stored legs; finish wallets stuck in `PROVISIONING`       |
| `audit-log`   | event                                     | Persist wallet events (created, deposit, swap, sent, received, failed, reconciled) to `audit_logs`           |
| `request-log` | every request                             | Persist request and response (headers redacted, bodies over 16 KB truncated) to `request_logs` by request id |

Audit and request logging are fire-and-forget: a Redis hiccup never fails a payment. Deterministic job ids and unique indexes make worker retries idempotent.
