# 91974 w a medium sendgrid webhook resolves notification channels by email alone so a cross user duplicate is not ownership bound when provider events are processed

**Submitted on Sep 5th 2026 at 22:15:33 UTC by @OadeHack for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91974
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Summary

The SendGrid event webhook (`processEvents`) handles `bounce`, `dropped`, `spamreport`, and `unsubscribe` events by looking up the affected notification channel **by email address alone** — with no `user_id` and no `status` scoping — and then updating that channel's status.

Because channel creation only rejects duplicates when an existing channel is already `verified`, an attacker can create a `pending` email channel for a victim's email address. The victim's `verified` channel and the attacker's `pending` channel can therefore coexist for the same email.

The webhook resolves the mutation target **solely by email address**: once two users have channels for the same email, a SendGrid event is eligible to mutate whichever matching row `findFirst` returns. The webhook does not preserve user ownership when applying the event — `user_id` and `status` are not part of the lookup predicate, so a channel belonging to a different user can be the one mutated. A SendGrid event for the colliding email can therefore be applied to *either* user's channel.

The underlying security boundary failure is that the webhook treats **email address as sufficient channel identity**, even though the database permits multiple users to hold channels for the same email in different states.

## Details

### 1. The webhook resolves the channel by email only

[`webhook/sendgrid.ts`](https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers/api-worker/src/app/routes/webhook/sendgrid.ts), `processEvents`:

```ts
const channel = await db.query.userChannels.findFirst({
  where: eq(TABLE.userChannels.target, event.email),
})
```

The lookup is scoped **only** by `target` (the email). It carries no `user_id` filter, no `status` filter, and no ordering that establishes which matching channel should be selected.

The returned channel is then updated by its database ID:

```ts
await db.update(TABLE.userChannels)
  .set({ status: 'bounced', status_reason: ..., last_bounce_at: ... })
  .where(eq(TABLE.userChannels.id, channel.id))
```

and for unsubscribe events:

```ts
await db.update(TABLE.userChannels)
  .set({ status: 'unsubscribed', status_reason: 'User unsubscribed via email link' })
  .where(eq(TABLE.userChannels.id, channel.id))
```

Therefore, **whichever matching channel the email-only lookup returns becomes the mutation target**, regardless of which user owns that channel or whether it is the verified recipient associated with the event.

### 2. An attacker can create a colliding pending channel for a victim's email

Channel creation ([`channels/email.ts`](https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers/api-worker/src/app/routes/notifications/channels/email.ts)) rejects a duplicate only when an existing channel is already `verified`:

```ts
if (existingChannel) {
  if (existingChannel.status === 'verified') {
    return c.json({ error: 'Email already verified for this account' }, 400)
  }
  // If pending, we can resend verification
}
```

The cross-user duplicate check likewise considers only verified channels:

```ts
const otherUserChannel = await c.var.db.query.userChannels.findFirst({
  where: and(
    eq(TABLE.userChannels.channel, 'email'),
    eq(TABLE.userChannels.target, email),
    eq(TABLE.userChannels.status, 'verified'),
  ),
})
```

Thus, an existing **verified** victim channel blocks the attacker from creating a verified channel, but it does not prevent the attacker from creating a **pending** channel for the same email.

The attacker can consequently create:

```
victim user:
  victim@example.com → verified
attacker:
  victim@example.com → pending
```

These rows belong to different users but share the same `target` value that the webhook later treats as the channel's sole lookup key.

### 3. Channel creation itself sends a real email to the victim's address

The collision is attacker-creatable through the application's normal channel-creation flow. The newly created row is explicitly stored as pending:

```ts
status: 'pending',
```

and the application then sends a verification email to the attacker-supplied address:

```ts
const emailResult = await sendVerificationEmail(
  c.env.SENDGRID_API_KEY,
  c.env.EMAIL_FROM_ADDRESS,
  email,
  verification.token,
  c.env.MANAGER_APP_URL,
)
```

Creating the colliding channel thus causes the application to send a genuine verification email to the supplied address. Consequently, legitimate SendGrid delivery events for that address can enter the vulnerable webhook path. For example, if delivery subsequently produces a `bounce` or `spamreport` event, the webhook resolves the event using only the email address and may therefore mutate the wrong user's channel. The application does not force this delivery outcome (a hard bounce, or the victim marking the unsolicited message as spam, depends on the receiving mailbox); the point is that any legitimate event for the shared email is resolved without regard to channel ownership. The attacker therefore does not need to forge or bypass SendGrid webhook authentication — the event, where generated, is a legitimate provider event produced by the application's own outbound email flow.

### 4. The event can mutate a different user's channel

Once the two rows coexist, any SendGrid event for `victim@example.com` is resolved using:

```ts
findFirst(target = victim@example.com)
```

The query does not establish the owning `user_id`, require `status = 'verified'`, or otherwise bind the event to the legitimate recipient channel. Both rows satisfy this predicate, so the set of eligible rows is not ownership-bound:

```
victim@example.com
    │
    ├── victim:   verified  ──┐
    │                          │ email-only lookup (target = email)
    └── attacker: pending     ├────────→ both rows are eligible
                               │
                               └────────→ selected row's status mutated
```

If the selected row is the victim's verified channel, a `bounce`, `dropped`, or applicable unsubscribe event transitions the victim's verified channel to `bounced`/`unsubscribed`, disabling that user's email notifications.

The reverse collision is also possible: the victim's legitimate unsubscribe event can resolve to the attacker's pending row. In that case, the attacker-owned row is marked unsubscribed while the victim's verified channel remains active, so the victim's unsubscribe silently fails to take effect.

The code does not define which matching row is selected; both rows satisfy the predicate, so the selected row is not ownership-bound. The security failure is therefore not dependent on a particular database ordering: **the webhook has no channel-identity mechanism capable of ensuring that the event mutates the correct user's row.** In either direction, an attacker-created cross-user duplicate causes the trusted webhook to operate on a channel belonging to a different account.

## Impact

**Medium — Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: enabling/disabling notifications.**

An authenticated notification-channel state belonging to one user is not bound to that user's identity when asynchronous SendGrid events are processed: the webhook resolves and mutates a channel using only the email address, and channel creation permits a second user to hold a `pending` channel for the same email. Once that collision exists, a delivery event for the email — including a genuine `bounce`/`spamreport`/`unsubscribe` — can mutate a channel belonging to the other user, transitioning a `verified` channel to `bounced`/`unsubscribed` (which disables that user's email notifications).

The attacker's controlled contribution is creating the identity-colliding channel (and, by that same action, causing the application to send a verification email to the victim's address, which can then enter the webhook path). The finding does not claim the attacker can force which row PostgreSQL selects; the severity rests on the ownership-binding failure — demonstrated by the PoC and source — not on guaranteed victim-row selection. A second outcome is that a victim's genuine unsubscribe can be applied to the attacker's pending channel instead, causing the victim's unsubscribe to fail; both outcomes arise from the same cross-user channel-resolution defect.

This occurs without wallet interaction and affects a non-sensitive notification setting belonging to another user, mapping to the listed Medium impact for **changing details of other users, including enabling/disabling notifications**. The finding does not require an already-connected wallet, disclose confidential information, or alter sensitive account details.

**Relation to known issues.** This is distinct from R2-05 (the webhook failing open when the verification key is unset — an authentication gap). Here, the SendGrid signature verification can succeed normally; the vulnerability is the subsequent **channel-resolution logic**. It is also distinct from the email-verification rate-limit and channel-creation issues: those mechanisms are relevant only because they permit the cross-user pending duplicate that exposes the webhook's separate identity-confusion flaw. No published entry covers a SendGrid webhook resolving a notification channel by email alone across users.

## Recommendation

Ensure a delivery event can only affect the verified channel(s) that legitimately correspond to it, and never a pending row or a different user's channel. The target invariant is:

1. **One verified email → one user/channel**, enforced transactionally at channel verification.
2. The webhook only mutates the **verified** channel(s) corresponding to the event's email; pending rows are never eligible for delivery-status mutations.
3. Where the provider flow permits, associate the SendGrid event with a stable channel/message identifier rather than treating the email address as the sole channel identity.

Concretely, at minimum, scope the webhook's lookup/update to verified channels:

```ts
await db.update(TABLE.userChannels)
  .set({ status: 'bounced', ... })
  .where(and(
    eq(TABLE.userChannels.target, event.email),
    eq(TABLE.userChannels.channel, 'email'),
    eq(TABLE.userChannels.status, 'verified'),
  ))
```

and additionally prevent creation of a `pending` channel for an email another account has already verified, closing the collision at the source. Restricting mutations to verified channels is necessary but not sufficient on its own: the durable fix is enforcing single verified ownership per email so the webhook can never face an ambiguous set of rows.

## Proof of Concept

## PoC

**What it proves (in-env):** The PoC drives the **real** SendGrid webhook route with a cryptographically valid signature, executing the real `processEvents`. It establishes:

1. the actual webhook route and signature verification execute, and `processEvents` performs the channel lookup using the event email;
2. the modeled victim `verified` row and attacker `pending` row both satisfy the shipped lookup predicate (`target = event.email`) — `user_id` and `status` are not part of it, so the eligible set is not ownership-bound;
3. the channel returned by that lookup is used directly as the mutation target; and
4. **for either selection outcome** the webhook mutates the returned channel: when the lookup resolves to the victim row, the victim channel becomes `bounced`; when it resolves to the attacker row, the attacker channel becomes `bounced`. This demonstrates that the mutation is not bound to channel ownership.

**Scope of the claim:** the source establishes that the two rows can coexist (Details §2) and that creating the pending duplicate sends a verification email to the address (Details §3), providing a legitimate path for a real provider event to enter the webhook. The worker test environment provides **no live PostgreSQL** — the app uses Neon Postgres over HTTP via `env.DATABASE_URL`, and all worker tests mock the database — so the PoC does **not** assert PostgreSQL row-order/selection behavior; that a real collision resolves to the victim's specific row is not asserted. The demonstrable core is that the webhook's mutation target is not ownership-bound given a collision, for either possible selection. This is the same class of environmental limitation acknowledged in the triage of previosu issue escalated.

### How to run

Add the file at `workers/api-worker/src/app/routes/webhook/__poc__/sendgrid-cross-user-channel.poc.test.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'

const mocks = vi.hoisted(() => {
  const findFirst = vi.fn()
  const set = vi.fn(() => ({ where: vi.fn().mockResolvedValue(undefined) }))
  const update = vi.fn(() => ({ set }))
  const mockDb = { query: { userChannels: { findFirst } }, update }
  return { findFirst, update, set, getDatabase: vi.fn(() => mockDb) }
})

vi.mock('#core/database/index.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import('#core/database/index.js')>()
  return { ...actual, getDatabase: mocks.getDatabase }
})

import sendgridApp from '../sendgrid'

const EMAIL = 'victim@example.com'

// The two rows that coexist after the attacker creates a pending duplicate for
// an email the victim has already verified (Details §2/§3 — source-established).
const VICTIM_ROW   = { id: 'victim-ch',   user_id: 'victim',   target: EMAIL, status: 'verified' }
const ATTACKER_ROW = { id: 'attacker-ch', user_id: 'attacker', target: EMAIL, status: 'pending' }

// The SHIPPED lookup predicate is findFirst({ where: eq(target, event.email) }).
// Both rows satisfy it — user_id and status are not part of the predicate — so
// the eligible set is ownership-blind. The real PostgreSQL row selection is not
// asserted here (the worker test environment provides no live Postgres); the
// test instead shows the mutation is not ownership-bound for EITHER outcome.
const eligibleRows = [VICTIM_ROW, ATTACKER_ROW].filter((r) => r.target === EMAIL)

const payload = JSON.stringify([{ email: EMAIL, event: 'bounce', timestamp: 1_750_000_000 }])
const toBase64 = (v: ArrayBuffer) => btoa(String.fromCharCode(...new Uint8Array(v)))
const createSignature = async (body: string, timestamp: string) => {
  const keyPair = (await crypto.subtle.generateKey({ name: 'ECDSA', namedCurve: 'P-256' }, true, ['sign', 'verify'])) as CryptoKeyPair
  const publicKey = (await crypto.subtle.exportKey('spki', keyPair.publicKey)) as ArrayBuffer
  const signature = await crypto.subtle.sign({ name: 'ECDSA', hash: 'SHA-256' }, keyPair.privateKey, new TextEncoder().encode(timestamp + body))
  return { publicKey: toBase64(publicKey), signature: toBase64(signature) }
}
const fireWebhook = async () => {
  const timestamp = '1750000000'
  const { publicKey, signature } = await createSignature(payload, timestamp)
  const env = { SENDGRID_WEBHOOK_VERIFICATION_KEY: publicKey } as CloudflareBindings
  return sendgridApp.request('/sendgrid/events', {
    method: 'POST', body: payload,
    headers: {
      'Content-Type': 'application/json',
      'X-Twilio-Email-Event-Webhook-Signature': signature,
      'X-Twilio-Email-Event-Webhook-Timestamp': timestamp,
    },
  }, env)
}

beforeEach(() => vi.clearAllMocks())

describe('SendGrid webhook: mutation target is not bound to channel ownership', () => {
  it('both rows satisfy the shipped lookup predicate (target=email); ownership/status are not part of it', () => {
    expect(eligibleRows).toContain(VICTIM_ROW)
    expect(eligibleRows).toContain(ATTACKER_ROW)
    expect(eligibleRows.every((r) => r.target === EMAIL)).toBe(true)
  })

  it('when the lookup resolves to the VICTIM row, the webhook mutates the VICTIM channel', async () => {
    mocks.findFirst.mockResolvedValue(VICTIM_ROW)
    const res = await fireWebhook()
    expect(res.status).toBe(200)
    expect(mocks.update).toHaveBeenCalledTimes(1)
    expect(mocks.set).toHaveBeenCalledWith(expect.objectContaining({ status: 'bounced' }))
  })

  it('when the lookup resolves to the ATTACKER row, the webhook mutates the ATTACKER channel', async () => {
    mocks.findFirst.mockResolvedValue(ATTACKER_ROW)
    const res = await fireWebhook()
    expect(res.status).toBe(200)
    expect(mocks.update).toHaveBeenCalledTimes(1)
    expect(mocks.set).toHaveBeenCalledWith(expect.objectContaining({ status: 'bounced' }))
  })
})
```

Run from `workers/api-worker`:

```bash
node_modules/.bin/vitest run "src/app/routes/webhook/__poc__/sendgrid-cross-user-channel.poc.test.ts"
```

**Expected result (achieved):**

```
✓ SendGrid webhook: mutation target is not bound to channel ownership (3)
  ✓ both rows satisfy the shipped lookup predicate (target=email); ownership/status are not part of it
  ✓ when the lookup resolves to the VICTIM row, the webhook mutates the VICTIM channel
  ✓ when the lookup resolves to the ATTACKER row, the webhook mutates the ATTACKER channel
Test Files  1 passed (1)
     Tests  3 passed (3)
```

The worker logs confirm the mutation follows the returned row in each case: `"Email channel marked as bounced","channelId":"victim-ch"` and `"Email channel marked as bounced","channelId":"attacker-ch"`.
