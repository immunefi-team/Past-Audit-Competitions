# 92661 w a medium missing ownership check in the sendgrid webhook s channel lookup lets any account silently disable another person s verified ens email notifications permanently

**Submitted on Sep 11th 2026 at 10:52:07 UTC by @Nyko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92661
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

`workers/api-worker` scopes email channels per user everywhere it writes them, except in the one place an untrusted party can trigger a write. The `user_channels` uniqueness constraint is on `(user_id, channel, target)`, so one address legitimately exists as several rows owned by different accounts. The SendGrid webhook then resolves a delivery event to a channel **by address alone** — no user, no status, no ordering — and writes the delivery status onto whatever row that lookup returns.

Any authenticated account can create a pending row for an address it does not own, which makes ENS send a genuine, ENS-branded verification mail to that address. The recipient does the ordinary thing with unsolicited mail — reports it as spam, or clicks unsubscribe — and the resulting signed SendGrid event lands on **their own** verified row, setting it to `bounced` or `unsubscribed`. Every delivery path selects `status = 'verified'`, so that person stops receiving ENS notifications, including name expiry reminders. Neither app renders a badge or a reason for those two states, so the change is silent. And because SendGrid suppresses the address at account level after that event, the person cannot recover by re-enrolling: the worker resets their row to `pending`, the verification mail is dropped before sending, and the `dropped` event flips the row back.

## Vulnerability Details

All line numbers verified against the frozen revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, clone verified unmodified.

**1. The schema says an address belongs to a user. The webhook treats it as global.**

`src/core/database/schema/notifications.ts:73-79`

```
unique('user_channel_unique').on(table.user_id, table.channel, table.target)
```

Uniqueness is per user, and the enrollment code expects multiple rows per address.

**2. Any account can make ENS mail any address.**

`src/app/routes/notifications/channels/email.ts` — POST at `:29` with `...requireAuth` at `:31`. Before writing, `:58-71` rejects only if the address is already `status: 'verified'` **for a different user** (`:69`, "This email address is already linked to another account"). While the address owner has not verified, that gate is empty. The handler upserts a pending `user_channels` row (`:76-96`), inserts a `channel_verifications` row (`:103-118`), and sends the verification mail with ENS's own sender and template (`:143-149`). The recipient gets a legitimate ENS message they never requested. An account costs one SIWE signature.

**3. The webhook resolves the address to a row with no ownership check.** This is the defect.

`src/app/routes/webhook/sendgrid.ts:172-174`

```
const channel = await db.query.userChannels.findFirst({
  where: eq(TABLE.userChannels.target, event.email),
})
```

No `user_id`, no `channel`, no `status`, no `orderBy`. The returned row is then written:

```
:185  case 'bounce': case 'dropped'                  -> status 'bounced'      (:191)
:205  case 'spamreport'                              -> status 'bounced'      (:210)
:222  case 'unsubscribe': case 'group_unsubscribe'    -> status 'unsubscribed' (:228)
```

Each update is `where eq(TABLE.userChannels.id, channel.id)` — the row the unscoped lookup chose.

This is not an authentication defect and does not depend on one. Verification is fail-closed at `:110-152`: a missing key returns 500, missing headers 401, a failed verification 401. The damaging event is a genuine, correctly signed SendGrid event. The row selection is what is wrong.

**4. The consequence is that ENS stops delivering to that person.** Every send path filters on verified status — `src/queues/event-ingestion.ts:283`, `src/services/notifications/create.ts:120` and `:312`. Once the row is `bounced` or `unsubscribed` it is excluded from all of them.

**5. The change is silent.** `src/app/routes/notifications/channels/index.ts:30` returns the channel list to its owner including `status` and `status_reason` (`:40`), so the worker's own output carries the changed state. The Manager renders none of it: `apps/manager/src/features/notifications/settings/contact-methods/email.tsx:255` matches on status, renders a badge for `'pending'` (`:256`) and `'verified'` (`:264`), and `:272` is `.otherwise(() => null)`. A repository-wide grep for `status_reason` across `apps/manager/src` and `apps/portal/src` returns zero hits. The user sees their address still listed, the "Verified" badge quietly gone, and no explanation.

**6. A deterministic enabler: the delete is scoped, its side effect is not.** `src/app/routes/notifications/channels/$id.ts:62` is the DELETE, correctly restricted to the caller's own row by `eq(TABLE.userChannels.user_id, userId)` at `:69`. Its SendGrid cleanup keys on the address alone (`:93-94`), so after the attacker deletes their own pending row the victim's row is the only match for that address and the unscoped lookup cannot select anything else.

**7. A second branch, same unsolicited mail: verify is unauthenticated.** The POST at `:29` carries `requireAuth`; the verify route at `:172-176` does not. It looks the token up (`:179-189`) and sets `status: 'verified'` on `verification.channel_id` (`:195-201`) — whichever channel that is, and whoever presents the token — then adds the address to the broadcast list under `verification.user_id` (`:233-243`), the attacker's id. So if the recipient has no channel yet and clicks "Verify" on the unsolicited mail, the attacker owns a verified ENS channel delivering to that inbox, and the address owner is thereafter refused by the `:58-71` gate whenever they try to enrol their own address.

## Impact Details

**The selected impact, stated affirmatively:** another user's notification enablement is changed by someone else, with no wallet interaction and at most one click. The victim's own verified channel goes from `verified` to `bounced` or `unsubscribed` because of an action taken by a different account. The only thing the victim does is respond to an unsolicited but entirely genuine ENS email the way mail clients invite — report spam, or click unsubscribe — and the click is made on ENS's own message, so nothing about it looks like an attack. Afterwards their ENS notifications stop, including name expiry reminders, which is the notification with consequences for a name owner, and they are not told.

**The denial is permanent, not temporary, and the application offers no way out.** This is the part that distinguishes it from a transient griefing defect. SendGrid's documented `dropped` reasons include "Unsubscribed Address", "Bounced Address" and "Spam Reporting Address", and its suppression lists are account-level and keep suppressing. So once that first event occurs:

* every later ENS send to the address is dropped before delivery, with no click from anyone;
* each `dropped` event reaches the same unscoped lookup (`:172-174`) and rewrites the row to `bounced` (`:185-194`);
* the victim's own re-enrollment does reset their row — `onConflictDoUpdate` sets `status: 'pending'` (`email.ts:88-92`) — but the verification mail is dropped, so they can never verify, and the `dropped` event flips the row back;
* the worker contains **no** suppression handling at all: a repository-wide grep across `src/services/` and `src/app/` for `suppress`, `bounces`, `invalid_emails`, `spam_reports`, `blocks` and `asm` returns zero hits. There is no in-application path to clear the state.

The suppression itself lives in SendGrid, which this program places out of scope; it is cited as the mechanism. The in-scope observables are the worker's row being written and rewritten by events it has no ownership relationship to, and the total absence of any code that could release it.

**Second branch, opposite direction, same click.** If the recipient clicks "Verify" instead, the attacker holds a verified ENS-branded channel into that inbox and can drive mail to it through any event on a name they own or favourite (`event-ingestion.ts:93-101`) or directly via `POST /notifications/channels/:id/test` (`$id.ts:113`). The address owner is then permanently refused when enrolling their own address, with no user-facing way to release it.

Who loses what: the owner of the email address loses their ENS notifications, silently and durably, and in the pre-claim branch loses the ability to enrol that address at all. The attacker gains nothing financial; in the pre-claim branch they gain an ENS-branded channel into a chosen inbox. They need the victim's email address and one SIWE signature.

**On tier, against this competition's own precedent.** Report #92305 — the email-verification rate limiter in this same worker — was confirmed at Low, and the stated reason was that "the victim's own account state is never modified, the writes land under the attacker's user id", making it "a targeted, repeatable denial of one feature, which is the Low row for temporarily disabling a user." Neither half of that reason applies here. The update lands on the victim's own row, and the denial is not temporary: after the first event it re-establishes itself with no further interaction and the application has no code to undo it. I am filing at Medium against the bullet whose condition is met exactly, and not claiming the zero-interaction Critical line, because the first event needs either the recipient's click or their mail server's rejection, and the attacker controls neither.

## Why this is not a duplicate

**Not R2-05.** R2-05 is the webhook failing open when the verification key is unset — a signature-verification defect whose fix is already present at `sendgrid.ts:110-116`, returning 500 rather than proceeding. This report *requires* a valid signature: the damaging event is genuine and verified. Fixing R2-05 changes nothing, and the vulnerable lines (`:172-174`) sit downstream of the verification R2-05 concerns.

**Not our own #92305, and not the rework it was tracked under.** #92305 is the rate limiter in `routes/notifications/channels/email.ts`: statement order relative to the two inserts, and the key the limiter uses. Confirming it, the project noted it was tracked "with the email channel handler rework". The defect here is in a different file, `routes/webhook/sendgrid.ts:172-174`, and its root cause is a lookup with no owner predicate. Reordering and re-keying the limiter leaves that lookup untouched; scoping the lookup does nothing for the limiter. The second branch is in a third place, the unauthenticated `/verify` route. If the rework widens into the webhook, the overlap will be visible in the diff — I am flagging it rather than leaving it to be found.

**Not an enumeration finding.** This does not use the 400 at `email.ts:69` as an existence oracle and learns nothing from the application; the attacker already knows the address they target.

**Nothing in the published known-issues list touches the notification worker's channel ownership,** the webhook's row selection, or the verify route's missing auth. R3-xx, SEC-MGR-xxx, EXP-xxx and the QA/WEB entries concern the transaction manager, smart account, and the two front-end apps.

## Recommended fix

Resolve the event to the row the message was actually sent to: carry the channel id or user id through the SendGrid message as a custom arg and look the channel up by that identifier. At minimum, constrain `processEvents` to `channel = 'email'` and apply the transition only to the row that was the recipient of that specific message. Require authentication on `POST /notifications/channels/email/verify` and check that the caller owns `verification.channel_id`, so a click by the address owner cannot verify another account's channel. Key the SendGrid contact cleanup in `$id.ts` on `(user, address)` rather than the address alone, and skip it while another row still holds that address. Render a badge and `status_reason` for `bounced` and `unsubscribed` in the Manager so the change is never silent. The same "address as global identity" shape also appears at `src/services/delivery/push.ts:55` and `:124` and `src/queues/dlq.ts:209` — noted so the fix can cover them, not claimed as separate findings.

## References

* `workers/api-worker/src/app/routes/webhook/sendgrid.ts` — `:13-17`, `:76-100`, `:110-152`, `:162`, `:172-174` (the defect), `:185-194`, `:205-213`, `:222-231`
* `workers/api-worker/src/app/routes/notifications/channels/email.ts` — `:20`, `:29`, `:31`, `:37`, `:58-71`, `:69`, `:76-96`, `:88-92`, `:103-118`, `:143-149`, `:172-176`, `:179-189`, `:195-201`, `:233-243`
* `workers/api-worker/src/app/routes/notifications/channels/$id.ts` — `:62`, `:69`, `:93-94`, `:113`
* `workers/api-worker/src/app/routes/notifications/channels/index.ts` — `:30`, `:40`
* `workers/api-worker/src/core/database/schema/notifications.ts` — `:73-79`
* `workers/api-worker/src/services/notifications/create.ts` — `:115-121`, `:120`, `:312`
* `workers/api-worker/src/queues/event-ingestion.ts` — `:93-101`, `:283`
* `apps/manager/src/features/notifications/settings/contact-methods/email.tsx` — `:255`, `:256`, `:264`, `:272`
* SendGrid Event Webhook reference — documented `dropped` reasons, including "Unsubscribed Address", "Bounced Address", "Spam Reporting Address": <https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event>
* SendGrid suppressions — account-level lists that keep suppressing: <https://www.twilio.com/docs/sendgrid/ui/sending-email/index-suppressions>
* Frozen revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

## Proof of Concept

## Proof of Concept

**What this establishes:** that a delivery event for a message sent to one account's channel is written onto a different account's channel row, and that after the write the victim's notifications stop. The end state is checked in the worker's own data and in the worker's own recipient query, not in a rendered screen.

Preconditions: one SIWE account, the attacker A. `victim@example.com` is an address A does not control; V is the account of the person who owns it. No wallet interaction by V, no forged signature, no live system touched.

### Branch 1 — the address owner's own verified channel is flipped by someone else

1. **As A:** `POST /notifications/channels/email` with `{ email: "victim@example.com" }`. No verified row exists for that address, so the gate at `email.ts:58-71` is empty and the request proceeds. A pending `user_channels` row is created for A (`:76-96`), a `channel_verifications` row is created (`:103-118`), and ENS sends the genuine verification mail to `victim@example.com` (`:143-149`).
2. **As A:** `DELETE /notifications/channels/{A's channel id}`. The delete is correctly scoped to A's own row (`$id.ts:62-71`). After it, no row for that address remains under A, so whatever row exists later is the sole match for the unscoped lookup. This step is what makes the outcome deterministic rather than dependent on which row the store happens to return first.
3. **As V:** enrol `victim@example.com` normally and click the link in V's own verification mail. The gate at `:58-71` finds no verified row and lets V through; V's row becomes `status = 'verified'` (`:195-201`). V now has exactly one verified email channel — and one unsolicited ENS message still sitting in the mailbox from step 1.
4. **The person at `victim@example.com`** does the ordinary thing with an ENS mail they did not request: clicks unsubscribe, or reports it as spam. This can happen at any time after step 1, which is what makes the branch deterministic. SendGrid posts the resulting event to `POST /sendgrid/events`, correctly signed; verification passes (`sendgrid.ts:110-152`).
5. **`processEvents` resolves the event by address only** (`:172-174`) and selects V's verified row. `unsubscribe` writes `status: 'unsubscribed'` (`:222-231`); `spamreport` writes `status: 'bounced'`, reason "User reported as spam" (`:205-213`); `bounce` and `dropped` write `status: 'bounced'` with the SMTP reason (`:185-194`).
6. **Observe the state change in the worker's own output.** `GET /notifications/channels` as V (`channels/index.ts:30`) now returns that channel with `status: "bounced"` or `"unsubscribed"` and the reason at `:40`.
7. **Observe the loss of delivery.** Run the recipient query from `create.ts:115-121`; it returns no channel for V. Every notification path uses that query (`event-ingestion.ts:283`, `create.ts:120`, `:312`), so V's expiry reminders and transfer notifications are no longer delivered.
8. **Observe that V is not told.** In the Manager settings panel, `email.tsx:255` matches the status, renders a badge only for `'pending'` (`:256`) and `'verified'` (`:264`), and returns null otherwise (`:272`). `status_reason` is rendered nowhere in either app.
9. **Observe that V cannot recover.** V re-enrols; `onConflictDoUpdate` resets the row to `'pending'` (`:88-92`). But the address is now on SendGrid's account-level suppression list, so the verification mail is `dropped` rather than sent, and that `dropped` event returns through `:172-174` and writes the row back to `bounced` (`:185-194`). The worker has no suppression handling anywhere — grep `src/services/` and `src/app/` for `suppress`, `bounces`, `invalid_emails`, `spam_reports`, `blocks`, `asm`: zero hits — so there is no in-application way to clear it.

### Branch 2 — pre-claim: V has not enrolled yet

1. **As A:** `POST /notifications/channels/email` with `{ email: "victim@example.com" }`. The gate at `:58-71` is empty, the request proceeds, ENS sends the genuine verification mail.
2. **The recipient clicks "Verify"** — the affirmative reading of an ENS message. The link posts the token to `POST /notifications/channels/email/verify`, which carries no `requireAuth` (`:172-176`), unlike the POST at `:31`.
3. **The handler** finds the verification by token (`:179-189`) and sets `status: 'verified'` on `verification.channel_id` (`:195-201`) — A's channel — then adds the address to the broadcast list under `verification.user_id` (`:233-243`), A's id.
4. **Result:** A owns a verified ENS channel delivering to the victim's inbox, drivable through any event on a name A owns or favourites (`event-ingestion.ts:93-101`) or via `POST /notifications/channels/{id}/test` (`$id.ts:113`).
5. **As V:** attempt to enrol `victim@example.com`. The gate at `:58-71` now finds a verified row under a different user and returns 400 (`:69`). V can never enrol their own address, and there is no user-facing way to release it.

### Local reproduction, no external service

Apply the five migrations under `workers/api-worker/drizzle/` to a local Postgres. Seed a `users` row for V and one `user_channels` row for it: `{ channel: 'email', target: 'victim@example.com', status: 'verified' }` — the victim's legitimately verified channel, and the only row for that address. Generate a P-256 key pair and set `SENDGRID_WEBHOOK_VERIFICATION_KEY` to its base64 SPKI, so verification passes on a genuine event rather than being bypassed. Sign `timestamp + rawBody` the way `verifySignature` expects (`sendgrid.ts:76-100`) and POST `[{"email":"victim@example.com","event":"spamreport","timestamp":1760000000}]` to `/sendgrid/events`.

Result: the response is `{ok:true}`, and V's row — a row the caller of that event has no relationship to — reads `status = 'bounced'`, `status_reason = 'User reported as spam'`. Run the recipient query from `create.ts:115-121` and it returns no channel for V.

### The row-selection step in isolation

For anyone who wants the mechanism without a database. Save as `poc.mjs`, run `node poc.mjs`; it exits 0 on reproduction.

```js
// The uniqueness constraint is per user, so one address can exist as several rows.
// schema/notifications.ts:73-79 -> unique('user_channel_unique').on(user_id, channel, target)
let userChannels = [
  { id: 'row-attacker', user_id: 'A', channel: 'email', target: 'victim@example.com', status: 'pending'  },
  { id: 'row-victim',   user_id: 'V', channel: 'email', target: 'victim@example.com', status: 'verified' },
]

// Step 2 above. DELETE /notifications/channels/{A's id} is correctly scoped to the
// caller's own row ($id.ts:62-71), and A uses it deliberately: once A's row is gone,
// the victim's row is the ONLY row for that address.
userChannels = userChannels.filter((r) => !(r.id === 'row-attacker' && r.user_id === 'A'))

// VERBATIM shape of sendgrid.ts:172-174 — no user_id, no channel, no status, no orderBy.
const findFirst = (target) => userChannels.find((r) => r.target === target)

// The event SendGrid posts for the message that was sent to A's channel, not to V's.
const event = { email: 'victim@example.com', event: 'spamreport' }

const picked = findFirst(event.email)
// sendgrid.ts:205-213 -> update ... where eq(TABLE.userChannels.id, channel.id)
picked.status = 'bounced'
picked.status_reason = 'User reported as spam'

// create.ts:115-121 -> every recipient query selects on status === 'verified'
const deliverableForV = userChannels.filter((r) => r.user_id === 'V' && r.status === 'verified')

const ok = (c) => (c ? 'PASS' : 'FAIL')
console.log('the message that produced this event was sent to :', 'A')
console.log('row the webhook wrote to                         :', picked.id, '(owner ' + picked.user_id + ')')
console.log('status written                                   :', picked.status, '/', picked.status_reason)
console.log("V's deliverable email channels after the write   :", deliverableForV.length)
console.log(ok(picked.user_id === 'V'),       'the write landed on a row the event has no relationship to')
console.log(ok(deliverableForV.length === 0), "V's notifications no longer have a deliverable channel")
process.exit(picked.user_id === 'V' && deliverableForV.length === 0 ? 0 : 1)
```

Output, exit code 0:

```
the message that produced this event was sent to : A
row the webhook wrote to                         : row-victim (owner V)
status written                                   : bounced / User reported as spam
V's deliverable email channels after the write   : 0
PASS the write landed on a row the event has no relationship to
PASS V's notifications no longer have a deliverable channel
```

### Note on method

A worker logic and ownership defect established by reading the shipped source at the frozen commit. No live system was touched, no mail was sent to anyone, no signature was forged, and no third-party service was tested. The three load-bearing facts are directly verifiable in the cited files: the uniqueness constraint is per user (`schema/notifications.ts:73-79`), the webhook lookup carries no owner or status predicate (`sendgrid.ts:172-174`), and the verify route carries no auth middleware while the sibling POST does (`email.ts:31` versus `:172-176`).
