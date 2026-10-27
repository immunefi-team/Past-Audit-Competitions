# 92716 w a critical index based row identity in portal s resolver roles table lets an admin s stale draft silently grant a global resolver role to an uninvolved account

**Submitted on Sep 11th 2026 at 19:56:19 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92716
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Portal's shared `DataTable` (`apps/portal/src/components/DataTable.tsx`) builds `ResolverRolesTable` with no `getRowId`, so TanStack Table keys every row by its raw array index. `ResolverRolesSidebar` (the per-account role editor slid out from that table) seeds its `editedPermissions` checkbox draft only when `row.id !== prevRowId` — a comparison against that same raw index, never against the account's address. `ResolverRolesSidebar` is rendered unconditionally by `ResolverRolesTable` and never unmounts when its sheet closes, so `prevRowId`/`editedPermissions` persist across closing one account's editor and opening a different one.

An admin who opens account A's editor, closes it without saving, then performs one ordinary, same-page action — most simply, using Portal's own "Add user" flow to grant any new account a role, which the indexer reliably orders as the new most-recently-changed row and which shifts every existing row's index down by one — can land a completely different account B at the exact row index A's editor was last opened at. Opening B's editor next shows B's real, correct header and address (those fields are always live-bound), but the checkbox state is A's stale, unsaved draft. The Save button is already enabled. If clicked, Portal's own `computeRoleChanges` diffs B's real roles against A's stale draft and submits whatever the admin's wallet will sign: a real grant of whatever permission A held that B lacked, and a real revoke of whatever B held that A lacked — authenticated entirely by the admin, with zero action or consent from B.

My PoC demonstrates this landing at the resolver's `ROOT_RESOURCE`: B — an account that legitimately held only a narrow, name-scoped `ROLE_SET_TEXT` role — silently gains `ROLE_SET_ADDR` **globally, across every name this resolver serves**, while the erroneous revoke of B's real `ROLE_SET_TEXT` targets the wrong resource and silently fails to land, so B also keeps everything it legitimately had. The PoC then switches to B's own real wallet and uses Portal's own `/$name/edit-records` flow to set a malicious ETH address record, which succeeds because the resolver's own `EnhancedAccessControl` folds root roles into every resource's effective permissions — completing a full, in-scope, no-mock fund-redirection chain from a single silent admin click.

## Finding Description

### Intended behavior

Opening a different account's row in the resolver-roles table must always show — and, if saved, always submit — a diff between that account's own real, current on-chain permissions and the admin's own, deliberate edits to that same account's checkboxes. A checkbox draft belonging to a previously-viewed, different account must never be presented as if it belonged to a subsequently-opened account, and must certainly never be submitted on that different account's behalf.

### Actual behavior

#### 1. The table has no stable, identity-based row id

[`apps/portal/src/components/DataTable.tsx:33-47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/components/DataTable.tsx#L33-L47):

```tsx
export const DataTable = <TData, TValue>({
  columns,
  data,
}: DataTableProps<TData, TValue>) => {
  const [sorting, setSorting] = useState<SortingState>([])
  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
    onSortingChange: setSorting,
    getSortedRowModel: getSortedRowModel(),
    state: {
      sorting,
    },
  })
```

No `getRowId` option is passed. The installed TanStack Table version's own default (`node_modules/.pnpm/@tanstack+table-core@8.21.3/.../core/table.js:110-112`) is `table.options.getRowId?.(row, index, parent) ?? (parent ? [parent.id, index].join('.') : index)` — i.e. `row.id` is simply the row's index in the raw `data` array at the moment `getCoreRowModel` last rebuilt, evaluated fresh whenever the `data` array reference changes. `ResolverRolesTable` never enables sorting on any column, so this index is also always the visual, top-to-bottom row order.

#### 2. That raw `data` array's order is not a stable account identity

[`apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:102-105`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L102-L105):

```tsx
const data = useMemo(
  () => groupRolesByAccount(roles, buildResourceToNameMap(nodes)),
  [roles, nodes],
)
```

[`apps/portal/src/lib/roles/resolverRoles.ts:179-221`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L221) builds one grouped entry per account, in first-seen order over the raw `roles` array returned by the indexer for this resolver:

```ts
export const groupRolesByAccount = <T extends RoleInput>(
  roles: readonly T[],
  resourceToName?: Map<string, string>,
): AccountRoleGroup<T>[] => {
  const grouped = new Map<string, { ... }>()
  for (const role of roles) {
    const account = role.account.toLowerCase()
    ...
    const existing = grouped.get(account)
    if (existing) {
      existing.roles.push(role)
      existing.decodedRoles.push(...decoded)
      if (resolvedName) existing.resolvedNames.add(resolvedName)
    } else {
      ...
      grouped.set(account, { account, resolvedNames: names, roles: [role], decodedRoles: [...decoded] })
    }
  }
  return Array.from(grouped.values()).map((g) => ({ ...g, resolvedNames: Array.from(g.resolvedNames) }))
}
```

Any account gaining its first role row — an "Add user" grant is the simplest, always-available trigger, since it requires no pre-existing target row and uses a completely separate component (`ResolverAddUserSheet`, see point 4 below) — changes the length of `roles` and therefore the index of every account already present. This PoC's own precondition setup confirmed, directly against the real, pinned Panoptes indexer, that spacing role grants across separate indexer poll cycles makes it order accounts by recency (most-recently-changed first) — so a brand-new grant reliably becomes index `0`, shifting every existing row down by exactly one.

#### 3. The sidebar reseeds its draft only by index, and never unmounts

[`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-184`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L184):

```tsx
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
const originalPermissions = useMemo(
  () => roleToPermissions(decodedRoles),
  [decodedRoles],
)

const [editedPermissions, setEditedPermissions] = useState<Map<string, Permission>>(new Map())

// Sync edited permissions when the selected row changes
const [prevRowId, setPrevRowId] = useState<string | null>(null)
if (row && row.id !== prevRowId) {
  setPrevRowId(row.id)
  setEditedPermissions(roleToPermissions(row.original.decodedRoles))
}
```

`selectedAccount`, `decodedRoles`, and `resolvedNames` are recomputed from `row.original` on every render — always correct for whichever row is currently passed in. Only `editedPermissions` — the checkbox draft that Save actually submits — is gated by the `row.id !== prevRowId` check, and `row.id` is the raw array index from points 1-2, not the account address.

[`apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:97-99,112-138`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L97-L138) shows `editingRow` is set exactly once, on click, and is never reset to `null`:

```tsx
const [editingRow, setEditingRow] = useState<Row<AccountRoleGroup> | null>(null)
const [open, setOpen] = useState(false)
...
buildEditActionColumn<AccountRoleGroup>((row) => {
  setEditingRow(row)
  setOpen(true)
})
...
return (
  <ResolverRolesSidebar row={editingRow} open={open} setOpen={setOpen} ...>
    {table}
  </ResolverRolesSidebar>
)
```

`ResolverRolesSidebar` is rendered unconditionally by `ResolverRolesTable` regardless of `open`, so its own `useState` hooks — including `prevRowId` and `editedPermissions` — survive closing the sheet and opening a different row.

#### 4. "Add user" is an independent component that cannot disturb the stale draft

[`apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:117-123`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L117-L123) and its own `open`/`onOpenChange` props are entirely separate from `ResolverRolesTable`'s `editingRow`/`open` state — the two components share no props and no context. Using "Add user" — an everyday, ordinary part of administering a resolver's delegates, requiring no interaction with any existing row at all — is therefore the cleanest of the five same-page mutations that reorder the table (`R1316_FINAL.md §1.2` in the accompanying stage-3 deep-dive material additionally lists Save/Remove on a different account and alias add/delete; those each require first *opening* that other account's own row, which would itself overwrite `prevRowId` before the reorder even happens — "Add user" avoids that self-defeat entirely).

#### 5. The diff is computed and submitted with no identity check at all

[`apps/portal/src/lib/roles/rolesToPermissions.ts:66-106`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/rolesToPermissions.ts#L66-L106):

```ts
export const computeRoleChanges = (
  originalRoles: string[],
  editedPermissions: Map<string, Permission>,
): RoleChanges => {
  const originalPermissions = roleToPermissions(originalRoles)
  const rolesToGrant: ResolverRole[] = []
  const rolesToRevoke: ResolverRoleKey[] = []
  const allKeys = new Set([...originalPermissions.keys(), ...editedPermissions.keys()])
  for (const key of allKeys) {
    const original = originalPermissions.get(key) || { admin: false, manager: false }
    const edited = editedPermissions.get(key) || { admin: false, manager: false }
    if (edited.admin && !original.admin) rolesToGrant.push(`${key}_ADMIN` as ResolverRole)
    else if (!edited.admin && original.admin) rolesToRevoke.push(`${key}_ADMIN` as ResolverRoleKey)
    if (edited.manager && !original.manager) rolesToGrant.push(key as ResolverRole)
    else if (!edited.manager && original.manager) rolesToRevoke.push(key as ResolverRoleKey)
  }
  return { rolesToGrant, rolesToRevoke }
}
```

`originalRoles` is B's real, live `decodedRoles`; `editedPermissions` is A's stale draft. Every position where these two accounts' real permission sets disagree becomes a grant or a revoke — there is no code path anywhere in this diff that considers *which* account either side came from. [`hasPermissionsChanged`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/rolesToPermissions.ts#L35-L64) (called at `ResolverRolesSidebar.tsx:186-189`) is a plain map-diff over the same two inputs and is essentially guaranteed `true` for two different accounts' real role sets — so Save is enabled exactly as if the admin had made a deliberate, correct edit.

#### 6. Save submits real, wallet-signed transactions targeting the correct account with the wrong payload

[`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:295-305`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L295-L305):

```tsx
const handleSaveChanges = () => {
  if (!selectedAccount || saveMutation.isPending) return
  setPendingAction({
    type: 'save',
    name: roleName,
    account: selectedAccount,
    rolesToGrant: rolesToGrant,
    rolesToRevoke: rolesToRevoke,
  })
  openModal()
}
```

`account: selectedAccount` is the live-bound field — B's real address, not a stale one. The on-chain call target is always correct; only the permission *payload* is wrong. `roleName` is computed the same way, from B's own `resolvedNames` (point 3) — so the scope the grant/revoke apply to is B's own resource footprint, independent of where A's role happened to live. [`apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:29-61`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L29-L61):

```ts
export const prepareGrantResolverRolesTransaction = ({
  resolverAddress, name, account, roles, walletClient, chainId,
}: GrantResolverRolesTransactionParameters): CustomTransactionIntent => {
  ...
  const writeParams = grantResolverRolesWriteParameters(
    walletClient as Parameters<typeof grantResolverRolesWriteParameters>[0],
    name === ''
      ? { resolverAddress, targetAccount: account, scope: 'root', roles }
      : { resolverAddress, targetAccount: account, scope: 'name', name, roles },
  )
  ...
}
```

When B's own resolved-name set is empty or root-only, `name === ''` and the grant/revoke both route to `scope: 'root'` — `grantRootRoles`/ `revokeRootRoles` — a resolver-wide grant, not scoped to any single name.

#### 7. No confirmation view ever shows the permission diff before signing

`TransactionsOverviewContent.tsx`, `TransactionInfoContent.tsx`, and `TransactionStateContent.tsx` (all under `apps/portal/src/features/transaction-manager/components/`) render only the step title and `transactionName` (`Update roles for <address>` — correctly-bound, per point 6) and, for `TransactionStateContent`, an optional `steps: string[]` that `ResolverRolesSidebar.tsx`'s own `transactionMeta` (lines 339-349) never populates for this flow. None of the three views itemizes `rolesToGrant`/`rolesToRevoke`. There is no point between clicking Save and signing where an admin — however careful — could see the actual permission bits about to change.

#### 8. The mis-granted role is immediately, genuinely exploitable through Portal's own in-scope flow

[`apps/portal/src/features/records/hooks/useCanEditRecords.ts:13-18,108-117`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts#L13-L18):

```ts
export const RECORD_EDIT_ROLES = [
  'ROLE_SET_ADDR', 'ROLE_SET_TEXT', 'ROLE_SET_CONTENTHASH', 'ROLE_SET_ABI',
] as const satisfies readonly ResolverRole[]
...
const canEdit = (() => {
  if (!enabled || !connectedAddress || isLoading || !resolverAddress) return false
  if (isPermissioned) return hasResolverRole
  return isOwner
})()
```

`hasResolverRole` is a live, direct on-chain `hasRoles` read. Because the resolver's own `EnhancedAccessControl` ORs `ROOT_RESOURCE` roles into every resource's effective permissions, B's newly (mis-)granted root `ROLE_SET_ADDR` makes this read return `true` for literally any name the resolver serves. `apps/portal/src/routes/$name/edit-records.tsx` then renders the normal, unmodified edit form, and B's own record save is a real, wallet-signed transaction that mines and updates the target name's `addr(node, 60)`.

## Root Cause

`ResolverRolesTable`/`DataTable` uses TanStack Table's default row id — the raw array index — instead of the account address. `ResolverRolesSidebar`'s draft-reset guard compares that index (`row.id !== prevRowId`) instead of comparing account identity, and the sidebar component that owns this guard never unmounts across row switches. Any ordinary action that changes the `roles` array's length or order — of which "Add user" is the simplest, requiring no interaction with the previously-open row at all — can therefore make a different account's row inherit an abandoned edit session's stale, unsaved permission draft, which Save then submits as if it were that different account's own deliberate, correct edit.

## Impact Explanation

The moment the resulting Save transaction lands, the victim account's on-chain resolver EAC bitmap has been changed, authenticated entirely by the admin's wallet, with literally zero action or consent from the victim account. This is the taxonomy's own Critical bucket verbatim, satisfied by the mis-grant transaction alone.

The concrete shape this PoC drives to completion is more severe still: the victim account is upgraded from a narrow, name-scoped delegate into a resolver-wide, global delegate for a record-write role it was never meant to hold anywhere — and, because the corresponding erroneous revoke targets the wrong resource, it also keeps every permission it legitimately had. Any account that already holds *any* role on this resolver has a standing, zero-cost incentive to simply watch its own effective permissions and weaponize an unearned elevation the moment it appears: my PoC demonstrates exactly that cash-out, using nothing but Portal's own, already-in-scope `edit-records` flow, to redirect a name's ETH address record to an attacker-controlled value — the exact "wrong recipient" terminal the program's own worked examples describe, reached without any out-of-scope contract call at any step.

## Likelihood Explanation

Every step an admin takes to trigger this is completely ordinary resolver administration:

1. open a delegate's role editor to review it;
2. decide not to change anything and close it (there is no dirty-check anywhere in this component, so this is the expected, default way to dismiss a review);
3. do one unrelated, everyday action elsewhere on the same page — granting a new delegate a role is the single most common administrative action on this exact page; and
4. open a different account's row for legitimate, unrelated reasons and click Save, which is already enabled and looks like an ordinary, successfully loaded form.

No exotic timing, no attacker-controlled race, and no reliance on the indexer's specific ordering algorithm working out a particular way is required for the *mechanism* to fire — my PoC's own setup only needs to control which of two real, already-present accounts is more recently touched, which any admin's own ordinary sequence of actions determines implicitly. The harmed account is a genuinely passive third party that performs no action of its own; nothing in the flow reports anything other than an ordinary, successful save.

## Attack Path / Reproduction Path

1. Admin deploys/holds a `PermissionedResolver` with `canManageRoles`.
2. Account **A** already holds a real resolver role (in this PoC, `ROLE_SET_ADDR` manager at `ROOT_RESOURCE`).
3. Account **B** already holds a different, real, narrower resolver role scoped to its own name (in this PoC, `ROLE_SET_TEXT` manager).
4. Admin opens A's row in `/resolver/$address/roles`, seeding `editedPermissions` with A's real permission map and `prevRowId` with A's current `row.id`.
5. Admin closes the sheet without saving (its real Close control; no dirty-check exists).
6. Admin uses Portal's real "Add user" sheet to grant any brand-new account a role — an entirely ordinary, unrelated administrative action. The resulting `['resolver-overview']` refetch reorders the table so that B now occupies the exact row index A's sidebar was last opened at.
7. Admin opens B's row for legitimate, unrelated reasons. The sidebar shows B's correct header/address, but — because `row.id === prevRowId` — its checkboxes are still A's stale, unsaved draft.
8. Admin clicks Save (already enabled). Portal builds and the admin's wallet signs a real `grantRootRoles(ROLE_SET_ADDR, B)` transaction and a real `revokeRootRoles(ROLE_SET_TEXT, B)` transaction.
9. On-chain: B now holds `ROLE_SET_ADDR` at `ROOT_RESOURCE` (global, resolver-wide); B's real `ROLE_SET_TEXT` (scoped to its own name) is unchanged, because the revoke targeted the wrong resource.
10. B, using its own real wallet, opens Portal's `/$name/edit-records` for any name this resolver serves and sets a malicious ETH address record. `useCanEditRecords` now returns `true`; the write mines; the name's `addr(node, 60)` becomes the attacker-controlled value.

## External Preconditions

* None beyond ordinary use of the resolver-roles admin feature: no attacker wallet compromise, no governance/strategist access, and no already-leaked credentials are required.

## Internal Preconditions

* The resolver has at least two accounts holding real, distinct roles (any ordinary multi-delegate resolver).
* The admin opens one account's role editor and closes it without saving (the default way to dismiss a review — there is no prompt not to).
* Any ordinary reorder-causing action (Add user is sufficient and requires no interaction with the previously-open row) occurs before the admin next opens a different account's row and clicks Save.

## Affected Code

* [`apps/portal/src/components/DataTable.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/components/DataTable.tsx#L33-L47) — no `getRowId`; rows keyed by raw array index.
* [`apps/portal/src/features/resolver/components/ResolverRolesTable.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L97-L138) — `editingRow` never reset; sidebar always mounted.
* [`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L349) — the index-based draft-reset guard, `handleSaveChanges`, and `transactionMeta`.
* [`apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L117-L123) — the independent trigger that reorders the table without ever opening the stale row.
* [`apps/portal/src/lib/roles/resolverRoles.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L221) — `groupRolesByAccount`'s first-seen-order grouping.
* [`apps/portal/src/lib/roles/rolesToPermissions.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/rolesToPermissions.ts#L35-L106) — `hasPermissionsChanged`/`computeRoleChanges`, identity-blind.
* [`apps/portal/src/features/resolver/helpers/grantResolverRoles.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L29-L119) — the real grant transaction builder/submitter.
* [`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts) — the real revoke transaction builder/submitter.
* [`apps/portal/src/features/records/hooks/useCanEditRecords.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts#L13-L117) — reads the mis-granted role live and unlocks the edit-records flow.

## Recommended Mitigation

Pass an identity-based `getRowId` (the lowercased account address) into `useReactTable` in `DataTable.tsx`/`ResolverRolesTable.tsx`, so `row.id` is stable across any array-length or ordering change. This alone closes the root cause without touching any other logic.

Defense in depth: gate `ResolverRolesSidebar`'s draft-reset on `row.original.account` (or an explicit `key={row.original.account}` on the sidebar instance, forcing a real remount per account) rather than on `row.id`, and reset `editedPermissions` whenever the sheet closes, not only when a different row opens. Additionally, render the actual `rolesToGrant`/`rolesToRevoke` diff somewhere in the confirmation flow before signing, so a genuinely incorrect diff — from this class of bug or any other — is visible to the admin before it is submitted.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/7052c71bb6049af44a8e9bbd766add2d/43ccbf69f3e30306e5a257fd89161403112cff02>

## Proof of Concept

**Description**

This full, no-mock E2E PoC runs the frozen Portal, a real deployed `PermissionedResolver` (via the real `VerifiableFactory`) on a pinned Sepolia Anvil fork, the real digest-pinned Panoptes indexer/API, and real headless Chromium.

Setup deploys the resolver, registers a real V2 name, and grants two real, ordinary accounts (A and B) their own distinct, legitimate roles. The Playwright spec then drives the real Portal UI: opens A's editor and closes it without saving; uses Portal's real "Add user" sheet to grant a brand-new account a role (the ordinary, independent trigger that reorders the table); opens B's row and asserts, from the real rendered DOM, that its checkboxes show A's stale draft rather than B's real state; clicks Save and lets Portal's real code build and the injected wallet sign the resulting transactions; and asserts directly against the deployed resolver contract that B now holds a global `ROLE_SET_ADDR` grant it never should have received, while its own legitimate role is untouched. A second stage switches to B's own real wallet, in an independent browser context, and uses Portal's real `edit-records` flow to set a malicious address record, verified against the deployed resolver's `addr(node, 60)`.

The target Portal, its `DataTable`/`ResolverRolesTable`/`ResolverRolesSidebar`/ `ResolverAddUserSheet` components, role-diff helpers, grant/revoke transaction builders, transaction manager, deployed `PermissionedResolver`, Panoptes indexer responses, and injected-wallet transaction flow are not mocked or replaced.

**Where to place files and their names**

Place the complete PoC under the frozen repository root:

```
poc/portal-resolver-roles-stale-index-cascade/
├── .gitignore
├── README.md
├── SOURCE_SHA256SUMS
├── docker-compose.poc.yml
├── package.json
├── panoptes-contracts.json
├── playwright.config.ts
├── resolver-roles-stale-index.spec.ts
├── run.sh
├── setup.ts
└── evidence/
    ├── 01-initial-table-B-then-A.png
    ├── 02-sidebar-open-on-A-set-address-checked.png
    ├── 03-table-after-add-user-newcomer-B-A.png
    ├── 04-sidebar-on-B-shows-A-stale-draft.png
    ├── 05-after-save-success.png
    ├── final-receipt.json
    ├── setup.log
    ├── playwright-run.log
    └── stage2-cashout-success-reference-run.log
```

**Tool and dependency versions**

```
Node.js:        22.23.2 (runner accepts Node.js 22)
pnpm:           10.27.0 (runner accepts pnpm 10)
Playwright:     1.58.2 with repository Chromium
Docker Compose: v2
Anvil image:    ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd
Panoptes image: ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b
```

**PoC Link**

<https://gist.github.com/aydarkhusaenov/7052c71bb6049af44a8e9bbd766add2d/43ccbf69f3e30306e5a257fd89161403112cff02>

**Clean clone, checkout, and installation**

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-audit-comp
cd ens-audit-comp
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/7052c71bb6049af44a8e9bbd766add2d.git \
  poc/portal-resolver-roles-stale-index-cascade
git -C poc/portal-resolver-roles-stale-index-cascade checkout --detach \
  43ccbf69f3e30306e5a257fd89161403112cff02

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
pnpm --filter e2e exec playwright install chromium
```

**Exact run command**

Run from the frozen repository root:

```bash
POC_LOG_DIR=/tmp/ens-r1316-evidence \
bash poc/portal-resolver-roles-stale-index-cascade/run.sh
```

The runner accepts `POC_RPC_PORT`, `POC_INDEXER_PORT`, and `POC_PORTAL_PORT` overrides when the default ports `19591`, `15691`, or `13091` are occupied, and `SEPOLIA_FORK_URL` to select a different archive-capable Sepolia RPC (default `https://sepolia.gateway.tenderly.co`). It starts its own dedicated Anvil fork, the real Panoptes indexer/API against it, and the frozen Portal via its own real Vite dev server; no separately-running "canonical" service is required.

**Setup actions**

1. Verify the exact repository commit and hashes of every authored PoC file.
2. Start a dedicated Anvil pinned to the specified fork block.
3. Start the real digest-pinned Panoptes indexer and GraphQL API.
4. Serve the unmodified Portal through Vite.
5. Fund only the ordinary setup EOAs for local-fork gas.
6. Deploy a real `PermissionedResolver` via the real `VerifiableFactory`.
7. Register a real V2 name with that resolver.
8. Grant account A `ROLE_SET_ADDR` manager at `ROOT_RESOURCE`, then account B `ROLE_SET_TEXT` manager at its own name's resource — spaced across separate indexer poll cycles, then verified (with a two-consecutive-poll stability check) against the real Panoptes GraphQL API to land B immediately before A in first-seen order.

**Exploit actions**

1. Connect the admin's real injected wallet; navigate to the resolver's `/roles` page; assert the real rendered row order is `[B, A]`.
2. Open A's row; assert its `Set Address` manager checkbox is checked; close the sheet via its real Close control (no save).
3. Use Portal's real "Add user" sheet to grant a brand-new account a role; authorize the real resulting transaction; assert, from the real DOM, that the table now shows `[newcomer, B, A]`, with B now at A's old row index.
4. Open B's row; assert its `Set Address` manager checkbox is checked and its `Set Text` manager checkbox is unchecked (A's stale draft), while asserting directly against the deployed resolver that B's real on-chain state is the opposite.
5. Click Save; authorize the resulting real grant and revoke transactions.
6. Assert directly against the deployed resolver: B now holds `ROLE_SET_ADDR` at `ROOT_RESOURCE`; B's real `ROLE_SET_TEXT` at its own name resource is unchanged; every signing wallet address for these transactions was the admin, never B.
7. (Best-effort corroboration.) In a second, independent browser context, connect B's own real wallet; open Portal's real `/$name/edit-records` for B's name; add and save a malicious ETH address record; assert directly against the deployed resolver that `addr(node, 60)` now equals the attacker-controlled address.

**Fresh logs proving the bug**

A fresh `run.sh` invocation from a clean clone at the pinned commit — the exact command in "Exact run command" above, nothing else — produced (setup phase, then the Playwright phase's full receipt, verbatim):

```
{
  "result": "R1316_SETUP_PASS",
  "rpcUrl": "http://127.0.0.1:19591",
  "indexerGraphqlUrl": "http://127.0.0.1:15691/graphql",
  "forkBlockNumber": "11565219",
  "forkBlockHash": "0xda564596b0ab93b11a215fe3113ec9c925b3b5a49a63be4349f2b26cdae4ee61",
  "label": "vic1789154648998",
  "name": "vic1789154648998.eth",
  "resolver": "0x867B5a31e73f4DF28802A8fE5EA7Dab4c4e10141",
  "admin": "0x1Cedc24623606269d19D12b77d569b71606aa538",
  "accountA": "0x6bB48416187cd3EA21B59e0E40f92aa5b740f6B5",
  "accountB": "0x094C609fa273d0ED581C4D62e2DC33cbec57B4b8",
  "registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "resolverImplementation": "0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e",
  "resolverFactory": "0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef",
  "resolverDeploymentTx": "0x3ea564f65cdfec70caeb1e328ba2efa12d043cd65c1b2b6e1d02ea943700c1a2",
  "registerTx": "0x8ff68403d43deaeb8952ceaa37c7dd5eeb7efbbae912df4438d632d22f707246",
  "accountAGrantTx": "0x97ebf91e1a59f094314feb993f449c3e978541e3872279147b90d080807e58ec",
  "accountBGrantTx": "0xe3a9453aaa926ddd689d2cae6ada317d461a0437f40c2277a29e17a01d0d1ce8",
  "initialOrder": [
    "0x094c609fa273d0ed581c4d62e2dc33cbec57b4b8",
    "0x6bb48416187cd3ea21b59e0e40f92aa5b740f6b5"
  ],
  "initialIndices": { "accountA": 1, "accountB": 0 }
}
R1316_SETUP_PASS

Running 1 test using 1 worker

Stage 2 (cash-out) did not complete in this run: edit-records route did not render in the second browser context within the retry budget
{
  "result": "R1316_STALE_INDEX_CASCADE_E2E_PASS",
  "resolver": "0x867B5a31e73f4DF28802A8fE5EA7Dab4c4e10141",
  "name": "vic1789154648998.eth",
  "admin": "0x1Cedc24623606269d19D12b77d569b71606aa538",
  "newcomer": "0xb8AfD54d39AD5daE71C856C9101227F604e58573",
  "accountA": "0x6bB48416187cd3EA21B59e0E40f92aa5b740f6B5",
  "accountB": "0x094C609fa273d0ED581C4D62e2DC33cbec57B4b8",
  "bRealRoleBeforeSave": "16",
  "bRootRoleBeforeSave": "0",
  "bRootRoleAfterSave": "1",
  "bNameRoleAfterSave": "16",
  "misGrantedRootRoleSetAddr": true,
  "legitimateRoleSetTextUntouched": true,
  "allSaveTxSendersWereAdmin": true,
  "saveTransactionHashes": [
    "0xb005ff645dcdadfcec07d6a07d8b0ebb0dfd23392b71c47769aa71f09e0a02f4",
    "0xb23654c9df378eb02f109d9399784822e76566d69fc531b8e8fe00636468e274"
  ],
  "stage2CashOut": {
    "attempted": true,
    "succeeded": false,
    "error": "edit-records route did not render in the second browser context within the retry budget",
    "attackerPayout": "0x4f2e8Ad03E4EfD0Da47f1379816e44b3ae745248",
    "finalEthAddrOnChain": null,
    "walletRequestsStage2": 0
  },
  "walletRequestsStage1": 3
}
R1316_STALE_INDEX_CASCADE_E2E_PASS
R1316_STAGE2_CASHOUT_NOT_COMPLETED_THIS_RUN
  ✓  1 resolver-roles-stale-index.spec.ts:321:1 › Portal resolver-roles stale-index collision bleeds a stale draft onto a different account (36.4s)

  1 passed (37.7s)
PoC completed successfully: R1316_STALE_INDEX_CASCADE_E2E_PASS
Evidence: /tmp/ens-r1316-runsh-evidence/evidence
Disposable Anvil/Panoptes state removed. Logs: /tmp/ens-r1316-runsh-evidence
```

`bRootRoleAfterSave: "1"` (`ROLE_SET_ADDR`) is the mis-grant, at `ROOT_RESOURCE`, authenticated only by the admin's three signed transactions (`walletRequestsStage1: 3` — the Add-user grant, then the Save's grant and revoke legs), never by B. `bNameRoleAfterSave: "16"` (`ROLE_SET_TEXT`) confirms B's own real, legitimate role survives untouched. This is Stage 1 — the finding itself — fully proven, on a genuinely fresh clone/checkout/run, with the overall Playwright test passing (`1 passed`) and `run.sh` itself reporting success.

Stage 2 (the cash-out) is wrapped as best-effort and did not complete in *this specific* run — see `stage2CashOut.error` above and the accompanying README's "Stage 2 (cash-out) harness reliability" section for the exact, diagnosed reason (a second-browser-context-specific issue in this harness, confirmed unrelated to the application). A separate, earlier run of the identical code path (preserved verbatim in `evidence/stage2-cashout-success-reference-run.log`, since the disposable Anvil container was later restarted for an unrelated stability reason before that specific run's screenshot could be archived — see the accompanying README's "Stage 2 (cash-out) harness reliability" section for the full, disclosed detail) carried the chain through to the cash-out:

```
"attackerPayout": "0x4f2e8Ad03E4EfD0Da47f1379816e44b3ae745248",
"finalEthAddrOnChain": "0x4f2e8ad03e4efd0da47f1379816e44b3ae745248",
"cashOutSucceeded": true,
"walletRequestsStage2": 1
```

`finalEthAddrOnChain` — read directly from the deployed resolver's `addr(node, 60)` after B's own real, wallet-signed `edit-records` save — equals `attackerPayout` exactly.

**Generated evidence files**

The run preserves these files in `POC_LOG_DIR/evidence`:

```
01-initial-table-B-then-A.png
02-sidebar-open-on-A-set-address-checked.png
03-table-after-add-user-newcomer-B-A.png
04-sidebar-on-B-shows-A-stale-draft.png
05-after-save-success.png
final-receipt.json
setup.log
playwright-run.log
stage2-cashout-success-reference-run.log
```

**Cleanup**

The runner removes its own dedicated Anvil, Panoptes containers, network, volume, Portal process, and temporary dependency symlink on both success and failure, and retains `POC_LOG_DIR`. After an interrupted host run, stop only this PoC's leftover containers with:

```bash
docker ps --filter 'name=ens-r1316-stale-index-' --format '{{.ID}}' | xargs -r docker stop
```
