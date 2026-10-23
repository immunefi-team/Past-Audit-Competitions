# 90896 w a critical apps portal remove user confirms removal from all roles the approved transaction revokes one name scope

**Submitted on Aug 27th 2026 at 22:49:19 UTC by @aurorazz for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90896
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

### Intro

The Explorer's resolver Roles page merges an account's global and name-scoped roles into a single table row, and its Remove user control revokes only the name-scoped part of that row while the confirmation dialog tells the operator they are removing the user from all roles. The operator approves the transaction on the strength of that description, the transaction succeeds, the application reports success, and the account's global grant is untouched. The operator has approved and submitted a transaction that does not do what the application told them it does.

### Vulnerability Details

**Scope**

apps/portal — the Explorer app (Files). Exactly three files:

* `src/lib/roles/resolverRoles.ts` — groups every scope an account holds into one table row
* `src/features/resolver/components/ResolverRolesSidebar.tsx` — picks the scope for every action taken on that row, and renders the confirmation the operator approves against
* `src/features/resolver/helpers/revokeResolverRoles.ts` — maps that pick to the on-chain call the operator signs

The fix is a one-line change in `ResolverRolesSidebar.tsx`; nothing outside apps/portal changes.

### The defect

One row per account, built by `groupRolesByAccount`. Every scope's decoded roles accumulate into `decodedRoles`; every scope's name accumulates into `resolvedNames`:

```ts
  for (const role of roles) {
    const account = role.account.toLowerCase()
    const resource = role.resource.toLowerCase()
    const decoded = decodeResolverRoleBitmap(role.roleBitmap)
    const resolvedName = resourceToName?.get(resource) ?? null

    const existing = grouped.get(account)

    if (existing) {
      existing.roles.push(role)
      existing.decodedRoles.push(...decoded)
      if (resolvedName) existing.resolvedNames.add(resolvedName)
    } else {
      const names = new Set<string>()
      if (resolvedName) names.add(resolvedName)
      grouped.set(account, {
        account,
        resolvedNames: names,
        roles: [role],
        decodedRoles: [...decoded],
      })
    }
  }
```

(`resolverRoles.ts:193-215`, inside [`groupRolesByAccount`, `resolverRoles.ts#L179-L221`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L221).)

The sidebar then treats the first non-root name in that merged list as the scope for every action on the row:

```ts
  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

([`ResolverRolesSidebar.tsx#L169`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169).)

And on Remove user it sends the union of `decodedRoles` with that single name:

```ts
  const handleRemoveUser = () => {
    if (
      !selectedAccount ||
      decodedRoles.length === 0 ||
      removeUserMutation.isPending
    )
      return

    removeUserMutation.reset()
    setPendingAction({
      type: 'remove',
      name: roleName,
      account: selectedAccount,
      roles: decodedRoles,
    })
    openModal()
  }
```

([`ResolverRolesSidebar.tsx#L307-L323`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323).)

Finally, the revoke helper picks the root path only when the picked name is the empty string:

```ts
  const writeParams = revokeResolverRolesWriteParameters(
    walletClient as Parameters<typeof revokeResolverRolesWriteParameters>[0],
    name === ''
      ? {
          resolverAddress,
          targetAccount: account,
          scope: 'root',
          roles: roles as ResolverRole[],
        }
      : {
          resolverAddress,
          targetAccount: account,
          scope: 'name',
          name,
          roles: roles as ResolverRole[],
        },
  )
```

([`revokeResolverRoles.ts#L45-L61`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L61).)

Union row, first non-root name picked as the scope for every action, and a mixed row can never take the root path: on an account holding both root and any name-scoped grant, `roleName` is that name, never the empty string, so the root removal branch is structurally unreachable from this flow.

### What the operator sees and approves

The portal was run live against an anvil fork, its indexer route serving the fork's readbacks per request, with the roles data mirroring those readbacks; the captures referenced here are unmodified screenshots from that run.

Clicking Remove user on the merged row opens a confirmation dialog whose text, in full:

```tsx
              <DialogTitle>Remove user</DialogTitle>
              <DialogDescription>
                Are you sure you want to remove this user from all roles? This
                action cannot be undone.
              </DialogDescription>
```

([`ResolverRolesSidebar.tsx#L444-L448`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L444-L448), copy at [`:446-L447`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L446-L447).)

Confirming opens the transaction modal titled `Remove resolver user`, transaction name `Remove user ${account}`, rendered with the removed account's address ([`:339-L344`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L339-L344)); in the run the modal sat at its waiting state while the wallet held the unsigned transaction the application built for that click.

Captures:

* `captures/0-sidebar-mixed-row.png` — the row before the removal, both scopes merged into one row
* `captures/1-confirm-dialog.png` — the dialog over that row
* `captures/1b-tx-modal-pending.png`
* `captures/1c-awaiting-wallet.png` — the modal pending on the wallet, the sidebar copy behind it

Everything quoted above is the application's own display at the moment of approval: this is the transaction detail the operator was shown.

## Impact Details

Critical. The executed state change was not the state change described. The application displayed an incorrect transaction detail, what the approved transaction would do, and the operator approved and submitted the transaction on that display (the displayed confirmation under Vulnerability Details and the built transaction under Proof of Concept). This is the selected impact's second listed example instantiated at the moment of approval: wrong detail shown on a transaction the user then approves, so that the transaction submitted is not the transaction described. The wrong detail here is the transaction's effect, which grants it revokes,and the difference is not cosmetic: the grant the operator believed they had revoked is exactly the one that survives, while the application confirms success.

The precondition is routine, and a delegate can create it deliberately. The Add-user sheet defaults its scope to the root resource [ResolverAddUserSheet.tsx#L127](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L127), so merged rows are the natural state of this page, not a corner case. A delegate holding a global grant requests one innocuous name-scoped grant on a name they already work on; from that point the Remove user control is structurally incapable of stripping their global grant, and every removal attempt confirms success while never touching it. What the delegate gains is not a permission but immunity from revocation. The authority that changes hands is the operator's.

Nothing after the approval exposes the false description. The flow ends in a success state and the sidebar closes on it [ResolverRolesSidebar.tsx#L503-L508](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L503-L508), the application waits for the indexer to sync and refetches [ResolverRolesSidebar.tsx#L278-L287](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L278-L287), and the account reappears in the table with its remaining global roles [ResolverRolesSidebar.tsx#L391-L396](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L396). In an application whose tables trail chain state, a row still present after a confirmed removal reads as indexer lag; the one signal that would expose the partial revoke is the one the interface trains operators to discount. The retained grant has no expiry.

## References

Repo pinned at [1c9b47f18fcddd2e864dfe385c4171061c9811ae](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae). All portal line ranges below were verified against the pinned clone.

* [resolverRoles.ts#L179-L221](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L221) — `groupRolesByAccount`, the union/merge (merge loop at L193-215)
* [ResolverRolesSidebar.tsx#L169](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169) — the scope pick
* [ResolverRolesSidebar.tsx#L307-L323](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323) — `handleRemoveUser`
* [ResolverRolesSidebar.tsx#L391-L396](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L396) — the `Global roles (all names)` subtitle
* [ResolverRolesSidebar.tsx#L446-L447](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L446-L447) — the remove-dialog copy
* [ResolverRolesSidebar.tsx#L339-L344](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L339-L344) — the removal transaction modal title and transaction name
* [ResolverRolesSidebar.tsx#L503-L508](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L503-L508) — `onDone` closes the sidebar on success
* [ResolverRolesSidebar.tsx#L278-L287](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L278-L287) — refetch after indexer sync
* [revokeResolverRoles.ts#L45-L61](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L61) — the scope ternary (grant side: `grantResolverRoles.ts`)
* [ResolverAddUserSheet.tsx#L127](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L127) — Add-user sheet defaults to the root scope (`ROOT_NODE_VALUE`, defined at L38-39)
* [roles/index.tsx#L41-L50](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/resolver/%24address/roles/index.tsx#L41-L50) — the roles route and its table query (L35-L39)

## Proof of Concept

### What the app builds for that approval

The vitest file below was written for this report. It imports the portal's source files unmodified and drives the application's own grouping, the sidebar's verbatim scope pick, and the application's own transaction builder with the two indexer records of a merged row; the wallet client is a structural stub carrying only the operator address and chain id. The last assertion of the first test byte-compares the built transaction against the removal encoding executed in the on-chain verification below, reproduced there independently of the application.

Run:

```sh
pnpm install && pnpm -C apps/portal exec vitest run scripts/remove-user-approval.test.ts
```

```ts
import type { WalletClient } from 'viem'
import { toFunctionSelector } from 'viem'
import { describe, expect, it } from 'vitest'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  type ResolverRoleKey,
} from '../src/lib/roles/resolverRoles'
import { prepareRevokeResolverRolesTransaction } from '../src/features/resolver/helpers/revokeResolverRoles'

const RESOLVER_ADDRESS =
  '0xa1c8256043f3bdd5b683f570db40bc936b7dd294' as const
const OPERATOR = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266' as const
const ACCOUNT = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as const

const ROOT_RESOURCE =
  '0x0000000000000000000000000000000000000000000000000000000000000000'
const HARMONIA_RESOURCE = computeNameResource('harmonia.eth')

const walletClient = {
  account: { address: OPERATOR },
  chain: { id: 11155111 },
} as WalletClient

const requestOf = (intent: unknown) =>
  (intent as { request: { to: string; data: `0x${string}` } }).request

// authorizeNameRoles(bytes,uint256,address,bool)
const AUTHORIZE_NAME_ROLES = toFunctionSelector(
  'authorizeNameRoles(bytes,uint256,address,bool)',
)
// revokeRootRoles(uint256,address)
const REVOKE_ROOT_ROLES = toFunctionSelector('revokeRootRoles(uint256,address)')

const word = (data: string, i: number) =>
  data.toLowerCase().slice(2 + 8 + i * 64, 2 + 8 + (i + 1) * 64)

// The exact sidebar scope selection (ResolverRolesSidebar.tsx:166-169):
const sidebarScopeFor = (row: { resolvedNames: readonly string[] }) =>
  row.resolvedNames.find((name) => name !== '(root)') ?? ''

const prepareRemove = (name: string, roles: readonly ResolverRoleKey[]) =>
  requestOf(
    prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER_ADDRESS,
      name,
      account: ACCOUNT,
      roles,
      walletClient,
      chainId: 11155111,
    }),
  )

// The removal encoding executed on the fork (the report reproduces it
// independently with cast); the app's own builder must emit it byte-for-byte.
const pocEquivalent =
  '0xbbd9abb5' +
  '0000000000000000000000000000000000000000000000000000000000000080' +
  '0000000000000000000000000000000000000000000000000000000000000011' +
  '00000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8' +
  '0000000000000000000000000000000000000000000000000000000000000000' +
  '000000000000000000000000000000000000000000000000000000000000000e' +
  '086861726d6f6e69610365746800000000000000000000000000000000000000'

describe('Remove-user approval on a merged row (root + name scope)', () => {
  it('Remove user on a mixed row prepares only a name-scoped removal', () => {
    const roleRecords = [
      { account: ACCOUNT, resource: ROOT_RESOURCE, roleBitmap: '1' },
      { account: ACCOUNT, resource: HARMONIA_RESOURCE, roleBitmap: '16' },
    ]
    const rows = groupRolesByAccount(
      roleRecords,
      buildResourceToNameMap([{ name: 'harmonia.eth' }]),
    )
    const row = rows[0]

    expect(row.decodedRoles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])
    expect(row.resolvedNames).toEqual(['(root)', 'harmonia.eth'])

    const roleName = sidebarScopeFor(row)
    expect(roleName).toBe('harmonia.eth')

    const { to, data } = prepareRemove(roleName, row.decodedRoles as ResolverRoleKey[])

    expect(to.toLowerCase()).toBe(RESOLVER_ADDRESS.toLowerCase())
    expect(data.slice(0, 10).toLowerCase()).toBe(AUTHORIZE_NAME_ROLES)
    // bitmap word = union of both scopes' displayed roles (0x11)
    expect(word(data, 1)).toBe(BigInt(0x11).toString(16).padStart(64, '0'))
    // the transaction the operator is handed for this click, byte-for-byte
    expect(data.toLowerCase()).toBe(pocEquivalent)

    console.log('built transaction data:', data)
    console.log('row roles:', row.decodedRoles)
    console.log('row scopes:', row.resolvedNames)
    console.log('scope selected by sidebar:', roleName === '' ? 'root' : 'name')
    console.log('global removal branch reached:', roleName === '')
  })

  it('CONTROL: a root-only row reaches the root removal branch', () => {
    const roleRecords = [
      { account: ACCOUNT, resource: ROOT_RESOURCE, roleBitmap: '17' },
    ]
    const rows = groupRolesByAccount(
      roleRecords,
      buildResourceToNameMap([{ name: 'harmonia.eth' }]),
    )
    const row = rows[0]

    expect(row.resolvedNames).toEqual(['(root)'])
    const roleName = sidebarScopeFor(row)
    expect(roleName).toBe('')

    const { data } = prepareRemove(roleName, row.decodedRoles as ResolverRoleKey[])

    expect(data.slice(0, 10).toLowerCase()).toBe(REVOKE_ROOT_ROLES)
    // args: [bitmap, account] — bitmap 0x11 for both roles, account word next
    expect(word(data, 0)).toBe(BigInt(0x11).toString(16).padStart(64, '0'))
    expect(word(data, 1)).toBe(ACCOUNT.toLowerCase().replace('0x', '').padStart(64, '0'))

    console.log('root-only row scopes:', row.resolvedNames)
    console.log('scope selected by sidebar: root')
    console.log('root removal branch reached: true (revokeRootRoles calldata)')
  })
})
```

Output of my run (vitest 4.1.5, repo pinned at the commit listed under References):

```
built transaction data: 0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001100000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000e086861726d6f6e69610365746800000000000000000000000000000000000000
row roles: [ 'ROLE_SET_ADDR', 'ROLE_SET_TEXT' ]
row scopes: [ '(root)', 'harmonia.eth' ]
scope selected by sidebar: name
global removal branch reached: false

root-only row scopes: [ '(root)' ]
scope selected by sidebar: root
root removal branch reached: true (revokeRootRoles calldata)

Test Files  1 passed (1)
Tests  2 passed (2)
```

Side by side, at the moment of approval:

* The application displayed: "Are you sure you want to remove this user from all roles? This action cannot be undone."
* The transaction it built for the click that confirmed those words: a revocation carrying both of the row's displayed roles in its mask, applied at one name scope only, with the revoke flag set. The account's global scope is not addressed by the call at all, and the global-removal branch of the application's own builder — the branch the control row reaches — is never taken.

### The executed state, as verification

The ENS smart contracts are not claimed defective; they behave exactly as designed and appear only as the execution environment for the transaction the application builds. This transcript exists for one purpose: to verify that the description the application displayed was false. After the confirmed removal, the account's global grant is still present on-chain.

The claim above is complete without this appendix: the displayed description and the application's own built transaction are products of apps/portal alone, established by the source, the captures, and the offline harness, none of which touches the chain. What follows only confirms the executed effect.

Environment: foundry (anvil + cast). Fork of live Sepolia pinned to block 11553325 (transcript executed at that block; deterministic).

Contracts used as the execution environment for the app's transaction:

```
VerifiableFactory 0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
PermissionedResolverImpl 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
```

{% stepper %}
{% step %}

## Start the fork

```sh
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --fork-block-number 11553325 --chain-id 11155111 --port 8545
```

{% endstep %}

{% step %}

## Set environment variables

```sh
RPC=http://127.0.0.1:8545
FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
IMPL=0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
A=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
PK_A=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
B=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
C2=0xa1c8256043f3bdd5b683f570db40bc936b7dd294
SALT=0xe31a7f93245359127604e49b10a834c7c995d4ecc83b2ca3e32df518871decf3
INIT=0x7058b559000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266111111111111111111111111111111111111111111111111111111111111100000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000000
DNS=0x086861726d6f6e69610365746800
RES=0x4234bb81fe55340abfd83a35ea9be4edc9ccb26e197dc842ec4137e9a03d1de1
```

{% endstep %}

{% step %}

## Set up the merged row displayed by the sidebar

```sh
# 1. deploy resolver via the app's own encoding (owner A, full root init bitmap)
cast send $FACTORY "deployProxy(address,uint256,bytes)" $IMPL $SALT $INIT --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 3000000
# -> status 0x1, resolver at $C2

# 2. operator grants B root ROLE_SET_ADDR (the Add-user sheet's default scope)
cast send $C2 "grantRootRoles(uint256,address)" 1 $B --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 300000

# 3. operator grants B name-scope ROLE_SET_TEXT on harmonia.eth (the innocuous request)
cast send $C2 "authorizeNameRoles(bytes,uint256,address,bool)" $DNS 0x10 $B true --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 300000

# pre-state: the mixed row the sidebar shows
cast call $C2 "roles(uint256,address)(uint256)" 0 $B --rpc-url $RPC
# -> 1
cast call $C2 "roles(uint256,address)(uint256)" $RES $B --rpc-url $RPC
# -> 16
```

{% endstep %}

{% step %}

## Submit the application's removal transaction

The app's removal send, the encoding the application builds (byte-identity below):

```sh
cast send $C2 "authorizeNameRoles(bytes,uint256,address,bool)" $DNS 0x11 $B false --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 300000
# -> status 0x1
```

{% endstep %}

{% step %}

## Verify the post-state

Expected zero everywhere after a removal from all roles; actual:

```sh
cast call $C2 "roles(uint256,address)(uint256)" 0 $B --rpc-url $RPC
# -> 1    <== ROOT ROLE STILL THERE
cast call $C2 "roles(uint256,address)(uint256)" $RES $B --rpc-url $RPC
# -> 0    <== only the name part died
```

Expected vs actual: the dialog says "remove this user from all roles". Expected after removal: `roles(0, B) = 0`. Actual: `roles(0, B) = 1`. The description the application displayed at approval time was false.
{% endstep %}
{% endstepper %}

### Byte-identity verification

The harness above drives the application's own grouping, scope pick, and transaction builder with the same two indexer records the pre-state readbacks print, and its assertion requires the produced transaction to equal this encoding byte-for-byte. The full encoding reproduces independently of the application:

```sh
cast calldata "authorizeNameRoles(bytes,uint256,address,bool)" 0x086861726d6f6e69610365746800 0x11 0x70997970c51812dc3a010c7d01b50e0d17dc79c8 false
```

```
0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001100000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000e086861726d6f6e69610365746800000000000000000000000000000000000000
```
