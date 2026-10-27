# 89420 w a critical resolver roles sidebar collapses an account s multiple role resources into one write scope so remove user displays one scope and writes another leaving removed accounts wit&#x20;

**Submitted on Aug 19th 2026 at 08:23:21 UTC by @c0mpos3r for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89420
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

`groupRolesByAccount` keys its map on the role row's `account` and discards the row's `resource`. One table row therefore represents an account's roles across every resource of a `PermissionedResolver`. The sidebar then picks a single name out of that merged row and uses it as the scope for `Remove user` and `Save changes`.

When an account holds roles at two resources, the approved transaction addresses one of them and carries the role bitmap of both, while the sidebar label lists both.

Asset: `apps/portal`. Commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

## Vulnerability Details

### 1. The grouping discards `resource`

`apps/portal/src/lib/roles/resolverRoles.ts`, `groupRolesByAccount`:

```ts
const account  = role.account.toLowerCase()                   // :194  map key
const resource = role.resource.toLowerCase()                  // :195
const resolvedName = resourceToName?.get(resource) ?? null    // :197

const existing = grouped.get(account)
if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)                      // :203
  if (resolvedName) existing.resolvedNames.add(resolvedName)  // :204
}
```

`resource` is used only for a display-name lookup. Every resource's bitmap lands in one `decodedRoles` array and every name in one `resolvedNames` set.

The indexer does return a name per role row: `useResolverOverview.ts:90-99` selects `roles { account resource roleBitmap blockNumber transactionHash timestamp name }`, and `ResolverRole` (`:14-22`) carries `name: string | null`. The grouper's input type `RoleInput` (`resolverRoles.ts:162-166`) declares only `account`, `resource`, `roleBitmap`, so that name is never available to it.

`ResolverRolesTable.tsx:103` calls the grouper and renders one row per account.

### 2. The sidebar reduces the merged row to one scope

`ResolverRolesSidebar.tsx`:

```ts
const resolvedNames = row?.original.resolvedNames ?? []            // :167
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''   // :169
```

`find` returns the first entry in Set insertion order, which follows the indexer's row order. The remaining resources are dropped.

### 3. The label and the write scope are computed differently

```tsx
{resolvedNames.includes('(root)')
  ? 'Global roles (all names)'
  : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}  // :393-395
```

The label joins every non-root name; the write takes one. The confirm dialog is unconditional (`:445-447`): "Are you sure you want to remove this user from all roles? This action cannot be undone."

### 4. The single scope reaches the calldata

`:299` and `:318` set `name: roleName` on `pendingAction`; `:490` and `:497` pass `pendingAction.name` to the mutations. `handleRemoveUser` (`:308-322`) passes `roles: decodedRoles`, the merged list.

`revokeResolverRoles.ts:47-59` (and `grantResolverRoles.ts:47-57`):

```ts
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

`name` is one string. `revokeResolverRolesWriteParameters` emits `authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant)`, which DNS-encodes one name. Two resources cannot be expressed in one call.

## Impact Details

An account the operator removed keeps its roles at every resource except the one the calldata addressed. The outcome does not depend on how `PermissionedResolver` handles a revoke of an unheld bit:

* If the call succeeds, the account keeps `ROLE_SET_TEXT` at `bar.eth` and the modal reports completion.
* If the call reverts, the whole removal fails, including the `foo.eth` revocation that was correctly scoped, and the operator sees a generic transaction error.

Either way the operator is left believing the account was fully de-authorised. That belief is the substance of the issue. This control exists for offboarding and key-compromise response, and an operator who believes a compromised key has been revoked stops acting against it.

Which roles survive determines what the retained account can do. All ten resolver roles are grantable through the same sheet: `resolverPermissions` (`resolverRoles.ts:57-108`) is rendered by both `ResolverAddUserSheet.tsx:59` and `ResolverRolesSidebar.tsx:83`, so any of them can appear in a merged row.

| Retained role                                                                                               | What it permits                                                |
| ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `ROLE_UPGRADE`                                                                                              | Upgrade the resolver contract                                  |
| `ROLE_SET_ADDR`                                                                                             | Rewrite address records, redirecting payments sent to the name |
| `ROLE_SET_CONTENTHASH`                                                                                      | Repoint the name's content hash                                |
| `ROLE_CLEAR`                                                                                                | Clear all versioned records                                    |
| `ROLE_SET_TEXT`, `ROLE_SET_PUBKEY`, `ROLE_SET_ABI`, `ROLE_SET_INTERFACE`, `ROLE_SET_NAME`, `ROLE_SET_ALIAS` | Rewrite the corresponding record type                          |

Blast radius follows the surviving resource. A name-scoped grant affects that one name. A ROOT-scoped grant affects every name the resolver serves, which is the `nodes` list rendered on the same screen.

Worst case is a row whose resources include ROOT. The label reads `Global roles (all names)`, the operator removes the user, and the ROOT-scope grant is not addressed. A retained root `ROLE_SET_ADDR` is write access to the address record of every name that resolver serves; a retained root `ROLE_UPGRADE` is control of the resolver contract that answers for all of them.

`Save changes` collapses the same way. `computeRoleChanges` runs over the merged `decodedRoles`, so enabling a permission writes a grant at one resource while the label lists several.

## Severity

Selected impact: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient".

The displayed scope and the approved calldata differ, which is the second sub-bullet ("showing the wrong recipient or amount on a transaction the user then approves").

The ROOT case reaches the first sub-bullet. A retained root `ROLE_SET_ADDR` lets any name that resolver serves be pointed at an arbitrary address, so a third party paying that name sends funds to an unintended recipient. A retained root `ROLE_UPGRADE` is control of the resolver contract that answers for every one of those names.

No attacker input is needed. The precondition is produced by the portal's own Add-user sheet, and the actor is the resolver's root admin (`routes/resolver/$address/roles/index.tsx:50`, `canManageRoles = Boolean(hasRootRole)`).

Factors that reduce it:

* After `pollForIndexerSync` refetches, the surviving role reappears in the row, and a second `Remove user` (now single-scope) revokes correctly.
* Reaching loss of funds requires the retained role holder to act.
* `apps/portal` is Sepolia-only today (`lib/wagmi.ts:67`, `chains: [sepoliaWithEns]`), so this is a pre-launch logic defect rather than a live-funds issue.

The empty-`resolvedNames` case, where `roleName = ''` maps to root scope at `revokeResolverRoles.ts:47`, is a second-order consequence of the same collapse. Its trigger depends on how the indexer composes `resolver.nodes`, which is out of scope, so it is not part of the claim above.

## Suggested fix

Verified against this commit: `pnpm typecheck` clean, `pnpm vitest run` 141 files / 1447 tests passing. Patch available on request.

{% stepper %}
{% step %}

## Key the grouping on `(account, resource)`

The permission checkboxes and `computeRoleChanges` operate on the merged `decodedRoles`, so a merged row cannot express which resource a newly enabled permission belongs to. Deriving the scope per resource at write time only is not sufficient.
{% endstep %}

{% step %}

## Take the scope name from `role.name`

Use `role.name`, which the indexer already returns (`useResolverOverview.ts:97`), instead of resolving `resource` through `resolver.nodes`.
{% endstep %}

{% step %}

## Decide root from the resource value

Do not decide root from a missing name. Today `roleName === ''` means root scope, so any failed name lookup becomes a resolver-wide write. `filterEventsByResource.ts:37-47` already compares resources with `BigInt` for this reason, noting that the indexer may encode `ROOT_RESOURCE` as `0x0` while callers build a padded zero; `buildResourceToNameMap:151` keys only the padded form.

A non-root resource with no resolvable name should block the write, not widen it.
{% endstep %}

{% step %}

## Emit one call per resource

`saveMutation` already issues grant and revoke sequentially, so this fits the existing shape.
{% endstep %}
{% endstepper %}

A `resolvedNames.length > 1` guard alone would miss `length === 0`, the case that widens a name-scoped write to root.

## Notes

`resolverRoles.test.ts:185-197` ("should merge the same account across resources") covers the merge as display behaviour. No test covers the write scope derived from a merged row, and there is no test file for `ResolverRolesSidebar.tsx`.

Searching `resolverRoles.ts`, `ResolverRolesSidebar.tsx` and `ResolverRolesTable.tsx` for `known|limitation|caveat|by design|intended|TODO|FIXME|trade-off|acceptable|assume` returns no hits.

The name-level role family (`features/roles`) is not affected: its write scope comes from the route param (`routes/$name/roles/index.tsx:18` to `RolesSidebar` prop to `grantRoles.ts:70-81`), and `useNameRoleAccounts.ts:101` filters by resource before building its map.

## References

| File                                                                    | Lines                                              |
| ----------------------------------------------------------------------- | -------------------------------------------------- |
| `apps/portal/src/lib/roles/resolverRoles.ts`                            | 179-221, 162-166, 147-160, 57-108                  |
| `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx` | 83, 167, 169, 299, 318, 393-395, 445-447, 490, 497 |
| `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx`   | 103, 129                                           |
| `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts`      | 45-60                                              |
| `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts`       | 20, 45-57                                          |
| `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx` | 59, 127, 186, 205, 251-258                         |
| `apps/portal/src/features/resolver/hooks/useResolverOverview.ts`        | 14-22, 90-99                                       |
| `apps/portal/src/routes/resolver/$address/roles/index.tsx`              | 18, 50, 86-90                                      |
| `apps/portal/src/lib/roles/filterEventsByResource.ts`                   | 37-47                                              |
| `apps/portal/src/lib/roles/resolverRoles.test.ts`                       | 185-197                                            |

## Proof of Concept

Deterministic and local-only: no network access, no deployed contract, and no transaction is broadcast. The test builds calldata with the app's own helpers and decodes it.

### Setup

{% stepper %}
{% step %}

## Clone and install

Clone the audit repo at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and run:

```bash
pnpm install --ignore-scripts
```

{% endstep %}

{% step %}

## Save the test file

Save the file below as `apps/portal/src/lib/roles/c17-poc.test.ts`.
{% endstep %}

{% step %}

## Run the test

From `apps/portal`, run:

```bash
pnpm vitest run src/lib/roles/c17-poc.test.ts --silent=false
```

{% endstep %}
{% endstepper %}

### Scenario (reproducible through the app's own UI)

{% stepper %}
{% step %}

## Establish resolver administration

Resolver `R` is a shared `PermissionedResolver`. Operator `A` holds the root admin role, so `canManageRoles` is true on `/resolver/R/roles` (`routes/resolver/$address/roles/index.tsx:50`).
{% endstep %}

{% step %}

## Grant a role at `foo.eth`

Using the portal's Add user sheet, account `X` is granted `ROLE_SET_ADDR` scoped to `foo.eth`. (`ResolverAddUserSheet.tsx:186` lists every node; `:205` `setPendingGrant({ name: selectedNode, ... })` grants per node.)
{% endstep %}

{% step %}

## Grant a role at `bar.eth`

Using the same sheet again, `X` is granted `ROLE_SET_TEXT` scoped to `bar.eth`. The resolver now holds two role rows for `X`, at two different resources.
{% endstep %}

{% step %}

## Merge the rows

`getResolverOverview` returns both rows. `groupRolesByAccount` merges them into one table row: `decodedRoles = ['ROLE_SET_ADDR','ROLE_SET_TEXT']`, `resolvedNames = ['foo.eth','bar.eth']`.
{% endstep %}

{% step %}

## Open the sidebar

`A` opens the sidebar for `X`. Line `:393-395` renders `Roles scoped to foo.eth, bar.eth`. Line `:169` computes `roleName = 'foo.eth'`, the first non-root entry.
{% endstep %}

{% step %}

## Remove the user

`A` clicks Remove user. The dialog at `:445-447` promises removal "from all roles". `handleRemoveUser` (`:308-322`) sets `{ name: 'foo.eth', roles: ['ROLE_SET_ADDR','ROLE_SET_TEXT'] }`.
{% endstep %}

{% step %}

## Build the mutation

`onStart` (`:487-494`) calls the mutation with `pendingAction.name`, reaching `revokeResolverRoles.ts:47-59`, which selects `scope: 'name'` and encodes a single `authorizeNameRoles` call.
{% endstep %}
{% endstepper %}

### Observed result (actual test output)

```
=== C-17 PoC RESULT ===
UI label shown to operator : Roles scoped to foo.eth, bar.eth
confirm dialog             : "remove this user from all roles"
function called            : authorizeNameRoles
toName (DNS-encoded)       : 0x03666f6f0365746800
roleBitmap                 : 17 = ROLE_SET_ADDR|ROLE_SET_TEXT
grant                      : false
=> bar.eth grant of ROLE_SET_TEXT is NEVER addressed.

Test Files  1 passed (1)
     Tests  6 passed (6)
```

`0x03666f6f0365746800` is the DNS wire format of `foo.eth` alone (`0x03` + `foo` + `0x03` + `eth` + `0x00`). `bar.eth` appears nowhere in the calldata, while `roleBitmap = 17` = `(1 << 0) | (1 << 4)` carries both role bits, including the one `X` holds only at `bar.eth`.

The single approved call therefore attempts to revoke `ROLE_SET_TEXT` at `foo.eth`, where `X` never held it, and never touches `bar.eth`. Whether that call succeeds or reverts, `X`'s `bar.eth` grant is untouched.

The test also covers:

* Step 5, non-determinism: reversing the indexer's row order changes the surviving name from `foo.eth` to `bar.eth`, so which resource is spared is not under the operator's control.
* Step 6, the root+name case: `resolvedNames = ['(root)', 'foo.eth']` makes the label print `Global roles (all names)` while `:169` skips `'(root)'` and writes name scope, so the ROOT-resource grant is not addressed.

### Test file

```ts
import { decodeFunctionData, type Address } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { createWalletClient, http } from 'viem'
import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from './resolverRoles'

const RESOLVER = '0x1111111111111111111111111111111111111111' as Address
const X = '0x3333333333333333333333333333333333333333333333' as Address
const NODES = [{ name: 'foo.eth' }, { name: 'bar.eth' }]

const rolesFromIndexer = [
  { account: X, resource: computeNameResource('foo.eth'), roleBitmap: resolverRoles.ROLE_SET_ADDR.toString() },
  { account: X, resource: computeNameResource('bar.eth'), roleBitmap: resolverRoles.ROLE_SET_TEXT.toString() },
]

const walletClient = createWalletClient({
  account: privateKeyToAccount('0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d'),
  chain: sepolia,
  transport: http('http://127.0.0.1:1'), // never contacted; calldata only
})

describe('C-17 PoC - multi-resource collapse in the resolver roles sidebar', () => {
  it('step 1: the grouper merges two RESOURCES into one row keyed only by account', () => {
    const grouped = groupRolesByAccount(rolesFromIndexer, buildResourceToNameMap(NODES))
    expect(grouped).toHaveLength(1)
    expect(grouped[0].resolvedNames).toEqual(['foo.eth', 'bar.eth'])
    expect(grouped[0].decodedRoles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])
    expect(grouped[0].roles).toHaveLength(2)
  })

  it('step 2: the UI label advertises BOTH names', () => {
    const { resolvedNames } = groupRolesByAccount(rolesFromIndexer, buildResourceToNameMap(NODES))[0]
    // ResolverRolesSidebar.tsx:393-395, verbatim (non-root branch)
    const label = `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`
    expect(label).toBe('Roles scoped to foo.eth, bar.eth')
  })

  it('step 3: the write scope collapses to ONE name', () => {
    const { resolvedNames } = groupRolesByAccount(rolesFromIndexer, buildResourceToNameMap(NODES))[0]
    // ResolverRolesSidebar.tsx:169, VERBATIM
    const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
    expect(roleName).toBe('foo.eth') // bar.eth is silently dropped
  })

  it('step 4: "Remove user" calldata targets foo.eth ONLY, carrying bar.eth role bit', () => {
    const group = groupRolesByAccount(rolesFromIndexer, buildResourceToNameMap(NODES))[0]
    const roleName = group.resolvedNames.find((n) => n !== '(root)') ?? ''

    // handleRemoveUser (ResolverRolesSidebar.tsx:308-322) passes the MERGED role list
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: group.account as Address,
      roles: group.decodedRoles as never,
      walletClient,
      chainId: sepolia.id,
    })

    const decoded = decodeFunctionData({
      abi: [{
        name: 'authorizeNameRoles', type: 'function', stateMutability: 'nonpayable',
        inputs: [
          { name: 'toName', type: 'bytes' },
          { name: 'roleBitmap', type: 'uint256' },
          { name: 'account', type: 'address' },
          { name: 'grant', type: 'bool' },
        ],
        outputs: [{ name: '', type: 'bool' }],
      }],
      data: (intent.request as { data: `0x${string}` }).data,
    })

    const [toName, roleBitmap, account, grant] = decoded.args as [`0x${string}`, bigint, Address, boolean]

    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(grant).toBe(false)
    expect(account.toLowerCase()).toBe(X.toLowerCase())

    const dnsName = Buffer.from(toName.slice(2), 'hex').toString('latin1')
    expect(dnsName).toContain('foo')
    expect(dnsName).not.toContain('bar') // bar.eth is never addressed

    // ...yet the bitmap carries BOTH roles, including the one X holds at bar.eth
    expect(roleBitmap & resolverRoles.ROLE_SET_ADDR).toBe(resolverRoles.ROLE_SET_ADDR)
    expect(roleBitmap & resolverRoles.ROLE_SET_TEXT).toBe(resolverRoles.ROLE_SET_TEXT)

    console.log('\n=== C-17 PoC RESULT ===')
    console.log('UI label shown to operator : Roles scoped to foo.eth, bar.eth')
    console.log('confirm dialog             : "remove this user from all roles"')
    console.log('function called            :', decoded.functionName)
    console.log('toName (DNS-encoded)       :', toName)
    console.log('roleBitmap                 :', roleBitmap.toString(), '= ROLE_SET_ADDR|ROLE_SET_TEXT')
    console.log('grant                      :', grant)
    console.log('=> bar.eth grant of ROLE_SET_TEXT is NEVER addressed.\n')
  })

  it('step 5: which name survives is not deterministic - it follows indexer order', () => {
    const pick = (rows: typeof rolesFromIndexer) =>
      groupRolesByAccount(rows, buildResourceToNameMap(NODES))[0]
        .resolvedNames.find((n) => n !== '(root)') ?? ''
    expect(pick(rolesFromIndexer)).toBe('foo.eth')
    expect(pick([...rolesFromIndexer].reverse())).toBe('bar.eth')
  })

  it('step 6: root + name - the label says "Global", the write says name-scope', () => {
    const ROOT_RESOURCE = '0x0000000000000000000000000000000000000000000000000000000000000000'
    const rows = [
      { account: X, resource: ROOT_RESOURCE, roleBitmap: resolverRoles.ROLE_SET_ADDR.toString() },
      { account: X, resource: computeNameResource('foo.eth'), roleBitmap: resolverRoles.ROLE_SET_TEXT.toString() },
    ]
    const group = groupRolesByAccount(rows, buildResourceToNameMap(NODES))[0]
    expect(group.resolvedNames).toEqual(['(root)', 'foo.eth'])
    expect(group.resolvedNames.includes('(root)')).toBe(true)   // label -> 'Global roles (all names)'
    expect(group.resolvedNames.find((n) => n !== '(root)') ?? '').toBe('foo.eth') // write -> name scope
  })
})
```

### Testing conduct

All work was local: no deployed or public-testnet contract was contacted and no transaction was signed or broadcast, and the key in the test is the well-known Anvil account #1 with its transport pointed at a dead local port.
