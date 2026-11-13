# YAPA

**YAPA** (_YAPA Is Another Payment API_) is a REST API for multi-currency stablecoin wallets: deposits, in-wallet swaps and FX-converted transfers across five mock stablecoins (**cNGN, cGHS, cXAF, USDx, EURx**). The core idea is a split of responsibilities: **TigerBeetle holds the money; MongoDB holds everything about the money.** Ledger transfer ids are fixed before anything is submitted, so a request can be retried or resumed after a crash between the two stores without money ever moving twice.

The name is a recursive acronym in the tradition of GNU and YACC: an honest description of what it is.

## Tech stack

- Node.js 24
- TypeScript
- NestJS 11
- TigerBeetle
- MongoDB (Mongoose)
- Redis
- BullMQ
- Jest
- Testcontainers
- Docker Compose
- OpenAPI (Swagger)

## Contents

- [What it does](#what-it-does)
- [Why this database setup](#why-this-database-setup)
- [Running it](#running-it)
- [Running the tests](#running-the-tests)
- [How it works](#how-it-works)
- [Stack choices](#stack-choices)
- [Assumptions](#assumptions)
- [Limitations](#limitations)
- [Next steps](#next-steps)

## What it does

- Create wallets identified by email and/or phone (E.164).
- Simulated deposits into any of the five currencies.
- Swap between currencies inside one wallet.
- Transfer to another wallet, in the same or a different currency, with automatic FX conversion.
- Live balances (with a USD total), cursor-paginated history, single-transaction lookup and live rates.
- Admin routes to pin or adjust FX rates and spreads per pair, and to trigger background jobs.

The full request and response walkthrough is in [docs/API.md](docs/API.md). A running instance also serves Swagger UI at `/docs`.

## Why this database setup

Balances and money movement live in TigerBeetle, a database built for exactly this:

- **Double-entry by construction.** Every transfer debits one account and credits another on the same ledger (one ledger per currency), so money can't appear or vanish.
- **No negative balances, enforced by the database.** User accounts carry the `debits_must_not_exceed_credits` flag. An overdraft is rejected atomically inside the ledger, not by a read-then-write check in application code.
- **Atomic multi-leg operations.** A swap or cross-currency transfer is 2–3 transfers submitted as one _linked chain_: all of them apply or none do.
- **Idempotent by id.** Resubmitting a transfer id returns `exists` instead of applying it twice, which is the foundation of the idempotency and recovery design.
- **Serial execution.** TigerBeetle applies transfers one at a time, so concurrency needs no row locks or retries in the API.

MongoDB stores the metadata TigerBeetle intentionally doesn't: wallet identity, which ledger accounts belong to which wallet, the business-level transaction record (rate applied, fee, source and destination, status, cached response), FX configuration, audit trail and request logs. These are document-shaped records that change shape more often than the ledger does, and its unique indexes give wallet-identity uniqueness and `unique_reference` idempotency.

The two stores can't share a transaction, so the write path is ordered to make every crash recoverable (see [Idempotency](docs/DESIGN.md#idempotency)).

Why not just PostgreSQL? A single Postgres with `SELECT … FOR UPDATE` and `CHECK (balance >= 0)` would also be correct. TigerBeetle makes the hard guarantees (no overdraft, atomic multi-leg, idempotent ids, double-entry) properties of the ledger rather than of code I wrote. For a payments system, that is the more interesting and more defensible design.

## Running it

Requirements: Docker with Compose. For local development without Docker you also need Node 24 and Corepack.

```bash
docker compose up --build
```

This starts the API (`http://localhost:3000`), MongoDB, Redis and TigerBeetle. There are no manual migration steps. On boot:

- TigerBeetle formats its data file on first start (handled by [`docker/tigerbeetle/entrypoint.sh`](docker/tigerbeetle/entrypoint.sh));
- Mongo indexes are built;
- the per-currency operator accounts (treasury, FX pool, fee) are created idempotently in Mongo and TigerBeetle;
- the `fx-refresh` worker fetches live rates immediately and then on a schedule.

Useful URLs:

- Swagger UI: http://localhost:3000/docs (raw spec at `/docs/docs.json`)
- Health: http://localhost:3000/v1/health (send `X-Admin-Key` to see the MongoDB, Redis and TigerBeetle checks)

> TigerBeetle uses io_uring and locks memory, so its container runs with `seccomp=unconfined`, `IPC_LOCK` and an unlimited memlock ulimit (see `docker-compose.yml`). On Docker Desktop, give the VM at least 4 GB of memory.

**Local development** (dependencies in Docker, API on the host):

```bash
corepack enable
yarn install
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d mongo redis tigerbeetle
cp .env.example .env
set -a && source .env && set +a
yarn start:dev
```

`docker-compose.dev.yml` publishes MongoDB, Redis and TigerBeetle on 27017, 6379 and 3001 (override with `MONGO_PORT`, `REDIS_PORT`, `TB_PORT`). The default `docker compose up` publishes only the API, so it does not clash with databases already running on the host.

Configuration is environment-driven and validated at boot ([`src/config/env.schema.ts`](src/config/env.schema.ts), example in [`.env.example`](.env.example)).

## Running the tests

```bash
yarn test          # everything: unit + e2e (needs a running Docker daemon)
yarn test:unit     # fast, no Docker: services wired to in-memory TigerBeetle/FX/repository fakes
yarn test:e2e      # Testcontainers: real MongoDB, Redis and TigerBeetle
yarn lint && yarn typecheck
```

- **Unit tests** cover money parsing and conversion (including 0-decimal cXAF), rate rounding, FX resolution (live, cached, static, fallback, 503), the ledger wrapper, the transaction service end to end, history pagination and caching, the reconciler, the worker framework, the HTTP envelope and validation, and log sanitising. The transaction tests run on an in-memory TigerBeetle fake that enforces linked-chain atomicity, `exists`, `id_already_failed` and the no-overdraft flag. They include concurrent overdraft attempts, concurrent duplicate requests and crash-between-stores recovery.
- **E2E tests** boot the real `AppModule` against containers started once per run ([`test/e2e/global-setup.ts`](test/e2e/global-setup.ts)). Each suite gets its own Mongo database and Redis logical database, and the live FX provider is replaced by a controllable stub.

## How it works

The short version. The detail, including the data model, is in [docs/DESIGN.md](docs/DESIGN.md).

- **Money is integer minor units end to end**: integer strings in the API, `bigint` in TypeScript, u128 in TigerBeetle. Each currency has its own scale (cXAF has 0 decimals, the rest 2). FX rates are decimal strings rounded half-up to 20 significant digits, and every conversion step rounds down, so a client never receives more than the exact value.
- **Idempotency**: `unique_reference` is required on every transaction. The quote, amounts and ledger legs, including pre-generated TigerBeetle transfer ids, are stored as a `PENDING` record before anything is submitted. A retry or the reconciler resubmits the same ids, and TigerBeetle answers `exists` for legs it already applied. Completed and failed requests replay their stored response verbatim.
- **Concurrency**: TigerBeetle applies transfers serially and enforces no-overdraft itself, so parallel transfers against one balance can't overdraw it, with no locks in the API. A unique index on `unique_reference` settles concurrent duplicates.
- **FX**: live rates from [open.er-api.com](https://open.er-api.com) are cached in Redis and also stored in MongoDB as the last known-good fallback. Pairs can be pinned to a static rate by an admin. Every transaction records the mid rate, effective rate, spread, fee and rate source it used.
- **Background jobs** (BullMQ): FX refresh, a reconciler that finishes stale `PENDING` transactions and half-provisioned wallets, and fire-and-forget audit and request logging.

## Stack choices

| Area           | Choice                                                                 | Why                                                                                                                         |
| -------------- | ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Runtime        | Node.js 24 LTS, TypeScript (strict, `noUncheckedIndexedAccess`)        | Current LTS. Strict mode plus `bigint` keeps money code honest.                                                             |
| Framework      | NestJS 11                                                              | Modules and DI keep the ledger, FX, wallets and jobs separate and easy to test. Interceptors and filters give one envelope. |
| Ledger         | **TigerBeetle**                                                        | A purpose-built double-entry accounting database. It is the source of truth for balances.                                   |
| Metadata store | **MongoDB** (Mongoose)                                                 | Wallets, accounts registry, the transaction read model, FX configuration, audit and request logs.                           |
| Cache / queue  | **Redis** + **BullMQ**                                                 | Live FX rate cache, transaction-history cache, and background workers.                                                      |
| Validation     | class-validator, zod (environment)                                     | Request DTOs go through a global validation pipe; the environment is validated once at boot.                                |
| Money maths    | `bigint` + decimal.js                                                  | Integer minor units end to end; decimal.js is used only for rates.                                                          |
| IDs            | ULID                                                                   | Time-ordered, used for every Mongo `_id` **and** as TigerBeetle u128 ids, so an id means the same thing in both stores.     |
| Tests          | Jest, Supertest, Testcontainers                                        | Unit tests on in-memory fakes, plus e2e tests against real MongoDB, Redis and TigerBeetle containers.                       |
| Tooling        | Yarn 4 (Corepack), ESLint, Prettier, commitlint, husky, GitHub Actions | Conventional commits are enforced by a commit-msg hook and checked in CI.                                                   |
| API docs       | OpenAPI 3 (`@nestjs/swagger`)                                          | Served at `/docs`, with the raw spec at `/docs/docs.json`.                                                                  |

## Assumptions

- Stablecoins are pegged 1:1 to their fiat currency, so FX between them is the fiat cross rate.
- Deposits are simulated: money is minted from a per-currency treasury account (which goes negative, representing the platform's issued supply).
- FX liquidity is unlimited: pool accounts are unbounded, so a swap never fails for lack of liquidity.
- There is no user authentication; wallets are identified only by a unique email or phone. Admin routes use a static shared key.
- A failed transaction is final for its `unique_reference`.
- Phone numbers must be in international format; there is no default country.

## Limitations

- **No user authentication or authorisation.** Anyone who knows a wallet id can read it and move its funds. Fine for a sandbox, and the first thing to add.
- **Mongo and TigerBeetle are eventually consistent** for the transaction record. The ledger is always correct, but a crash can leave a transaction `PENDING` for up to `RECONCILER_STALE_AFTER_MS` + one reconciler interval before it shows `COMPLETED`.
- **Single-replica TigerBeetle** in Compose. Production would run a 3- or 6-replica cluster.
- **No rate locking or quotes.** The rate is taken at request time. Real FX flows usually issue a quote with an expiry that the client accepts.
- **Live provider rates update daily** on the free tier, so "live" is coarse. The source and refresh time are always exposed.
- **History cache** is per wallet and per query with a short TTL, invalidated by a version bump. A page fetched while a transaction is still `PENDING` refreshes when that transaction finalizes.
- **No withdrawals, reversals, limits, KYC or multi-tenant separation.** The scope is deliberately limited to getting money movement right first.

## Next steps

- Authentication (API keys or OAuth) with wallet ownership checks, and per-key rate limiting.
- FX quotes: `POST /quotes` returns a rate locked for N seconds, and transactions reference the quote.
- Withdrawals and reversals, using TigerBeetle pending/two-phase transfers.
- Velocity limits and AML-style checks before submission.
- Reconcile ledger balances against the Mongo read model on a schedule, with alerting on drift.
- OpenTelemetry tracing across HTTP → Mongo → TigerBeetle → BullMQ, plus Prometheus metrics.
- A multi-replica TigerBeetle cluster in Compose, and load tests on the transfer path.
- Webhooks for transaction status changes.
