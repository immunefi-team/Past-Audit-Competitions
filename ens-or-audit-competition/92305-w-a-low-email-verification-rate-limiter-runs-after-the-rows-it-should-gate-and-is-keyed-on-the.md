# 92305 w a low email verification rate limiter runs after the rows it should gate and is keyed on the target address so any account can burn a chosen email s hourly quota and block that person fr&#x20;

**Submitted on Sep 8th 2026 at 20:27:59 UTC by @Nyko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92305
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

The email-verification endpoint in `workers/api-worker` enforces its "3 verification emails per hour" limit with a check that (a) runs AFTER it has already committed the two database rows the limit is meant to gate, and (b) is keyed on the target email address rather than on the authenticated caller. Because the key is the recipient address, any authenticated account can spend a chosen address's hourly quota by naming it three times. The owner of that address is then unable to receive their own verification email and cannot enable email notifications, without ever having interacted with the attacker. A secondary consequence is that every request, including rate-limited ones, still inserts a row into an unindexed `channel_verifications` table.

All line numbers below were checked against the frozen revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, clone verified unmodified.

## Vulnerability Details

**1. The limiter runs after the writes it should gate.** `workers/api-worker/src/app/routes/notifications/channels/email.ts`, the `POST` handler:

```
:76   insert into user_channels (…onConflictDoUpdate…)   → committed
:105  insert into channel_verifications { token: generateToken(), … }  → committed  (:111)
:126  const rateLimit = await checkAndConsumeEmailVerificationRateLimit(c.env.KV, email)
:131  if (!rateLimit.isAllowed) return c.json({ … }, 429)   (:138)
```

The 429 at `:138` is returned only after both inserts at `:76` and `:105` have already been persisted. A rate-limited request is not prevented; it is reported as rejected while its rows remain written. The limiter can throttle the email SEND but not the state changes.

**2. The quota is keyed on the victim, not the caller.** `workers/api-worker/src/services/notifications/email-verification-rate-limit.ts`:

```
export const checkAndConsumeEmailVerificationRateLimit = async (kv, email) => {
  const normalizedEmail = normalizeEmailForRateLimit(email)
  const key = KV_KEY.NOTIFICATIONS.EMAIL_VERIFICATION(normalizedEmail)
  …
  EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS = 3, window = 3600 s
```

The counter is stored under the normalized recipient address. There is no per-account or per-caller quota anywhere on this path. Accounts are SIWE-authenticated, so obtaining one costs a signature, not an invitation.

**3. The collision gate does not stop the cross-account case.** Before the writes, the handler checks (`:59-70`) whether the SAME email is already `status:'verified'` for a DIFFERENT user, and only then rejects. While the victim has not yet verified — which is precisely the window this defect targets — that gate is empty, so the attacker's request proceeds to the writes and the quota consumption.

**4. The written table is unindexed.** `workers/api-worker/drizzle/0000_vengeful_golden_guardian.sql:51-63` creates `channel_verifications`; across all five migrations the only index is `users_address_index` (`:129`) and the unique constraints are on `users.address`, `notification_preference`, `notifications.idempotency_key`, and `user_channels(user_id,channel,target)`. `channel_verifications` has no index and no unique constraint on `token`, yet the verification-click path looks a row up by exactly that column (`email.ts:181`, `eq(channelVerifications.token, token)`).

## Impact Details

**Primary (the selected impact): cross-account denial of a person's email-notification enrollment, zero victim interaction.** An authenticated attacker sends three `POST` requests naming the victim's email address. Each commits the attacker's own pending `user_channels` row and a `channel_verifications` row, and increments the KV counter keyed on that address. After the third, the counter is at the limit for the hour. When the victim later tries to enrol the same address for ENS notifications, their request reaches `:126`, the limiter reports the quota exhausted, and they receive a 429 and no verification email. Repeating three requests per hour keeps the address permanently unenrollable. The victim never interacted with the attacker; the attacker needs only the victim's email address. This maps to the selected impact, changing another user's notification enablement without their interaction, and is in fact reached with zero clicks rather than the one the tier allows.

**Secondary: unbounded growth of an unindexed table.** Because the writes at `:76`/`:105` are never gated by the limiter, the 3/hour cap governs only email sends; `channel_verifications` rows are inserted on every request, limited or not. Every legitimate verification click then runs an unindexed `findFirst` by `token` (`:181`) over a table that only grows. This is a consequence of the same statement-order defect, not a separate volumetric claim, and is given here as severity context for the primary impact.

## Why this is not a published known issue

Nothing in the known-issues list touches rate limiting, statement ordering, or the `channel_verifications` schema. **R2-05** is the SendGrid webhook failing open when the verification key is unset, a different file and a different mechanism. The limiter's own documented weakness in this repo is the "non-atomic read-modify-write on KV" comment, which concerns overshoot bounding, not the ordering or the victim-keying described here.

## References

* `workers/api-worker/src/app/routes/notifications/channels/email.ts:59-70, 76, 105, 111, 126, 131, 138, 181`
* `workers/api-worker/src/services/notifications/email-verification-rate-limit.ts` (keyed on the recipient; MAX\_SENDS=3, window 3600 s)
* `workers/api-worker/drizzle/0000_vengeful_golden_guardian.sql:51-63` (`channel_verifications`), `:129` (the sole index)
* Frozen revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

## Proof of Concept

Preconditions: two SIWE accounts (attacker A; the "victim" account V is only used to show the denial). `victim@example.com` is the address to be blocked; A does not control it.

1. As A, `POST /notifications/channels/email` with `{ target: "victim@example.com" }`. The handler finds no verified row for that address under another user (`:59-70` empty), commits a pending `user_channels` row for A (`:76`) and a `channel_verifications` row (`:105`), then consumes the quota for `victim@example.com` (`:126`). Count = 1.
2. Repeat twice more. Count = 3 = `EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS`.
3. As V (the real owner of `victim@example.com`), `POST /notifications/channels/email` with `{ target: "victim@example.com" }`. The writes at `:76`/`:105` still execute, then the limiter at `:126` reports the quota exhausted and V receives `429` with "Too many verification emails sent to this address" (`:138`). No verification email is sent to V.
4. V cannot enrol email notifications for the rest of the window, and A repeating step 1 three times per hour keeps it that way indefinitely.

Observed vs expected: the limiter is intended to cap how often an address is emailed; instead it lets any caller exhaust a chosen address's budget and, separately, never actually prevents the row writes. Expected behaviour is that the limit is keyed on the authenticated caller (or the (caller, target) pair) and is checked before the rows are committed and the send is attempted.

Note on method: this is a worker logic/ordering defect established by reading the shipped source at the frozen commit; no live system was touched. The ordering (`:76`/`:105` before `:126`), the victim-keying of the limiter, and the absence of any index/unique on `channel_verifications.token` are all verifiable directly in the three files cited.

## Recommended fix

Key the limiter on the authenticated user (or the (user, target) pair), and call it — and the collision checks — before inserting `user_channels`/`channel_verifications` and before attempting the send, returning 429 without side effects. Add a unique or indexed constraint on `channel_verifications.token` (and bound rows per user) so the verification-click lookup is not a sequential scan.
