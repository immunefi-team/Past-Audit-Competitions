# 91256 w a low unauthenticated verify endpoint binds a victim s email address to the attacker s ens account

## #91256 \[W\&A-Low] Unauthenticated verify endpoint binds a victim's email address to the attacker's ENS account

**Submitted on Aug 31st 2026 at 03:37:40 UTC by @UptightChipmunk990 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91256
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email

### Description

## Unauthenticated verify endpoint binds a victim's email address to the attacker's ENS account

**Asset:** `workers/api-worker` · branch `audit-comp-ready` @ `1c9b47f` · Route: `POST /notifications/channels/email/verify`\
**Selected impact (High):** *Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email*

### Brief/Intro

The endpoint that promotes an email notification channel to `verified` has no authentication and no ownership check. An attacker registers a victim's email address on the attacker's own ENS account; ENS then emails the victim a genuine verification link, and the victim's single click binds their address to the attacker's account. In production this lets any anonymous user, at the cost of one free login, permanently seize a chosen email address: the victim can no longer register it for their own notifications, ENS begins delivering the attacker's notifications to the victim's inbox, and the victim's global SendGrid contact record is retagged to the attacker.

### Vulnerability Details

In `workers/api-worker/src/app/routes/notifications/channels/email.ts` the add handler is authenticated but the verify handler, in the same file importing the same middleware, is not:

```ts
 4  import { requireAuth } from '#app/middleware/auth.js'
29  .post('/', ...requireAuth, vValidator('json', addEmailChannelBodySchema), ...)    // authenticated
172 .post('/verify', vValidator('json', verifyEmailChannelBodySchema), injectDb, ...) // NO requireAuth
```

The verify handler resolves a channel row from the token alone and promotes it, checking neither the caller's identity nor whether the address is already verified on another account (`:179-201`):

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

It then upserts the global SendGrid contact, keyed by email and tagged with the row owner's id (`:233-243` → `services/sendgrid/contacts.ts:53-78`):

```ts
addContactToList(env, verification.channel.target /* victim email */, verification.user_id /* attacker */)
```

The guard that should stop this lives only on the add path and matches only addresses already `verified` elsewhere (`:59-72`), and per-user uniqueness lets any account hold a `pending` row for any address (`core/database/schema/notifications.ts:74-78`, `unique(user_id, channel, target)`). The verify path applies none of it.

**Attacker and preconditions.** An unprivileged remote party with one SIWE login (a fresh keypair — no funds, name, or on-chain state). They must know the victim's email address but never access it; the attacker sends the victim no communication of their own — ENS's system sends the verification email through its normal flow.

**Recommended fix.** Apply `requireAuth` to `/verify` and require `verification.user_id === c.var.user_id`; re-run the add-handler uniqueness guard at verification time; reject promotion of a non-`pending` row.

### Impact Details

A verified email channel is bound to the attacker's account with one victim click:

* **The victim loses their own email address for notifications, permanently.** Their subsequent `POST /notifications/channels/email` hits the `:59-72` guard and returns *"This email address is already linked to another account."* There is no self-service reclaim — the row belongs to the attacker, and deletion requires row ownership.
* **ENS delivers the attacker's notifications to the victim's inbox.** Delivery enumerates every `status='verified'` channel (`services/notifications/create.ts:150-190`), so the attacker's name expiries, transfers and transactions are mailed to the victim.
* **The victim's global SendGrid contact record is mutated** — its `user_id` custom field now identifies the attacker, and the address is re-added to the broadcast list even if the victim had unsubscribed.

This matches the selected in-scope impact exactly: a sensitive detail of another user — their email — is changed with one click of user interaction and no connected-wallet interaction. One click is within that tier's stated budget; the zero-interaction Critical variant does not apply here, because the verification token is 128-bit CSPRNG, emailed only to the victim, and returned by no endpoint, so the attacker cannot complete the flow without the victim's click.

This is not a known issue. R2-05 is the *inbound* SendGrid webhook signature check (fails closed at this commit); this is a first-party route with no signature. SEC-MGR-003 is client-side token exfiltration; this requires no token. SEC-MGR-008/011 are debug routes; `/verify` is a production feature route. No audit finding, QA item, or WEB bug-tracker ticket concerns notification channels or email verification. It is distinct from the separately reported channel-deletion bug: a different route, a different missing control (absent authentication vs. absent status check), and a different SendGrid primitive.

### References

* Verify handler (no auth): `workers/api-worker/src/app/routes/notifications/channels/email.ts#L172-L258`
* Auth applied on the add handler for contrast: `…/channels/email.ts#L29-L33`
* Add-handler uniqueness guard the verify path skips: `…/channels/email.ts#L59-L72`
* Global SendGrid upsert: `workers/api-worker/src/services/sendgrid/contacts.ts#L53-L78`
* Per-user uniqueness on a global identifier: `workers/api-worker/src/core/database/schema/notifications.ts#L74-L78`
* Delivery to all verified channels: `workers/api-worker/src/services/notifications/create.ts#L150-L190`
* Repo: <https://github.com/immunefi-team/audit-comp-ens> (branch `audit-comp-ready`, commit `1c9b47f`)

### Proof of Concept

### Steps to Reproduce

{% stepper %}
{% step %}

#### Register the victim's email address

As the attacker (Wallet A), register the victim's email address via notification settings. This creates a pending channel owned by the attacker and causes ENS to send a genuine verification link to the victim's inbox.
{% endstep %}

{% step %}

#### Verify the channel without authentication

The `/verify` endpoint performs no authentication and no check that the caller owns the channel — it verifies whatever channel the token maps to. Submitting the token with no credentials succeeds:

```javascript
fetch('https://app-api.ens.dev/notifications/channels/email/verify', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ token: '<token from the verification email>' })
})
```

```
→ 200 {"message":"Email verified successfully"}
[Screenshot 1 — request carries no authorization header]
```

In a real attack the token is delivered only to the victim's inbox, so the victim supplies it by clicking the genuine ENS verification link (the single click of user interaction). This PoC submits the token directly to prove the endpoint requires no authentication.
{% endstep %}

{% step %}

#### Confirm verification on the attacker's account

The email is now VERIFIED on the attacker's account (Wallet A).

\[Screenshot 2]
{% endstep %}

{% step %}

#### Attempt to register the same address as the victim

The victim (Wallet B, a different account) attempts to register the SAME address and is permanently refused:

```
POST /notifications/channels/email  {"email":"rdhagenstar+poc@gmail.com"}
```

```
[Screenshot 3 — payload]
→ 400 {"error":"This email address is already linked to another account"}
[Screenshot 4 — response]
```

Because `/verify` never checks that the caller owns the channel being verified, the victim's own verification click binds their email to the attacker's account rather than their own. The victim can never reclaim the address (Step 4), and ENS notifications for the attacker's account are delivered to the victim's inbox.
{% endstep %}
{% endstepper %}
