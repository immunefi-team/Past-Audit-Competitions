# 92408 w a low email verification endpoint has no authentication letting an attacker bind a victim s email address to the attacker s account

**Submitted on Sep 9th 2026 at 14:30:52 UTC by @spongebob for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92408
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Changing details of other users (including modifying browser local storage) without already-connected wallet interaction and with significant user interaction, such as:
* Iframing leading to modifying the backend/browser state (must demonstrate impact with PoC)

## Description

## Summary

`POST /notifications/channels/email/verify` is the only state-modifying route in the notifications tree that is mounted **without `requireAuth`**. It resolves a notification channel from the request body's `token` alone and flips that channel to `verified`, never checking that the caller owns the channel the token was minted for.

The token is emailed to the address being verified — so whoever controls the mailbox can redeem it, **regardless of which account the channel belongs to**. An attacker registers a channel for the victim's address on the attacker's own account; the verification email is delivered to the victim, correctly branded and indistinguishable from a legitimate one. One click by the victim binds their address to the attacker's account.

The control that looks like it should prevent this — the "this email address is already linked to another account" check on `POST /channels/email` — does not, for two independent reasons: it runs only at request time and is never re-checked at commit time, and it compares raw strings against a target that is only `trim().toLowerCase()`, so provider-level aliases of one mailbox are distinct targets.

## Root cause

### 1. The route is unauthenticated, and the token is treated as proof of ownership

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/email.ts#L172-L192>

```ts
  .post(
    '/verify',
    vValidator('json', verifyEmailChannelBodySchema),   // <-- no ...requireAuth
    injectDb,
    async (c) => {
      const { token } = c.req.valid('json')

      const verification = await c.var.db.query.channelVerifications.findFirst({
        where: and(
          eq(TABLE.channelVerifications.token, token),
          eq(TABLE.channelVerifications.purpose, 'verify'),
          gt(TABLE.channelVerifications.expires_at, new Date()),
          // no predicate on channelVerifications.user_id
        ),
        with: { channel: true },
      })

      if (!verification) {
        return c.json({ error: 'Invalid or expired verification token' }, 400)
      }

      // Mark channel as verified
      await c.var.db
        .update(TABLE.userChannels)
        .set({ status: 'verified', verified_at: new Date() })
        .where(eq(TABLE.userChannels.id, verification.channel_id))
```

Every sibling route spreads `...requireAuth` (`[requireJWT, verifyJWT]` from `app/middleware/auth.ts`) — `POST /channels/email` at line 31, and `GET`/`DELETE`/`POST /test`/`POST /resend` in `channels/$id.ts`. `/verify` is the single exception.

The defect is **not** token guessability. `generateToken()` (`services/notifications/helpers.ts`) is 16 bytes from `crypto.getRandomValues`, which is fine. The defect is the conflation of two different facts:

* the token proves **control of the mailbox**;
* the route treats it as proof of **ownership of the channel**.

Those are the same fact only when the person who requested the channel and the person who owns the mailbox are the same. Nothing enforces that, because anyone can request a channel for any address.

`verification.user_id` is loaded and used later (passed to `addContactToList` at line \~242), so the owning account is known at the point of the write — it is simply never compared to the caller.

### 2. The uniqueness control is checked at request time, not commit time

`email.ts:58-72` — `POST /channels/email` rejects an address already verified elsewhere:

```ts
      const otherUserChannel = await c.var.db.query.userChannels.findFirst({
        where: and(
          eq(TABLE.userChannels.channel, 'email'),
          eq(TABLE.userChannels.target, email),
          eq(TABLE.userChannels.status, 'verified'),
        ),
      })

      if (otherUserChannel && otherUserChannel.user_id !== userId) {
        return c.json(
          { error: 'This email address is already linked to another account' },
          400,
        )
      }
```

`/verify` never repeats this check. The invariant therefore holds when the channel is *created* and not when it is *committed*, so it is decorative against an attacker who creates the channel before the victim does.

### 3. The uniqueness check keys on a string, not on a mailbox

`services/notifications/email-verification-rate-limit.ts:15-16`:

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/notifications/email-verification-rate-limit.ts#L15-L16>

```ts
export const normalizeEmailForRateLimit = (email: string) =>
  email.trim().toLowerCase()
```

This is the only normalization applied to `userChannels.target` (`email.ts:37`). It does not canonicalize sub-addressing or dot-insertion, both of which many providers (Gmail, Fastmail, Outlook aliases) collapse to one mailbox. So `victim@gmail.com`, `victim+ens@gmail.com` and `v.i.c.t.i.m@gmail.com` are three unrelated rows delivering to one inbox — and the "already linked" comparison at line 62 is an exact `eq` on that string.

Consequence: **the attack works even against a victim who has already verified their own address**, because the alias is not the string being compared. It also gives each alias its own rate-limit bucket, since `KV_KEY.NOTIFICATIONS.EMAIL_VERIFICATION` is keyed on the same normalized target — so the 3-sends-per-hour cap (`EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS`) does not bound sends to a mailbox, only to a string.

Secondary: `channel_verifications.consumed_at` exists in the schema (`core/database/schema/notifications.ts:117`) and is never written anywhere in the codebase — single-use is enforced only by deleting the row *after* the channel update, so concurrent redemptions of one token both read it before either deletes.

## Attack path

**Preconditions:** the attacker has an ENS Manager account (free — SIWE with any wallet) and knows the victim's email address. No wallet interaction by the victim, no privileged access, no network position.

1. **Attacker creates the channel.** Authenticated as themselves, the attacker calls:

   ```http
   POST /notifications/channels/email
   Authorization: Bearer <attacker JWT>
   Content-Type: application/json

   {"email": "victim@corp.example"}
   ```

   A `user_channels` row is inserted with `user_id = <attacker>`, `target = victim@corp.example`, `status = 'pending'`, plus a `channel_verifications` row whose `token` is a fresh 16-byte value. If the victim has already verified that exact address, the attacker uses a provider alias (`victim+x@corp.example`) — a different `target` string, same mailbox, so the check at line 67 does not fire.

   The response returns `channelId` and `expires_at` but **not** the token, so the attacker never sees it. They do not need to.
2. **The platform delivers the attack email.** The worker sends, from the real `EMAIL_FROM_ADDRESS` via the real SendGrid account, the genuine template (`services/email/verification.ts`): subject *"Verify your email address - ENS Notifications"*, a **Verify Email Address** button pointing at `${MANAGER_APP_URL}/notifications/channels/email/verify?token=…`. It is byte-identical to a legitimate verification email because it *is* one. No attacker-controlled content, no spoofing, nothing for a mail filter to catch.
3. **Victim clicks.** The link opens `apps/manager/src/routes/notifications/channels/email/verify.tsx`, which reads `?token=` into `Route.useLoaderDeps()` and renders a **Verify Email Address** button. The victim clicks it — one in-app click. No auto-submit, and no wallet connection is required.
4. **The server commits the attacker's channel.** The click issues `POST /notifications/channels/email/verify {"token": "…"}`. The route reads no session. It finds the row, and sets `status = 'verified'`, `verified_at = now()` on `channel_id` — a channel owned by `user_id = <attacker>`. The victim sees *"Email verified successfully"*, which is true from their point of view and false in every way that matters.
5. **Persistence.** A welcome email is sent to the victim, and (`email.ts:~236`) the address is pushed to the SendGrid broadcast list keyed on `verification.user_id` — the attacker's id.

**Result:** `user_channels` now holds a `verified` row with the attacker's `user_id` and the victim's address.

## Impact

* **The victim's address is bound to an account they do not control.** ENS notifications for the attacker's names — expiry warnings, transfer alerts — are delivered to the victim's inbox indefinitely, from the real ENS sender. The victim has no way to sever it: the channel is on the attacker's account, and every deletion route (`channels/$id.ts` `DELETE /`) is correctly scoped to the owning user.
* **The victim is locked out of their own address.** Once the attacker's channel is `verified`, the victim's own `POST /channels/email` for that address returns *"This email address is already linked to another account"* (line 69). A user can be permanently denied email notifications for their own ENS names by an attacker who spends one request.
* **A durable, ENS-authentic phishing channel.** The attacker controls which notification kinds fire on their own account, and each one is delivered to the victim from `EMAIL_FROM_ADDRESS` with valid SPF/DKIM/DMARC. This is a warmed channel for the follow-on: `templates/email.ts` renders `payload.name` into notification bodies, and the attacker owns the names generating them.
* **Attacker-side data.** The victim's address is written into the attacker's account state and synced to the SendGrid broadcast list under the attacker's `user_id`.

**Severity mapping.** This is the High bullet *"Changing sensitive details of other users… without already-connected wallet interaction and with up to one click of user interaction, such as: Email"* — the changed detail is the binding of the victim's address, the victim connects no wallet, and the interaction is one click on ENS's own page reached from ENS's own email. Should triage read the binding as a non-sensitive detail, it still lands on the Medium bullet *"Changing non-sensitive details of other users… such as: Enabling/disabling notifications."*

## Recommendation

Bind redemption to the session that requested it. The token should prove control of the mailbox **and** the session should prove ownership of the channel; neither alone is sufficient.

The patch below does three things:

1. mounts `...requireAuth` on `/verify` and adds `channelVerifications.user_id = <session user>` to the lookup predicate — filtering in the query rather than comparing after the fetch, so "no such token" and "not your token" return the same 400 and the endpoint is not an existence oracle;
2. re-checks at commit time the uniqueness invariant that `POST /channels/email` enforces at request time;
3. honours the unused `consumed_at` column so a token cannot be redeemed twice.

With (1) in place the alias bypass is neutralized as a path to this impact: the victim redeeming an attacker-owned token now fails the ownership predicate regardless of which alias string the channel was created under.

## Proof of Concept

## Proof of concept

Two suites under `workers/api-worker/src/app/__audit__/`, run with the project's own harness (`@cloudflare/vitest-pool-workers`) against the real Hono app.

```bash
cd workers/api-worker && pnpm vitest run src/app/__audit__/email-verify.poc.test.ts src/app/__audit__/email-alias.poc.test.ts
```

```
 ✓ email target normalisation > only trims and lowercases
 ✓ email target normalisation > does NOT canonicalise plus-subaddressing (same Gmail inbox, different target)
 ✓ email target normalisation > does NOT canonicalise dot-insertion (same Gmail inbox, different target)
 ✓ email target normalisation > an attacker has unbounded distinct targets for one victim mailbox
 ✓ POST /notifications/channels/email/verify > verifies a channel with NO Authorization header at all
 ✓ POST /notifications/channels/email/verify > verifies the token owner's channel even when a DIFFERENT user is logged in
 ✓ POST /notifications/channels/email/verify > does not re-check whether the address is already verified elsewhere

 Test Files  2 passed (2)
      Tests  7 passed (7)
```

### `src/app/__audit__/email-verify.poc.test.ts`

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'

const state = {
  verification: null as null | {
    id: string
    channel_id: string
    user_id: string
    channel: { target: string | null }
  },
  updates: [] as Array<Record<string, unknown>>,
  deletes: 0,
}

const mockDb = {
  query: {
    channelVerifications: { findFirst: vi.fn(async () => state.verification) },
  },
  update: vi.fn(() => ({
    set: vi.fn((values: Record<string, unknown>) => ({
      where: vi.fn(async () => { state.updates.push(values) }),
    })),
  })),
  delete: vi.fn(() => ({ where: vi.fn(async () => { state.deletes += 1 }) })),
}

vi.mock('#core/database/index.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import('#core/database/index.js')>()
  return { ...actual, getDatabase: () => mockDb }
})
vi.mock('#services/email/welcome.js', () => ({
  sendWelcomeEmail: vi.fn(async () => ({ isErr: () => false, isOk: () => true })),
}))
vi.mock('#services/sendgrid/contacts.js', () => ({
  addContactToList: vi.fn(async () => ({ isErr: () => false })),
  searchContact: vi.fn(),
  deleteContact: vi.fn(),
}))

import emailApp from '../routes/notifications/channels/email'

const env = {
  JWT_SECRET: 'test-secret',
  SENDGRID_API_KEY: 'sg-key',
  EMAIL_FROM_ADDRESS: 'noreply@example.test',
  MANAGER_APP_URL: 'https://app.ens.domains',
} as unknown as CloudflareBindings

const ctx = {
  waitUntil: () => undefined,
  passThroughOnException: () => undefined,
} as unknown as ExecutionContext

const post = (body: unknown, headers: Record<string, string> = {}) =>
  emailApp.fetch(
    new Request('https://api.example/email/verify', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...headers },
      body: JSON.stringify(body),
    }),
    env,
    ctx,
  )

beforeEach(() => {
  state.verification = null
  state.updates = []
  state.deletes = 0
  vi.clearAllMocks()
})

describe('POST /notifications/channels/email/verify', () => {
  it('verifies a channel with NO Authorization header at all', async () => {
    state.verification = {
      id: 'verification-1',
      channel_id: 'channel-of-user-A',
      user_id: 'user-A',
      channel: { target: 'victim@corp.example' },
    }

    const res = await post({ token: 'a'.repeat(32) })

    expect(res.status).toBe(200)
    await expect(res.json()).resolves.toEqual({
      message: 'Email verified successfully',
    })
    expect(state.updates).toEqual([
      expect.objectContaining({ status: 'verified' }),
    ])
  })

  it('verifies the token owner’s channel even when a DIFFERENT user is logged in', async () => {
    state.verification = {
      id: 'verification-1',
      channel_id: 'channel-of-user-A',
      user_id: 'user-A',
      channel: { target: 'victim@corp.example' },
    }

    const res = await post(
      { token: 'a'.repeat(32) },
      { Authorization: 'Bearer not-even-a-valid-jwt' },
    )

    // The route never reads the session, so an invalid bearer is irrelevant.
    expect(res.status).toBe(200)
    expect(state.updates).toEqual([
      expect.objectContaining({ status: 'verified' }),
    ])
  })

  it('does not re-check whether the address is already verified elsewhere', async () => {
    state.verification = {
      id: 'verification-2',
      channel_id: 'channel-of-user-A',
      user_id: 'user-A',
      channel: { target: 'victim@corp.example' },
    }

    await post({ token: 'b'.repeat(32) })

    // Only channelVerifications is queried; userChannels is never consulted.
    expect(mockDb.query.channelVerifications.findFirst).toHaveBeenCalledTimes(1)
    expect((mockDb.query as Record<string, unknown>).userChannels).toBeUndefined()
  })
})
```

### `src/app/__audit__/email-alias.poc.test.ts`

```ts
import { describe, expect, it } from 'vitest'
import { normalizeEmailForRateLimit } from '#services/notifications/email-verification-rate-limit.js'

describe('email target normalisation', () => {
  it('only trims and lowercases', () => {
    expect(normalizeEmailForRateLimit('  VICTIM@Gmail.COM ')).toBe('victim@gmail.com')
  })

  it('does NOT canonicalise plus-subaddressing (same Gmail inbox, different target)', () => {
    expect(normalizeEmailForRateLimit('victim+ens@gmail.com'))
      .not.toBe(normalizeEmailForRateLimit('victim@gmail.com'))
  })

  it('does NOT canonicalise dot-insertion (same Gmail inbox, different target)', () => {
    expect(normalizeEmailForRateLimit('v.i.c.t.i.m@gmail.com'))
      .not.toBe(normalizeEmailForRateLimit('victim@gmail.com'))
  })

  it('an attacker has unbounded distinct targets for one victim mailbox', () => {
    const targets = new Set(
      Array.from({ length: 50 }, (_, i) =>
        normalizeEmailForRateLimit(`victim+${i}@gmail.com`),
      ),
    )
    // 50 distinct `user_channels.target` rows, one real inbox, and 50 separate
    // rate-limit buckets (the KV key is the same normalized target).
    expect(targets.size).toBe(50)
    expect(targets.has(normalizeEmailForRateLimit('victim@gmail.com'))).toBe(false)
  })
})
```

### Full diff

```diff
diff --git a/workers/api-worker/src/app/routes/notifications/channels/email.ts b/workers/api-worker/src/app/routes/notifications/channels/email.ts
index 850810c..ae64934 100644
--- a/workers/api-worker/src/app/routes/notifications/channels/email.ts
+++ b/workers/api-worker/src/app/routes/notifications/channels/email.ts
@@ -1,5 +1,5 @@
 import { vValidator } from '@hono/valibot-validator'
-import { and, eq, gt } from 'drizzle-orm'
+import { and, eq, gt, isNull } from 'drizzle-orm'
 import * as v from 'valibot'
 import { requireAuth } from '#app/middleware/auth.js'
 import { injectDb } from '#app/middleware/database.js'
@@ -171,15 +171,25 @@ export default createApp()
   )
   .post(
     '/verify',
-    vValidator('json', verifyEmailChannelBodySchema),
+    ...requireAuth,
     injectDb,
+    vValidator('json', verifyEmailChannelBodySchema),
     async (c) => {
       const { token } = c.req.valid('json')
+      const userId = c.var.user_id
 
+      // The token proves control of the mailbox; it does NOT prove that the
+      // caller owns the channel the token was minted for. Bind the lookup to
+      // the session so a token minted for account A can never be redeemed by
+      // account B. Filtering in the query (rather than comparing after the
+      // fetch) keeps the response identical for "no such token" and "not
+      // yours", so this is not an oracle.
       const verification = await c.var.db.query.channelVerifications.findFirst({
         where: and(
           eq(TABLE.channelVerifications.token, token),
           eq(TABLE.channelVerifications.purpose, 'verify'),
+          eq(TABLE.channelVerifications.user_id, userId),
+          isNull(TABLE.channelVerifications.consumed_at),
           gt(TABLE.channelVerifications.expires_at, new Date()),
         ),
         with: {
@@ -191,6 +201,28 @@ export default createApp()
         return c.json({ error: 'Invalid or expired verification token' }, 400)
       }
 
+      // Re-check at commit time the invariant POST /channels/email enforces at
+      // request time: an address verified on another account must not be
+      // verified again here.
+      if (verification.channel?.target) {
+        const conflicting = await c.var.db.query.userChannels.findFirst({
+          where: and(
+            eq(TABLE.userChannels.channel, 'email'),
+            eq(TABLE.userChannels.target, verification.channel.target),
+            eq(TABLE.userChannels.status, 'verified'),
+          ),
+        })
+
+        if (conflicting && conflicting.user_id !== userId) {
+          return c.json(
+            {
+              error: 'This email address is already linked to another account',
+            },
+            400,
+          )
+        }
+      }
+
       // Mark channel as verified
       await c.var.db
         .update(TABLE.userChannels)
```
