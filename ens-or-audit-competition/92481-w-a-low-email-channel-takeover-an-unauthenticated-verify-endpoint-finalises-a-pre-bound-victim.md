# 92481 w a low email channel takeover an unauthenticated verify endpoint finalises a pre bound victim address

**Submitted on Sep 10th 2026 at 07:23:30 UTC by @SaiyanVegeta for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92481
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email

## Description

## Bug Description

**Brief.** Adding an email notification channel is a two-step flow: `POST /notifications/channels/email` (authenticated) creates a *pending* channel and mails a verification link; `POST /notifications/channels/email/verify` finalises it. Three facts combine into an account-level defect:

1. The **conflict check that protects an email from being claimed by two accounts fires only against `verified` channels** — a *pending* row for someone else's email is accepted.
2. The **channel is written to the database under the caller's `user_id` before the verification email is sent** — so the binding exists the moment the attacker calls the endpoint.
3. The **verification endpoint carries no `requireAuth` and no ownership check** — it flips the channel to `verified` on token possession alone, regardless of who is authenticated.

An attacker (any signed-in user) can therefore register the **victim's** email under the **attacker's** account. ENS then sends the victim a genuine verification email; the victim, seeing an official ENS message about "their" email, clicks it — and that click verifies the **attacker's** channel. The victim is then locked out of registering their own address, and notifications the system directs to that address are attached to the attacker's account.

**Details (verified at HEAD, and measured end to end — see PoC).**

*The conflict check only considers verified channels* — `workers/api-worker/src/app/routes/notifications/channels/email.ts:58-72`:

```ts
const otherUserChannel = await c.var.db.query.userChannels.findFirst({
  where: and(
    eq(TABLE.userChannels.channel, 'email'),
    eq(TABLE.userChannels.target, email),
    eq(TABLE.userChannels.status, 'verified'),   // ← pending rows do not block
  ),
})
if (otherUserChannel && otherUserChannel.user_id !== userId) {
  return c.json({ error: 'This email address is already linked to another account' }, 400)
}
```

*The channel and its token are persisted before any email is sent* — `email.ts:75-119` insert the `pending` channel and the `channel_verifications` row; the verification email is only attempted afterwards at `:143-150`. So the binding survives even if the send fails.

*The verify endpoint is unauthenticated and never checks the clicker* — `email.ts:172-207`:

```ts
.post(
  '/verify',
  vValidator('json', verifyEmailChannelBodySchema),   // ← no ...requireAuth
  injectDb,
  async (c) => {
    const { token } = c.req.valid('json')
    const verification = await c.var.db.query.channelVerifications.findFirst({ where: … token … })
    …
    await c.var.db.update(TABLE.userChannels)
      .set({ status: 'verified', verified_at: new Date() })
      .where(eq(TABLE.userChannels.id, verification.channel_id))   // ← whoever's channel the token points at
```

Contrast the sibling routes in the same file, which all spread `...requireAuth` (`email.ts:32`, and every route in `channels/index.ts` e.g. `:30`). I confirmed there is **no compensating middleware**: the app mounts sub-apps with `.route('/', …)` (`app/index.ts:16-22`, `notifications/index.ts:31`, `channels/index.ts:53`), which in Hono applies **no** parent middleware to the mounted routes; the only global middleware is `cors()` (`app/index.ts:15`). So `/verify` is genuinely reachable with no JWT.

*The verify handler also attaches the victim's address to the attacker's marketing contact* — `email.ts:232-243` calls `addContactToList(target, verification.user_id)` with the attacker's `user_id`.

The verification token is 128-bit (`services/notifications/helpers.ts` `generateToken`), so this is **not** a token-guessing attack — the token reaches the attacker's benefit specifically because the victim is the one who receives and clicks it.

***

## Impact

The endpoint finalises an authenticated binding that the caller never proved they own:

* **The victim's email is verified under the attacker's account.** ENS notifications the system sends toward that address are now delivered against the attacker's `user_id`, and the address is added to the broadcast list under the attacker.
* **The victim is denied their own channel — durably and opaquely.** After takeover, the victim's `POST /email` for their own address returns `400 "already linked to another account"` (now the `verified` conflict check fires — against the attacker's row). And the victim cannot clear it: both `DELETE /channels/:id` and `GET /channels` are scoped to the caller's own `user_id` (`channels/$id.ts:69`, `:40`), so the victim can neither remove the attacker's channel nor even see that it exists. The lockout persists until the attacker or a DB admin removes it, with no diagnostic path for the victim — so, in the limit, the victim's name can lapse without the expiry alert the feature exists to provide.
* **The attacker gains an official-sender channel into the victim's inbox — measured end to end (G2).** Delivery resolves recipients per event as the name's owner ∪ its favouriters (`queues/event-ingestion.ts:85,100-101`), then delivers to each recipient's **verified** channels' targets (`:279-289`). Because the hijacked channel is `{ user_id: attacker, target: victim_email, status: verified }`, any event for a name the **attacker owns or merely favourites** produces a notification for the attacker's `user_id` that is delivered to the **victim's** inbox, from the genuine ENS sender. This was run against the real worker (see PoC, second measurement): with the takeover in place, the attacker **merely favourited** an expiring name and enabled the favourite-expiry toggle, then the real expiry-discovery cron ran — and the worker wrote a `notification_deliveries` row with `target = victim_email`, `channel = email`, owned by the **attacker's** `user_id`. That `target` is exactly the value the email service places in SendGrid's `to:` (`services/delivery/email.ts:64`). The attacker does not *read* the victim's notifications; they *drive* official ENS email into the victim's inbox on demand. **Grade note:** what is measured is the worker's routing decision — the delivery row it writes addresses the victim's inbox under the attacker's account. The one unmeasured step is SendGrid actually accepting and delivering the message, which is genuine third-party infrastructure and out of test scope.
* **Aggravation by chaining (flagged, not relied on).** That controllable official-sender channel composes with the notification-template escaping gap (the ENS name is interpolated into the Telegram/email templates without escaping — a separate goodwill item): an attacker who favourites a name crafted to carry a link can have that link delivered to the victim's inbox inside an authentic ENS message. On its own this report stands as the High channel-takeover; the chain is noted as why the takeover is worth more than a nuisance.
* **The victim's own click is the finaliser.** The verification email is authentic — same official ENS sender the product always uses — so the social step is far stronger than a phishing link: the victim is doing exactly what the product tells them to do.

This instantiates the enumerated **High** category "changing sensitive details of other users … with up to one click of user interaction … Email": the attacker causes the victim's email to be bound and verified against the attacker's account, and denies the victim the same action for themselves — entirely through the worker's own endpoints, with no on-chain component.

***

## Risk Breakdown

**Where the defect lives, and why it is fully in scope.** Cause, effect and fix are all in `workers/api-worker` (an in-scope asset). No smart contract, indexer or metadata service is involved at any step. There is no out-of-scope component on the data path to invite a scope challenge — unlike a finding whose untrusted value arrives through the indexer, this one is pure backend logic.

**Precondition, stated plainly: the attacker supplies an email they already know — the flaw discloses none.** The attack does not reveal any address; the attacker provides the target email as input (a contact, a published address, or an entry from a breach list). This is a low bar but a real precondition, and I flag it rather than let triage find it. Two facts widen it rather than narrow it: the target **need not have an ENS account** (any email can be bound; if its owner ever tries to enable ENS notifications they are pre-locked-out), and with a leaked list the attacker can bind and lock out **many** addresses at once. The change is still to *another user's* sensitive detail, which is exactly what the High category enumerates.

**The application creates the binding; it does not merely fail to remove one.** The pending channel under the attacker's `user_id` did not exist until the attacker called `POST /email`; the verified state did not exist until `/verify` was called without auth. Both are states the worker's own endpoints produce.

**Which category, and where the tier comes from.** The severity is not argued from first principles; it is read off the enumerated category the finding instantiates. Two adjacent categories bracket it, and the program's own wording draws the line on **interaction**:

* **Critical** — *"Taking state-modifying authenticated actions … on behalf of other users **without any interaction by that user**, such as: … Notification/settings changes."* This does **not** apply: completing the takeover requires the victim's click on the verification email. Before that click the victim is not yet harmed — a *pending* attacker row does not trip the (verified-only) conflict check, so the victim could still add their own address at that point.
* **High** — *"Changing sensitive details of other users … with **up to one click of user interaction**, such as: **Email**."* This applies exactly: the victim's one click flips the sensitive detail (their email, bound and verified under the attacker) and locks them out. One click ⇒ High, which is the program's own boundary, not my judgement.

**The one non-application step, stated plainly.** The victim must open the verification email. But that email is genuine and ENS-sent — materially stronger than "victim clicks an attacker link" — and the category already prices this in by allowing "up to one click." I state it rather than hide it.

**A secondary weakness alongside.** The channel and token are inserted **before** the rate-limit check (`email.ts:76-140`: insert `:76`/`:105`, rate-limit `:126`), so rate-limiting does not prevent the pending binding. Noted, not relied on.

**Not a Known Issue.** Checked against *ENS Apps: Known Security Issues* (2026-08-14): no entry concerns channel ownership, the verification endpoint's auth, or pre-verification binding. The SendGrid webhook items (`WK-*`) concern inbound delivery status, not channel creation.

**Anticipated objection: "the victim consented by clicking."** They consented to verifying **their** email for notifications, which the official message tells them is what they are doing. They did not consent to binding it to a stranger's account — and could not have, because nothing in the flow discloses that the pending channel belongs to someone else. Consent to an action the interface misdescribes is not consent to the action performed.

***

## Recommendation

Two independent fixes; the first is the root cause.

1. **Do not let a pending row bind another account's email.** This is the real fix: an email must not become bound to an account until that account has proven control of it. Concretely — treat a *pending* row as blocking too (widen the conflict check beyond `status: 'verified'`), so `POST /email` cannot create a cross-account pending binding in the first place. This kills the attack at its source and does not depend on the verification step at all.
2. **At verification time, tie the flip to proven ownership — but note that `requireAuth` alone is the wrong shape here.** Your own verify page is explicitly built for a logged-out clicker (`apps/manager/src/routes/notifications/channels/email/verify.tsx:65,70-71,232` reads `isAuthed` and branches its post-verify destination, label and rendering on it), so simply spreading `...requireAuth` onto `/verify` would break a flow you deliberately support. The durable guard is (1): if a pending row can only ever belong to the address's real owner, then token possession is already equivalent to ownership and the flip is safe without authenticating the clicker.

   **What the negative control actually measured, precisely.** It models an ownership comparison at verify time — the clicker's `user_id` against `channel.user_id` — and with that guard in place the attacker's channel is *not* verified by the victim's click. So it demonstrates the **principle** (binding the finaliser's identity to the channel owner defeats the takeover); it does **not** validate fix (1), which is reasoned from source and was not separately measured. I separate the two rather than let the negative control appear to cover both.

Also move the channel/token insert to **after** the rate-limit check so a rate-limited caller cannot leave a pending binding behind.

***

## References

* `workers/api-worker/src/app/routes/notifications/channels/email.ts:58-72` (verified-only conflict check), `:75-119` (channel+token persisted pre-send), `:143-150` (email sent after), `:172-207` (unauthenticated verify + status flip), `:232-243` (broadcast attach under caller's user\_id)
* `workers/api-worker/src/app/index.ts:15-22`, `.../notifications/index.ts:31`, `.../channels/index.ts:53` (mounting confirms no compensating auth middleware)
* `workers/api-worker/src/app/middleware/auth.ts:35` (`requireAuth = [requireJWT, verifyJWT]`, HS256)
* `workers/api-worker/src/services/notifications/helpers.ts` (`generateToken`, 128-bit)

***

## Proof of Concept

## Proof of Concept

All testing is local. The **real api-worker** was run unmodified via its own `pnpm dev` (Cloudflare Vite plugin / workerd) on `:2999`, reading a local `.dev.vars`. Its database calls (`@neondatabase/serverless` → the repo's own `db.localtest.me:4444/sql` dev endpoint, `core/database/index.ts:10`) were served by an embedded **pglite** Postgres behind a shim that speaks the neon `/sql` protocol. **The only substitutions are the DB transport (pglite instead of the repo's containerised Postgres, because Docker was unavailable) and the unreachable SendGrid sandbox; neither touches the route logic under test.** The schema is the repo's own five drizzle migrations (`drizzle/0000…0004`), with `uuid_generate_v7()` provided as a pure-SQL function since the `pg_uuidv7` extension is not in pglite — this only affects primary-key generation, not the logic.

Two JWTs were forged with the `JWT_SECRET` the worker loaded — the ordinary authenticated-user credential both parties legitimately hold. No token was guessed.

**Scope of the measurement, stated up front.** Two things are measured against the real worker: **(A) the takeover** — the attacker binds the victim's email, `/verify` is accepted with no auth, the channel becomes verified under the attacker, the victim is locked out; and **(B) the routing** — with the takeover in place, the attacker favourites an expiring name and the real expiry-discovery cron makes the worker write an email delivery addressed to the victim's inbox under the attacker's account. Both run through the real worker's HTTP endpoints, cron handler and queue consumers against a real database. Two steps are *not* executed and are marked as such: the "victim receives the email" input is bridged by reading the persisted token from the database (standing in for the click), and **SendGrid's actual acceptance/delivery of the message is out of test scope** — it is genuine third-party infrastructure. The routing in run (B) is therefore measured where it is decided — the `notification_deliveries` row the worker writes, whose `target` is the victim's inbox and whose owner is the attacker's `user_id`.

### Result (single clean run, fresh DB)

```json
{
  "step1_attacker_adds_victim_email": { "status": 500, "body": "Failed to send verification email" },
  "channels_after_add":  [{ "user_id": "…attacker…", "target": "victim.real.person@example.com", "status": "pending" }],
  "token_belongs_to":    "…attacker…",
  "step2_verify_no_auth": { "status": 200, "body": "Email verified successfully" },
  "channels_after_verify":[{ "user_id": "…attacker…", "target": "victim.real.person@example.com", "status": "verified" }],
  "step3_victim_adds_own_email": { "status": 400, "body": "already linked to another account" },
  "VERDICT": {
    "channel_owned_by_attacker": true,
    "victim_locked_out":         true,
    "verify_needed_no_auth":     true
  }
}
```

Reading of the run:

* **step1** returns 500 **only** because SendGrid is unreachable in the harness — but `channels_after_add` and `token_belongs_to` prove the pending channel and its token are already persisted **under the attacker**, exactly because the insert precedes the send.
* **step2** — `POST /verify` with **no Authorization header** returns **200**. (In an earlier run before this leg worked, the same call returned **500 from inside the handler**, not `401` — the request had already passed the middleware chain, which is itself proof the endpoint is unauthenticated.)
* **step2 aftermath** — the channel is now `verified` **under the attacker's `user_id`**.
* **step3** — the victim's attempt to add their own address returns **400 "already linked to another account"**: locked out.

### Result of run (B) — routing to the victim's inbox (same live worker)

`drive-routing.mjs` chains onto the takeover, then has the attacker do two ordinary things — favourite the name `attacked.eth` and enable the favourite-expiry toggle — and triggers the real cron via the worker's `scheduled` handler (`/cdn-cgi/handler/scheduled`). A local stub stands in for the ENS indexer (its URL is env-configurable, `expiry-discovery/indexer.ts:65`, so production is never contacted) and returns `attacked.eth` inside the 7-day expiry window. The owner address is deliberately unknown to the worker, so the **only** recipient the worker resolves is the favouriting attacker:

```json
{
  "hijacked_channel":        [{ "user_id": "…attacker…", "target": "victim.…@example.com", "status": "verified" }],
  "deliveries_before_cron":  [{ "n": 0 }],
  "scheduled_trigger":       [{ "url": ".../cdn-cgi/handler/scheduled?cron=…", "status": 200 }],
  "deliveries_after_cron":   [{ "target": "victim.…@example.com", "channel": "email",
                               "status": "queued", "notif_owner": "…attacker…", "kind": "name-expiry" }],
  "VERDICT": {
    "takeover_channel_owner_is_attacker":     true,
    "ens_notification_routed_to_victim_inbox": true,
    "routed_delivery_owned_by_attacker":       true,
    "routed_channel_is_email":                 true
  }
}
```

Reading of the run:

* **Baseline** — zero deliveries to the victim's address before the cron.
* **After the cron** — the worker inserted exactly one `notification_deliveries` row: `target = victim_email`, `channel = email`, owned by the **attacker's** `user_id`, kind `name-expiry`. That `target` is precisely the value the email service places in SendGrid's `to:` (`services/delivery/email.ts:64`).
* The attacker never owned `attacked.eth` and never touched the victim's account after the takeover — favouriting a name was enough to route an official-ENS-sender email to the victim's inbox.

### Negative control (models the fix)

`negctl.mjs` models the fixed endpoint's decision — verify only if the authenticated caller owns the channel — against the same live pglite state:

```json
{ "channelOwner": "…attacker…", "sessionUser": "…victim…",
  "fixAllowsVerify": false, "channelStatusAfter": "pending", "takeoverBlocked": true }
```

With ownership enforced, the victim's click does **not** verify the attacker's channel and the victim is **not** locked out. **Honesty on grade:** the positive result is a true runtime measurement of the real worker (**G2**); this negative control **models** the fix's guard rather than running a patched worker, because the engagement forbids modifying the audited source. It establishes the counterfactual logically, not by executing patched code.

### Reproduction

1. `pnpm i @electric-sql/pglite` in a scratch dir; copy the repo's `drizzle/0000…0004` into `schema.sql`, replacing `CREATE EXTENSION pg_uuidv7` + `uuid_generate_v7()` with a pure-SQL uuid function.
2. `node neon-shim.mjs` — serves neon `/sql` on `:4444` over pglite (archived alongside this report).
3. In `workers/api-worker`, create `.dev.vars` with `JWT_SECRET`, `DATABASE_URL=postgres://…@db.localtest.me/postgres`, a dummy `SENDGRID_API_KEY`; run `pnpm dev`.
4. `node drive.mjs` for run (A). For run (B): also run `node indexer-stub.mjs` (local GraphQL stand-in on `:4455`), set `ENS_INDEXER_GRAPHQL_URL=http://localhost:4455/` in `.dev.vars`, then `node drive-routing.mjs`. All scripts archived verbatim in `poc-lanes-345/t49-email-channel-takeover/`. Note: run (B) triggers the email-delivery consumer, which will attempt outbound calls to `api.sendgrid.com`; block egress to that host if the harness must stay fully offline — the routing proof (the `notification_deliveries` row) is written upstream of that call and does not require it.
