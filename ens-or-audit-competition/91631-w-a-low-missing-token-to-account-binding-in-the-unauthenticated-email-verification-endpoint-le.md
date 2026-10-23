# 91631 w a low missing token to account binding in the unauthenticated email verification endpoint leads to cross account mailbox binding victim lock out and an email membership oracle

**Submitted on Sep 3rd 2026 at 06:36:32 UTC by @Bee55357 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91631
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

The API worker lets any authenticated user register an arbitrary email address as a notification channel on their own account, then verifies that channel through an endpoint that requires no authentication and binds the verification token to nothing but itself. The account that receives the notifications is chosen by whoever started the request, while the person who proves control of the mailbox is someone else entirely. In production this lets an attacker attach a victim's mailbox to the attacker's own ENS account after the victim opens an unsolicited but entirely genuine ENS verification email and presses its Verify button. That redirects the account's name-expiry notifications into the victim's inbox from the official ENS sender, blocks the victim from registering their own address on their own account until they unsubscribe or file a spam report, and turns the resulting error string into an oracle for whether a given address is already verified as a notification channel on another account.

## Vulnerability Details

Two endpoints in `workers/api-worker/src/app/routes/notifications/channels/email.ts` combine to produce the flaw.

**Registration accepts any address.** `POST /notifications/channels/email` is authenticated, but it accepts any email address the caller supplies. The only rejection path that guards a third party's address is the check at lines 58-72, with the status filter at line 63. The handler's other rejections do not block this attack: a same-account duplicate check that fires only when the caller already holds that address verified (lines 48-54), and a per-address rate limit of three verification sends per hour returning 429 (lines 126-140; constants at `services/notifications/email-verification-rate-limit.ts` lines 3-4), which runs only after the channel row and the token have already been inserted. A pending row is inserted as `{user_id: <caller>, channel: 'email', status: 'pending', target: <arbitrary address>}` (lines 75-97) and the worker mails that address a genuine verification link (lines 143-149).

**The verification email carries no account context.** `services/email/verification.ts` builds a message that names no wallet address, no ENS name, and no account identifier, in both the text/plain body (lines 30-38) and the HTML body (lines 42-83). The only value interpolated into either body is the verification URL (line 16); `sendVerificationEmail` is never even passed an account identifier. A recipient therefore has nothing against which to judge whether the request was theirs. Both bodies carry the standard notice "If you didn't request this email, you can safely ignore it" (lines 36 and 73). That notice is the only recipient-facing control present, and it is weakened here because the message is genuinely generated and sent by the platform's own sender, so none of the usual signals of a forged message are available to the recipient.

**Verification is unauthenticated and unbound.** `POST /notifications/channels/email/verify` carries no `requireAuth` middleware (lines 172-176). It resolves the channel purely from the 128-bit token (lines 179-188), marks that channel `'verified'` (lines 195-201), sends the welcome mail (lines 208-230), and enrols the address in the SendGrid broadcast list with `custom_fields.user_id` set to the **original requester** (lines 232-258).

The root cause is the absence of any relation between the token and the identity that redeems it. The handler never compares the redeeming session to the owner of `verification.channel_id`, and the token is not hash-bound to that owner. Because `/verify` is unauthenticated, there is not even a session available to compare against.

Downstream, both fan-out paths select channels at status `'verified'` and treat the mis-bound channel as fully legitimate: the production name-expiry path at `queues/event-ingestion.ts` lines 279-291, and `services/notifications/create.ts` lines 116-123. Delivery then goes to `channel.target` at `services/delivery/email.ts` lines 61-76. The victim's escape is to unsubscribe or file a spam report, which flips a `user_channels` row matching that address through the SendGrid event webhook (spam report at `webhook/sendgrid.ts` lines 205-220, unsubscribe at lines 222-237; the lookup at lines 172-174 matches on `target` alone).

## Impact Details

This maps to the in-scope impact **"Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications."**

The detail of another user that changes is the victim's notification-enablement state. After the attack the victim cannot enable email notifications on their own account for their own address, and their mailbox begins receiving ENS notification mail they never subscribed to. No interaction with the victim's wallet occurs at any point; the only signature in the flow is the attacker's own SIWE login to the attacker's own account. The victim's involvement is opening the emailed link, which is a navigation, and then one click on the "Verify Email Address" button on the ENS-hosted page (`apps/manager/src/routes/notifications/channels/email/verify.tsx` lines 218-225, onClick at line 220). I read that button press as the single click of user interaction contemplated by this impact, since arriving at the page is navigation rather than an in-application action, and I state the count plainly rather than asserting a bare one click. If the panel counts the link open as a second interaction, I ask that the finding be graded on the lock-out effect at whatever tier the panel considers correct. On the ENS side the row written is the attacker's own `user_channels` row, inserted with the attacker's `user_id` (lines 75-82) and flipped to `'verified'` (lines 195-201); no row owned by the victim is created or modified, and no code path in the worker ever rewrites `user_channels.user_id`.

The consequences:

1. **Cross-account mailbox binding.** The attacker's account holds a verified email channel pointing at a mailbox the attacker does not control. Subsequent notifications for the attacker's account are delivered to the victim from the official ENS sender using the template selection at `services/delivery/templates/email.ts` (lines 18-40). This is a durable, ENS-branded message channel into a chosen inbox, and a materially better phishing primitive than a spoofed email because the sender, template, and infrastructure are authentic.
2. **Victim lock-out.** Once the address is verified on the attacker's account, the victim signing in with their own wallet and attempting to register the same address receives `400 {"error":"This email address is already linked to another account"}` (lines 67-71). The victim cannot enable email notifications on their own account for their own address until they unsubscribe or spam-report.
3. **Email-membership oracle.** That same rejection string answers, for any address an attacker supplies, whether it is currently verified as a channel on another account. The oracle is asymmetric: a positive result returns at line 67 before any write and consumes no rate-limit budget, while a negative result falls through to the channel insert (lines 75-97), the token insert (lines 104-120) and `sendVerificationEmail` (lines 143-149), so each miss writes a pending row and mails the probed address a genuine ENS verification email, capped at three per address per hour. Enumeration is therefore noisy rather than silent, and I state that rather than overstating the primitive.
4. **Pre-binding.** Because registration only rejects `'verified'` collisions and not `'pending'` ones, an attacker can pre-stage pending rows against many addresses in advance.

The limits of this finding: there is no account takeover, and no funds or names are at risk. Email is used for outbound delivery only. Authentication is SIWE plus JWT and never keys on email, so binding a mailbox grants no authority over any account. I claimed Medium rather than a higher tier for that reason.

Two adjacent impacts I considered and deliberately did **not** claim:

* **"Taking state-modifying authenticated actions on behalf of other users without any interaction by that user"** names notification and settings changes and is otherwise a close match, but it requires no interaction by the victim. This attack requires the victim's click, so it does not qualify.
* **"Changing sensitive details of other users ... such as: Email"** is arguable, since after the attack the victim's mailbox is bound to an account they do not control and they cannot register it themselves. I did not claim it because the row written on the ENS side is the attacker's own, and the uniqueness constraint is per-user on `(user_id, channel, target)` rather than global on the address (`core/database/schema/notifications.ts` lines 73-79). The change to the victim is the lock-out and the unconsented enrolment, not an edit of a record they own. If the panel reads it the other way, a High would follow.

## References

Repository: `https://github.com/immunefi-team/audit-comp-ens`, branch `audit-comp-ready`, commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. Asset: **Workers**.

Primary file, `workers/api-worker/src/app/routes/notifications/channels/email.ts`: same-account duplicate check at lines 48-54; cross-account check at lines 58-72 with the status filter at line 63 and the rejection string at lines 67-71; channel insert at lines 75-97; token insert at lines 104-120; rate limit at lines 126-140; verification send at lines 143-149; success body at lines 165-169; unauthenticated verify route at lines 172-176; token lookup at lines 179-188; status flip at lines 195-201; welcome mail at lines 208-230; SendGrid enrolment at lines 232-258.

Supporting files:

* `workers/api-worker/src/services/email/verification.ts`, line 16 and lines 30-38 and 42-83: URL construction, and text and HTML bodies with no account context
* `workers/api-worker/src/services/notifications/email-verification-rate-limit.ts`, lines 3-4: three sends per address per hour
* `workers/api-worker/src/queues/event-ingestion.ts`, lines 225, 279-291, 341-349: production name-expiry fan-out, channel selection filtered to verified, settings gate
* `workers/api-worker/src/services/notifications/create.ts`, lines 116-123: selects verified channels
* `workers/api-worker/src/services/notifications/helpers.ts`, lines 95-115: `toPublicChannel` response shaping
* `workers/api-worker/src/services/delivery/email.ts`, lines 61-76: delivery to `channel.target`
* `workers/api-worker/src/services/delivery/templates/email.ts`, lines 18-40: template and subject selection
* `workers/api-worker/src/app/routes/webhook/sendgrid.ts`, lines 172-174, 205-220, 222-237: target-only lookup, spam report, unsubscribe
* `workers/api-worker/src/app/routes/notifications/channels/index.ts`, line 46: channel listing filtered to the authenticated caller
* `workers/api-worker/src/app/routes/auth/index.ts`, lines 13-82: SIWE nonce and login
* `workers/api-worker/src/core/database/schema/notifications.ts`, lines 73-79: per-user uniqueness on `(user_id, channel, target)`
* `apps/manager/src/routes/notifications/channels/email/verify.tsx`, lines 52-60 and 218-225: verify route declares no auth guard and no loader; the POST fires only from the "Verify Email Address" button's onClick at line 220
* `apps/manager/src/utils/backend-client.ts`, lines 143-158: client omits the Authorization header entirely when signed out

Severity framework: Immunefi Vulnerability Severity Classification System v2, Websites and Applications, Medium (Level 3).

## Proof of Concept

## Proof of Concept

Method note: this was derived by source analysis of the competition repository `immunefi-team/audit-comp-ens` at branch `audit-comp-ready`, commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. No traffic was sent to any live ENS, Immunefi, or third-party system at any point, in line with the program's prohibition on testing against deployed environments. The steps below are written against the in-scope worker source and reference the exact handlers that produce each state change.

**Setup.** The attacker controls wallet A. The mailbox they wish to weaponise is `victim@example.com`, which the victim has not yet verified on their own account.

**Step 1: Attacker authenticates.** `POST /auth/nonce`, then `POST /auth/login` with `{address, message, signature, nonce}` (`app/routes/auth/index.ts` lines 13-82). The attacker receives a bearer token, `JWT_A`.

**Step 2: Attacker registers the victim's address on the attacker's own account.**

```
POST /notifications/channels/email
Authorization: Bearer JWT_A
Content-Type: application/json

{"email":"victim@example.com"}
```

Expected response: `200 {"message":"Verification email sent","expires_at":"<ISO 8601, 24h out>","channelId":"C"}` (lines 165-169). The token is not returned, which is why the attacker cannot redeem it and the victim's click is required.

The handler's only check against other accounts is the query at lines 58-72 (filter at line 63); the check at lines 48-54 rejects only an address the caller has already verified, and the rate limit at lines 126-140 permits three sends per address per hour. Finding no verified row on another account, it inserts `{user_id: <attacker>, channel: 'email', status: 'pending', target: 'victim@example.com'}` (lines 75-97) and dispatches the verification mail (lines 143-149).

**Step 3: Victim receives a genuine ENS email.** Subject `Verify your email address - ENS Notifications`, from the official ENS sender, containing `MANAGER_APP_URL/notifications/channels/email/verify?token=T` (built at `services/email/verification.ts` line 16). Per lines 30-38 and 42-83, neither the text nor the HTML body names a wallet, an ENS name, or an account, so the victim has no way to tell the request did not originate from them.

**Step 4: Victim opens the link, then presses "Verify Email Address".** The emailed link is a plain GET navigation to the manager app. The page does not verify on load: its route declares only `component`, `validateSearch` and `loaderDeps` with no `loader` (`apps/manager/src/routes/notifications/channels/email/verify.tsx` lines 52-60), the file contains no `useEffect`, and the request fires only from `handleVerify` (lines 77-92), reached from the button's `onClick` at line 220 (label `Verify Email Address` at line 224). The worker exposes verification only as POST (lines 172-176), so the GET from the mail client cannot verify anything. The route carries no `beforeLoad` guard and sits outside the `_authenticated` layout, and `apps/manager/src/utils/backend-client.ts` omits the Authorization header entirely when no auth key is stored, so a signed-out visitor completes it and the victim need not have an account at all. On that button press the page issues:

```
POST /notifications/channels/email/verify
Content-Type: application/json
(no Authorization header)

{"token":"T"}
```

The handler has no `requireAuth` (lines 172-176), resolves the channel from the token alone (lines 179-188), marks channel `C` as `'verified'` (lines 195-201), sends the welcome mail (lines 208-230), and enrols `victim@example.com` on the broadcast list with `custom_fields.user_id` set to the **attacker** (lines 232-258).

**Step 5: Confirm the mis-binding.** `GET /notifications/channels` with `Authorization: Bearer JWT_A`. Expected: channel `C` present with `"channel":"email"`, `"status":"verified"`, a non-null `verified_at`, and `"label":"victim@example.com"`. The response is shaped by `toPublicChannel` (`services/notifications/helpers.ts` lines 95-115), which emits `label` in place of `target` and never returns `user_id`. Ownership by the attacker follows from the handler filtering on the authenticated caller's own id (`app/routes/notifications/channels/index.ts` line 46), so the row appearing under `JWT_A` is itself the proof.

**Step 6: Demonstrate the victim lock-out.** The victim signs in with their own wallet, obtaining `JWT_V`, and issues `POST /notifications/channels/email` with `{"email":"victim@example.com"}`. Expected: `400 {"error":"This email address is already linked to another account"}` (lines 67-71). The victim cannot use their own address on their own account.

**Step 7: Demonstrate the membership oracle.** Repeat Step 6's request with any arbitrary third-party address. The presence or absence of that same 400 string discloses whether the address is verified as a channel on another account. A negative probe is not passive: it creates a pending row on the probing account and sends a genuine ENS verification email to the probed address, bounded at three sends per address per hour, which is why this step was reasoned from source rather than executed.

**Step 8: Demonstrate the persistent channel.** The attacker first enables their own expiry preference, since `user_notification_settings` defaults to false in code and in the schema (`queues/event-ingestion.ts` lines 341-343). A name the attacker owns then approaches expiry. The production expiry path is `queues/event-ingestion.ts`: it creates the `name-expiry` notification (line 225), loads that user's channels filtered to `status: 'verified'` (lines 279-291), passes the settings gate (lines 341-349), and queues a delivery to `channel.target`. `services/delivery/email.ts` lines 61-76 then calls SendGrid with `to: [{ email: deliveryJob.target }]` (line 66), now `victim@example.com`, from the configured ENS sender, using the template id and subject selected in `services/delivery/templates/email.ts` (`name-expiry` lines 18-29). This persists until the victim unsubscribes or files a spam report, which flips a `user_channels` row matching that address (`webhook/sendgrid.ts` lines 205-220 and 222-237).

**Preconditions.** The victim opens one unsolicited verification link and then presses the "Verify Email Address" button on the page it opens; the page does not verify on load. The victim's address must not already be verified on the victim's own account.

**Suggested fix.** Bind the token to the account that owns the channel. Require `/verify` to be performed by the same `user_id` that owns `verification.channel_id` (lines 179-201), or hash-bind the token to that `user_id` so a token cannot verify a channel owned by a different account. Include the requesting wallet or account identifier in the verification email so the recipient can see whose account they are confirming. Reject registration of an address already `'pending'` on another account, not only one already `'verified'`.

**Known-issue check.** Not listed among the entries itemized in Part 1 of the Public Disclosure of Known Issues. The nearest listed entries are R2-05 (SendGrid event webhook fails open when the verification key is unset, which is fixed in this fork at `webhook/sendgrid.ts` lines 110-152) and EXP-INPUT-009 (push notification click opens a backend-supplied URL). Neither concerns channel-to-account binding, which is the root cause here. Under a root-cause rather than entry-point test the nearest neighbour is EXP-4337-003 (`request.from` is not checked against the connected signer), which shares the shape of a requester-supplied identity never checked against the authenticated one. It is a different asset and a different subsystem: ERC-4337 userop construction in the transaction manager, not the API worker's notification channels.
