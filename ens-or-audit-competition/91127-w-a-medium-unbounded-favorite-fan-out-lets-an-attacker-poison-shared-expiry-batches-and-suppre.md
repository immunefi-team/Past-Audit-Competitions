# 91127 w a medium unbounded favorite fan out lets an attacker poison shared expiry batches and suppress cross user notifications

**Submitted on Aug 29th 2026 at 22:58:05 UTC by @OadeHack for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91127
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes

## Description

## Summary

The api-worker's expiry-notification pipeline fans out one database row per recipient of an expiry event, where recipients include **every user who has favorited the name**. The favoriter count per name is entirely attacker-controlled and unbounded — there is no per-name cap, no per-user cap, and no rate limit on `PUT /favorites/:name`, and SIWE authentication is free and unthrottled. The rows are inserted in a **single, unchunked** `INSERT`, so once a batch's combined recipient count exceeds 16,383, the statement crosses PostgreSQL's hard limit of 65,535 bind parameters and throws. Because expiry events are processed in **shared batches** and the handler retries every message in a failed batch (then dead-letters), one poisoned name atomically fails the whole batch and suppresses expiry notifications for the **unrelated users** whose names were grouped into it. Because the poisoned favorite state persists in the database, every future batch containing that name reproduces the failure. This is a deterministic, attacker-controlled availability attack on a security-relevant subsystem — **not** a large-dataset performance concern.

## Details

### 1. Recipients scale 1:1 with attacker-controlled favoriters

For each event, [`collectRecipientsForEvent`](https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers/api-worker/src/queues/event-ingestion.ts) adds the owner plus **every favoriter** of the name to the recipient map:

```ts
// event-ingestion.ts:75-106
export function collectRecipientsForEvent(
  event: ExpiryEvent,
  ownerToUserId: Map<string, string>,
  favoriteUsersByName: Map<string, Set<string>>,
): RecipientMap {
  const recipients: RecipientMap = new Map()
  if (event.owner) {
    const ownerUserId = ownerToUserId.get(event.owner.toLowerCase())
    if (ownerUserId) recipients.set(ownerUserId, 'owned')
  }
  if (!event.includeFavorites) return recipients
  const favoriteUserIds = favoriteUsersByName.get(event.name)
  if (!favoriteUserIds) return recipients
  for (const favoriteUserId of favoriteUserIds) {
    if (!recipients.has(favoriteUserId)) recipients.set(favoriteUserId, 'favourited')
  }
  return recipients
}
```

The favoriters are loaded with **no `LIMIT`** — every favoriter of the name is returned and grouped into a per-name set:

```ts
// event-ingestion.ts:159-185
const favorites =
  favoriteNames.length > 0
    ? yield* intoDbResult(
        db.query.favorites.findMany({
          where: inArray(TABLE.favorites.name, favoriteNames),
          columns: { user_id: true, name: true },
        }),
      )
    : []
// ...
const favoriteUsersByName = new Map<string, Set<string>>()
for (const favorite of favorites) {
  if (!favoriteUsersByName.has(favorite.name)) {
    favoriteUsersByName.set(favorite.name, new Set())
  }
  favoriteUsersByName.get(favorite.name)?.add(favorite.user_id)
}
```

`processExpiryEvents` then pushes **one row per recipient, across all events in the batch**, into a single array (`notificationsToInsert`), four fields per row — `user_id`, `kind`, `payload`, `idempotency_key`.

### 2. The insert is unchunked and crosses PostgreSQL's hard parameter limit

The entire array is handed to a **single** insert with no chunking:

```ts
// event-ingestion.ts:247-260
const insertedNotifications = yield* intoDbResult(
  db
    .insert(TABLE.notifications)
    .values(notificationsToInsert)          // ← unchunked; 4 bind params per row
    .onConflictDoNothing({
      target: TABLE.notifications.idempotency_key,
    })
    .returning({
      id: TABLE.notifications.id,
      user_id: TABLE.notifications.user_id,
      kind: TABLE.notifications.kind,
      payload: TABLE.notifications.payload,
    }),
)
```

Each row binds **4 columns**. PostgreSQL rejects any statement with more than **65,535** bind parameters, giving a deterministic overflow threshold:

```
floor(65535 / 4) = 16,383 rows  →  16,384+ recipients overflows the statement
17,000 favoriters → 68,000 bind parameters → throws
```

Drizzle's `.values([...])` does not auto-chunk, so array length translates directly into bind-parameter count. The same file *does* chunk elsewhere — the queue-send path uses a `chunk` utility — so the unchunked inserts are the anomaly, not the norm.

The subsequent **deliveries** insert uses the identical unchunked pattern and is independently vulnerable for the same reason; it should be fixed alongside the primary sink as defense-in-depth (not a competing crash):

```ts
// event-ingestion.ts:405-410
if (deliveriesToInsert.length > 0) {
  yield* intoDbResult(
    db.insert(TABLE.notificationDeliveries).values(deliveriesToInsert),
  )
}
```

### 3. The favoriter count is attacker-controlled, uncapped, and free to inflate

`PUT /favorites/:name` is gated by `requireAuth` **only** — no per-user cap, no per-name cap, no rate limit:

```ts
// app/routes/favorites/index.ts:21-36
.put('/:name', ...requireAuth, injectDb, async (c) => {
  const { name } = c.req.param()
  const { user_id } = c.var
  await c.var.db.insert(schema.favorites)
    .values({ name, user_id })
    .onConflictDoNothing()
  return c.json({ message: 'Favorite added' }, 200)
})
```

The [favorites schema](https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers/api-worker/src/core/database/schema/favorites.ts) has PK `(user_id, name)`, so **N distinct users produce N rows for one name**:

```ts
// core/database/schema/favorites.ts
primaryKey({ columns: [table.user_id, table.name] })
```

The identities are free: [`POST /auth/login`](https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers/api-worker/src/app/routes/auth/index.ts) is SIWE (nonce + signature) with **no throttle, captcha, or proof-of-work**. Generating 17,000 keypairs, signing in for 17,000 JWTs, and issuing 17,000 `PUT /favorites/target.eth` requests is fully scriptable and gasless.

### 4. Shared-batch atomic failure suppresses unrelated users' notifications

Because `notificationsToInsert` accumulates the recipients of **all events in the batch** into one array (Section 1), a single poisoned name makes the whole batch's insert throw. On any processing error, the handler retries **every message in the batch**:

```ts
// event-ingestion.ts:595-609
    for (const { message } of validMessages) {
      message.retry()
    }
    return
  }
  for (const { message } of validMessages) {
    message.ack()
  }
```

Per [`wrangler.jsonc`](https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers/api-worker/wrangler.jsonc), the ingestion queue is configured `max_retries: 3` with a `dead_letter_queue`. So the poisoned batch fails, all its messages retry three times (crashing identically each time), and the batch is dead-lettered — and **every unrelated name grouped into that batch loses its expiry notification**. Because the poisoned favorite rows persist, the name recurs at each expiry stage (30-day, 7-day, 1-day, expiry), reproducing the failure on every future batch it lands in. The effect is **persistent and recurring**: the attacker maintains the poisoned state and each batch containing the poison name is atomically failed, though any individual batch is eventually dead-lettered rather than wedged forever.

This finding does not correspond to any entry in the published Known Issues document: there is no favorites-related audit finding, QA item, or bug-tracker entry, and the api-worker notification-queue fan-out path is outside the transaction-manager/smart-account packages covered by the Round 3 source audit. It is not a consequence of any listed root cause — the availability entries (R3-01..R3-04) are client-side transaction-manager state-machine defects with no external attacker, whereas this is a server-side, attacker-controlled database-parameter overflow with cross-user collateral.

## Impact

**Critical – Taking state-modifying authenticated actions on behalf of other users without any interaction by that user (explicitly including “Notification/settings changes”).**

Under normal processing, an expiry event for an unrelated user's name causes the api-worker to create notification and delivery records for that authenticated user. By poisoning a shared expiry batch with more than 16,383 attacker-controlled favorites, an attacker causes the batch-wide notification insert to fail before those records are created. The queue retries the batch and ultimately dead-letters it, so the affected users' expected expiry notifications are suppressed without any interaction from them.

The attacker therefore causes the application's **notification state/outcome for unrelated authenticated users to change** using only attacker-controlled favorite records. The affected users do not need to log in, approve anything, change their notification settings, or otherwise interact with the application; their notifications are unilaterally suppressed as a consequence of the attacker's input.

This is directly aligned with the program's explicit **“Notification/settings changes”** example in this impact category. The change here is a suppression rather than an affirmative preference change, but the security-relevant effect is the same category of cross-user state modification: an attacker causes the application's notification behavior for other authenticated users to differ from the state they would have had absent the attack.

The effect is also **persistent and recurring**: the attacker's favorite rows remain in the database, so every subsequent expiry-processing batch containing the poisoned name can reproduce the failure at the 30-day, 7-day, 1-day, and expiry stages.

This finding does **not** claim that the entire ENS application becomes unavailable. The impact is the attacker-controlled modification/suppression of notification processing for other authenticated users, which is why the finding is mapped to the program's authenticated state-modification category rather than the separate “Taking down the application/website” category.

## Recommendation

Chunk both inserts so no single statement approaches the parameter limit, reusing the `chunk` utility already used for queue sends in this file:

```ts
const CHUNK = 5000 // 5000 * 4 = 20000 params, well under 65535
for (const rows of chunk(notificationsToInsert, CHUNK)) {
  await db.insert(TABLE.notifications).values(rows)
    .onConflictDoNothing({ target: TABLE.notifications.idempotency_key })
}
// apply the same chunking to the deliveriesToInsert insert
```

Defense-in-depth: cap favorites per user and/or per name; add a rate limit to `PUT /favorites/:name`; and isolate per-event failures in the queue handler so one poisoned name cannot atomically fail an entire batch (process/insert per event, or catch and dead-letter only the offending event rather than the whole batch).

## Proof of Concept

**What it proves:** driving the **real, shipped** `collectRecipientsForEvent` with 17,000 distinct favoriter accounts for a single name produces a recipient set of 17,000, which `processExpiryEvents` turns into 17,000 `notificationsToInsert` rows handed to the single unchunked `.insert().values()` — 17,000 × 4 = 68,000 bind parameters, exceeding PostgreSQL's 65,535 limit and throwing at runtime. A control with a normal favoriter count stays well under the limit, confirming the overflow is driven purely by attacker-controlled favoriter volume.

The test asserts the overflow condition on the real recipient-assembly code and the documented 4-column insert; it does not execute against a live PostgreSQL because the worker test pool exposes no Postgres binding. The 65,535 limit is a fixed, documented PostgreSQL protocol invariant, and the throwing sink is the unchunked `.values(notificationsToInsert)` call — so the PoC establishes the attacker-controllable, deterministic overflow (the contestable part), while the throw follows from a known database limit.

### How to run

{% stepper %}
{% step %}

## Add the PoC test file

Add the file at `workers/api-worker/src/queues/__poc__/favorites-param-overflow.poc.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { collectRecipientsForEvent } from '../event-ingestion'

const PARAMS_PER_NOTIFICATION_ROW = 4
const PG_MAX_BIND_PARAMS = 65535
const OVERFLOW_ROW_THRESHOLD = Math.floor(PG_MAX_BIND_PARAMS / PARAMS_PER_NOTIFICATION_ROW) // 16383
type ExpiryEvent = Parameters<typeof collectRecipientsForEvent>[0]

describe('M8 — unbounded favorites overflow the unchunked notifications insert', () => {
  it('17,000 distinct favoriters of ONE name yield an insert array that exceeds Postgres 65,535 params', () => {
    const POISON_NAME = 'target.eth'
    const ATTACKER_ACCOUNTS = 17_000
    const favoriteUsersByName = new Map<string, Set<string>>()
    const attackerIds = new Set<string>()
    for (let i = 0; i < ATTACKER_ACCOUNTS; i++) {
      attackerIds.add(`00000000-0000-4000-8000-${i.toString(16).padStart(12, '0')}`)
    }
    favoriteUsersByName.set(POISON_NAME, attackerIds)
    const event = { name: POISON_NAME, owner: null, includeFavorites: true, stage: '30d', expiryDate: 1_900_000_000 } as unknown as ExpiryEvent
    const recipients = collectRecipientsForEvent(event, new Map(), favoriteUsersByName)
    expect(recipients.size).toBe(ATTACKER_ACCOUNTS)
    expect(recipients.size).toBeGreaterThan(OVERFLOW_ROW_THRESHOLD)
    expect(recipients.size * PARAMS_PER_NOTIFICATION_ROW).toBeGreaterThan(PG_MAX_BIND_PARAMS)
  })

  it('control: a normal name (few favoriters) stays well under the limit', () => {
    const favoriteUsersByName = new Map<string, Set<string>>([['normal.eth', new Set(['u1', 'u2', 'u3'])]])
    const event = { name: 'normal.eth', owner: null, includeFavorites: true, stage: '30d', expiryDate: 1_900_000_000 } as unknown as ExpiryEvent
    const recipients = collectRecipientsForEvent(event, new Map(), favoriteUsersByName)
    expect(recipients.size * PARAMS_PER_NOTIFICATION_ROW).toBeLessThan(PG_MAX_BIND_PARAMS)
  })
})
```

{% endstep %}

{% step %}

## Run the test

Run from `workers/api-worker`:

```bash
node_modules/.bin/vitest run "src/queues/__poc__/favorites-param-overflow.poc.test.ts"
```

{% endstep %}

{% step %}

## Verify the expected result

```
✓ M8 — unbounded favorites overflow the unchunked notifications insert (2)
  ✓ 17,000 distinct favoriters of ONE name yield an insert array that exceeds Postgres 65,535 params
  ✓ control: a normal name (few favoriters) stays well under the limit
Test Files  1 passed (1)
     Tests  2 passed (2)
```

{% endstep %}
{% endstepper %}
