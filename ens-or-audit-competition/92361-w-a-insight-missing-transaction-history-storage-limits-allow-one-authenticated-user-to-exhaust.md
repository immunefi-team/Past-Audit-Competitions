# 92361 w a insight missing transaction history storage limits allow one authenticated user to exhaust shared database capacity

**Submitted on Sep 9th 2026 at 08:02:45 UTC by @Pig46940 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92361
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

The ENS API Worker accepts an authenticated transaction-history `payload` as an arbitrary JSON object and stores it in PostgreSQL without a per-record byte limit, per-user storage quota, or retained-row limit. Because the client also controls `txId`, an authenticated user can create many independent large rows instead of repeatedly replacing one bounded record.

The supplied local full-stack PoC exercises the real `POST /transactions` route with one ordinary authenticated attacker and sequential requests. On a deliberately bounded shared PostgreSQL instance, valid transaction-history writes consumed the remaining database capacity until PostgreSQL could no longer write WAL, failed recovery, and exited. Database-backed API requests for a separate authenticated victim then returned HTTP 500 or timed out.

Selected asset: **Workers**.

Selected published impact:

> **Critical — Taking down the application/website**

The PoC uses a disposable 256 MiB database only to exercise the capacity boundary safely. It does not claim that production uses the same capacity or that the same number of requests would stop the live deployment.

## Root cause

The shared transaction schema accepts arbitrary object content:

```ts
export const TransactionPayloadSchema = v.record(v.string(), v.unknown())
```

[Shared transaction schema](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/shared-schema/src/transactions/index.ts#L30-L40)

The authenticated transaction-history route stores the complete object in PostgreSQL and returns the stored row. There is no application-level limit on encoded payload bytes, nesting, key count, retained rows, or cumulative stored bytes:

[Transaction-history route](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/transactions/index.ts#L28-L80)

The transaction table stores `payload` as `jsonb`. The primary key includes client-selected `tx_id`, so changing `txId` creates another persistent row for the same authenticated user:

[Transaction table](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/database/schema/transactions.ts#L37-L68)

The resulting path is:

```
authenticated user
→ POST /transactions
→ large schema-valid JSON payload
→ full jsonb persistence
→ new txId creates another retained row
→ repeated writes consume shared database capacity
```

The read endpoint also returns the user's complete transaction history without pagination, but the Critical impact claimed here is based on persistent shared-database exhaustion rather than response size alone.

## Reproduction

The attacker uses only their own wallet and backend session.

1. Obtain a normal authenticated ENS backend session.
2. Repeatedly call `POST /transactions`.
3. Use a new `txId` for each request.
4. Supply a large valid JSON object in `payload`.
5. Each successful request stores another persistent row in the shared database.
6. Continue until the bounded database has insufficient space for the next write and WAL activity.
7. Verify that unrelated database-backed API requests fail after PostgreSQL terminates.

No victim credential, administrative access, direct database connection, public-chain write, or concurrent request flood is required.

## PoC result

The PoC runs the fixed revision through the real Worker entrypoint, Hono routes, authentication middleware, Valibot validation, Drizzle Neon HTTP client, and JSON serialization. The managed database boundary is replaced with a loopback Neon-protocol adapter backed by disposable PostgreSQL 17.

Two fresh runs produced the same result:

| Observation                |                          Run 1 |                          Run 2 |
| -------------------------- | -----------------------------: | -----------------------------: |
| Database volume            |                  256 MiB tmpfs |                  256 MiB tmpfs |
| Initial available space    |                     \~228.1 MB |                     \~228.1 MB |
| Payload per attacker row   |                8,388,608 bytes |                8,388,608 bytes |
| Successful attacker writes |                             12 |                             12 |
| First failed write         |                             13 |                             13 |
| PostgreSQL result          | PANIC; recovery failed; exit 1 | PANIC; recovery failed; exit 1 |
| OOM-killed                 |                          false |                          false |

PostgreSQL reported:

```
PANIC: could not write to file "pg_wal/xlogtemp...": No space left on device
database system was not properly shut down; automatic recovery in progress
PANIC: could not write to file "pg_wal/xlogtemp...": No space left on device
shutting down due to startup process failure
```

After the database stopped, the separate victim received failures from the tested database-backed routes:

| Victim request                   | Result             |
| -------------------------------- | ------------------ |
| `GET /transactions`              | HTTP 500 / timeout |
| `GET /favorites`                 | HTTP 500           |
| `GET /notifications/preferences` | HTTP 500           |
| `GET /notifications`             | HTTP 500           |

The Worker process itself remained able to answer a non-database route. The reproduced failure is therefore specifically the loss of shared database-backed ENS functionality.

## Impact Details

The demonstrated consequence is not limited to the attacker's own oversized request. One authenticated user can persist attacker-selected bytes into shared storage without an application quota, and the local full-stack reproduction shows that exhausting that shared database prevents unrelated users from using multiple database-backed ENS functions.

The local capacity witness does not establish the exact production attack budget. The repository does not disclose:

* live Neon storage capacity or quota configuration;
* current free storage headroom;
* deployed WAF or route-specific rate limits;
* provider autoscaling or recovery behavior.

Those deployment details determine how much data would be required to reproduce the same outage in production.

The application-level issue remains that transaction-history writes have no per-user cumulative storage bound, while the data is persisted in shared database infrastructure used by unrelated users.

## Suggested remediation

1. Reject transaction-history requests above a small explicit body-size limit before persistence.
2. Replace `record(string, unknown)` with a narrow transaction-summary schema and bound nested/string/array sizes.
3. Enforce per-user retained-row and cumulative stored-byte quotas.
4. Paginate transaction-history reads and avoid returning full stored payloads when they are not required.
5. Add route-specific write-rate limits and tests showing one user's storage usage cannot affect other users.

## References

* [Shared transaction schema](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/shared-schema/src/transactions/index.ts)
* [Transaction-history Worker route](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/transactions/index.ts)
* [Transaction database schema](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/database/schema/transactions.ts)

## Link to Proof of Concept

<https://gist.github.com/HiroyukiNaito/e83ca8b2ae18f3d23b14e0e3d8aab39b>

## Proof of Concept

## Proof of Concept

The PoC is provided through an unlisted Secret Gist:

`https://gist.github.com/HiroyukiNaito/e83ca8b2ae18f3d23b14e0e3d8aab39b`

### Executable files

* `run-local-poc.sh` — creates the isolated database, starts the local API stack, executes both fresh runs, performs victim controls, and cleans up.
* `capacity-fill.mjs` — submits valid large transaction-history records.
* `neon-http-proxy.mjs` — provides the loopback Neon HTTP protocol boundary.
* `post-failure-controls.mjs` — verifies the effect on unrelated victim routes.
* `database-schema.sql` — initializes the disposable database.

The JSON results, PostgreSQL logs and SHA-256 manifest are generated evidence, not additional executable inputs.

### Prerequisites

* Linux or WSL
* Node.js `22.16.0`
* pnpm
* Docker
* Git
* At least 2 GB of free local disk space

### Download the PoC and audited source

Run from an empty working directory:

```bash
git clone https://gist.github.com/HiroyukiNaito/e83ca8b2ae18f3d23b14e0e3d8aab39b.git transaction-history-db-poc

git clone \
  https://github.com/immunefi-team/audit-comp-ens.git \
  audit-comp-ens

git -C audit-comp-ens checkout \
  1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

### Install the audited repository’s locked dependencies

```bash
corepack enable

cd audit-comp-ens
pnpm install --frozen-lockfile
cd ..
```

### Run the PoC

```bash
cd transaction-history-db-poc

bash ./run-local-poc.sh \
  "$(realpath ../audit-comp-ens)"
```

Do not invoke `capacity-fill.mjs` against an external database or deployed service. The runner must use only its disposable loopback PostgreSQL container.

### Expected result

Both fresh runs should report:

```yaml
successfulAttackerWrites: 12
firstFailedWrite:
  index: 13
  result: timeout
database:
  allServerProcessesTerminated: true
  automaticRecoveryAttempted: true
  automaticRecoveryFailed: No space left on device
  containerExitCode: 1
  oomKilled: false
```

The PostgreSQL log should contain:

```
PANIC: could not write to file "pg_wal/xlogtemp.*": No space left on device
LOG: terminating any other active server processes
LOG: all server processes terminated; reinitializing
PANIC: could not write to file "pg_wal/xlogtemp.*": No space left on device
```

The post-failure controls should show that the Worker shell remains reachable while the unrelated victim’s database-backed routes return HTTP 500 or time out.
