# 92587 w a critical explorer s resolver permission editor keys its checkbox state to the table row index so re opening a row position grants the previous occupant s permissions to a different acc&#x20;

**Submitted on Sep 10th 2026 at 19:12:12 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92587
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

`ResolverRolesSidebar` seeds its editable checkbox map from the selected row exactly once per distinct `row.id`:

```tsx
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:179-184
  // Sync edited permissions when the selected row changes
  const [prevRowId, setPrevRowId] = useState<string | null>(null)
  if (row && row.id !== prevRowId) {
    setPrevRowId(row.id)
    setEditedPermissions(roleToPermissions(row.original.decodedRoles))
  }
```

`row.id` is not an identity. The rows come from the shared `DataTable`, which calls `useReactTable` with no `getRowId` (`apps/portal/src/components/DataTable.tsx:38-47`), so TanStack's default applies:

```tsx
// @tanstack/table-core@8.21.3/src/core/table.ts:375-377
    _getRowId: (row: TData, index: number, parent?: Row<TData>) =>
      table.options.getRowId?.(row, index, parent) ??
      `${parent ? [parent.id, index].join('.') : index}`,
```

`row.id` is the array index — `"0"`, `"1"`, … — a **position**.

So whenever the roles list changes such that a given index holds a different account, re-opening that index leaves `editedPermissions` seeded from the previous occupant. Everything else in the sheet updates: the header, the scope line, `selectedAccount`, and `decodedRoles` all read from the new row. Only the checkbox grid is stale — and the checkbox grid is what gets written:

```tsx
// :190-193
  const { rolesToGrant, rolesToRevoke } = computeRoleChanges(
    decodedRoles as string[],        // the NEW account's on-chain roles
    editedPermissions,               // the PREVIOUS account's checkbox state
  )
```

`hasChanges` (`:186-189`) compares the new row's real roles against that stale map, so **Save is already enabled before the admin touches anything**, and `handleSaveChanges` (`:295-305`) snapshots the diff with `account: selectedAccount` — the new account.

Three other tables in this app already pass a stable `getRowId` — `EditRecordsTable.tsx:256` (`getRecordId(row)`), `SubnamesTable.tsx:185` (`row.name`) and `resolver/$address/aliases.tsx:199` (`row.fromName`). The shared `DataTable` does not, and the resolver roles sidebar is the one consumer that persists per-row state across a data change. The root cause is therefore in `components/DataTable.tsx` — a file #91306 never cites — and the sidebar is the consumer that turns it into a mis-write.

The two sibling sheets in the same codebase do it correctly. `useEditedPermissions` (`apps/portal/src/features/roles/hooks/useEditedPermissions.ts:15-20`) re-seeds on the row **object**, and `RegistryEditUserSheet` (`apps/portal/src/features/registry/components/v2/RegistryEditUserSheet.tsx:104-110`) re-seeds on `[open, currentRoles]`. Only the resolver sidebar keys on the index string.

***

## Impact

Precondition: a resolver with more than one role holder, and one list change between two edits. Both are ordinary.

The list changes as a direct result of the admin's own work. Both `Save` and `Remove user` in this very sheet call `pollForIndexerSync` and invalidate `['resolver-overview']` (`:240-248`, `:278-286`), and `groupRolesByAccount` (`apps/portal/src/lib/roles/resolverRoles.ts:179-221`) rebuilds the array from a `Map` keyed in the indexer's return order. Removing or adding any holder shifts every index below it.

The sequence:

1. The admin opens the sheet for the delegate at row 0 — a trusted operations account holding `ROLE_SET_ADDR`, `ROLE_SET_NAME`, `ROLE_CLEAR` — reviews it, and closes without saving.
2. Any list change puts a different account at row 0: a low-privilege contributor with only `ROLE_SET_TEXT`.
3. The admin opens row 0 again. The header shows the contributor. The scope line shows the contributor's name. The checkboxes show the operations account's three roles.
4. Save is enabled with no toggle. Pressing it grants `ROLE_SET_ADDR`, `ROLE_SET_NAME` and `ROLE_CLEAR` to the contributor and revokes their `ROLE_SET_TEXT`.

`ROLE_SET_ADDR` is the consequential one: it lets the grantee repoint the name's ETH address record, which is the app's own definition of a name resolving to an address the owner did not choose.

Nothing in the flow looks wrong. The admin is shown a set of checkboxes and presses Save; the app writes what it displayed. The wallet prompt carries `authorizeNameRoles(dnsEncode(name), <bitmap>, <account>, true)` — an opaque bitmap and an address the admin does have selected — so it cannot contradict the screen either.

***

## Root cause

| File:line                                                                                | Code                                                                                                                    |
| ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:179-184`          | the seed guard, keyed on `row.id`                                                                                       |
| `apps/portal/src/components/DataTable.tsx:38-47`                                         | `useReactTable` with no `getRowId`                                                                                      |
| `@tanstack/table-core@8.21.3/src/core/table.ts:375-377`                                  | the default `_getRowId` is the array index                                                                              |
| `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:97-115`             | `editingRow` state; `data` memoized on `[roles, nodes]`, so the same index yields the same `row.id` across close/reopen |
| `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:186-193`          | `hasChanges` and `computeRoleChanges` diff the new row's roles against the stale map                                    |
| `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:295-305`          | the pending action is snapshotted with the **new** `selectedAccount`                                                    |
| `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:212-238`          | `saveMutation` writes it via `grantResolverRoles` / `revokeResolverRoles`                                               |
| `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:240-248, 278-286` | both paths invalidate `['resolver-overview']`, which is what re-orders the list                                         |

`:179-184` is the only writer of `editedPermissions` besides the user's own `handlePermissionChange` (`:325-336`). There is no `useEffect` on `open`, on `selectedAccount`, or on `row`, and `onDone` (`:503-508`) closes the sheet without resetting it.

***

## Preempting the likely closure arguments

**"The admin can see the checkboxes before pressing Save."** They can see checkboxes — the ones the app pre-filled, next to the correct account name. There is nothing on the screen indicating the pre-fill belongs to a different account, and no other surface in the app to compare against. Pre-filling a form with the wrong party's data and then writing it is exactly the shape the impact row names.

**"It needs the list to re-order."** The sheet's own actions cause the re-order: `Save` and `Remove user` both invalidate `['resolver-overview']` (`:240-248`, `:278-286`), and the array is rebuilt from an indexer-ordered `Map`. Reviewing several delegates in one sitting is the ordinary use of this page.

**"This is #91306."** #91306 is `:169` — `roleName = resolvedNames.find(n => n !== '(root)') ?? ''`, which decides **which resource** the write targets. This is `:179-184`, which decides **which role bits** the write carries and where they came from. Different line, different value, independent fixes: canonicalising the scope does not re-seed the checkbox map, and re-seeding it does not fix the root sentinel.

**Recoverability.** The grant is on-chain and takes effect immediately. It can be revoked afterwards, if the admin notices — but they have no reason to, because the app told them they were editing that account's permissions and then wrote what it showed. A grantee holding the admin variant of a granted role can also re-grant it.

***

## Not a duplicate

* **#91306** (*Resolver-roles sidebar signs a different scope than it displays*): the scope sentinel at `:169`. This is the one that must be settled, because it is the same component and it is Confirmed Critical. #91306's own published fix is a typed `ResolverRoleScope` on the row, the sidebar reading `row.original.scope`, and removal of the `''`→ROOT sentinel at the helper boundary. Apply all three and `:179-184` is byte-identical: re-opening index 0 after the list changes still skips the re-seed, `hasChanges` still fires against the wrong base, and `computeRoleChanges` still emits the previous occupant's grants against the new `selectedAccount`. Conversely `getRowId` does nothing about the scope sentinel. The two defects sit in different argument positions of the same call — `name` (scope) versus `roles` (bits) — and this one is not even resolver-specific: it originates in the shared `DataTable`.
* **#92542** (*"Remove user" in Explorer's roles editor revokes transfer rights it never discloses*): `RolesSidebar.tsx:125-141`, the **registry** roles surface, and an over-revoke of a disclosed-vs-undisclosed bitmap. This is the **resolver** sidebar, and a stale-state defect rather than a bitmap-composition one.
* **#92540** (`useRoleHistory` querying by label hash with no registry filter): the History panel's query.
* The resolver role-table gap reported separately (`resolverRoles.ts:24-45`, missing `ROLE_SET_DATA`) is a different line and a different failure: that one is about which roles the app can *express*, this one about which account's set it *applies*. Worth noting they interact — when the stale map contains an `_ADMIN` entry the encode throws first, turning this instance into a failed transaction rather than a silent mis-grant. The silent mis-grant is the manager-only case, which is what `ResolverAddUserSheet` hands out.
* No known issue touches the roles sidebars. QA-01 / WEB-1257 / WEB-1305 are registry role combinations after a transfer.

***

## Recommendation

1. Give the table a stable identity: `getRowId: (row) => row.account` on the shared `DataTable`, or at minimum on `ResolverRolesTable`.
2. Re-seed on identity, not position. Key the guard on `row.original.account`, or follow `useEditedPermissions.ts:15-20` and key on the row object.
3. Reset on close. Clear `editedPermissions` in `onDone` / on `open === false` so a stale map cannot survive into the next selection.
4. Fail closed on a mismatch. Before submitting, assert that the account the edits were seeded from is the account being written to.
5. Audit the other consumers of `DataTable` for the same assumption — any component that persists per-row state across a data change is exposed to the identical defect.

## Proof of Concept

## Proof of Concept

Executed with the app's own helpers.

### Environment

No network is required — the PoC imports the app's own modules and runs them directly.

```bash
cd apps/portal && ./node_modules/.bin/vitest run .poc/roles-rowid.test.ts --reporter=verbose
```

### The guard, replayed against the app's real helpers

`apps/portal/.poc/roles-rowid.test.ts` replays the guard from `:179-184` verbatim and then runs the app's real `roleToPermissions` and `computeRoleChanges`.

```tsx
/**
 * PoC — ResolverRolesSidebar keys its editable permission state to the table
 * ROW INDEX, so the same index occupied by a different account later shows, and
 * then writes, the previous occupant's permission set.
 *
 * Uses the app's own roleToPermissions / computeRoleChanges and replays the
 * exact guard from ResolverRolesSidebar.tsx:179-184.
 */
import { it } from 'vitest'
import { computeRoleChanges, roleToPermissions } from '@/lib/roles/rolesToPermissions'
import type { ResolverRoleKey } from '@/lib/roles/resolverRoles'

type Row = { id: string; original: { account: string; decodedRoles: ResolverRoleKey[]; resolvedNames: string[] } }

// Two role holders on the same resolver. DataTable.tsx:38-47 calls
// useReactTable with no `getRowId`, so tanstack's default applies:
//   _getRowId: (row, index, parent) => options.getRowId?.(...) ?? `${index}`
const A = { account: '0xAAAAaaaaAAAAaaaaAAAAaaaaAAAAaaaaAAAAaaaa', decodedRoles: ['ROLE_SET_ADDR','ROLE_SET_NAME','ROLE_CLEAR'] as ResolverRoleKey[], resolvedNames: ['shop.eth'] }
const B = { account: '0xBBBBbbbbBBBBbbbbBBBBbbbbBBBBbbbbBBBBbbbb', decodedRoles: ['ROLE_SET_TEXT'] as ResolverRoleKey[], resolvedNames: ['shop.eth'] }

it('the sidebar writes one account\'s permission set onto another', () => {
  // --- the component's state, exactly as the file declares it ---
  let prevRowId: string | null = null
  let editedPermissions = new Map()
  const openRow = (row: Row) => {
    // ResolverRolesSidebar.tsx:179-184
    if (row && row.id !== prevRowId) {
      prevRowId = row.id
      editedPermissions = roleToPermissions(row.original.decodedRoles)
    }
  }

  // --- the admin opens row 0, which is A ---
  let row: Row = { id: '0', original: A }
  openRow(row)
  console.log('open row 0 ->', A.account)
  console.log('  seeded checkboxes:', [...editedPermissions.values()].filter((p: never) => (p as {enabled?: boolean}).enabled !== false).length, 'entries from', A.decodedRoles.join(', '))

  // --- the list re-orders (any role change invalidates ['resolver-overview'],
  //     and groupRolesByAccount re-keys a Map from the indexer's order).
  //     Index 0 is now B. row.id is still "0". ---
  row = { id: '0', original: B }
  openRow(row)

  console.log('\nopen row 0 again ->', B.account)
  console.log('  header / selectedAccount :', row.original.account)
  console.log('  scope line               : Roles scoped to', row.original.resolvedNames[0])
  console.log('  B\'s actual on-chain roles:', B.decodedRoles.join(', '))

  // ResolverRolesSidebar.tsx:190-193
  const { rolesToGrant, rolesToRevoke } = computeRoleChanges(
    row.original.decodedRoles as string[],
    editedPermissions,
  )
  console.log('\nSave ->')
  console.log('  rolesToGrant :', JSON.stringify(rolesToGrant))
  console.log('  rolesToRevoke:', JSON.stringify(rolesToRevoke))
  console.log('  written against account:', row.original.account)
})
```

Actual output:

```
open row 0 -> 0xAAAAaaaaAAAAaaaaAAAAaaaaAAAAaaaaAAAAaaaa
  seeded checkboxes: 3 entries from ROLE_SET_ADDR, ROLE_SET_NAME, ROLE_CLEAR

open row 0 again -> 0xBBBBbbbbBBBBbbbbBBBBbbbbBBBBbbbbBBBBbbbb
  header / selectedAccount : 0xBBBBbbbbBBBBbbbbBBBBbbbbBBBBbbbbBBBBbbbb
  scope line               : Roles scoped to shop.eth
  B's actual on-chain roles: ROLE_SET_TEXT

Save ->
  rolesToGrant : ["ROLE_SET_ADDR","ROLE_SET_NAME","ROLE_CLEAR"]
  rolesToRevoke: ["ROLE_SET_TEXT"]
  written against account: 0xBBBBbbbbBBBBbbbbBBBBbbbbBBBBbbbbBBBBbbbb

 ✓ .poc/roles-rowid.test.ts  1ms
```

Account B is granted the three roles that belong to account A, and loses the one role it actually had, from a sheet whose header, scope line and account field all correctly say B.

### Driven through the running app, in a real browser

The logic PoC above uses the app's helpers directly. This section drives the real `/resolver/$address/roles` page on the Explorer dev server, against the local fork, and never constructs a component or a piece of state by hand.

Setup (`.poc/setup-07.mjs`): a contract is placed at the resolver address whose runtime is `0x600160005260206000f3` — it answers every staticcall with `1`, so the page's on-chain `hasRoles` gate passes and the edit control renders. Two role holders are served by an indexer stand-in (`.poc/stub-07.mjs`) answering the Explorer's real `getResolverOverview` query:

| account       | roleBitmap   | decodes to                                     |
| ------------- | ------------ | ---------------------------------------------- |
| `0xAAAA…aaaa` | `4311744513` | `ROLE_SET_ADDR`, `ROLE_SET_NAME`, `ROLE_CLEAR` |
| `0xBBBB…bbbb` | `16`         | `ROLE_SET_TEXT`                                |

The stub's `/flip` endpoint swaps their order — the one thing the indexer does when the app refetches after a role change. The refetch itself is triggered with the app's own `invalidateQueries(['resolver-overview'])`, which is the exact call the sidebar makes at `ResolverRolesSidebar.tsx:240-248` and `:278-286` after every Save and every Remove.

```bash
node .poc/setup-07.mjs && node .poc/stub-07.mjs &
node .poc/drive-07.mjs
```

Actual output:

```
GOTO <http://127.0.0.1:9623/resolver/0x00000000000000000000000000000000000f0e51/roles>

[roles table as rendered]
  | Roles
  | Add user
  | User	Name	Role
  | Address  0xaaaa…aaaa   shop.eth   Set Address  Set Name  Clear

edit controls found: 2

--- 1. open the sheet at ROW 0 (account A) ---

[step1-rowzero-is-A]
  account shown in sheet : 0xaaaa…aaaa
  scope line             : shop.eth
  checkboxes CHECKED     : ["ROLE_SET_ADDR-manager","ROLE_SET_NAME-manager","ROLE_CLEAR-manager"]
  Save enabled           : false

--- 2. close the sheet without changing anything ---

--- 3. the indexer now returns the two holders in the other order ---
    flipped: role list order is now [B, A]

--- 4. the app refetches, exactly as it does after its own Save/Remove ---
    (ResolverRolesSidebar.tsx:240-248 -> invalidateQueries(['resolver-overview']))
    invalidateQueries dispatched: true

[roles table after the refetch]
  | Roles
  | Add user
  | User	Name	Role
  | Address  0xbbbb…bbbb   shop.eth   Set Text

--- 5. open the sheet at ROW 0 again — it is account B now ---

[step5-rowzero-is-B]
  account shown in sheet : 0xbbbb…bbbb
  scope line             : shop.eth
  checkboxes CHECKED     : ["ROLE_SET_ADDR-manager","ROLE_SET_NAME-manager","ROLE_CLEAR-manager"]
  Save enabled           : true
```

Read the two sheet states together. The table correctly shows that row 0 is now `0xbbbb…bbbb`, whose only role is `Set Text`. The sheet header agrees — it shows `0xbbbb…bbbb`. The checkbox grid does not: it still carries `0xaaaa…aaaa`'s three permissions, because `row.id` was `"0"` both times and the guard at `:179-184` never re-seeded.

`Save enabled: true` is the part that matters. The operator has toggled nothing, and the button is already live — because `hasChanges` is comparing B's real roles against A's stale checkbox map. One click grants `ROLE_SET_ADDR`, `ROLE_SET_NAME` and `ROLE_CLEAR` to B and revokes the `ROLE_SET_TEXT` B actually had.

Screenshots: `07-step1-rowzero-is-A.png`, `07-step5-rowzero-is-B.png`.

### The row-id claim, from the installed library

```
$ sed -n '374,378p' node_modules/.pnpm/@tanstack+table-core@8.21.3/node_modules/@tanstack/table-core/src/core/table.ts
    _getRowId: (row: TData, index: number, parent?: Row<TData>) =>
      table.options.getRowId?.(row, index, parent) ??
      `${parent ? [parent.id, index].join('.') : index}`,
```

`apps/portal/src/components/DataTable.tsx:38-47` passes no `getRowId`, so the `??` branch is taken.

***
