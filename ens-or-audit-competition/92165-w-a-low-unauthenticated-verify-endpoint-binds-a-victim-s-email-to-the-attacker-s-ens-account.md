# 92165 w a low unauthenticated verify endpoint binds a victim s email to the attacker s ens account

**Submitted on Sep 7th 2026 at 17:48:56 UTC by @UptightChipmunk990 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92165
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email

## Description

## Brief / Intro

`POST /notifications/channels/email/verify` has no authentication and no ownership check. An unprivileged remote party, at the cost of one free SIWE login, registers a victim's email address on the **attacker's** account; ENS then emails the victim a genuine verification link, and the victim's single click promotes the channel to `verified` **on the attacker's account**. The address is then permanently bound to the attacker: the victim can no longer register it for their own notifications, ENS begins delivering the attacker's notifications to the victim's inbox, and the victim's global SendGrid contact record is retagged to the attacker. The proof of concept below **carries this out end-to-end across two independent accounts**, with the promoting request issued from a **logged-out** session.

## Vulnerability Details

In `workers/api-worker/src/app/routes/notifications/channels/email.ts`, the add handler is authenticated but the verify handler — in the same file, importing the same middleware — is not:

```ts
 4  import { requireAuth } from '#app/middleware/auth.js'
29  .post('/', ...requireAuth, vValidator('json', addEmailChannelBodySchema), ...)    // authenticated
172 .post('/verify', vValidator('json', verifyEmailChannelBodySchema), injectDb, ...) // NO requireAuth
```

The verify handler (`L179-L201`) resolves a channel row from the token alone and promotes it, checking **neither the caller's identity nor whether the address is already verified on another account**:

```ts
const verification = await c.var.db.query.channelVerifications.findFirst({
  where: and(eq(token, token), eq(purpose, 'verify'), gt(expires_at, new Date())),
  with: { channel: true },
})
if (!verification) return c.json({ error: 'Invalid or expired verification token' }, 400)
await c.var.db.update(TABLE.userChannels)
  .set({ status: 'verified', verified_at: new Date() })
  .where(eq(TABLE.userChannels.id, verification.channel_id))
```

It then upserts the global SendGrid contact, keyed by email and tagged with the row owner's id (`L233-L243` → `services/sendgrid/contacts.ts:L53-L78`): `addContactToList(env, verification.channel.target /* victim email */, verification.user_id /* attacker */)`. The guard that should stop this lives only on the add path and matches only addresses already `verified` elsewhere (`L59-L72`); per-user uniqueness (`core/database/schema/notifications.ts:L74-L78`, `unique(user_id, channel, target)`) lets any account hold a `pending` row for any address. The verify path applies none of it.

## Impact Details

With one victim click and one free login, a verified email channel is bound to the attacker's account:

* **The victim permanently loses their own email for notifications.** Their subsequent `POST /notifications/channels/email` hits the `L59-L72` guard and returns *"This email address is already linked to another account"* (proven, panel 6). There is no self-service reclaim — the row belongs to the attacker and deletion requires row ownership.
* **ENS delivers the attacker's notifications to the victim's inbox.** Delivery enumerates every `status='verified'` channel (`services/notifications/create.ts:L150-L190`), so the attacker's name expiries, transfers and transactions are mailed to the victim.
* **The victim's global SendGrid contact record is mutated** — its `user_id` custom field now identifies the attacker, and the address is re-added to the broadcast list even if the victim had unsubscribed. This matches the selected in-scope impact exactly: a sensitive detail of another user — their email — is changed with **one click** of user interaction and **no connected-wallet interaction**. One click is within that tier's stated budget; the zero-interaction Critical variant does not apply, because the token is a 128-bit CSPRNG value emailed only to the victim and returned by no endpoint, so the victim's click is required.

**Not a known issue.** R2-05 is the *inbound* SendGrid webhook signature check (fails closed at this commit); this is a first-party route with no signature. SEC-MGR-003 is client-side token exfiltration; this requires no token theft. SEC-MGR-008/011 are debug routes; `/verify` is a production feature route. No audit finding, QA item, or WEB ticket concerns notification channels or email verification. Distinct from the channel-deletion bug: different route, different missing control (absent authentication vs. absent status check), different SendGrid primitive.

**Recommended fix.** Apply `requireAuth` to `/verify` and require `verification.user_id === c.var.user_id`; re-run the add-handler uniqueness guard at verification time; reject promotion of a non-`pending` row.

## References

All permalinks pinned to commit `1c9b47f` (line ranges verified against that commit):

* Verify handler (no auth): <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f/workers/api-worker/src/app/routes/notifications/channels/email.ts#L172-L261>
* Auth applied on the add handler for contrast: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f/workers/api-worker/src/app/routes/notifications/channels/email.ts#L29-L33>
* Add-handler uniqueness guard the verify path skips: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f/workers/api-worker/src/app/routes/notifications/channels/email.ts#L59-L72>
* Global SendGrid upsert: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f/workers/api-worker/src/services/sendgrid/contacts.ts#L53-L78>
* Per-user uniqueness on a global identifier: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f/workers/api-worker/src/core/database/schema/notifications.ts#L74-L78>
* Delivery to all verified channels: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f/workers/api-worker/src/services/notifications/create.ts#L150-L190>

## Proof of Concept

## Proof of Concept — the cross-account seizure, carried out

The attached composite (`ens-verify-crossaccount-poc.png`) shows the full flow across **two genuinely separate accounts** — different SIWE logins, different Bearer tokens — with the promoting call made from a logged-out session. Panels:

1. **The `/verify` request carries NO `Authorization` header**, submitted from a logged-out session (`POST /notifications/channels/email/verify`).
2. **That unauthenticated `/verify` returns `200 OK`** (`{"message":"Email verified successfully"}`).
3. **The address is now `Verified` on attacker account A** — connected wallet `0x3cD4…027D`, shown top-right.
4. **A separate account B** submits the same address — note the **different Bearer token** (a distinct account, not account A).
5. **B's request payload:** the same address, `rdhagenstar+poc5@gmail.com`.
6. **B is refused:** `{"error":"This email address is already linked to another account"}` → the address is bound to A, and B is locked out. Steps to reproduce:
7. **Account A (attacker)** — one SIWE login from a fresh keypair (no funds, name, or on-chain state). `POST /notifications/channels/email {"email":"<victim address>"}` → accepted (no verified row exists); a `pending` row is created under A's id and ENS emails the address a genuine verification link.
8. **Promote from a logged-out session** — `POST /notifications/channels/email/verify` with only a `content-type` header and the token from the email; **no `Authorization` header** (panel 1) → `200 OK` (panel 2). The address is now `verified` on account A (panel 3).
9. **Account B (any other user / the address's real owner)** — a separate login with a different Bearer token (panel 4) attempts to register the same address (panel 5) → refused, *"already linked to another account"* (panel 6).

### Why the reporter's own inbox does not limit this finding

The first review objected that the token was verified from an inbox the reporter controls. It does not matter, because **the endpoint never reads the caller's identity.** `/verify` has no `requireAuth` (`email.ts:L172`) and its handler (`L179-L201`) resolves and promotes the channel from the **token alone**. The HTTP request that promotes the channel is therefore **byte-identical** whether it originates from the victim clicking the genuine ENS link or from any other party who holds that token — the submitter's identity is not an input to the vulnerability. In production the token is a 128-bit CSPRNG value delivered only to the victim's inbox and returned by no endpoint, so the victim's own click is the natural trigger; the PoC reproduces that exact unauthenticated request (panel 1). Demonstrating against a non-consenting third party's real inbox would be both unnecessary (the server cannot distinguish the source) and unethical. The cross-account binding is proven directly: account B — a different account — is locked out because the row is owned by account A (panels 4–6), which is only possible if the address bound to A.
