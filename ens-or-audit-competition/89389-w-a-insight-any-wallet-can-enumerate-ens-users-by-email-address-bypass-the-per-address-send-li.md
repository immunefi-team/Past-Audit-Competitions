# 89389 w a insight any wallet can enumerate ens users by email address bypass the per address send limit and get a victim s own expiry alerts switched off

**Submitted on Aug 19th 2026 at 00:43:20 UTC by @sikoh for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89389
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Improperly disclosing confidential user information, such as: Email address
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

The email notification channel treats an address as owned by whoever types it first, and nothing in the worker rate-limits anything except a single counter keyed on the address string itself. Three consequences follow, none of which needs the victim to do anything. Any wallet holder can ask the server, for free and without leaving a trace, whether a given email address belongs to an ENS user. Any wallet holder can push an unbounded number of ENS-sent emails into one inbox by varying the address in a way the provider ignores but the server does not. And a victim who reacts to that mail the way anyone would, by reporting it as spam, can have their own verified channel switched off, which silently stops the expiry alerts that are the whole point of the feature. On top of that, the verification endpoint takes no authentication at all, so a single click on a genuine ENS email hands the attacker a permanent claim on that address.

## Vulnerability Details

Four defects, in the order they matter.

{% stepper %}
{% step %}

## The rejection path is a free oracle

`POST /notifications/channels/email` looks for a channel held by *another* user with `status = 'verified'` and returns 400 if it finds one:

```typescript
// channels/email.ts:59-70
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

That return is at line 68. The first write is at line 76, the rate-limit consumption at line 125, the send at line 142 — all of them after it. So a probe writes no row, sends no mail and consumes no quota. There is no rate limiting to fall back on either: `app/middleware/` contains only `auth`, `database`, `eth` and `hono`.
{% endstep %}

{% step %}

## The one rate limit is keyed on a string the mail provider ignores

```typescript
// services/notifications/email-verification-rate-limit.ts:3-4,15-16
export const EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS = 3
export const EMAIL_VERIFICATION_RATE_LIMIT_WINDOW_SECONDS = 60 * 60
export const normalizeEmailForRateLimit = (email: string) => email.trim().toLowerCase()
```

Trim and lowercase, nothing else. `victim+1@gmail.com` and `victim@gmail.com` are one inbox to the provider and two separate quotas here — and two separate rows, since the uniqueness constraint is `(user_id, channel, target)` on the literal target (`schema/notifications.ts:74-78`). The same mismatch walks past the guard in defect 1: a victim holding a verified `victim@gmail.com` does not block `victim+1@gmail.com`.
{% endstep %}

{% step %}

## The SendGrid webhook changes another user's row

```typescript
// webhook/sendgrid.ts:172-174
const channel = await db.query.userChannels.findFirst({
  where: eq(TABLE.userChannels.target, event.email),
})
```

No `user_id`, no ordering. Whatever row that returns is then written: `spamreport` sets `status: 'bounced'` (`:205-217`), `unsubscribe` sets `'unsubscribed'` (`:222-232`). When an attacker's pending row and the victim's verified row share the address, which one is hit is not decided by this code. And delivery only ever goes to verified channels (`services/notifications/create.ts:117-122` and `:309-314`), so flipping that status is what actually stops the alerts.
{% endstep %}

{% step %}

## Verification takes no authentication, and removal is owner-scoped

`POST /notifications/channels/email/verify` (`email.ts:168`) is registered with only `vValidator` and `injectDb` — no `requireAuth`, unlike every sibling route — and sets `status: 'verified'` on whatever channel the token points at. `DELETE /notifications/channels/:id` filters on `eq(user_id, userId)` (`$id.ts:62-78`), so the person whose address it is cannot remove it. The token itself is fine: 16 bytes of `crypto.getRandomValues` (`helpers.ts:173-176`). Nothing here depends on guessing it.

That asymmetry is visible on the live API without any credentials:

```shell
$ curl -s -o /dev/null -w '%{http_code}\n' -X POST \
    https://app-api.ens.dev/notifications/channels/email \
    -H 'content-type: application/json' -d '{"email":"x@example.com"}'
401
```

```shell
$ curl -s -X POST https://app-api.ens.dev/notifications/channels/email/verify \
    -H 'content-type: application/json' \
    -d '{"token":"0000000000000000000000000000000000000000"}'
{"error":"Invalid or expired verification token"}
```

Adjacent routes, same host, no credentials on either. The first is stopped by authentication. The second reaches the token lookup and answers about it.
{% endstep %}
{% endstepper %}

## Impact Details

### Enumeration of your users by email address

One SIWE session is enough to test any address and learn whether an ENS user has verified it, at no cost and with nothing recorded against the address probed. Point it at a breach dump, or at a list of named individuals, and it returns the subset who use ENS. Three responses are distinguishable:

* *Email already verified for this account* means the address is the caller's own.
* *This email address is already linked to another account* means it belongs to some other user.
* Success means nobody holds it.

### Unbounded mail into a single inbox, sent by ENS

The advertised ceiling of three per hour is three per hour per string. Because the provider collapses `+tag` variants and the server does not, the number of strings resolving to one inbox is unbounded, and every one of them carries the real ENS sender address (`noreply@ens.dev`, `wrangler.jsonc:98`) and a link into `https://app.ens.dev` (`:100`). Measured below: an address already blocked for the hour absorbed 30 further sends through ten variants.

### A victim's own expiry alerts silenced, by the victim

They receive mail they never asked for, report it as spam as anyone would, and the webhook flips a row for that address to `bounced`. If it is theirs, delivery stops. This is recoverable — re-registering upserts the row back to `pending` — but nothing tells them it happened, and the attacker can trigger it again. For a feature whose only job is warning you before your name expires, silent failure is the damage.

### A permanent claim on the address, after one ordinary click

The verification mail is genuinely from ENS and the button is the real one. Clicking it verifies whichever channel the token belongs to, which may be the attacker's. From then on the victim gets *This email address is already linked to another account* on their own address, cannot delete the channel, and the attacker's notifications are delivered to their inbox.

Two things worth having rather than not. There is no cap anywhere on how many channels one account may hold. And the SendGrid contact paths — adding the address to the broadcast list on verification (`email.ts:232`), and deleting the contact when the channel is removed (`$id.ts:82-95`, which searches by `channel.target`, the victim's address) — are both gated on `SENDGRID_BROADCAST_LIST_ID`, which is `""` today (`wrangler.jsonc:107`). They are dormant, not absent: the day that variable is populated, the same flow that subscribes a stranger also lets one delete their contact record.

{% hint style="info" %}
One question rather than a claim: expiry mail is rendered by a SendGrid dynamic template referenced only by id (`delivery/templates/email.ts:18-27`, `wrangler.jsonc:108`), and no template source is versioned in this repository. If those templates interpolate `name` with triple braces rather than double, the name — which reaches them unvalidated, since there is no ENSIP-15 check anywhere in the worker — would render as HTML inside an ENS-branded email delivered to an address of the attacker's choosing. I cannot verify that from the repository and am not asserting it; it is worth five minutes of your time to check.
{% endhint %}

## References

* `workers/api-worker/src/app/routes/notifications/channels/email.ts` — guard and 400 at 59-70, insert 76, rate check 125, send 142, unauthenticated verify 168
* `workers/api-worker/src/services/notifications/email-verification-rate-limit.ts` — 3-4, 15-16
* `workers/api-worker/src/core/database/schema/notifications.ts:74-78` — `unique(user_id, channel, target)`
* `workers/api-worker/src/app/routes/webhook/sendgrid.ts:172-174, 205-217, 222-232`
* `workers/api-worker/src/services/notifications/create.ts:117-122, 309-314` — delivery is verified-only
* `workers/api-worker/src/app/routes/notifications/channels/$id.ts:62-78, 82-95`
* `workers/api-worker/src/services/notifications/helpers.ts:173-176`
* `workers/api-worker/wrangler.jsonc:98, 100, 107, 108`

Bug found and verified by me. — Sikoh

## Proof of Concept

Two parts: three unauthenticated requests against the live API, and one test file against the real app.

### On production, no credentials, nothing written

The base URL is `https://app-api.ens.dev` (`VITE_API_URL` in `apps/manager/.env.example`; the worker is `app-api-worker`). There is no `/api` and no `/v1` segment — those return 404.

```shell
$ curl -s -o /dev/null -w '%{http_code}\n' -X POST \
    https://app-api.ens.dev/notifications/channels/email \
    -H 'content-type: application/json' -d '{"email":"x@example.com"}'
401
```

```shell
$ curl -s -X POST https://app-api.ens.dev/notifications/channels/email/verify \
    -H 'content-type: application/json' -d '{}'
{"typed":false,"success":false,"issues":[{"kind":"schema","expected":"\"token\"","received":"undefined", ...}]}
```

```shell
$ curl -s -X POST https://app-api.ens.dev/notifications/channels/email/verify \
    -H 'content-type: application/json' \
    -d '{"token":"0000000000000000000000000000000000000000"}'
{"error":"Invalid or expired verification token"}
```

The second and third requests carry no authentication, pass schema validation and reach the database lookup. A token of all zeroes simply misses, so no row is touched and no mail is sent.

### Against the real app

Attached `poc-F001.test.ts`. It mounts the actual Hono app from `src/app/index.ts`, goes through the real JWT middleware with a token signed by `hono/jwt`, and uses the real KV binding from `cloudflare:test`. Only the database and the two SendGrid send helpers are stubbed. Drop it in `workers/api-worker/src/` and run `pnpm vitest run src/poc-F001.test.ts`:

```
✓ CONTROL: the endpoint distinguishes states, so success is not unconditional
✓ an attacker binds a THIRD-PARTY address to their own account and ENS mails the victim
✓ the "already linked" 400 is a free, silent oracle: no write, no mail, no quota
✓ CONTROL: an unused address returns success, so the two states are distinguishable
✓ /verify is unauthenticated: a bare token flips the attacker channel to verified
✓ CONTROL: the limit really does bite on a single address
✓ BYPASS: +tag variants of the SAME inbox each get a fresh quota

Test Files  1 passed (1)
     Tests  7 passed (7)
```

What each one pins down:

The oracle test asserts the three things that would have made a probe costly or noticeable, and asserts them positively: `insert` is never called, no send is recorded, and the KV quota key for that address reads byte-identical before and after the request. Its control shows an unused address returns 200 through the same code path, so the two states really are distinguishable by an outsider.

The rate-limit test runs against the real KV. Its control exhausts one address and confirms the fourth send is refused. The bypass then takes ten `+tag` variants of that same inbox and lands 30 further sends, each variant getting its own untouched quota.

The verify test posts a bare token with no authorization header and gets 200 with the channel update performed.

The suite is 134/134 before and after; the file is standalone and leaves no trace when removed.

Verified at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, the head of `audit-comp-ready`.
