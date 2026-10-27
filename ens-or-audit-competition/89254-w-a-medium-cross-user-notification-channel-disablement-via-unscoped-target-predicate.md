# 89254 w a medium cross user notification channel disablement via unscoped target predicate

**Submitted on Aug 18th 2026 at 09:00:43 UTC by @shadowHunter for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89254
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

Three statements that change a notification channel's `status` select the row to modify using the `target` column alone — the email address or push endpoint — with **no `user_id` predicate and no channel-type predicate**. Two of the three are `UPDATE`s with no `LIMIT`; the third is a `findFirst` and therefore bounded to a single row.

The database schema explicitly permits two different users to hold the same `target`: the uniqueness constraint is the composite `unique('user_channel_unique').on(user_id, channel, target)`. When a collision exists, a bounce, spam-report, or unsubscribe event for that address disables **an arbitrary one** of the colliding rows — which may be the victim's.

An attacker can deliberately create the collision, because the guard preventing an address being attached to two accounts rejects only addresses already in `verified` state. Attaching a victim's address while it is unclaimed is permitted, and doing so causes the system to send a genuine ENS verification email to the victim — the very message whose bounce or spam-report then fires the vulnerable code path.

## Root cause

The identity of a notification channel is `(user_id, channel, target)` — as the schema itself declares. Three code paths treat `target` alone as the identity. Because the read is unordered and the database is PostgreSQL (`drizzle-orm/neon-http`, `pg-core`), which guarantees no row order absent `ORDER BY`, the row selected among colliding candidates is unspecified.

## 3. Affected code

### 3.1 The unscoped selection — `workers/api-worker/src/app/routes/webhook/sendgrid.ts:171-174`

```ts
const channel = await db.query.userChannels.findFirst({
  where: eq(TABLE.userChannels.target, event.email),
})
```

No `user_id`. No `channel` type. No ordering. Every branch below then mutates `channel.id`:

```ts
case 'bounce':
case 'dropped':
  await db.update(TABLE.userChannels)
    .set({ status: 'bounced',
           status_reason: event.reason ?? `${event.event}: ${event.type ?? 'unknown'}`,
           last_bounce_at: new Date(event.timestamp * 1000) })
    .where(eq(TABLE.userChannels.id, channel.id))

case 'spamreport':
  await db.update(TABLE.userChannels)
    .set({ status: 'bounced', status_reason: 'User reported as spam',
           last_bounce_at: new Date(event.timestamp * 1000) })
    .where(eq(TABLE.userChannels.id, channel.id))

case 'unsubscribe':
case 'group_unsubscribe':
  await db.update(TABLE.userChannels)
    .set({ status: 'unsubscribed', status_reason: 'User unsubscribed via email link' })
    .where(eq(TABLE.userChannels.id, channel.id))
```

### 3.2 Mass update — `workers/api-worker/src/queues/dlq.ts` (hard-bounce handling)

```ts
// mark channel as bounced, prevents future sends. only status='verified' channels are queried
await db
  .update(TABLE.userChannels)
  .set({ status: 'bounced',
         status_reason: `Hard bounce: ${delivery.channel}`,
         last_bounce_at: new Date() })
  .where(eq(TABLE.userChannels.target, delivery.target))
```

This is an `UPDATE`, not a single-row mutation: it disables **every** row sharing that target, across all users. The inline comment refers to which channels are *queried for sending*; it does not scope this write.

### 3.3 Mass update — `workers/api-worker/src/services/delivery/push.ts` (410 Gone handling)

```ts
if (response.status === 410) {
  await db
    .update(TABLE.userChannels)
    .set({ status: 'unsubscribed', status_reason: 'Push subscription expired' })
    .where(eq(TABLE.userChannels.target, deliveryJob.target))
```

### 3.4 The schema permits the collision — `workers/api-worker/src/core/database/schema/notifications.ts`

```ts
unique('user_channel_unique').on(
  table.user_id,
  table.channel,
  table.target,
),
```

Uniqueness is per `(user_id, channel, target)`, so the same `target` under two different `user_id`s is legal.

### 3.5 The guard that fails to prevent it — `channels/email.ts:58-72`

```ts
const otherUserChannel = await c.var.db.query.userChannels.findFirst({
  where: and(
    eq(TABLE.userChannels.channel, 'email'),
    eq(TABLE.userChannels.target, email),
    eq(TABLE.userChannels.status, 'verified'),   // <-- only 'verified' rows block
  ),
})
if (otherUserChannel && otherUserChannel.user_id !== userId) { return 400 }
```

Only a **verified** row belonging to someone else blocks the attach. An unclaimed or pending address is attachable by anyone.

### 3.6 The heap-order primitive — `channels/email.ts:74-93`

```ts
.onConflictDoUpdate({
  target: [TABLE.userChannels.user_id, TABLE.userChannels.channel, TABLE.userChannels.target],
  set: { status: 'pending', last_verification_sent_at: new Date() },
})
```

Re-POSTing the address issues an `UPDATE` against the attacker's own row. `$id.ts:215` (`/resend`) does the same. In PostgreSQL MVCC an `UPDATE` writes a new tuple version, so this is a repeatable primitive for changing where the attacker's row sits in physical scan order.

## Preconditions

1. The attacker knows the victim's email address.
2. The victim's address is not yet `verified` on their own account at the moment the attacker attaches it.
3. The attacker holds an ordinary account — permissionless SIWE sign-up, no gas.

## Vulnerability Details

{% stepper %}
{% step %}

## The attacker creates an account

Sign-in is Sign-In-With-Ethereum; any wallet works, no funds required.
{% endstep %}

{% step %}

## The attacker attaches the victim's email to the attacker's own account

```http
POST /notifications/channels/email
Authorization: Bearer <attacker JWT>
Content-Type: application/json

{"email":"victim@example.com"}
```

This is accepted, because `channels/email.ts:58-72` rejects only addresses already `verified` under another account. A row is created with `user_id = attacker`, `status = 'pending'`, `target = victim@example.com`.
{% endstep %}

{% step %}

## ENS sends the victim a genuine verification email

Correctly branded, correctly signed — a real ENS message. The victim did not request it.
{% endstep %}

{% step %}

## The victim attaches and verifies their own address

Both rows now exist, and both are legal under `unique(user_id, channel, target)`:

```
id=…0001  user_id=ATTACKER  channel=email  target=victim@example.com  status=pending
id=…0002  user_id=VICTIM    channel=email  target=victim@example.com  status=verified
```

{% endstep %}

{% step %}

## The attacker re-submits the same address

The attacker re-submits the same address (or calls `POST /notifications/channels/:id/resend`). `onConflictDoUpdate` issues an `UPDATE` on the attacker's row, moving its tuple in PostgreSQL heap order, and sends the victim another unsolicited ENS verification email.
{% endstep %}

{% step %}

## The trigger fires

Any of these causes SendGrid to post a webhook event carrying `email = victim@example.com`:

* The victim clicks "Report spam" on the unsolicited ENS email — the correct security response.
* The victim clicks the unsubscribe link.
* The victim's mailbox transiently rejects a message (full mailbox, greylisting), producing `bounce` or `dropped`.
  {% endstep %}

{% step %}

## The server disables an arbitrary colliding row

```sql
-- selection (sendgrid.ts:171)
select … from "user_channels" where "user_channels"."target" = $1
-- mutation
update "user_channels" set "status" = 'bounced', … where "user_channels"."id" = <selected>
```

With no `ORDER BY` and no `user_id`, the selected row may be the victim's.
{% endstep %}

{% step %}

## The victim's notifications are silently off

Nothing informs the victim of the status change. Their `status` reads `bounced`, with a reason — `"User reported as spam"` or `"Hard bounce: email"` — that points at the victim themselves.
{% endstep %}
{% endstepper %}

## Impact Details

The victim stops receiving ENS name-expiry warnings. ENS names lapse if not renewed, and the expiry notification is the mechanism the product provides to prevent that. A victim relying on it and receiving no warning can lose a name and pay premium-priced re-registration, or lose it entirely to a third party.

Secondary: the same unscoped predicate in `dlq.ts` and `push.ts` is an `UPDATE` without `LIMIT`, so those two paths do disable **every** colliding row rather than one. This does not apply to the `sendgrid.ts` path, which is bounded to one row.

## Recommendation

Scope every channel-status write by the full channel identity.

## Proof of Concept

### Prerequisites

* Node.js v22 (tested on v22.21.1)
* `drizzle-orm@0.45.2` — the exact version in the repo lockfile
* No database, no network access, no repo checkout required

### Placement

```
poc-channel-writes/
├── package.json
├── poc.mjs
└── node_modules/drizzle-orm/     (npm install, or copy from the repo pnpm store)
```

### Setup and run

```sh
mkdir poc-channel-writes && cd poc-channel-writes
# create package.json and poc.mjs from sections 10.5 and 10.6
npm install
node poc.mjs
```

### `package.json`

```json
{
  "name": "poc-channel-writes",
  "private": true,
  "type": "module",
  "dependencies": { "drizzle-orm": "0.45.2" }
}
```

### `poc.mjs`

```js
// PoC — unscoped channel-status writes, ENS api-worker @ commit 1c9b47f
// Uses drizzle-orm 0.45.2 (the exact version in the repo lockfile) with the
// pg-proxy driver, so every statement below is REAL drizzle-generated SQL.
import { pgTable, uuid, text, timestamp, unique } from 'drizzle-orm/pg-core'
import { drizzle } from 'drizzle-orm/pg-proxy'
import { eq } from 'drizzle-orm'
import { readFileSync } from 'node:fs'

// Columns transcribed from workers/api-worker/src/core/database/schema/notifications.ts.
// The composite unique key is reproduced EXACTLY: it is what permits two
// different users to hold the same `target`.
const userChannels = pgTable('user_channels', {
  id: uuid('id').primaryKey(),
  user_id: uuid('user_id').notNull(),
  channel: text('channel').notNull(),
  target: text('target'),
  status: text('status'),
  status_reason: text('status_reason'),
  last_bounce_at: timestamp('last_bounce_at', { withTimezone: true }),
}, (t) => [unique('user_channel_unique').on(t.user_id, t.channel, t.target)])

const captured = []
const db = drizzle(async (sql, params) => { captured.push({ sql, params }); return { rows: [] } },
  { schema: { userChannels } })

const report = (label, sql, params) => {
  const s = sql.toLowerCase()
  // Only the predicate matters: user_id also appears in a SELECT column list.
  const where = s.includes(' where ') ? s.slice(s.indexOf(' where ')) : ''
  console.log(`\n--- ${label} ---`)
  console.log(sql)
  console.log('params        :', JSON.stringify(params))
  console.log('WHERE clause     :', where.trim() || '(none)')
  console.log('user_id in WHERE :', where.includes('user_id'))
  console.log('channel in WHERE :', where.includes('"channel"'))
  console.log('has order by  :', s.includes('order by'))
  console.log('has limit     :', s.includes('limit'))
}

const VICTIM = 'victim@example.com'
console.log('='.repeat(74))
console.log(' ENS api-worker — unscoped channel-status writes (commit 1c9b47f)')
console.log(' drizzle-orm', JSON.parse(readFileSync('./node_modules/drizzle-orm/package.json','utf8')).version)
console.log('='.repeat(74))

// 1. webhook/sendgrid.ts:171 — selects the row that will be mutated
captured.length = 0
await db.select().from(userChannels).where(eq(userChannels.target, VICTIM))
report('SELECT — sendgrid.ts:171  findFirst({ where: eq(target, event.email) })',
  captured[0].sql, captured[0].params)

// 2. queues/dlq.ts:~201 — hard-bounce mass update
captured.length = 0
await db.update(userChannels)
  .set({ status: 'bounced', status_reason: 'Hard bounce: email', last_bounce_at: new Date(0) })
  .where(eq(userChannels.target, VICTIM))
report('UPDATE — dlq.ts:~201  set status=bounced where target=?',
  captured[0].sql, captured[0].params)

// 3. services/delivery/push.ts:~117 — 410 Gone mass update
captured.length = 0
await db.update(userChannels)
  .set({ status: 'unsubscribed', status_reason: 'Push subscription expired' })
  .where(eq(userChannels.target, VICTIM))
report('UPDATE — push.ts:~117  set status=unsubscribed where target=?',
  captured[0].sql, captured[0].params)

// 4. Behavioural: two users share one target; whichever row the SELECT
//    returns is the one the handler mutates. No disambiguation exists.
console.log('\n' + '='.repeat(74))
console.log(' Two accounts legitimately share one target (composite unique key)')
console.log('='.repeat(74))
const rows = [
  { id: 'aaaa1111-0000-0000-0000-000000000001', user_id: 'ATTACKER', channel: 'email', target: VICTIM, status: 'pending'  },
  { id: 'bbbb2222-0000-0000-0000-000000000002', user_id: 'VICTIM',   channel: 'email', target: VICTIM, status: 'verified' },
]
console.log('rows present:')
for (const r of rows) console.log(`  id=${r.id}  user_id=${r.user_id.padEnd(8)}  status=${r.status}`)
console.log('\nunique key (user_id, channel, target) => BOTH rows are legal.')
for (const first of [rows[0], rows[1]]) {
  console.log(`\nif the unordered SELECT returns user_id=${first.user_id}:`)
  console.log(`  -> UPDATE ... WHERE id = '${first.id}'  => ${first.user_id}'s channel set to 'bounced'`)
  if (first.user_id === 'VICTIM') console.log("  -> the VICTIM stops receiving name-expiry alerts.")
}
console.log('\nThe handler applies no user_id check and no tie-break: whichever row')
console.log('the unordered SELECT yields is the row that gets disabled.')
```

### Actual output

```
==========================================================================
 ENS api-worker — unscoped channel-status writes (commit 1c9b47f)
 drizzle-orm 0.45.2
==========================================================================

--- SELECT — sendgrid.ts:171  findFirst({ where: eq(target, event.email) }) ---
select "id", "user_id", "channel", "target", "status", "status_reason", "last_bounce_at" from "user_channels" where "user_channels"."target" = $1
params        : ["victim@example.com"]
WHERE clause     : where "user_channels"."target" = $1
user_id in WHERE : false
channel in WHERE : false
has order by  : false
has limit     : false

--- UPDATE — dlq.ts:~201  set status=bounced where target=? ---
update "user_channels" set "status" = $1, "status_reason" = $2, "last_bounce_at" = $3 where "user_channels"."target" = $4
params        : ["bounced","Hard bounce: email","1970-01-01T00:00:00.000Z","victim@example.com"]
WHERE clause     : where "user_channels"."target" = $4
user_id in WHERE : false
channel in WHERE : false
has order by  : false
has limit     : false

--- UPDATE — push.ts:~117  set status=unsubscribed where target=? ---
update "user_channels" set "status" = $1, "status_reason" = $2 where "user_channels"."target" = $3
params        : ["unsubscribed","Push subscription expired","victim@example.com"]
WHERE clause     : where "user_channels"."target" = $3
user_id in WHERE : false
channel in WHERE : false
has order by  : false
has limit     : false

==========================================================================
 Two accounts legitimately share one target (composite unique key)
==========================================================================
rows present:
  id=aaaa1111-0000-0000-0000-000000000001  user_id=ATTACKER  status=pending
  id=bbbb2222-0000-0000-0000-000000000002  user_id=VICTIM    status=verified

unique key (user_id, channel, target) => BOTH rows are legal.

if the unordered SELECT returns user_id=ATTACKER:
  -> UPDATE ... WHERE id = 'aaaa1111-0000-0000-0000-000000000001'  => ATTACKER's channel set to 'bounced'

if the unordered SELECT returns user_id=VICTIM:
  -> UPDATE ... WHERE id = 'bbbb2222-0000-0000-0000-000000000002'  => VICTIM's channel set to 'bounced'
  -> the VICTIM stops receiving name-expiry alerts.

The handler applies no user_id check and no tie-break: whichever row
the unordered SELECT yields is the row that gets disabled.
```

### What the output shows

* **`drizzle-orm 0.45.2`** — the version pinned by the repo lockfile, so the emitted SQL is what the application emits.
* **SELECT block** — `WHERE clause : where "user_channels"."target" = $1`, with `user_id in WHERE : false` and `channel in WHERE : false`. This is the statement at `sendgrid.ts:171` that chooses which row gets disabled. `has order by : false` is why that choice is unspecified. (`user_id` does appear in the *column list*; the check deliberately inspects only the predicate.)
* **Two UPDATE blocks** — `dlq.ts` and `push.ts`. Both carry the same bare `target` predicate and `has limit : false`. These two are faithful to production: as `UPDATE`s they modify every matching row, not one.
* **The SELECT block's `has limit : false` is WRONG for production.** Production calls `findFirst()`, which compiles with `LIMIT 1`. The harness used `db.select()`. See §1a.
* **Collision section** — the two rows are legal simultaneously under `unique(user_id, channel, target)`. The two branches show the consequence of each possible selection; in one, the victim's verified channel is disabled. No code path distinguishes them.
