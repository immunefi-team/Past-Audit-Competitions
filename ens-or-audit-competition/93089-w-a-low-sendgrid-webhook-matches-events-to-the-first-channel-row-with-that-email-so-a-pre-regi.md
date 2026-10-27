# 93089 w a low sendgrid webhook matches events to the first channel row with that email so a pre registered pending duplicate absorbs another user s unsubscribe or bounce

**Submitted on Sep 14th 2026 at 08:10:05 UTC by @abarbatei for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93089
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

The api-worker's SendGrid event webhook resolves each event with `findFirst` on `userChannels.target == event.email`, with no user, channel-type, status or ordering constraint, and updates only that one row. The add-email route lets a second account hold the same address as a *pending* channel, because the uniqueness index is per `(user_id, channel, target)` and the cross-user guard only rejects a *verified* duplicate, and pending rows are never expired. An authenticated user who adds a victim's address before the victim's own channel is verified therefore owns the older row, and the victim's later unsubscribe, spam report or bounce is applied to the attacker's row. The victim's channel stays `verified` and keeps receiving email after opting out.

## Vulnerability Details

Intended behavior: an unsubscribe, spam report or bounce reported by SendGrid for an address must disable the channel of the user whose message produced it. The `unsubscribe` branch exists precisely to honour consent withdrawal.

Root cause:

1. [`sendgrid.ts::processEvents`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/workers/api-worker/src/app/routes/webhook/sendgrid.ts#L167-L237) looks the channel up by email only:

```ts
    const channel = await db.query.userChannels.findFirst({
      where: eq(TABLE.userChannels.target, event.email),
    })
```

Every branch then updates `where: eq(userChannels.id, channel.id)`. There is no `user_id`, no `channel = 'email'`, no status filter and no `orderBy`, so the physical first row wins.

2. [`email.ts` lines 36-72](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/workers/api-worker/src/app/routes/notifications/channels/email.ts#L36-L72): the add-email route rejects an address held by another user only when that other row is `verified`:

```ts
      const otherUserChannel = await c.var.db.query.userChannels.findFirst({
        where: and(
          eq(TABLE.userChannels.channel, 'email'),
          eq(TABLE.userChannels.target, email),
          eq(TABLE.userChannels.status, 'verified'),
        ),
      })
```

A pending row belonging to someone else is not a conflict, and the insert at lines 74-98 upserts on `(user_id, channel, target)`.

3. [`notifications.ts#L74-L79`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/workers/api-worker/src/core/database/schema/notifications.ts#L74-L79): the unique index is `user_channel_unique on (user_id, channel, target)`, so two users may hold the same target.
4. `scheduled/index.ts` has no expiry of pending channels, so the attacker's row persists indefinitely.

Sequence:

1. Attacker A (any signed-in user) calls `POST /notifications/channels/email` with the victim's address E. Row `(A, email, E, pending)` is inserted (a verification mail goes to E; rate-limited to three per hour per address).
2. Victim V later adds E and verifies it. Allowed, because A's row is pending. Row `(V, email, E, verified)` now exists, newer than A's.
3. V receives an ENS notification email and clicks unsubscribe, or the mailbox bounces or reports spam. SendGrid posts `{event, email: E}` with a valid signature.
4. `processEvents` finds A's row first and sets it to `unsubscribed` (or `bounced`). V's row is untouched.
5. `createNotification` still selects V's channel because its status is `verified`, and email delivery to E continues.

Precondition: A must add E before V's row is verified (before V adds it, or while V's row is pending). Against an already-verified victim row the guard blocks A.

## Impact Details

* The victim's own consent withdrawal (unsubscribe) and the platform's bounce and spam-report handling are silently applied to an unrelated account's row, so the victim keeps receiving email and a bouncing or spam-reporting mailbox is never disabled.
* Any authenticated user can set this up for any address they know, with one API call, and the state persists because pending rows are never cleaned.
* Bounded: the attacker must act before the victim verifies, the effect is "keeps receiving" rather than reading or changing the victim's data, and the unordered `findFirst` is insertion-ordered in practice rather than by guarantee. Filed as Low under the Medium wording for enabling/disabling notifications of other users.

## Mitigation

* Send the worker's own `channel_id` to SendGrid as `custom_args` and match on it in the webhook. Fall back to email only when absent, filtered to `channel = 'email'` and `status = 'verified'`, updating all matching rows.
* Reject adding an address that another user holds in any non-deleted state, or expire pending rows after the 24-hour token lifetime from the scheduled job.

## References

* [`sendgrid.ts::processEvents`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/workers/api-worker/src/app/routes/webhook/sendgrid.ts#L167-L237): email-only lookup and single-row update.
* [`email.ts#L36-L98`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/workers/api-worker/src/app/routes/notifications/channels/email.ts#L36-L98): verified-only cross-user guard and upsert.
* [`notifications.ts#L74-L79`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/workers/api-worker/src/core/database/schema/notifications.ts#L74-L79): per-user uniqueness.
* [`create.ts#L115-L123`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/workers/api-worker/src/services/notifications/create.ts#L115-L123): fan-out selects `status = 'verified'` channels.

## Proof of Concept

## Proof of Concept

A Vitest test for the api-worker that drives the real `/webhook/sendgrid/events` route with a validly signed payload (test-generated P-256 key placed in `SENDGRID_WEBHOOK_VERIFICATION_KEY`). The database is mocked with two rows sharing the victim's address, the attacker's pending row inserted first, and `findFirst` returns the first inserted match, which is what an unordered query yields in practice. The test asserts that the unsubscribe event updates only the attacker's row and that the victim's row stays `verified`.

Audited revision `63772fd872af472ced58b009499355f3430c2a86`; the cited files are unchanged in the audit-comp-ready branch. Node 22, pnpm 10, the repo's existing install and the worker's `vitest.config.ts` (Cloudflare vitest pool).

Place the file at:

```
workers/api-worker/test/poc/M07/ProofSendgridEventEmailCollision.test.ts
```

Run from the repository root:

```bash
pnpm install --frozen-lockfile
pnpm --filter api-worker exec vitest run test/poc/M07/ProofSendgridEventEmailCollision.test.ts
```

Expected: 1 test file passed, 1 test passed, exit code 0.

### Proof source

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'

// Two rows share target = E: an attacker-created pending row inserted FIRST,
// and the victim's verified row inserted later. The mock returns rows in
// insertion order for the same target, which is what an unordered findFirst
// yields in practice. The route under test is the REAL sendgrid webhook.
const mocks = vi.hoisted(() => {
  const EMAIL = 'victim@example.com'
  const rows = [
    { id: 'attacker-pending', user_id: 'attacker', channel: 'email', target: EMAIL, status: 'pending' },
    { id: 'victim-verified', user_id: 'victim', channel: 'email', target: EMAIL, status: 'verified' },
  ]
  const extractParams = (cond: unknown): unknown[] => {
    const out: unknown[] = []
    const walk = (n: any) => {
      if (!n || typeof n !== 'object') return
      if ('value' in n && !('queryChunks' in n)) out.push(n.value)
      if (Array.isArray(n.queryChunks)) n.queryChunks.forEach(walk)
    }
    walk(cond)
    return out
  }
  const findFirst = vi.fn(async ({ where }: { where: unknown }) => {
    const email = extractParams(where).find((p) => typeof p === 'string')
    return rows.find((r) => r.target === email) ?? null
  })
  const updatedIds: string[] = []
  const updateWhere = vi.fn(async (cond: unknown) => {
    const id = extractParams(cond).find((p) => typeof p === 'string') as string
    updatedIds.push(id)
    const row = rows.find((r) => r.id === id)
    if (row) row.status = pendingSet.status
  })
  const pendingSet: { status: string } = { status: '' }
  const updateSet = vi.fn((values: { status: string }) => {
    pendingSet.status = values.status
    return { where: updateWhere }
  })
  const db = { query: { userChannels: { findFirst } }, update: vi.fn(() => ({ set: updateSet })) }
  return { EMAIL, rows, findFirst, updatedIds, db, getDatabase: vi.fn(() => db) }
})

vi.mock('#core/database/index.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import('#core/database/index.js')>()
  return { ...actual, getDatabase: mocks.getDatabase }
})

import sendgridApp from '#app/routes/webhook/sendgrid.js'

const toBase64 = (value: ArrayBuffer) => btoa(String.fromCharCode(...new Uint8Array(value)))
const sign = async (body: string, timestamp: string) => {
  const keyPair = (await crypto.subtle.generateKey({ name: 'ECDSA', namedCurve: 'P-256' }, true, ['sign', 'verify'])) as CryptoKeyPair
  const publicKey = (await crypto.subtle.exportKey('spki', keyPair.publicKey)) as ArrayBuffer
  const signature = await crypto.subtle.sign({ name: 'ECDSA', hash: 'SHA-256' }, keyPair.privateKey, new TextEncoder().encode(timestamp + body))
  return { publicKey: toBase64(publicKey), signature: toBase64(signature) }
}

describe('M07 SendGrid event lands on the first row sharing the email', () => {
  beforeEach(() => vi.clearAllMocks())

  it("routes the victim's unsubscribe to the attacker's older pending row", async () => {
    const timestamp = String(Math.floor(Date.now() / 1000))
    const body = JSON.stringify([
      { email: mocks.EMAIL, event: 'unsubscribe', timestamp: Number(timestamp), sg_event_id: 'evt-1' },
    ])
    const signed = await sign(body, timestamp)
    const env = { SENDGRID_WEBHOOK_VERIFICATION_KEY: signed.publicKey } as CloudflareBindings

    console.log('1. rows for', mocks.EMAIL, mocks.rows.map((r) => `${r.id}:${r.status}`).join(', '))
    const res = await sendgridApp.request('/sendgrid/events', {
      method: 'POST',
      body,
      headers: {
        'Content-Type': 'application/json',
        'X-Twilio-Email-Event-Webhook-Signature': signed.signature,
        'X-Twilio-Email-Event-Webhook-Timestamp': timestamp,
      },
    }, env)
    expect(res.status).toBe(200)
    console.log('2. signed unsubscribe for', mocks.EMAIL, 'accepted; updated ids =', mocks.updatedIds)

    expect(mocks.updatedIds).toEqual(['attacker-pending'])
    const attacker = mocks.rows.find((r) => r.id === 'attacker-pending')!
    const victim = mocks.rows.find((r) => r.id === 'victim-verified')!
    expect(attacker.status).toBe('unsubscribed')
    expect(victim.status).toBe('verified')
    console.log(`3. attacker row -> ${attacker.status}, victim row stays ${victim.status}: victim keeps receiving email`)
  })
})
```
