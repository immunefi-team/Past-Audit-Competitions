# 91925 w a low email verification binds another user s mailbox to the requesting account unauthenticated

**Submitted on Sep 5th 2026 at 16:09:21 UTC by @ChaseTheLight for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91925
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email

## Description

## Brief/Intro

The API worker states a one-account-per-address rule for notification email channels and enforces it in the wrong request. `POST /notifications/channels/email` is authenticated and, before writing anything, asks whether the address is already `verified` at all (`workers/api-worker/src/app/routes/notifications/channels/email.ts:59-65`) and, if the row it finds belongs to a different account (`:67`), refuses with "This email address is already linked to another account" (`:67-72`). The row it then writes is `pending` (`:80`). The state the rule governs is created later, by `POST /notifications/channels/email/verify`, which matches a token, loads the channel row and sets `status = 'verified'` (`:195-201`) without asking the question again. So the check runs against state that has not yet formed, and the commit runs without the check. Two things follow. An account can enrol an address it does not control, the worker mails the project's own verification message to the real owner, and the row that flips to `verified` on their click belongs to the requesting account. And two registered accounts can both reach `verified` for one address, since neither add is refused while both rows are `pending`.

The only ownership conditions on the address are that the caller has not already verified it themselves (`email.ts:48-53`) and that nobody else has (`:59-72`). The verification mail comes from `noreply@ens.dev` and carries a genuine `https://app.ens.dev/...` URL; the recipient's press of "Verify Email Address" is what commits the flip (`:195-201`), on the strength of the token alone. The handler consults no caller identity, and it cannot: the `/verify` route carries no auth middleware at all (`:172-176`), while the add handler, immediately preceding it in the same route chain (`:29-33`), does spread `...requireAuth` (`:31`).

From that point the mailbox is bound, as `verified`, to an account its owner never chose. The verify hop itself sends the welcome mail to that mailbox from the project's sending domain (`email.ts:209-215`), and the owner is refused their own address for as long as that row stays `verified` (`:59-72`; see Impact).

Three drop-in test files built on the worker's own vitest harness drive the real Hono app the worker exports. They read and write through an in-memory stand-in that renders the drizzle predicate each handler actually passes and matches rows from that rendered SQL, so which rows a query selects is measured against production's own predicate rather than asserted. The delivered bytes were run on 2026-09-05 in a namespace holding no interface but loopback: eight of eight passed. Both fix patches were applied to the pinned snapshot and measured, the bind patch on both of its hunks. The stub's boundary and the counterfactuals are in the Proof of Concept field; reachability, the duplicate check, the fix and references are below.

## Vulnerability Details

### Root cause

The rule is enforced in the request that creates only a `pending` row, before the `verified` state it governs has formed, and the request that forms that state does not enforce it. Two facts follow from that placement; the observations after them record what the surrounding schema, middleware, configuration and client declare.

**The add handler binds an address it never proves the caller owns.** `email.ts:29-33` requires auth and validates the body, then writes a `user_channels` row carrying `user_id: userId` and `target: email` (`:75-97`). Ownership of the mailbox is meant to be established later, by the verification round trip.

**The verify handler never closes the loop.** `email.ts:179-201`, read whole, is three predicates and one write (reflowed here; `with` and `.set` are multi-line in source):

```ts
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
  .where(eq(TABLE.userChannels.id, verification.channel_id))
```

Token match, purpose, expiry. The handler loads one row plus its `channel` relation and consults no other row before flipping. It does not re-run the `:59-72` cross-account query. It does not compare anything about the requester to `verification.user_id`, and it has no requester to compare: there is no `...requireAuth` in the `/verify` handler's middleware chain (`:172-176`).

So the round trip proves that *somebody who could read the mailbox pressed a button*. The handler then treats that as proof that *the account which requested the binding owns the mailbox*. Those are different propositions.

**The column that was meant to carry the missing fact is declared and never written.** `core/database/schema/notifications.ts:109`:

```ts
target: text('target'), // email during email verification, null for Telegram until bot callback
```

There are exactly two inserts into `channel_verifications` in the worker, `email.ts:105-114` and `channels/$id.ts:191-200`, and neither sets `target`. Two more columns on the same path are declared in the schema and never read or written by any code in `workers/`: `channel_verifications.consumed_at` (`schema:117`) and `user_channels.verification_attempts` (`schema:71`). Single use is enforced by deleting the row (`email.ts:204-206`) rather than by the `consumed_at` column that was declared for it; the attempt-counting design was declared and never built at all.

* **Global middleware.** `app/index.ts:14-40` is the entire composition root; its only `.use()` is `cors()` at `:15`, with no origin list, and the rest of the chain is seven `.route()` mounts at `:16-22` and an `onError` handler at `:23-40`. No auth, CSRF, rate or ownership middleware sits above the router. `createApp` (`app/middleware/hono.ts:14-24`) returns a bare `new Hono(...)`.
* **Subtree middleware.** Neither `routes/notifications/index.ts:29-32` nor `channels/index.ts:53-56` calls `.use()`. `...requireAuth` is spread per handler throughout the tree (`notifications/index.ts:44,155,198,276,352`; `channels/index.ts:30`; `$id.ts:22,62,113,146`), which is why the one state-changing handler that omits it has nothing above it to compensate. Seventeen handlers are registered under `routes/notifications/`, and exactly two omit `...requireAuth`: this one, and `push.ts:44`, a `GET /vapid-public-key` that returns `c.env.VAPID_PUBLIC_KEY` and writes nothing (`:44-46`).
* **Edge configuration.** The checked-in `wrangler.jsonc` declares no route restriction: no `routes`, `route`, `pattern` or `custom_domain` entry appears in it. That is a fact about the configuration in this snapshot, not about the live edge, which this report does not observe.
* **The database.** The only uniqueness the ORM declares on `user_channels`, other than the surrogate primary key on `id`, is `unique('user_channel_unique').on(table.user_id, table.channel, table.target)` (`schema:74-78`), scoped per user. All five migration files were read; no partial unique index on `(channel, target) WHERE status = 'verified'` exists.

**This is not a declared design.** The verification page does branch on whether a session is present, but only to choose where its button navigates afterwards: `apps/manager/src/routes/notifications/channels/email/verify.tsx:65` reads the session flag, `:70-71` pick `/notifications/settings` or `/` and the matching label, and `:232-238` render "Back to Settings" or "Back to Home". Nothing in that file or in `email.ts` states that a third party's click should bind an address to someone else's account; the one comment that touches the verify transition, `email.ts:194`, only labels the write, and the page itself says "This page can only verify your email when opened from the link in the verification email" (`verify.tsx:106-107`). The only end-to-end exercise of this flow the project ships, `e2e/projects/manager/tests/notification.spec.ts:24-62`, adds the address and follows the verification link inside the same signed-in session, so in that test the requester and the recipient are one person (it runs only with Mailinator credentials, `notification.spec.ts:11-21`, and `e2e/` is cited as evidence of intent, not as an in-scope asset).

### The same defect with no third party

The gap also breaks the rule between two registered accounts, with no outside mailbox involved, and that variant is worth stating separately because it shows the rule failing with no outside party at all.

The add handler's cross-account check selects on `status = 'verified'`, with no term for the requesting account (`email.ts:59-65`). While two rows for one address are both `pending`, there is nothing for it to find, so neither add is refused. Both accounts receive a token, each valid for 24 hours (`email.ts:112`). Each redeems its own. `/verify` matches the token, loads the channel row, and sets `verified` on it (`email.ts:195-201`) without re-asking the question the add path asked. Two accounts now hold a `verified` row for one address, and nothing in storage objects: the only uniqueness on `user_channels` other than its surrogate key is per user, as set out in Root cause.

See "The exclusivity break, measured" in the Proof of Concept field.

### Trigger and preconditions

Preconditions: one SIWE login, free. The victim's email address, the only thing the attacker must know. No ENS name, no funds, no privilege, no stake are needed to bind the address.

The six reproduction steps are the first thing in the Proof of Concept field.

**What the victim sees at step 4:** the sender, the domain, the link and the landing page are all genuine, and no surface in the flow names the account the binding lands on. The first sentence after the title line tells the recipient they signed up for the notifications, in both parts: "Thank you for signing up for ENS notifications!" (`services/email/verification.ts:32` text/plain, `:54` html). The line that would cut against it, "If you didn't request this email, you can safely ignore it" (`:71-74`; the plain-text copy is at `:36`) names no account either. Across the whole message the worker builds (`verification.ts:19-86`) and across all 247 lines of the landing page (`verify.tsx`), the requesting account is never named: not by address, not by user id. The recipient is asked to confirm a binding without being shown what it binds to.

Every precondition is either free to the attacker or supplied by the shipped configuration. The one that is not attacker-controlled is the victim's click.

1. **A bearer token is free.** `POST /auth/login` gates on a nonce round-trip and a valid SIWE signature over an allow-listed domain, and on nothing about the caller's identity or history; `addUserIfNotExists` mints the user row on first sight (`services/auth/index.ts:109`). No name ownership, no allowlist on the caller's address (the only allowlist on that route is on the SIWE `domain`, `services/auth/index.ts:79-86`), no funding requirement. Any freshly generated EOA becomes a `user_id`.
2. **The route is live on the deployed worker.** Unbroken chain from the entry point: `app/index.ts:19` mounts `notificationsApp`, `routes/notifications/index.ts:31` mounts `channels`, `channels/index.ts:53` mounts `emailRoutes`. No feature flag and no dev gate on this handler. The contrast is in the same tree: `notifications/index.ts:362` **is** dev-gated, so the codebase knows how to gate a route and did not gate this one.
3. **The verify surface is reachable by a signed-out victim, on both halves.** Client side, the boundary is a fact about the file tree: TanStack file-based routing parents a route by its directory, `routes/notifications/_authenticated/route.tsx:10` is where `RequireBackendAuth` is applied, and `verify.tsx` does not live under `_authenticated/`. The generated tree agrees: `routeTree.gen.ts:523-529` shows `parentRoute: typeof rootRouteImport`, not the authenticated route. Server side, the handler carries no auth middleware (`email.ts:172-176`), and the client sends the request without an `Authorization` header rather than throwing when there is no session (`apps/manager/src/utils/backend-client.ts:148-152`).
4. **The shipped configuration carries the values the repository deploys with.** `wrangler.jsonc:98,100` set `EMAIL_FROM_ADDRESS: "noreply@ens.dev"` and `MANAGER_APP_URL: "https://app.ens.dev"`, so the victim sees a genuine link from a genuine sender (`services/email/verification.ts:16`). Unlike `SENDGRID_BROADCAST_LIST_ID` (`wrangler.jsonc:107`, empty), this path depends on no empty variable. `SENDGRID_API_KEY` is declared as a Cloudflare secret rather than a var (`.dev.vars.example:7`; it does not appear in `wrangler.jsonc`), which is where a live deployment keeps it. Citing these values is not an impact *on* a configuration file: the impact is in application code and the config is only what makes it reachable.
5. **Rate limiting does not bound it.** Three sends per hour, keyed on the target address (`services/notifications/email-verification-rate-limit.ts:3-4,29-30`). The attacker needs one. The same limit is the victim's quota, so a sprayer also burns the owner's ability to resend.
6. **The click cannot be forged.** The token is 16 CSPRNG bytes (`services/notifications/helpers.ts:173-177`), so there is no brute-force route to a zero-interaction variant, and the add handler withholds the token from its own caller (`email.ts:165-169`), so the attacker cannot self-verify. CORS is open (`app/index.ts:15`, `cors()` with no origin list) but buys nothing here, because an attacker-hosted page still cannot post a token it does not possess.

**Which victims the attack reaches, and why the guard does not cover them.** The `:59-72` guard makes the attack structurally impossible against a mailbox already verified to its owner, so at the moment of exploitation the victim has by construction no verified ENS record for that address. Three states:

| Victim state when the attacker sprays  | Outcome                                                                                                                                                                                                                                                                          |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Address already verified by its owner  | Attacker refused at `:67` with 400. No attack.                                                                                                                                                                                                                                   |
| Address pending on the owner's account | Attacker's row flips. The owner's row stays `pending` and they can still self-heal through `POST /notifications/channels/:id/resend` (`$id.ts:146-171`, gated on their own row being `pending`, subject to the same shared rate limit at `:173-176`). No lockout in this branch. |
| Address not enrolled at all            | Attacker's row flips to `verified`. The owner is refused at `:67` from then on, until and unless one of the releases below fires.                                                                                                                                                |

**The release paths, and their limits.** Two paths bear on this report, and each stops the row matching the `:59-72` lookup; only the first occurs without the attacker's cooperation. The first is the SendGrid event webhook, which finds the channel by `target` alone with no user scoping (`app/routes/webhook/sendgrid.ts:172-174`) and flips it to `bounced` on a recipient spam report (`:205-214`). The welcome mail fires the instant the binding lands (`email.ts:209-215`), so a victim who marks it as spam releases their own address without being told that is what they are doing. That path's limits: nothing tells the mailbox's owner which account the binding landed on (the whole mail object at `services/email/welcome.ts:18-92` names none), so the release is reached by accident or not at all; and the webhook's `findFirst` lookup, with no `orderBy`, picks an arbitrary row when several share the target, so with the victim's own pending row also present the event may land on the wrong one. The second is the attacker deleting the channel themselves: `DELETE /notifications/channels/:id` (`$id.ts:62-111`, the delete at `:77-79`) removes the row, so the guard's lookup finds nothing and `:67` stops firing. Its limit is who can reach it: the route spreads `...requireAuth` and scopes its own lookup to the caller's rows (`$id.ts:62`, `:66-70`), so it is the attacker's to use and the victim asking for that channel gets the 404 at `:73-75`, which makes it a lever rather than a relief path. The denial therefore has no guaranteed end: every exit is contingent, and the ones the attacker does not control are events that may never occur. This report claims it as a secondary effect on those terms rather than as a permanent lockout. The denial is keyed on the trimmed, lower-cased address (`email.ts:37`, `email-verification-rate-limit.ts:15-16`).

### Recommended fix

Two changes, one per limb of the defect. Both patches are reproduced in full in the Proof of Concept field, and each was applied to the pinned snapshot and measured there.

**Bind the transition to the requester. This closes the third-party binding.** Make the verify handler check the proposition it currently assumes. Spread `...requireAuth` on `/verify`, exactly as the add handler in the same file already does at `:31`, and refuse when the caller is not the account the verification row belongs to:

```ts
if (verification.user_id !== c.var.user_id) {
  return c.json(
    { error: 'This verification link belongs to a different account' },
    403,
  )
}
```

`bind_verify_to_the_requesting_account.patch` is that change. The attacker owns the row but never receives the token, and the victim receives the token but does not own the row, so the binding cannot complete for either of them. It does nothing about two registered accounts each redeeming their own token, since each is the requester of its own row.

**Re-run the rule in the request that commits the state. This closes the two-account variant.** `recheck_exclusivity_on_commit.patch` inserts, before the write at `:195`, the lookup the add path runs at `:59-65` (`channel = 'email'`, the address, `status = 'verified'`), reading the address through the `channel` relation the handler already loads (`:185-187`), and refuses with the add path's own error when the row it finds belongs to a different account than the verification row. The second of two accounts is then refused at its commit. It does not stop the first binding of an unclaimed address, because at that moment there is no `verified` row for the lookup to find.

Both are needed. The bind patch is two hunks, `...requireAuth` at the `/verify` registration (`:172-176`) and a requester comparison after the token lookup (`:190-192`); the recheck patch is one hunk at that second anchor. They insert at the same place, so applying them in sequence fails; apply one, or hand-merge the recheck lookup under the bind patch's comparison.

**Partial mitigation, if the signed-out flow must be preserved.** Two changes keep it: the commit-time recheck on its own, and naming the requesting account in both the verification mail and the landing page, so the recipient is deciding something rather than dismissing something. Neither closes the first-time binding this report describes: for a mailbox with no prior `verified` row the recheck finds nothing, and with the account named the app still writes the row (`email.ts:75-97`), still sends the mail (`:143-149`) and still flips the attacker's channel if the recipient clicks anyway.

Also worth fixing: the two declared-and-unused columns named in Root cause.

## Impact Details

### The in-scope impact this maps to

The governing rung, quoted verbatim from the programme's eligible-impact list:

> Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email

This report files at **High** on that rung. The object of the attack is an email address, which the rung names as its example. This report models the victim as an existing ENS user whose account has no email channel yet, which is the third row of the victim-state table above: the attacker's row flips to `verified`, and from then on the owner is refused their own address at `email.ts:67`. The affected user cannot see or remove that row: the channel list is scoped to the caller (`channels/index.ts:30`, `:46`) and so is the delete lookup (`$id.ts:62`, `:66-70`), so the row is invisible in their own settings and a delete request for it returns the 404 at `:73-75`. The rung asks what detail changed, not whose row records it. What changed is the affected user's own address: it now sits in the `target` column of a row whose `status` is `verified`, and the application begins acting on it without them. The verify hop itself sends them a welcome mail (`email.ts:209-215`).

**The discriminator against Medium is in the source, not in the adjective.** The Medium rung is the same sentence with "non-sensitive" and the example "Enabling/disabling notifications". In this worker those two examples name two different objects, on two different routes, in two different tables. Enabling or disabling notifications is `PATCH /notifications/preferences` (`routes/notifications/preferences.ts:62-66`), writing `user_notification_settings` (`core/database/schema/notifications.ts:138,143`); that route spreads `...requireAuth` at `preferences.ts:64`, binds the caller at `:68`, and scopes its only write to that value at `:74`, and this finding never reaches it. There is no cross-account write path to `user_notification_settings` anywhere in this worker. What this finding writes is a `user_channels` row whose `target` is a third party's mailbox, which is the High rung's example by name. Within the same tier the programme also lists "Improperly disclosing confidential user information, such as: Email address": email appears twice in this tier as protected user data (once as a "sensitive detail" and once as "confidential user information") and nowhere as non-sensitive.

**One click of user interaction, grounded in the rung's own Email example.** The one click of user interaction that commits the state is the press of "Verify Email Address" on `app.ens.dev` (`apps/manager/src/routes/notifications/channels/email/verify.tsx:220-224`), inside the app's own verification flow. Reaching that page means opening the link the worker mailed, and the rung's own example is "Email", that is, an emailed-verification flow, so following the emailed link is intrinsic to the class of impact the rung describes rather than an additional click charged against its budget. The press is the whole of the interaction the page asks for: `verify.tsx` has no `useEffect`, no `beforeLoad` and no `loader` function (only `loaderDeps` at `:57`, which reads the search parameter), and `handleVerify` (`:77-92`) is bound only to `onClick` at `:192` and `:220`, so the emailed link alone submits nothing.

### Loss mechanism and its bound

**The state change is the impact, and it is a write to the project's own database inside the `workers` asset.** `email.ts:195-201` sets `status: 'verified'` on a `user_channels` row whose `user_id` is the attacker's and whose `target` is a third party's mailbox that account never proved it controls. The rung claim rests on that write (`email.ts:195-201`), the welcome mail the verify hop sends to the mailbox (`:209-215`) and the refusal the owner then meets (`:67-71`), all inside `workers/api-worker`.

The scope boundaries clause excludes bugs *in* third-party services like Cloudflare or the mail provider. It does not exclude a first-party defect whose impact happens to be delivered through one. The change this report claims completes entirely inside `workers/api-worker`, which is the whole of the listed `workers` asset: the `email.ts:195-201` write, on the project's own database. No SendGrid or Cloudflare behavior is alleged to be defective; the mail provider is only the transport the High rung's own 'Email' example presupposes.

One further effect, on the mailbox owner:

* **Enrolment denial.** For a mailbox with no prior ENS record, the owner is refused at `email.ts:67` from then on; the release paths that bear on this report, and their limits, are set out above. The webhook's bounce and unsubscribe handlers (`sendgrid.ts:185-203`, `:222-237`) release it the same way, by moving the row's `status` off `verified`.

**The boundary of the claim.** Email is not an authentication or recovery factor here: `app/routes/auth/index.ts` is SIWE-only and no email-based reset exists, so the claim is bounded to the notification surface and account takeover is not asserted.

### Why not lower, and the out-of-scope entries to weigh

**Not Low.** The axis separating the High rung from the Low rung has two values, and the Low rung requires "significant user interaction", exemplified by "Iframing leading to modifying the backend/browser state", an attacker-controlled surface. A single press on the project's own domain, on the page the project's own verification mail links to, is not that, because it is not a surface the attacker controls.

**Not excluded.** "Impacts causing only the enumeration or confirmation of the existence of users or tenants" does not bite, because the harm claimed is the binding and not the 400/200 oracle. "Impacts caused by vulnerabilities requiring un-prompted, in-app user actions that are not part of the normal app workflows" does not bite either: the verification page exists for no other purpose, the worker's own mail links to it (`services/email/verification.ts:16`), and pressing "Verify Email Address" is the page's primary action (`verify.tsx:220-224`). Both rungs on this axis fund an emailed click on their face. Social engineering appears on this programme under Prohibited Activities, which governs what a researcher may do while testing, not which impacts are eligible; none was performed, and the attacker supplies an address and no content. Every word of the subject and body the recipient reads is generated by the worker itself and sent from the project's own domain, including "Thank you for signing up for ENS notifications!" (`services/email/verification.ts:32`, `:54`), so the misleading text is authored by first-party code. This handler commits the requesting account's binding on possession of the token alone, with no session and no comparison between the caller and the account the row belongs to (`email.ts:172-176`, `:195-201`); the address is bound to the requester's row whoever pressed the button.

### Not a duplicate, and scope

**The published register.** All 25 Part-1 entries of "ENS Apps: Known Security Issues" (last updated 2026-08-14) and its Part 2 lists were read; none touches notification channels or email verification, and the one entry attributed to the API worker, **R2-05** ("SendGrid event webhook fails open when the verification key is unset", Low), is a different defect on a different route, closed in this snapshot (`webhook/sendgrid.ts:113-116` returns 500 when `SENDGRID_WEBHOOK_VERIFICATION_KEY` is unset and `:128-152` enforces an ECDSA signature).

**The audit coverage boundary.** The register's Round 3 source-level audit (August 2026) covered `packages/transaction-manager` and `packages/smart-account`, not `workers/api-worker`, where this finding lives; across the same twenty-five Part-1 entries, exactly one (R2-05) is attributed to the API worker.

## References (pinned to archive `sha256:bf6ca914f817ccbf22e340813630ba2c69315884e777fbac4ac34a1e4ba542c5`)

The delivered export ships no `.git`, so there is no commit to link. Paths are repository-relative.

1. `workers/api-worker/src/app/routes/notifications/channels/email.ts` - `:29-33` (add handler, `...requireAuth`), `:59-72` (the cross-account guard, scoped to `verified`), `:67` (the refusal), `:75-97` (the channel row, `user_id` = caller), `:105-114` (token mint, no `target` written), `:165-169` (token withheld from the caller), **`:172-176` (the verify handler's middleware chain, no auth)**, `:179-188` (the token lookup), `:190-192` (the verify handler's only refusal), **`:195-201` (the flip, keyed on `verification.channel_id`)**, `:204-206` (the spent row deleted), `:209-215` (welcome mail)
2. `workers/api-worker/src/app/index.ts` - `:14-40` (composition root; the only `.use()` is `cors()` at `:15`)
3. `workers/api-worker/src/app/middleware/hono.ts` - `:14-24` (`createApp` attaches nothing)
4. `workers/api-worker/src/app/middleware/auth.ts` - `:35` (`requireAuth = [requireJWT, verifyJWT]`)
5. `workers/api-worker/src/app/routes/notifications/index.ts` - `:29-32` (no `.use()`), `:44,155,198,276,352` (per-handler `...requireAuth`), `:362` (a route that **is** dev-gated)
6. `workers/api-worker/src/app/routes/notifications/channels/index.ts` - `:23-24,53-56` (mount, no `.use()`)
7. `workers/api-worker/src/app/routes/notifications/channels/$id.ts` - `:22,62,113,146` (per-handler auth), `:146-171` (resend, the pending-row self-heal), `:191-200` (the second write that omits `target`)
8. `workers/api-worker/src/core/database/schema/notifications.ts` - `:71` (`verification_attempts`, never written), `:74-78` (per-user unique constraint), `:109` (`target`, declared with a comment and never written), `:117` (`consumed_at`, never written)
9. `workers/api-worker/drizzle/0000_vengeful_golden_guardian.sql` - `:114` (the only uniqueness constraint on `user_channels` other than the surrogate primary key on `id`, across all five migrations), `:124` and `:128` (the only other statements touching that table, both foreign keys)
10. `workers/api-worker/src/services/notifications/helpers.ts` - `:173-177` (`generateToken`, 16 CSPRNG bytes)
11. `workers/api-worker/src/services/notifications/email-verification-rate-limit.ts` - `:3-4,29-30` (3 per hour at `:3-4`, keyed on the normalised target at `:29-30`)
12. `workers/api-worker/src/services/email/verification.ts` - `:16` (the link the victim sees), `:19-86` (the whole message the worker builds; it names no account), `:32,54` ("Thank you for signing up for ENS notifications!", text/plain and html), `:36,71-74` (the "safely ignore it" line), `:90-93` (token logged)

## Link to Proof of Concept

<https://gist.github.com/ChaseTheLight01/24b39f6946c0c16836d0889baa711d3c>

## Proof of Concept

## Proof of Concept

1. The attacker signs in with a throwaway EOA. `addUserIfNotExists` mints the user row on first sight (`services/auth/index.ts:109`).
2. The attacker calls `POST /notifications/channels/email` with the victim's email address. The `email.ts:59-72` guard passes because nobody has verified that mailbox. A `pending` row is written with `user_id` = attacker (`:75-97`) and a 128-bit token is minted (`:105-114`, `helpers.ts:173-177`).
3. The worker mails `https://app.ens.dev/notifications/channels/email/verify?token=...` from `noreply@ens.dev` (`services/email/verification.ts:16`; `wrangler.jsonc:98,100` carry both values).
4. The victim opens the link and presses "Verify Email Address" (`apps/manager/src/routes/notifications/channels/email/verify.tsx:220-224`). The verify route is parented on the root, not on the authenticated subtree (`routeTree.gen.ts:523-529`), so a signed-out stranger completes it.
5. `email.ts:195-201` flips the attacker's row to `verified`. A welcome mail goes to the victim (`:209-215`).
6. The victim is refused at `:67` on every attempt to enrol their own address, for as long as that row stays `verified` (Impact sets out what can end that state, and who those levers belong to).

Three drop-in test files carry this, all built on the worker's own vitest harness and driven against the real Hono app the worker exports (`src/worker.ts:16`). `channel_bind_without_owner_consent.poc.test.ts` holds five tests: an authenticated account enrols an address it never proves it controls; the add path's cross-account check runs in the request that writes only a pending row; an unauthenticated verify commits the binding to the requesting account; the owner is then refused their own address; and the send rate limit is consumed only after both rows are written. `verified_address_is_not_exclusive.poc.test.ts` holds one, the same defect between two registered accounts with no third party. `verify_ignores_the_callers_session.poc.test.ts` holds two: the binding still lands on the requesting account when the mailbox owner is a registered user who redeems while signed in as themselves, and the same request without a session reaches the same state. All three read and write through `notification_tables.ts`, an in-memory stand-in that renders the drizzle predicate each handler actually passes, using the project's own dialect import, and decides which rows match from that rendered text (`notification_tables.ts:52-58`, `:88-136`). It hardcodes no column filter of its own, so what rows a query selects is measured against production's SQL rather than asserted.

The bytes delivered in this pack were run on 2026-09-05 in a namespace holding no interface but loopback. In every capture below the run path (a read-only overlay mount of the pinned tree) is shown as `<repo>`, the userns and loopback-raise lines and the `Start at` / `Duration` lines are elided, and the `=== VITEST_EXIT=1 ===` line is shown without its `===` delimiters; every elision is marked `[...]`.

```
[...]
=== TEST IDENTITY (shipped bytes) ===
977ff004cb240e630af842a7e147e6083bd725b585e13c830a96d0e6ae827800  <repo>/workers/api-worker/src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts
6552ace1dcab641a51505b2d28e7cd07727344ed73e48b5d293d0f05c3b6295a  <repo>/workers/api-worker/src/app/routes/notifications/channels/verified_address_is_not_exclusive.poc.test.ts
e44b21f7c629fbff0d5988b30025e98a6fda53ab70cbeed0886204074311d295  <repo>/workers/api-worker/src/app/routes/notifications/channels/verify_ignores_the_callers_session.poc.test.ts
8a0409f2aff823990b9e2cbccf8c0e766ea85e36a795e2601b882dbfd2f1b191  <repo>/workers/api-worker/src/app/routes/notifications/channels/notification_tables.ts
[...]
 ✓ src/app/routes/notifications/channels/verified_address_is_not_exclusive.poc.test.ts > one address across two accounts > lets both accounts reach verified for the same address 46ms
[...]
 ✓ src/app/routes/notifications/channels/verify_ignores_the_callers_session.poc.test.ts > the caller session the verify handler never reads > binds to the requesting account even when the mailbox owner redeems while signed in as themselves 68ms
 ✓ src/app/routes/notifications/channels/verify_ignores_the_callers_session.poc.test.ts > the caller session the verify handler never reads > reaches the same state when the same request carries no session at all 7ms
[...]
 ✓ src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > accepts an address the caller has not proved it controls, and keeps the token out of the response 41ms
 ✓ src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > runs its cross-account check only against rows that already reached verified 6ms
 ✓ src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > completes verification with no caller identity and commits the binding to the requesting account 15ms
 ✓ src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > refuses the mailbox owner their own address once the foreign row is verified 5ms
 ✓ src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > is not stopped by the send rate limit, which is consumed after both rows are written 2ms
[...]
 Test Files  3 passed (3)
      Tests  8 passed (8)
     Errors  1 error
[...]
VITEST_EXIT=1
```

The one error is not a failing assertion. A control inside the binding file issues `GET /notifications/channels` with no session and asserts a 401 (`channel_bind_without_owner_consent.poc.test.ts:249-254`); `hono/jwt` raises that 401 as an unhandled rejection, which vitest surfaces once for the run and attributes to the enclosing test. The rejection block is under "The captures". Every one of the eight tests passed.

Copy the three test files and `notification_tables.ts` to `workers/api-worker/src/app/routes/notifications/channels/` and run them from that workspace:

```bash
cp channel_bind_without_owner_consent.poc.test.ts \
   verified_address_is_not_exclusive.poc.test.ts \
   verify_ignores_the_callers_session.poc.test.ts \
   notification_tables.ts \
  <repo>/workers/api-worker/src/app/routes/notifications/channels/
cd <repo> && corepack enable && pnpm install --frozen-lockfile
cd workers/api-worker
pnpm exec vitest run \
  src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts \
  src/app/routes/notifications/channels/verified_address_is_not_exclusive.poc.test.ts \
  src/app/routes/notifications/channels/verify_ignores_the_callers_session.poc.test.ts \
  --reporter=verbose
```

Commands and the pinned environment are in the Gist's `README.md` (<https://gist.github.com/ChaseTheLight01/24b39f6946c0c16836d0889baa711d3c>, also in the submission's link field). No handler is reimplemented; the rate limiter is not mocked and runs at its shipped constants (`email-verification-rate-limit.ts:3-4`); the KV store it reads its counter from is a stub.

### Where this was run, and what it never touched

The run was inside a fail-closed network namespace holding no interface but loopback. The capture carries the interface inventory the process could see and an egress probe against `api.sendgrid.com`, the host `services/email/utils.ts:30` would reach, taken before and after the suite (abridged, elisions marked `[...]`):

```
netns interfaces  :
    1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
[...]
=== EGRESS PROBES, BEFORE loopback ===
  [tcp 1.1.1.1:443] BLOCKED
  [dns api.sendgrid.com] BLOCKED
  [dns app-api.ens.dev] BLOCKED
[...]
=== EGRESS PROBES, AFTER loopback (must still be blocked) ===
  [tcp 1.1.1.1:443] BLOCKED
  [dns api.sendgrid.com] BLOCKED
```

Loopback is raised only because `@cloudflare/vitest-pool-workers` runs `workerd`, which binds a local socket; a loopback-only namespace reaches nothing off-host. The archive itself was mounted read-only as the lower layer of an overlay, so the source tree could not be written even by a runaway process. Inside the process the paths are closed too: `getDatabase` is stubbed in both files, global `fetch` is stubbed in `beforeEach`, and no real secret is read: the JWT signing key and the other env values, `DATABASE_URL` and `SENDGRID_API_KEY` among them, are placeholders declared in the test files. Either the namespace or the module stubs is sufficient; both were in force.

### The transition commits the binding, measured against production's own SQL

The headline test drives `POST /notifications/channels/email/verify` with no credential and reads back only what production wrote or mailed. It asserts over the whole `user_channels` table, and it reads the commit's predicate as drizzle rendered it, so "keyed on the channel id, with no account term" is measured rather than taken from the source:

```ts
// channel_bind_without_owner_consent.poc.test.ts:236-313
  it('completes verification with no caller identity and commits the binding to the requesting account', async () => {
    const tables = createNotificationTables()
    mocks.getDatabase.mockReturnValue(tables.db)

    await addChannel(
      await bearer(REQUESTING_USER, REQUESTING_ADDRESS),
      MAILBOX,
    )
    const channelId = tables.rows.user_channels[0].id
    const token = verificationTokenFromMail()

    // Control, same application, same request shape, no Authorization header:
    // a route that requires a session refuses.
    const authenticatedRoute = await app.request(
      '/notifications/channels',
      { method: 'GET' },
      env,
    )
    expect(authenticatedRoute.status).toBe(401)

    const queriesBefore = tables.queries.length
    const response = await verifyChannel(token)

    expect(response.status).toBe(200)
    expect(await response.json()).toEqual({
      message: 'Email verified successfully',
    })

    // The row that flipped belongs to the account that made the request the
    // mailbox owner never saw.
    const forMailbox = tables.rows.user_channels.filter(
      (row) => row.target === MAILBOX,
    )
    expect(forMailbox).toHaveLength(1)
    expect(forMailbox[0].user_id).toBe(REQUESTING_USER)
    expect(forMailbox[0].status).toBe('verified')
    expect(forMailbox[0].verified_at).toBeInstanceOf(Date)

    // Over the whole table, not the expected row: nothing verified belongs to
    // the person who reads the mailbox.
    const verified = tables.rows.user_channels.filter(
      (row) => row.status === 'verified',
    )
    expect(verified.map((row) => row.user_id)).toEqual([REQUESTING_USER])
    expect(
      tables.rows.user_channels.some(
        (row) => row.user_id === MAILBOX_OWNER_USER,
      ),
    ).toBe(false)

    // The write, as production keyed it: the channel row id off the token
    // record, with no account in the predicate.
    const commit = tables.writes.find(
      (write) => write.kind === 'update' && write.table === 'user_channels',
    )
    expect(commit?.set).toMatchObject({ status: 'verified' })
    expect(commit?.predicate?.sql).toContain('"user_channels"."id" = ')
    expect(commit?.predicate?.sql).not.toContain('"user_channels"."user_id"')
    expect(commit?.predicate?.params).toEqual([channelId])

    // Asserted over every read the commit request made, not over the read we
    // expected to be missing: it looked at the token record and at nothing
    // else. The cross-account check is not among them.
    const duringVerify = tables.queries.slice(queriesBefore)
    expect(duringVerify.map((entry) => entry.table)).toEqual([
      'channel_verifications',
    ])
    expect(
      duringVerify.some((entry) => entry.predicate?.params.includes('verified')),
    ).toBe(false)

    // The welcome message goes to the mailbox as well, on the same sender.
    const welcome = sent.find(
      (message) => message.subject === 'Welcome to ENS Notifications',
    )
    expect(welcome?.to).toBe(MAILBOX)
    expect(welcome?.from).toBe('noreply@ens.dev')
  })
```

The `GET /notifications/channels` control at `:249-254` is the same absent-credential request against a route that does require a session; it returns 401, which is what makes the verify transition's 200 the absence of a check rather than the app admitting everyone. The commit request issues one relational read, of the token record; the handler loads its `channel` relation in that same statement (`email.ts:185-187`), which the stand-in resolves without recording a second query. No read it makes carries the param `'verified'` (`channel_bind_without_owner_consent.poc.test.ts:303-305`), so the add path's cross-account query is not among them.

### The check runs in the request that writes only a pending row

The add handler's cross-account query is read as drizzle rendered it: it keys on the address and on `status = 'verified'`, and carries no term for the account, so a second account's proposed row is outside what it can see. Measured, not argued:

```ts
// channel_bind_without_owner_consent.poc.test.ts:202-234
  it('runs its cross-account check only against rows that already reached verified', async () => {
    const tables = createNotificationTables()
    mocks.getDatabase.mockReturnValue(tables.db)

    await addChannel(
      await bearer(REQUESTING_USER, REQUESTING_ADDRESS),
      MAILBOX,
    )

    // Every predicate the add handler issued against user_channels, rendered by
    // drizzle. The second is the cross-account check: it is keyed on the
    // address and on status, and the status it asks for is 'verified'.
    const channelReads = tables.queries.filter(
      (entry) => entry.table === 'user_channels',
    )
    expect(channelReads).toHaveLength(2)

    const crossAccount = channelReads[1].predicate
    expect(crossAccount?.sql).toContain('"user_channels"."target" = ')
    expect(crossAccount?.sql).toContain('"user_channels"."status" = ')
    expect(crossAccount?.params).toContain('verified')
    // It does not ask about the account, so a proposed row under another
    // account is outside what it can see.
    expect(crossAccount?.sql).not.toContain('"user_channels"."user_id"')

    // Measured, not argued: a second account adding the same address while the
    // first row is still pending is not refused.
    const second = await addChannel(
      await bearer(MAILBOX_OWNER_USER, MAILBOX_OWNER_ADDRESS),
      MAILBOX,
    )
    expect(second.status).toBe(200)
  })
```

### The mailbox owner is then refused their own address

```ts
// channel_bind_without_owner_consent.poc.test.ts:315-350
  it('refuses the mailbox owner their own address once the foreign row is verified', async () => {
    const tables = createNotificationTables()
    mocks.getDatabase.mockReturnValue(tables.db)

    await addChannel(
      await bearer(REQUESTING_USER, REQUESTING_ADDRESS),
      MAILBOX,
    )
    await verifyChannel(verificationTokenFromMail())

    const ownerAttempt = await addChannel(
      await bearer(MAILBOX_OWNER_USER, MAILBOX_OWNER_ADDRESS),
      MAILBOX,
    )

    expect(ownerAttempt.status).toBe(400)
    expect(await ownerAttempt.json()).toEqual({
      error: 'This email address is already linked to another account',
    })

    // Control on the same code path: the same check does not refuse an address
    // nobody holds, so the refusal above is the check firing rather than the
    // route rejecting everything.
    const otherAddress = await addChannel(
      await bearer(MAILBOX_OWNER_USER, MAILBOX_OWNER_ADDRESS),
      'owner.own.mailbox@example.com',
    )
    expect(otherAddress.status).toBe(200)

    // And the row the owner cannot reach is still the requesting account's.
    expect(
      tables.rows.user_channels
        .filter((row) => row.target === MAILBOX)
        .map((row) => [row.user_id, row.status]),
    ).toEqual([[REQUESTING_USER, 'verified']])
  })
```

### The caller's session is never read, even when there is one

The mailbox owner in this file is a registered account, not a stranger: they hold a session the worker itself minted and they enrol an address of their own first, so the account plainly exists. The requesting account then enrols the owner's other mailbox and the owner opens the link while signed in as themselves. The row that flips is still the requesting account's. The `/verify` chain carries `vValidator` and `injectDb` and no `...requireAuth` (`email.ts:172-176`), so the Authorization header this test sends is never parsed and there is no `c.var.user_id` for the handler to compare against.

```ts
// verify_ignores_the_callers_session.poc.test.ts:165-212
  it('binds to the requesting account even when the mailbox owner redeems while signed in as themselves', async () => {
    const { tables, ownerSession } = await setUp()

    const queriesBefore = tables.queries.length
    const response = await verifyChannel(tokenFor(OWNER_MAILBOX), ownerSession)

    // The request carried a valid session for the mailbox owner and succeeded.
    expect(response.status).toBe(200)
    expect(await response.json()).toEqual({
      message: 'Email verified successfully',
    })

    // Over the whole table: the address the owner reads is held, verified, by
    // the other account, and the owner holds no row for it at all.
    const forMailbox = verifiedRowsFor(tables, OWNER_MAILBOX)
    expect(forMailbox).toHaveLength(1)
    expect(forMailbox[0].user_id).toBe(REQUESTING_USER)
    expect(
      tables.rows.user_channels.some(
        (row) => row.target === OWNER_MAILBOX && row.user_id === OWNER_USER,
      ),
    ).toBe(false)

    // The owner's own enrolment is untouched, so this is the binding landing
    // on the wrong account rather than the run mangling every row.
    expect(
      tables.rows.user_channels
        .filter((row) => row.target === OWNER_OTHER_MAILBOX)
        .map((row) => [row.user_id, row.status]),
    ).toEqual([[OWNER_USER, 'pending']])

    // The write, as production keyed it: the channel row id off the token
    // record, with no account term, so the session could not have narrowed it.
    const commit = tables.writes.find(
      (write) => write.kind === 'update' && write.table === 'user_channels',
    )
    expect(commit?.set).toMatchObject({ status: 'verified' })
    expect(commit?.predicate?.sql).toContain('"user_channels"."id" = ')
    expect(commit?.predicate?.sql).not.toContain('"user_channels"."user_id"')

    // Asserted over every read the commit request made, not over the read we
    // expected to be missing: it looked at the token record and nothing else.
    // No table carrying an identity was consulted.
    const duringVerify = tables.queries.slice(queriesBefore)
    expect(duringVerify.map((entry) => entry.table)).toEqual([
      'channel_verifications',
    ])
  })
```

The second test in the file is the control: the same request with no session at all reaches the same state, so the session was not overridden, it was never read.

### The exclusivity break, measured

The variant with no third party is measured by the second file, which drives the same two handlers and asserts over the whole `user_channels` table rather than over an expected row. It carries its own control in the same run: an address only one account enrolled ends held by exactly one, so the count is the rule breaking rather than the harness duplicating rows.

```ts
// verified_address_is_not_exclusive.poc.test.ts:119-181
describe('one address across two accounts', () => {
  it('lets both accounts reach verified for the same address', async () => {
    const tables = createNotificationTables()
    mocks.getDatabase.mockReturnValue(tables.db)

    const first = await addChannel(
      await bearer(FIRST_USER, FIRST_ADDRESS),
      SHARED_MAILBOX,
    )
    const second = await addChannel(
      await bearer(SECOND_USER, SECOND_ADDRESS),
      SHARED_MAILBOX,
    )

    // Neither add is refused: while both rows are proposed, the check the add
    // path runs has nothing to find.
    expect(first.status).toBe(200)
    expect(second.status).toBe(200)

    const mails = verificationMails(SHARED_MAILBOX)
    expect(mails).toHaveLength(2)

    // Nothing in either message distinguishes them. Blanking the token makes
    // the two bodies identical, so the recipient has no fact with which to
    // choose.
    const tokens = mails.map((mail) => tokenIn(mail.text ?? ''))
    expect(new Set(tokens).size).toBe(2)
    const blanked = mails.map((mail, index) =>
      (mail.text ?? '').replaceAll(encodeURIComponent(tokens[index]), 'TOKEN'),
    )
    expect(blanked[0]).toBe(blanked[1])

    for (const token of tokens) {
      expect((await verifyChannel(token)).status).toBe(200)
    }

    // The claim, over the whole table rather than over an expected row: the
    // address is held, verified, by two different accounts at once.
    const verified = tables.rows.user_channels.filter(
      (row) => row.status === 'verified',
    )
    expect(verified).toHaveLength(2)
    expect(verified.every((row) => row.target === SHARED_MAILBOX)).toBe(true)
    expect(new Set(verified.map((row) => row.user_id))).toEqual(
      new Set([FIRST_USER, SECOND_USER]),
    )

    // Control, same table, same run: an address only one account enrolled is
    // held by exactly one account, so the count above is the rule breaking
    // rather than the harness duplicating rows.
    const third = await addChannel(
      await bearer(FIRST_USER, FIRST_ADDRESS),
      OTHER_MAILBOX,
    )
    expect(third.status).toBe(200)
    await verifyChannel(tokenIn(verificationMails(OTHER_MAILBOX)[0].text ?? ''))

    expect(
      tables.rows.user_channels.filter(
        (row) => row.status === 'verified' && row.target === OTHER_MAILBOX,
      ),
    ).toHaveLength(1)
  })
```

### The counterfactual: the fix patches

Each patch was applied to the pinned tree in a read-only overlay of the same kind, with the egress probe denied before and after. Each capture carries the `APPLYING` line and the shipped-byte `TEST IDENTITY` block. Each patch closes one limb of the defect.

**The bind patch closes the third-party binding.** With `bind_verify_to_the_requesting_account.patch` applied, `POST /notifications/channels/email/verify` sits behind `requireAuth`, so an unauthenticated redemption is rejected. Both of the patch's hunks are exercised: this arm reaches the middleware hunk, and the arm on `verify_ignores_the_callers_session.poc.test.ts` below reaches the requester comparison, which the middleware alone cannot reach. Here the two tests that redeem a token without a session stop; the three add-only controls are untouched (abridged):

```
[...]
=== APPLYING bind_verify_to_the_requesting_account.patch ===
[...]
 ✓ src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > accepts an address the caller has not proved it controls, and keeps the token out of the response 18ms
 ✓ src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > runs its cross-account check only against rows that already reached verified 5ms
 × src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > completes verification with no caller identity and commits the binding to the requesting account 15ms
   → expected 401 to be 200 // Object.is equality
 × src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > refuses the mailbox owner their own address once the foreign row is verified 7ms
   → expected 200 to be 400 // Object.is equality
 ✓ src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts > email channel enrolment > is not stopped by the send rate limit, which is consumed after both rows are written 2ms
[...]
 Test Files  1 failed (1)
      Tests  2 failed | 3 passed (5)
     Errors  3 errors
[...]
VITEST_EXIT=1
```

Each patched arm was run against the single test file its patch bears on, which is why this summary counts five tests. A reader who applies the bind patch and runs all three files with the command above sees five of the eight fail: the exclusivity test redeems without a session too, and both tests in `verify_ignores_the_callers_session.poc.test.ts` are aimed at this patch by design.

Test 3 is the binding itself: it stops at the assertion on the verify call, which now returns 401 where it returned 200 (`channel_bind_without_owner_consent.poc.test.ts:259`). Test 4 depends on the binding having happened, so it stops too (`:330`). Tests 1, 2 and 5 exercise only the authenticated add handler, which this patch does not touch, and are unchanged. The three errors are the three now-rejected unauthenticated requests, all the same `hono/jwt` 401: the `GET` control (`:249`) and the two verify calls (`:257`, `:323`). The patch does not close the two-account variant, since each of those accounts is the authenticated requester of its own row; that limb is the recheck patch below.

The same patch, run against `verify_ignores_the_callers_session.poc.test.ts`, reaches its second hunk. The mailbox owner there redeems with a valid session of their own, so `requireAuth` admits the request and the requester comparison is what refuses it, with 403 rather than 401. The control in the same file, which sends no session, still stops at the middleware with 401, so the two hunks are told apart by the two results (abridged):

```
[...]
=== APPLYING bind_verify_to_the_requesting_account.patch ===
[...]
 × src/app/routes/notifications/channels/verify_ignores_the_callers_session.poc.test.ts > the caller session the verify handler never reads > binds to the requesting account even when the mailbox owner redeems while signed in as themselves 33ms
   → expected 403 to be 200 // Object.is equality
 × src/app/routes/notifications/channels/verify_ignores_the_callers_session.poc.test.ts > the caller session the verify handler never reads > reaches the same state when the same request carries no session at all 13ms
   → expected 401 to be 200 // Object.is equality
[...]
 Test Files  1 failed (1)
      Tests  2 failed (2)
[...]
VITEST_EXIT=1
```

```diff
Bind the email verification transition to the account that requested it.

The `/verify` handler currently proves that somebody who could read the mailbox
pressed a button, and then treats that as proof that the account which
requested the binding owns the mailbox. This makes the handler actually check
the proposition that the requesting account owns the mailbox: the caller must
be authenticated as the user whose verification row the token belongs to.

The middleware spread is copied from the add handler in the same file (`:31`),
so nothing new is introduced. `verifyJWT` already sets `c.var.user_id`.

Cost: a user who opens the link in a
browser where they are not signed in now has to sign in before the button
works. That is the same posture `/notifications/settings` already takes
(`apps/manager/src/routes/notifications/_authenticated/route.tsx:10`), and the
manager's verify route would need moving under `_authenticated/` to match. If
that trade is unacceptable, the report's 'Partial mitigation' section
describes two changes that keep the signed-out flow, but they do not close
the first-time-binding attack this patch closes, only a later re-binding by
a second account.

--- a/workers/api-worker/src/app/routes/notifications/channels/email.ts
+++ b/workers/api-worker/src/app/routes/notifications/channels/email.ts
@@ -171,6 +171,7 @@
   )
   .post(
     '/verify',
+    ...requireAuth,
     vValidator('json', verifyEmailChannelBodySchema),
     injectDb,
     async (c) => {
@@ -190,6 +191,17 @@
       if (!verification) {
         return c.json({ error: 'Invalid or expired verification token' }, 400)
       }
+
+      // The token proves the mailbox was reachable. It does not prove that the
+      // account which asked for the binding owns that mailbox, and those are
+      // different propositions. Refuse unless the caller is the requester, so
+      // a channel can only go live on the account that will receive its mail.
+      if (verification.user_id !== c.var.user_id) {
+        return c.json(
+          { error: 'This verification link belongs to a different account' },
+          403,
+        )
+      }

       // Mark channel as verified
       await c.var.db
# end of patch
```

**The recheck patch closes the two-account variant.** With `recheck_exclusivity_on_commit.patch` applied, the exclusivity test reaches its second redemption and receives 400 where it expected 200: the commit-time lookup finds the first account's `verified` row and refuses. One welcome mail is logged, for the commit that went through, and none after it (abridged):

```
[...]
=== APPLYING recheck_exclusivity_on_commit.patch ===
[...]
 × src/app/routes/notifications/channels/verified_address_is_not_exclusive.poc.test.ts > one address across two accounts > lets both accounts reach verified for the same address 42ms
   → expected 400 to be 200 // Object.is equality
[...]
 ❯ src/app/routes/notifications/channels/verified_address_is_not_exclusive.poc.test.ts:152:51
[...]
 Test Files  1 failed (1)
      Tests  1 failed (1)
[...]
VITEST_EXIT=1
```

```diff
Re-evaluate the one-account-per-address rule in the request that commits it.

Apply from the repository root: `patch -p1 < recheck_exclusivity_on_commit.patch`
or `git apply recheck_exclusivity_on_commit.patch`.

What it closes: two accounts can no longer both hold a verified row for one
address, and a commit for an address another account already holds is refused.

What it does not close on its own: the first account to bind an unclaimed
address still binds it, because at that moment there is no verified row for the
conflict query to find. That limb is the consent one, and the companion patch
in this directory addresses it.

--- a/workers/api-worker/src/app/routes/notifications/channels/email.ts
+++ b/workers/api-worker/src/app/routes/notifications/channels/email.ts
@@ -191,6 +191,25 @@
         return c.json({ error: 'Invalid or expired verification token' }, 400)
       }
 
+      const target = verification.channel?.target
+
+      if (target) {
+        const conflicting = await c.var.db.query.userChannels.findFirst({
+          where: and(
+            eq(TABLE.userChannels.channel, 'email'),
+            eq(TABLE.userChannels.target, target),
+            eq(TABLE.userChannels.status, 'verified'),
+          ),
+        })
+
+        if (conflicting && conflicting.user_id !== verification.user_id) {
+          return c.json(
+            { error: 'This email address is already linked to another account' },
+            400,
+          )
+        }
+      }
+
       // Mark channel as verified
       await c.var.db
         .update(TABLE.userChannels)
# end of patch
```

Both patches insert at the same anchor in `email.ts` (between the `if (!verification)` block at `:190-192` and the `// Mark channel as verified` comment at `:194`). Under `git apply`, applying them in sequence fails: it refuses the second in either order (`patch failed: ...email.ts:191` for the recheck after the bind, `...email.ts:190` for the bind after the recheck). Apply one, or hand-merge the recheck lookup under the bind patch's comparison. Each passes `git apply --check -p1` alone.

### What is measured, and what is argued from source

Every row below marked **Measured** is an assertion one of the three shipped test files makes against the pinned tree, which passed on the 2026-09-05 run captured above; the three counterfactual rows name their own patched run. No row rests on a mock that discards the predicate it is about: the stand-in renders production's SQL and matches rows from it.

| Claim                                                                                                                                                                                                                                    | Status                                                                                                                                                                                                                                                                              |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The verify transition sets `status: 'verified'` keyed on the channel id off the token record, with no account term in the predicate                                                                                                      | **Measured.** `commit?.predicate?.sql` contains `"user_channels"."id" =` , does not contain `"user_channels"."user_id"`, and its params are `[channelId]`                                                                                                                           |
| The commit request issues one relational read of the token record, and runs no separately-scoped cross-account lookup                                                                                                                    | **Measured**, as the list of top-level reads during the request (`['channel_verifications']`) and the absence of any read carrying the param `'verified'`                                                                                                                           |
| The flipped row belongs to the requesting account; over the whole table nothing verified belongs to the mailbox owner, and after the owner's own attempt the address's only row is still the requester's                                 | **Measured**, in test 3 over the whole table and in test 4 after the owner is refused                                                                                                                                                                                               |
| The welcome mail is addressed to the bound mailbox, from the configured sender, which `wrangler.jsonc:98` sets to `noreply@ens.dev`                                                                                                      | **Measured** (the recipient and the sender pass-through); the sender's production value is source                                                                                                                                                                                   |
| The verification mail names no account, in either the text or the HTML body                                                                                                                                                              | **Measured** over both bodies                                                                                                                                                                                                                                                       |
| The written row carries the caller's account and the mailbox, at `status: 'pending'`                                                                                                                                                     | **Measured**                                                                                                                                                                                                                                                                        |
| The add path's cross-account check keys on the address and `status = 'verified'` and carries no account term, so a second `pending` add is not refused                                                                                   | **Measured**, reading the rendered predicate, plus the second add returning 200                                                                                                                                                                                                     |
| Once the foreign row is verified, the mailbox owner is refused their own address; an unheld address is not refused                                                                                                                       | **Measured**, with the control in the same run                                                                                                                                                                                                                                      |
| Two registered accounts both reach `verified` for one address; an address only one enrolled is held by one                                                                                                                               | **Measured**, with the control in the same run                                                                                                                                                                                                                                      |
| With the per-address counter at the shipped ceiling, the limiter refuses the next send with 429, and the channel row and the token row are already written when it does                                                                  | **Measured**; that the limit does not bound the attack follows from the shipped ceiling of three sends per hour against the one send a binding needs                                                                                                                                |
| The binding lands on the requesting account even when the mailbox owner is a registered user redeeming while signed in as themselves; over the whole table the owner holds no row for that address, and their own enrolment is untouched | **Measured**                                                                                                                                                                                                                                                                        |
| The same request carrying no session at all reaches the same state, so the session is not overridden but never read                                                                                                                      | **Measured**, the control in the same run                                                                                                                                                                                                                                           |
| Bind patch: the two tests that redeem a token without a session stop; the three add-only controls pass                                                                                                                                   | **Measured**, counterfactual, 2026-09-05                                                                                                                                                                                                                                            |
| Bind patch, second hunk: with a valid session on the request the requester comparison refuses it with 403, where the sessionless control still stops at the middleware with 401                                                          | **Measured**, counterfactual, 2026-09-05                                                                                                                                                                                                                                            |
| Recheck patch: the second account's commit is refused, 400 where it was 200                                                                                                                                                              | **Measured**, counterfactual, 2026-09-05                                                                                                                                                                                                                                            |
| The checked-in `wrangler.jsonc` declares no route restriction at the Cloudflare edge                                                                                                                                                     | Argued from source; see the Description                                                                                                                                                                                                                                             |
| No database constraint refuses a second verified row on one mailbox                                                                                                                                                                      | Argued from source; see the Description                                                                                                                                                                                                                                             |
| The verify landing page is reachable by a signed-out visitor                                                                                                                                                                             | Argued from source. TanStack file-based routing parents a route by its directory; `RequireBackendAuth` is at `apps/manager/src/routes/notifications/_authenticated/route.tsx:10` and `verify.tsx` does not live under `_authenticated/`; corroborated by `routeTree.gen.ts:523-529` |
| The emailed link alone does not auto-submit                                                                                                                                                                                              | Argued from source; see the Description                                                                                                                                                                                                                                             |
| The webhook resolves a channel by address with no account term                                                                                                                                                                           | Argued from source (`webhook/sendgrid.ts:172-174`)                                                                                                                                                                                                                                  |

### The captures

The full run records, including the unhandled-rejection block and the patched arms, are the transcripts this pack was built from. The rejection that produces the suite's exit code, abridged (elisions marked `[...]`; the `?v=<hash>` suffixes vitest appends to module paths are stripped):

```
⎯⎯⎯⎯ Unhandled Rejection ⎯⎯⎯⎯⎯
Error: no authorization included in request
 ❯ jwt2 ../../node_modules/.pnpm/hono@4.12.34/node_modules/hono/dist/middleware/jwt/jwt.js:60:13
 ❯ src/app/middleware/auth.ts:13:10
[...]
 ❯ src/app/routes/notifications/channels/channel_bind_without_owner_consent.poc.test.ts:249:32
[...]
The latest test that might've caused the error is "completes verification with no caller identity and commits the binding to the requesting account". [...]
```

The stack resolves to `:249`, the `GET /notifications/channels` control, not to a binding assertion.

### What a green run does not settle

* **That mail arrives in an inbox.** Nothing is transmitted. The tests measure the request the worker hands to the mail provider, at the module boundary where the network call is made.
* **That the enrolment denial is permanent.** The release paths that can end it, and their limits, are set out in the Description under "The release paths, and their limits"; none is exercised here.

### Placing the files

The runnable copies, all three test files, the support module and both patches, are in the Gist linked above, byte-identical to the excerpts in this field; its `README.md` carries the run and patch commands. The sha256 of the four files the suite ran as is in the TEST IDENTITY block at the top of this field, so `sha256sum` on the Gist copies settles that the transcript above is of those bytes.
