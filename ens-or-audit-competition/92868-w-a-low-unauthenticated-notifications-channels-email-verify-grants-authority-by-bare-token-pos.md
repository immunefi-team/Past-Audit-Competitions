# 92868 w a low unauthenticated notifications channels email verify grants authority by bare token possession cross user email channel hijack and permanent victim lockout

**Submitted on Sep 12th 2026 at 22:53:50 UTC by @rector for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92868
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email
  * Improperly disclosing confidential user information, such as: Email address

## Description

## Brief/Intro

The api-worker's email-channel verification endpoint (`POST /notifications/channels/email/verify`) is **unauthenticated** and grants authority by **bare possession of the emailed token string**: its handler resolves which user's channel row to flip from `token + purpose + expires_at` alone, performing **zero** cross-user or row-state re-checks. Because the request-time exclusivity check reads only *verified* rows, an attacker can hold a *pending* row for any victim's email address; when the victim clicks the genuine verification link that ENS's own SendGrid infrastructure sends them, the **attacker's** row flips to `verified` — permanently blocking the victim from ever binding their own email address and redirecting ENS-attributed email into the victim's inbox under the attacker's account. Severity: High, per the program's in-scope row "Changing sensitive details of other users … without already-connected wallet interaction and with up to one click of user interaction, such as: Email" (with the sibling row "Improperly disclosing confidential user information, such as: Email address" covering the notification-redirection payoff).

## Vulnerability Details

Verification authority is **bare possession of the emailed token string**. Three independent gaps compose:

1. **The consume route has no authentication and no ownership re-check.** `/channels/email/verify` is mounted without `requireAuth` (route chain: `channels/index.ts:53` → `notifications/index.ts:29-31` → `app/index.ts:19`, with no global middleware in `hono.ts:14-24`; the Manager page is a public root-level route, `routeTree.gen.ts:163-167,522-529`), and its handler resolves authority from `token + purpose + expires_at` alone (`email.ts:179-188`) — the consume path (`email.ts:179-206`) performs **zero** `user_channels` reads: no user identity, no row state, no cross-user check before flipping whichever row the token points at (`email.ts:195-201`).
2. **Request-time exclusivity reads verified rows only.** The add route blocks an email only if another user holds a **verified** row (`email.ts:59-65`), so two different users can legally hold rows for the same inbox while pending (per-user uniqueness only: `unique(user_id, channel, target)`, `schema/notifications.ts:73-79`).
3. **Minting never supersedes and nothing rate-limits consumption.** Both the add route and the resend route mint a fresh 128-bit token without invalidating prior live tokens (`$id.ts:190-205`); `attempts` is inserted as 0 (`email.ts:113`) and has no writers repo-wide, as do `verification_attempts` and `consumed_at`; the verify endpoint is unthrottled. A token stays fully authoritative until expiry (24 h, `email.ts:112`) or until its own row is consumed — "consumption" is per-token-row, not per-channel.

## Vulnerable code (branch permalinks)

**1. The consume route — no auth, token-only authority, unconditional flip** — [`workers/api-worker/src/app/routes/notifications/channels/email.ts#L172-L206`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/workers/api-worker/src/app/routes/notifications/channels/email.ts#L172-L206):

```ts
.post(
  '/verify',
  vValidator('json', verifyEmailChannelBodySchema),
  injectDb,                                                  // <-- no `...requireAuth` — contrast the add route (email.ts:31)
  async (c) => {
    const { token } = c.req.valid('json')

    const verification = await c.var.db.query.channelVerifications.findFirst({
      where: and(
        eq(TABLE.channelVerifications.token, token),         // <-- authority = possession of the emailed string
        eq(TABLE.channelVerifications.purpose, 'verify'),
        gt(TABLE.channelVerifications.expires_at, new Date()),
      ),                                                     // <-- no user_id, no status, no consumed flag anywhere in the clause
      with: { channel: true },
    })

    // … invalid-token 400 (email.ts:190-192) …

    // Mark channel as verified
    await c.var.db
      .update(TABLE.userChannels)
      .set({ status: 'verified', verified_at: new Date() })
      .where(eq(TABLE.userChannels.id, verification.channel_id))   // <-- flips whichever user's row the token points at: zero cross-user re-check
```

**2. Enabler — the exclusivity check reads verified rows only** — [`workers/api-worker/src/app/routes/notifications/channels/email.ts#L58-L72`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/workers/api-worker/src/app/routes/notifications/channels/email.ts#L58-L72):

```ts
// Check if email is linked to another user
const otherUserChannel = await c.var.db.query.userChannels.findFirst({
  where: and(
    eq(TABLE.userChannels.channel, 'email'),
    eq(TABLE.userChannels.target, email),
    eq(TABLE.userChannels.status, 'verified'),               // <-- a pending row for the same email is invisible to this check
  ),
})

if (otherUserChannel && otherUserChannel.user_id !== userId) {
  return c.json(
    { error: 'This email address is already linked to another account' },
    400,
  )
}
```

**3. Enabler — minting never supersedes** — [`workers/api-worker/src/app/routes/notifications/channels/$id.ts#L189-L205`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/workers/api-worker/src/app/routes/notifications/channels/%24id.ts#L189-L205):

```ts
// Create new verification token
const verification = await c.var.db
  .insert(TABLE.channelVerifications)
  .values({
    user_id: userId,
    channel_id: channelId,
    channel: channel.channel,
    purpose: 'verify',
    token: generateToken(),                                  // <-- new 128-bit token; prior live tokens for the channel are untouched
    expires_at: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 hours
    attempts: 0,                                             // <-- never incremented anywhere; `consumed_at` has no writers repo-wide
  })
```

**4. Attribution side effect of the flip** — [`workers/api-worker/src/app/routes/notifications/channels/email.ts#L233-L258`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/workers/api-worker/src/app/routes/notifications/channels/email.ts#L233-L258), fired unconditionally on every successful consume:

```ts
addContactToList(
  { SENDGRID_API_KEY: c.env.SENDGRID_API_KEY, SENDGRID_BROADCAST_LIST_ID: c.env.SENDGRID_BROADCAST_LIST_ID },
  verification.channel.target,
  verification.user_id,                                      // <-- the OWNER OF THE TOKEN's user_id: the victim's inbox is attributed to the attacker's account
)
```

**Attack path (step-by-step)**

1. Attacker signs into the Manager app with any wallet (SIWE) and calls `POST /notifications/channels/email {"email":"victim@example.com"}`. The exclusivity check passes because it reads verified rows only (gap 2) → an attacker-owned **pending** row is created for the victim's address (`email.ts:74-97`, stored normalized — trim + lowercase, `email.ts:20,37,81`, `email-verification-rate-limit.ts:15-16`).
2. ENS's own SendGrid account sends the victim a genuine verification email — authentic sender domain, authentic template (`services/email/verification.ts:26-38`) — with the 128-bit CSPRNG token in the URL (`verification.ts:16`), valid 24 h (`email.ts:112`). The attacker can mint additional tokens on demand via `/channels/:id/resend` without invalidating any (gap 3, `$id.ts:190-205`).
3. The victim clicks the email link → public Manager page `verify.tsx` (no auth wrapper; the title/description show "Verify Your Email" with **no account identity**, `verify.tsx:139-153`) → clicks "Verify Email Address" (`verify.tsx:216-225`) → the page POSTs the token to `/channels/email/verify` **unauthenticated** (gap 1, `email.ts:172-176`).
4. The handler's token lookup passes (`email.ts:179-188`) → the **attacker's** row is flipped to `verified` with the victim's address (`email.ts:195-201`) → the token row is deleted (`email.ts:204-206`) → a welcome email is sent to the victim's inbox and the victim's address is added to the SendGrid broadcast list **under the attacker's user\_id** (`email.ts:209-230,233-258`).
5. Persistent payoffs, all attacker-side-complete:
   * **Squat:** the victim's own attempt to bind their email now returns `400 "This email address is already linked to another account"` (`email.ts:67-72`) — and keeps returning it, because the block keys on the attacker's verified row. Only the attacker can release it (channel DELETE, `channels/$id.ts:62-79`).
   * **Redirection:** the attacker can point their own account's notification email at the victim's inbox, and ENS broadcast campaigns reach the victim's mailbox attributed to the attacker's account (gap 1's side effect, excerpt 4).
   * **Duplicate-verified state:** when both rows are pending, both tokens verify — two distinct users end `status:'verified'` on one inbox (proven by PoC case C2), which is precisely the precondition that any inbound per-email row matching on the shared inbox would monetize (see Impact).

## Impact Details

An attacker of any authenticated account, for the cost of one API call and one induced email, takes ownership of the victim's email identity inside the notification system and locks the victim out of their own:

* **Permanent cross-user squat.** The victim's attempts to bind their own email return 400 indefinitely (`email.ts:67-72`); release requires a channel deletion only the attacker's account can perform (`$id.ts:62-79`). The victim is denied the email notification feature — including name-expiry and transfer warnings — for as long as the squat persists, with no self-service recovery path.
* **Misattributed email identity.** The victim's inbox is verified onto the attacker's account: the attacker can direct their account's notification email into the victim's mailbox, and the victim's address is added to ENS's SendGrid broadcast list under the attacker's user\_id (`email.ts:242`), producing ENS-originated email the mailbox owner never associated with any account they control. Zombie tokens compound the confusion after a hijack: stale tokens keep returning `200 Email verified successfully` with duplicate side effects (PoC C4), complicating victim and Support diagnosis.
* * **Impact amplifier — duplicate-verified inbox state.** Case C2 ends with two distinct users simultaneously `verified` on one inbox. That is exactly the coexistence state any inbound per-email row matching (e.g. the SendGrid event webhook's unscoped `findFirst`, `sendgrid.ts:172-174`) would resolve ambiguously — on this surface, bounce/spam/unsubscribe events for the shared inbox could then mutate a row belonging to the *other* user, with zero further interaction by either party. (That inbound mechanism is a separate code path from this report and is not claimed here; the composition is noted because the C2 state this report proves is what makes it exploitable.)- **Scale and persistence.** Targets are arbitrary (any address the attacker can cause to be emailed); the per-address rate limit bounds only verification-email sends (3/hour), not victims or consumes; squats and misattributions persist until the attacker voluntarily releases them.

**Recommended fix**

Make the channel flip and the cross-user guard **one atomic statement**, supersede superseded tokens on mint, and gate the exclusivity check on pending rows as well:

```diff
// channels/email.ts — consume: lookup + cross-user guard + flip in ONE statement.
// The stack is drizzle-orm/neon-http (core/database/index.ts:4,18) — an
// HTTP-per-statement driver with NO interactive transactions and no `.for('update')`;
// a `db.transaction(...)` sketch would make every email verify throw. Atomicity must
// therefore come from a single UPDATE … WHERE NOT EXISTS … RETURNING.
- const verification = await c.var.db.query.channelVerifications.findFirst({ … token+purpose+expires_at … })
- await c.var.db
-   .update(TABLE.userChannels)
-   .set({ status: 'verified', verified_at: new Date() })
-   .where(eq(TABLE.userChannels.id, verification.channel_id))
+ const flip = await c.var.db.execute(sql`
+   UPDATE user_channels uc
+   SET status = 'verified', verified_at = now()
+   WHERE uc.id = (
+     SELECT cv.channel_id FROM channel_verifications cv
+     WHERE cv.token = ${token} AND cv.purpose = 'verify' AND cv.expires_at > now()
+   )
+   AND NOT EXISTS (
+     SELECT 1 FROM user_channels other
+     WHERE other.channel = 'email'
+       AND other.target = uc.target
+       AND other.status = 'verified'
+       AND other.id <> uc.id
+   )
+   RETURNING uc.id
+ `)
+ if (flip.rows.length === 0) {
+   return c.json({ error: 'Invalid or expired verification token, or email already linked to another account' }, 409)
+ }
```

Zero rows returned distinguishes neither failure mode by itself; if the UX needs the distinction, a follow-up read on the token row (still safe — the `NOT EXISTS` guard is what enforces the invariant, and a failed flip leaves the token burnable) can separate "invalid/expired" from "already held". Burn the consumed token row after the flip, and — per the zombie-token finding (PoC C4) — do not leave sibling live tokens behind:

```diff
// mint sites: add route (email.ts:104-124) AND resend ($id.ts:190-205) — supersede on mint
+ await c.var.db
+   .delete(TABLE.channelVerifications)
+   .where(and(
+     eq(TABLE.channelVerifications.channel_id, channel.id),
+     ne(TABLE.channelVerifications.token, newToken),
+   ))
```

Supersession has a known behavior tradeoff, accepted deliberately: it voids the user's own still-in-flight token from a *prior* request (requested on phone, re-requested on laptop, then clicks the first email → 400). Closing the token window is worth that regression; alternatively keep multiple live tokens but add a consumed/status predicate to the lookup so a token for an already-verified channel is rejected.

Finally, tighten the request-time exclusivity (`email.ts:59-65`) to consider **pending** rows of other users too (409 with a re-verification path), so coexistence is refused at bind time and the consume-time guard never has to arbitrate a race it can only lose one side of.

## Known-issue position (no overlap with the disclosure)

Checked against the program's full 55-row known-issues disclosure :

* **No known-issue row covers this surface.** The disclosure contains no notifications/email-channel entry at all; this finding does **not** rely on the QA-01/QA-03-style carve-outs (those belong to the role-combo and price-display findings respectively). What carries the dup position is the carve-out *posture* those rows exemplify — the program's known-issue entries explicitly carve out "a demonstrated security consequence … would be a new finding", and this filing ships exactly that: a demonstrated cross-user hijack with a 4/4 PoC on real handler code.
* **R2-05** (the only prior api-worker item; SendGrid webhook failing open when the verification key is unset, fix already shipped): distinct route (inbound webhook vs the verify consume) and distinct root (auth-config fail-open vs absent consume-time enforcement); the shipped R2-05 fix never touched `channels/email.ts` or `$id.ts`.
* The SendGrid webhook/DLQ row-matching surface: distinct route and distinct root (inbound event trust vs outbound token possession); no shipped fix in the disclosure ever touched `channels/email.ts` or `channels/$id.ts`.
* **QA-02/WEB-699, EXP-INPUT-009:** different subsystems (wallet-connect session UI; input validation classes) — no overlap.
* The High row's "(e.g. email)" wording is a severity-tier question (the NV above), not a known-issue fence.

**Tools used**

Manual source review, adversarially cross-checked (design-space alternatives, attack-direction review, stand-alone correctness pass), vitest PoC on the real route modules (`@cloudflare/vitest-pool-workers`, real HS256 `requireAuth` via the repo's `signJWT`, real `generateToken`, drizzle-clause-evaluating DB spy), drizzle-orm 0.45.2 driver-capability verification for the fix shape (neon-http transaction limitation), SendGrid email-template verification from source.

## References

* Vulnerable route: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/workers/api-worker/src/app/routes/notifications/channels/email.ts> (verify handler L172–L206; exclusivity L58–L72; attribution L233–L258)
* Token minting without supersession: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/workers/api-worker/src/app/routes/notifications/channels/%24id.ts> (L189–L205)
* Public verify page (no account identity shown): <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/routes/notifications/channels/email/verify.tsx> (L139–L153, L216–L225)
* Uniqueness constraint (per-user only): <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/workers/api-worker/src/app/schema/notifications.ts> (L73–L79)
* Live check (read-only, stateless, 2026-09-13): `POST https://app-api.ens.dev/notifications/channels/email/verify` with an arbitrary token returns `400 {"error":"Invalid or expired verification token"}` with **no Authorization header** — the route is reachable unauthenticated on production; the add route (`POST .../channels/email`) returns 401 unauthenticated, confirming the intended auth split.

## Proof of Concept

## Proof of concept

`poc-email-token.test.ts` (vitest 4.1.5, `@cloudflare/vitest-pool-workers`). Runs the **real** route modules (`email.ts` add + verify, `$id.ts` resend) with the **real** `requireAuth` (HS256 JWT signed through the repo's own `signJWT`), real valibot validation, real `injectDb`, and the real `generateToken()` 128-bit CSPRNG. Stubbed, with justification: the DB edge (Neon Postgres unreachable locally; the finding is precisely about *which queries the handlers issue*, so a stateful in-memory emulator evaluates the handlers' real drizzle `where` clauses — column identity via `getTableColumns`, `and`/`eq`/`gt` chunk walking — and records every query into a spy log, making assertions against what the handlers actually asked the database) and the SendGrid HTTP boundary (external API; recipient/token/list arguments captured so every claim about what would be sent is evidence-backed).

Reproduce:

```bash
cd <repo>/workers/api-worker   # at branch audit-comp-ready, commit 1c9b47f (Node >= 22)
corepack pnpm vitest run src/app/routes/notifications/channels/poc-email-token.test.ts
```

Run log: `Test Files 1 passed (1) · Tests 4 passed (4)` — deterministic across repeated runs.

* **C1 — cross-user hijack:** attacker (user A) adds `victim@example.com` while the email is unbound; the spy confirms the exclusivity query's constraint set is exactly `channel + status + target` (no `user_id`). A real 128-bit token is minted and "emailed" to the victim. The victim's click — one `POST /email/verify` with **no Authorization header** (none exists) — returns `200 Email verified successfully`, flips the **attacker's** row to `verified`, burns the token, and the DB spy proves the consume path ran **zero** `userChannels` reads: every matched row belongs to user A, no constraint value references user B anywhere, and the authoritative mutation is exactly `eq(userChannels.id, <attacker row>)`. The victim's email then lands in the broadcast list attributed to user A (spy-asserted `addContactToList(..., victim, USER_A)`).
* **C2 — duplicate-verified:** a second add of the same email by user B passes the exclusivity check (spy: the verified-only query matched `[]` because A's row is pending — `email.ts:59-65` is the gap); two live tokens coexist. Both unauthenticated consumes return 200 and **both rows end `status:'verified'` for the same inbox, with distinct `user_id`s** — each consume mutated exactly its own row; nothing prevented the pair.
* **C3 — permanent block:** after C1's hijack, the victim adds their *own* email to their *own* account → `400 {"error":"This email address is already linked to another account"}` (`email.ts:67-72`); the spy shows the block fired from the **attacker's** verified row (B has no row at all) — a persistent squat only the attacker can release (`$id.ts:62-79`).
* **C4 — zombie token (bonus):** resend mints token2 without invalidating token1; after token1 verifies and burns, the **stale** token2 re-verifies the already-verified row — a misleading `200`, `verified_at` re-set, duplicate welcome email and duplicate broadcast-list add (worker log confirms the doubled side effects). The lookup constraints are exactly `token + purpose + expires_at` — no status/consumption join — while the burned token1 meanwhile 400s: consumption is per-token-row only.

Plausibility: step 1 is one authenticated call; the verification email is indistinguishable from the flow the victim opted into (authentic sender, authentic template, matching CTA); the consume is one unauthenticated request. Magnitude: any authenticated account can hijack the email channel of **any** address it can cause a verification email for — arbitrary victim counts (the 3/h limiter is per address), persistent until the attacker voluntarily releases the squat, with the victim locked out of their own binding throughout.

## Regression test

The three regression cases below are the PoC cases C1–C3, verbatim and drop-in against the same harness (real route modules + real JWT signing as described above; the spy records every emitted query). They pass today, demonstrating the bug; after the fix in the Recommendation they define the contract: C1's consume must be refused (no flip, no side effects), C2's second bind must be refused (or its consume must refuse the second flip), C3's victim add must succeed.

```ts
it('C1 — unauthenticated consume verifies the ATTACKER row; no query ever references user B; email lands in attacker-attributed broadcast list', async () => {
  // Setup through REAL handlers: attacker adds victim@x.com while unbound —
  // request-time exclusivity (email.ts:59-65) sees no VERIFIED row → passes.
  const token = await attackerAddsVictimEmail()
  const attackerRow = store.channels.find((r) => r.user_id === USER_A)!
  expect(attackerRow.status).toBe('pending')
  expect(store.verifications).toHaveLength(1)

  // Request-time check ran and matched nothing BECAUSE it reads verified rows only
  expect(exclusivityQuery().constraints.map((c) => c.colKey).sort()).toEqual(['channel', 'status', 'target']) // no user_id scoping

  ops.length = 0 // ← spy window: only the victim's consume click follows

  // The victim's click: NO auth header, token from the email URL.
  const res = await verifyRequest(token)

  expect(res.status).toBe(200)
  expect(await res.json()).toEqual({ message: 'Email verified successfully' })

  // THE HIJACK: attacker's row flipped verified.
  expect(attackerRow.status).toBe('verified')
  expect(attackerRow.verified_at).toBeInstanceOf(Date)

  // Token record burned.
  expect(store.verifications).toHaveLength(0)

  // DB SPY — the consume path NEVER re-checked ownership:
  expect(ops.filter((o) => o.kind === 'findFirst' && o.tableKey === 'userChannels')).toHaveLength(0) // zero userChannels reads (email.ts:179-206)
  expect(ops.every((o) => o.matched.every((r) => r.user_id === USER_A))).toBe(true)
  const everyConstraintValue = ops.flatMap((o) => o.constraints.map((c) => String(c.value)))
  expect(everyConstraintValue).not.toContain(USER_B)
  // the authoritative mutation is exactly eq(userChannels.id, attackerRow) — nothing user-scoped exists beside it
  const update = userChannelsUpdates()
  expect(update).toHaveLength(1)
  expect(update[0].setArgs).toMatchObject({ status: 'verified' })
  expect(update[0].where).toEqual(eq(TABLE.userChannels.id, attackerRow.id))

  // Payoff: the victim's email enters the SendGrid broadcast list attributed to the ATTACKER's user_id (email.ts:233-258, esp. :242).
  await flushBroadcast()
  expect(mocks.addContactToList).toHaveBeenCalledWith(
    expect.objectContaining({ SENDGRID_BROADCAST_LIST_ID: 'list-1' }),
    VICTIM_EMAIL,
    USER_A,
  )
  expect(mocks.sendWelcomeEmail).toHaveBeenCalledWith('SG.poc', 'no-reply@ens.example', VICTIM_EMAIL, 'https://manager.example')
})

it('C2 — pending-pending coexistence → both tokens verify → BOTH rows verified for one inbox (finding-01 duplicate-verified precondition)', async () => {
  // Attacker adds first: passes (victim unbound).
  expect((await addRequest(await bearerFor(USER_A), VICTIM_EMAIL)).status).toBe(200)
  const tokenA = mocks.sendVerificationEmail.mock.calls.at(-1)![3] as string
  const attackerRow = store.channels.find((r) => r.user_id === USER_A)!

  ops.length = 0
  // Victim adds the SAME email to their OWN account: the request-time check
  // reads VERIFIED rows only — attacker's row is pending → check matches nothing.
  const bAdd = await addRequest(await bearerFor(USER_B), VICTIM_EMAIL)
  expect(bAdd.status).toBe(200)
  expect(exclusivityQuery().matched).toEqual([]) // pending attacker row invisible to the check
  const tokenB = mocks.sendVerificationEmail.mock.calls.at(-1)![3] as string
  expect(tokenB).not.toBe(tokenA)

  const victimRow = store.channels.find((r) => r.user_id === USER_B)!
  expect(victimRow.status).toBe('pending')
  expect(store.verifications).toHaveLength(2) // two live tokens coexist

  ops.length = 0
  // Both clicks verify — consume has no exclusivity/supersession either.
  expect((await verifyRequest(tokenA)).status).toBe(200)
  expect((await verifyRequest(tokenB)).status).toBe(200)

  // THE STATE FINDING 01 MONETIZES: two users, VERIFIED, same target.
  expect(attackerRow.status).toBe('verified')
  expect(victimRow.status).toBe('verified')
  expect(store.channels.filter((r) => r.target === VICTIM_EMAIL && r.status === 'verified')).toHaveLength(2)
  expect(new Set(store.channels.map((r) => r.user_id))).toEqual(new Set([USER_A, USER_B]))

  // Each consume mutated exactly its own row — but nothing prevented the pair.
  const updates = userChannelsUpdates()
  expect(updates.map((u) => u.matched[0]!.id).sort()).toEqual([attackerRow.id, victimRow.id].sort())
})

it('C3 — after the hijack, the victim’s own add attempt 400s forever (cross-user block)', async () => {
  const token = await attackerAddsVictimEmail()
  const attackerRow = store.channels.find((r) => r.user_id === USER_A)!
  expect((await verifyRequest(token)).status).toBe(200) // victim's click verified the attacker's row
  expect(attackerRow.status).toBe('verified')

  ops.length = 0
  // Victim tries to add their OWN email to their OWN account.
  const res = await addRequest(await bearerFor(USER_B), VICTIM_EMAIL)

  expect(res.status).toBe(400)
  expect(await res.json()).toEqual({ error: 'This email address is already linked to another account' }) // email.ts:67-72
  // The block fired from the ATTACKER's verified row — B has no row at all.
  expect(exclusivityQuery().matched).toEqual([attackerRow])
  expect(store.channels.some((r) => r.user_id === USER_B)).toBe(false)
})
```

The full runnable test file and its passing run log (`Test Files 1 passed (1) · Tests 4 passed (4)`, deterministic across repeated runs) are available on request.
