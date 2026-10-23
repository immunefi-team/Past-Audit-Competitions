# 92578 w a medium notification channel state is read and written by target address alone with no user id predicate over a schema that deliberately allows the same target on multiple accounts

**Submitted on Sep 10th 2026 at 18:16:34 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92578
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Summary

Four statements in the api-worker read or update `user_channels` filtered on the `target` column alone — no `user_id`, and in three of them no `channel` either. The table's own uniqueness constraint is `(user_id, channel, target)`, which means the same target on two different accounts is a state the schema explicitly permits.

Where a duplicate exists, one account's delivery failure silently changes the other account's channel. A verified email channel belonging to the victim can be flipped to `bounced` or `unsubscribed`, after which they stop receiving ENS notifications — expiry warnings included — with nothing shown to them.

## The schema permits duplicate targets

`workers/api-worker/src/core/database/schema/notifications.ts:73-79`:

```ts
unique('user_channel_unique').on(
  table.user_id,
  table.channel,
  table.target,
),
```

The constraint is on the triple. A target is deliberately not unique across users — which makes sense, since two people can legitimately share a mailbox — but it means any statement keyed on `target` alone is under-specified.

## The four statements

**1. Push 410 handler** — `workers/api-worker/src/services/delivery/push.ts:118-124`:

```ts
await db
  .update(TABLE.userChannels)
  .set({
    status: 'unsubscribed',
    status_reason: 'Push subscription expired',
  })
  .where(eq(TABLE.userChannels.target, deliveryJob.target))
```

**2. Hard-bounce handler** — `workers/api-worker/src/queues/dlq.ts:202-209`:

```ts
// mark channel as bounced, prevents future sends. only status='verified' channels are queried
await db
  .update(TABLE.userChannels)
  .set({
    status: 'bounced',
    status_reason: `Hard bounce: ${delivery.channel}`,
    last_bounce_at: new Date(),
  })
  .where(eq(TABLE.userChannels.target, delivery.target))
```

The comment on that first line is itself the evidence: the author is reasoning about a single row.

**3. SendGrid event webhook** — `workers/api-worker/src/app/routes/webhook/sendgrid.ts:171-174`:

```ts
const channel = await c.var.db.query.userChannels.findFirst({
  where: eq(TABLE.userChannels.target, event.email),
})
```

`findFirst` with no `ORDER BY`, so among duplicates the handler acts on whichever row the planner returns.

**4. Push key lookup** — `workers/api-worker/src/services/delivery/push.ts:54-55` resolves the subscription's encryption keys by target alone, so with a duplicate present a notification can be encrypted under the wrong row's keys.

Statements 1 and 2 update **every** matching row across every user. Statement 3 picks an arbitrary one.

## How a duplicate comes to exist

The email path allows it directly. `workers/api-worker/src/app/routes/notifications/channels/email.ts:58-72` refuses an address only when a row for it already exists **and** that row is `status='verified'`:

```ts
const otherUserChannel = await c.var.db.query.userChannels.findFirst({
  where: and(
    eq(TABLE.userChannels.channel, 'email'),
    eq(TABLE.userChannels.target, email),
    eq(TABLE.userChannels.status, 'verified'),
  ),
})
```

So any authenticated user can attach a third party's address to their own account as a `pending` row. Authentication requires only a wallet — `POST /auth/nonce` then `POST /auth/login`, with no name, balance or allowlist check.

## Attack path

1. The victim has a verified email channel on their own account.
2. The attacker authenticates with any wallet and posts the victim's address to `POST /notifications/channels/email`. The guard above passes, because the attacker's row will be `pending`, and the check only looks for `verified` rows. Two rows now exist for that target.
3. ENS sends a verification email to the victim, unsolicited. This is the natural trigger for the next step and the attacker can repeat it.
4. The victim marks that mail as spam. SendGrid posts a `spamreport` event for the address.
5. `sendgrid.ts:171-174` resolves the address with an unordered `findFirst` across both rows and marks whichever it gets. If that is the victim's verified row, the victim's channel is switched off.
6. Separately, any genuine hard bounce for that address takes the `dlq.ts` path, which updates **both** rows unconditionally — the victim's verified channel is marked `bounced` regardless of which account caused the bounce.

Afterwards `createNotification` only ever queries `status='verified'` channels (`workers/api-worker/src/services/notifications/create.ts:116-123`), so the victim silently receives nothing. Nothing in the UI explains it.

## Impact

This is the listed impact of changing another user's non-sensitive details without any wallet interaction on their part, whose named example is enabling or disabling notifications. That is precisely what happens: the victim's notification channel is disabled by an event caused on a different account.

The practical consequence is that ENS expiry warnings stop reaching a user who believes they are subscribed.

## Scope correction

I want to be exact about which half is attacker-driven, so this is not read as broader than it is.

The **push** variants (statements 1 and 4) are not independently reachable: the target there is the browser's push endpoint, which is a secret, and `channels/push.ts:12-39` restricts endpoints to four vendor prefixes. `toPublicChannel` deliberately returns only a SHA-256 `endpointHash` (`services/notifications/helpers.ts:141-162`), so I found no way for an attacker to learn a victim's endpoint. I am reporting those two as the same defect class, not as separately exploitable.

The **email** target is the one an attacker can duplicate today.

Step 5 depends on Postgres row ordering, which I cannot make deterministic — that is why I have set out the `dlq.ts` path alongside it, which has no such ambiguity because it updates every matching row. What the `dlq.ts` path does require is a genuine hard bounce, which the attacker cannot force directly.

So: the defect I am reporting is the four missing predicates, which is unambiguous and verifiable by reading the code. The spam-report chain is the most plausible reachable consequence, and I am flagging its one probabilistic step rather than claiming a guaranteed hit.

## Why this is not an already-known issue

R2-05 concerns the SendGrid event webhook **failing open when the verification key is unset**. That is an authentication defect, it is in a different part of the same file, and it is already fixed at this revision — `webhook/sendgrid.ts:110-116` returns 500 when the key is absent, and `:131-152` rejects a missing or invalid signature. This finding sits downstream of a *valid* signature and is a row-selection defect, not an authentication one.

The `push.ts` and `dlq.ts` statements are not covered by any listed entry. Nothing in the known issues touches channel row selection.

## Recommended fix

Add `user_id` and `channel` to all four predicates. The delivery paths already carry enough context to do this — `BaseDeliveryJob` includes `user_id` (`services/notifications/create.ts:207-212`) — or carry `channel_id` on `notification_deliveries` and key on that instead, which removes the ambiguity entirely.

## Proof of Concept

**Revision note.** The competition Resources tab names the audited revision as `63772fd872af472ced58b009499355f3430c2a86`. This repository's `audit-comp-ready` branch imports that revision at commit `cda79ac`, and its head is `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, which the scope page's asset links point at. `git diff cda79ac 1c9b47f` touches only `apps/manager/package.json`, `workers/api-worker/package.json`, `packages/dqa-server/package.json` and `pnpm-lock.yaml` — a Hono version bump. No source file in any in-scope asset differs, so every line number below is identical at either revision.

Tested against the competition repository at `audit-comp-ready` HEAD `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

This is a database predicate defect, so the proof is the schema constraint set against the four statements that ignore it, plus the request sequence that creates the duplicate. All of it is below; nothing outside this report is needed.

## Part 1 — the constraint and the statements, side by side

The uniqueness constraint (`workers/api-worker/src/core/database/schema/notifications.ts:73-79`, and in the migration at `drizzle/0000_vengeful_golden_guardian.sql`):

```
UNIQUE("user_id", "channel", "target")
```

The four statements that key on `target` alone:

```
services/delivery/push.ts:54-55      SELECT keys        WHERE target = ?
services/delivery/push.ts:118-124    UPDATE ... status='unsubscribed'  WHERE target = ?
queues/dlq.ts:202-209                UPDATE ... status='bounced'       WHERE target = ?
app/routes/webhook/sendgrid.ts:171-174   findFirst          WHERE target = ?   (no ORDER BY)
```

None of the four includes `user_id`. Three of the four also omit `channel`.

## Part 2 — create the duplicate

Run the worker locally:

```
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
pnpm install --frozen-lockfile
pnpm --filter api-worker dev
```

Confirm the victim has a verified email channel:

```
select user_id, channel, target, status from user_channels
  where target = 'victim@example.com';
-- user-VICTIM | email | victim@example.com | verified
```

As the ATTACKER, authenticate with any wallet and claim the same address:

```
curl -sX POST http://127.0.0.1:8787/auth/nonce
curl -sX POST http://127.0.0.1:8787/auth/login \
  -H 'content-type: application/json' \
  -d '{"address":"0x<attacker>","message":"<siwe>","signature":"0x<sig>","nonce":"<nonce>"}'

curl -sX POST http://127.0.0.1:8787/notifications/channels/email \
  -H 'authorization: Bearer <attacker jwt>' \
  -H 'content-type: application/json' \
  -d '{"email":"victim@example.com"}'
```

Observed: `200`. The guard at `channels/email.ts:58-72` only looks for rows with `status='verified'`, and the attacker's new row is `pending`, so it does not fire.

Two rows now share the target:

```
select user_id, channel, target, status from user_channels
  where target = 'victim@example.com';
-- user-ATTACKER | email | victim@example.com | pending
-- user-VICTIM   | email | victim@example.com | verified
```

## Part 3 — one row's failure changes both

Drive the hard-bounce path. In a local run, enqueue a delivery for the attacker's row and let it fail terminally so `queues/dlq.ts` handles it, or invoke the handler directly with a `delivery` whose `target` is `victim@example.com`.

The statement executed is:

```
UPDATE user_channels
   SET status = 'bounced',
       status_reason = 'Hard bounce: email',
       last_bounce_at = now()
 WHERE target = 'victim@example.com';
```

Observed afterwards:

```
select user_id, channel, target, status from user_channels
  where target = 'victim@example.com';
-- user-ATTACKER | email | victim@example.com | bounced
-- user-VICTIM   | email | victim@example.com | bounced   <-- changed
```

Two rows updated, across two accounts, from an event belonging to one of them.

Then confirm the victim stops receiving notifications. `services/notifications/create.ts:116-123` selects only `status='verified'` channels when building deliveries, so the victim's channel is no longer considered:

```
-- create a notification for user-VICTIM, then:
select count(*) from notification_deliveries
  where user_id = 'user-VICTIM' and channel = 'email';
-- 0
```

Nothing is written to the victim's UI to explain the change.

## Part 4 — the webhook variant

For the SendGrid path, post a signed `spamreport` event for `victim@example.com` to `/webhook/sendgrid/events`. `sendgrid.ts:171-174` resolves it with:

```
findFirst({ where: eq(userChannels.target, 'victim@example.com') })
```

With two rows present and no `ORDER BY`, the row returned is whichever the planner yields. When it is the victim's, the victim's verified channel is the one marked.

I am flagging that this step is not deterministic — that is exactly why Part 3 is included, since the `dlq.ts` statement has no such ambiguity and updates both rows unconditionally.

## Observing it in the application

1. Victim adds and verifies their email in Manager notification settings.
2. Attacker signs in with a different wallet and adds the same address. It is accepted.
3. Attacker's row is allowed to hard-bounce, or the victim marks the resulting unsolicited ENS mail as spam.
4. The victim's notification settings still show their email as present, but no further ENS notifications arrive — including name-expiry warnings.
