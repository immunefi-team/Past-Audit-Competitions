# 90931 w a critical resolver row index reuse carries a removed user s root upgrade permission into another collaborator and enables shared resolver takeover

**Submitted on Aug 28th 2026 at 08:04:17 UTC by @sl1de for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90931
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

Portal's resolver role editor binds its mutable checkbox state to a TanStack table row's default numeric index instead of to the resolver, resource, and account being edited. If the resolver administrator removes one collaborator, the next collaborator shifts into the removed row's index. When that shifted row is opened, Portal can retain the removed collaborator's checked permissions while changing the target account to the shifted collaborator.

I reproduced a concrete Critical consequence with two genuine root-scoped rows:

* A trusted collaborator occupies row `0` and has root `Set Text | Upgrade`.
* A low-privilege collaborator occupies row `1` and has only root `Set Text`.
* The administrator removes the trusted collaborator through Portal's normal connected-wallet workflow.
* Authentic Panoptes omits the zero-bitmap row, so the low-privilege collaborator shifts from row `1` to row `0`.
* Portal's mounted sidebar retains `prevRowId = "0"` and the removed collaborator's permission state.
* When the administrator opens the shifted collaborator, the table correctly says only `Set Text`, but the editor also shows stale `Upgrade` as checked.
* The administrator deliberately changes only `Set ABI` from unchecked to checked and saves.
* Portal compares the shifted collaborator's real roles against the stale editor state and signs `grantRootRoles(ROLE_SET_ABI | ROLE_UPGRADE, collaborator)`.

The stale `Upgrade` checkbox is visibly checked in the editor. The victim can notice it and stop. However, the final Portal transaction modal only says `Update roles for <collaborator>` and enumerates neither `Set ABI` nor `Upgrade`. A wallet that independently decodes and explains the bitmap can also expose the extra permission.

The proof then exercises the newly granted authority. Before the Portal transaction, the collaborator's resolver upgrade attempt is rejected. After approval, the collaborator successfully replaces the shared PermissionedResolver implementation. Two names owned by different accounts and using that resolver subsequently resolve through ENSjs to the collaborator address.

All state changes were executed on a private disposable Sepolia fork. No public-chain exploitation is claimed.

## Vulnerability Details

### Actors and setup

There are three relevant accounts:

* The victim is a legitimate root role administrator for a shared PermissionedResolver and uses Portal's role-management interface.
* A trusted collaborator has root `ROLE_SET_TEXT | ROLE_UPGRADE` and is intentionally removed by the victim.
* The attacker-controlled collaborator has only benign root `ROLE_SET_TEXT` and appears immediately after the trusted collaborator in the indexed table order.

The shared resolver serves at least two names owned by different accounts. Before the vulnerable workflow begins, the attacker-controlled collaborator has no root `ROLE_SET_ABI`, no root `ROLE_UPGRADE`, and cannot upgrade the resolver.

The expected invariant is:

```
selected resolver + selected resource + selected account
    == editor permission source
    == diff baseline
    == confirmed role delta
    == signed role delta
```

The vulnerable sequence changes the selected account but keeps the previous row's editor permissions:

```
Initial indexed rows
  row.id "0" -> trusted account -> Set Text | Upgrade
  row.id "1" -> attacker       -> Set Text
                         |
                         v
Victim removes trusted row and approves the wallet transaction
                         |
                         v
Panoptes omits trusted zero-bitmap role
  row.id "0" -> attacker       -> Set Text
                         |
                         v
Sidebar still stores
  prevRowId = "0"
  editedPermissions = Set Text | Upgrade
                         |
                         v
Victim opens attacker row and changes only Set ABI
                         |
                         v
Portal diffs attacker actual Set Text against
stale Set Text | Set ABI | Upgrade
                         |
                         v
Victim wallet signs
  grantRootRoles(Set ABI | Upgrade, attacker)
                         |
                         v
Attacker upgrades the shared resolver implementation
                         |
                         v
Names owned by separate accounts resolve to attacker
```

### Root cause 1: the generic table uses positional row IDs

`ResolverRolesTable` groups roles into account rows, stores the selected TanStack `Row`, and passes the resulting data to the generic `DataTable`:

```tsx
const [editingRow, setEditingRow] = useState<Row<AccountRoleGroup> | null>(
  null,
)

const data = useMemo(
  () => groupRolesByAccount(roles, buildResourceToNameMap(nodes)),
  [roles, nodes],
)

// ...

buildEditActionColumn<AccountRoleGroup>((row) => {
  setEditingRow(row)
  setOpen(true)
})

// ...

<DataTable columns={columns} data={data} />
```

`DataTable` calls `useReactTable` without supplying `getRowId`:

```tsx
const table = useReactTable({
  data,
  columns,
  getCoreRowModel: getCoreRowModel(),
  onSortingChange: setSorting,
  getSortedRowModel: getSortedRowModel(),
  state: { sorting },
})
```

TanStack therefore assigns the current positional index as `row.id`. In the verified sequence, the trusted account has ID `"0"` and the attacker-controlled collaborator has ID `"1"`. Once the trusted account disappears, the collaborator receives ID `"0"`.

The role grouping function preserves the first-seen account order from the indexed role list. The proof creates distinct role-event timestamps and waits for authentic Panoptes to return exactly `[trusted, attacker]`, making the shift deterministic rather than assuming a convenient ordering.

### Root cause 2: sidebar state is synchronized only by the positional ID

The sidebar derives the currently selected account and its actual decoded roles from `row.original`:

```tsx
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
```

However, its mutable checkbox state is reset only when `row.id` changes:

```tsx
const [editedPermissions, setEditedPermissions] = useState<
  Map<string, Permission>
>(new Map())

const [prevRowId, setPrevRowId] = useState<string | null>(null)
if (row && row.id !== prevRowId) {
  setPrevRowId(row.id)
  setEditedPermissions(roleToPermissions(row.original.decodedRoles))
}
```

Opening the trusted row stores:

```
prevRowId          = "0"
editedPermissions  = Set Text | Upgrade
```

After removal, the sidebar component stays mounted. Its transaction `onDone` closes the modal and sheet, but does not clear `prevRowId`, `editedPermissions`, or `editingRow`:

```tsx
onDone: () => {
  closeModal()
  clearTransaction()
  setPendingAction(null)
  setOpen(false)
}
```

When the attacker-controlled collaborator shifts into row `"0"` and the victim opens that row, `row.original.account` changes but `row.id !== prevRowId` is false. The permission reset is skipped.

The resulting editor object combines:

```
selectedAccount      = attacker
decodedRoles         = Set Text
editedPermissions    = Set Text | Upgrade  (retained from trusted account)
```

### Root cause 3: the normal removal workflow creates and retains the vulnerable state

The sidebar's Remove user action snapshots the trusted account and its roles, then submits them through the production resolver-role helper:

```tsx
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
openModal()
```

After the removal succeeds, Portal polls for indexer synchronization and invalidates all `resolver-overview` queries:

```tsx
onSuccess: async () => {
  await pollForIndexerSync({
    invalidateQueries: () =>
      queryClient.invalidateQueries({
        queryKey: ['resolver-overview'],
        refetchType: 'all',
      }),
  })
}
```

Authentic Panoptes no longer returns the trusted account once its complete role bitmap is zero. The table rerenders with the attacker-controlled collaborator as its only row, but the enclosing `ResolverRolesSidebar` remains the same mounted component and retains the previous editor state.

This is not a forged-indexer or stale-indexer assumption. Panoptes returns the correct initial two-row order and the correct one-row post-removal state. The application mishandles the legitimate change in row position.

### Root cause 4: diff computation compares the new account against the old account's edited state

The sidebar recomputes `originalPermissions` from the currently selected row, but computes the delta against the retained `editedPermissions` map:

```tsx
const originalPermissions = useMemo(
  () => roleToPermissions(decodedRoles),
  [decodedRoles],
)

const { rolesToGrant, rolesToRevoke } = computeRoleChanges(
  decodedRoles as string[],
  editedPermissions,
)
```

`computeRoleChanges` iterates over the union of the current row's actual roles and the editor map. Any permission that is true in the retained editor but false for the shifted collaborator becomes a grant:

```ts
const allKeys = new Set([
  ...originalPermissions.keys(),
  ...editedPermissions.keys(),
])

// ...

if (edited.manager && !original.manager) {
  rolesToGrant.push(key as ResolverRole)
}
```

In the verified flow:

```
attacker original permissions = Set Text
retained editor permissions   = Set Text | Upgrade
victim's deliberate change    = Set ABI false -> true

computed rolesToGrant         = Set ABI | Upgrade
computed rolesToRevoke        = []
```

The victim never clicks the Upgrade checkbox in the attacker editor. It is already visibly checked due to the state bleed.

### Root cause 5: the pending action and transaction modal bind the stale delta to the new account

When Save is activated, the sidebar snapshots the newly selected account together with the stale-derived role delta:

```tsx
setPendingAction({
  type: 'save',
  name: roleName,
  account: selectedAccount,
  rolesToGrant: rolesToGrant,
  rolesToRevoke: rolesToRevoke,
})
openModal()
```

Because both source and target rows are genuine root rows, `roleName` is the application's normal empty-string representation of root scope. The grant helper correctly translates that root scope to the root ABI operation. The incorrect field is the role bitmap, not the target account or resource scope.

The prepared intent receives the stale-derived `rolesToGrant` and passes it to `prepareGrantResolverRolesTransaction`. That helper uses ENSjs to encode the root grant and supplies it to the production transaction manager:

```tsx
prepareGrantResolverRolesTransaction({
  resolverAddress,
  name,
  account,
  roles: rolesToGrant,
  walletClient,
  chainId,
})
```

Portal's visible transaction metadata identifies only the account:

```tsx
{
  title: 'Save resolver role changes',
  transactionName: `Update roles for ${pendingAction?.account ?? ''}`,
}
```

The production transaction state renders that `transactionName`. It does not enumerate the grant bitmap or permission names. In the captured run, the modal contained neither `Upgrade` nor `Set ABI`.

### Root cause 6: the stale permission becomes usable root implementation authority

Portal defines:

```
ROLE_SET_TEXT = 1 << 4
ROLE_SET_ABI  = 1 << 16
ROLE_UPGRADE  = 1 << 124
```

and describes `ROLE_UPGRADE` as permission to upgrade the resolver contract.

The exact browser-signed transaction decoded as:

```
grantRootRoles(
  roleBitmap = 21267647932558653966460912964485578752,
  account    = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
)
```

The signed bitmap is:

```
21267647932558653966460912964485578752
  = (1 << 124) | (1 << 16)
  = ROLE_UPGRADE | ROLE_SET_ABI
```

Exact mined input from the captured private-fork run:

```
0x072d5d77000000000000000000000000000000001000000000000000000000000001000000000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
```

Before this grant, the collaborator's complete root bitmap was exactly `16` (`ROLE_SET_TEXT`) and a direct `upgradeToAndCall` simulation was rejected. After the grant, the complete bitmap was exactly `ROLE_SET_TEXT | ROLE_SET_ABI | ROLE_UPGRADE`, and the collaborator's `upgradeToAndCall` transaction succeeded.

The implementation slot changed while the resolver proxy bytecode stayed unchanged. A deterministic proof implementation then caused two names owned by different accounts to return the collaborator address through ENSjs resolution.

### Full production call and state path

```
Panoptes resolver.roles
  [trusted: ROOT SetText|Upgrade, attacker: ROOT SetText]
        |
        v
useResolverOverview()
        |
        v
groupRolesByAccount() preserves indexed first-seen order
        |
        v
ResolverRolesTable -> DataTable
        |
        v
useReactTable() without getRowId
  trusted.id = "0", attacker.id = "1"
        |
        v
victim edits trusted row
        |
        v
ResolverRolesSidebar stores
  prevRowId = "0"
  editedPermissions = SetText|Upgrade
        |
        v
victim removes trusted user and approves wallet
        |
        v
pollForIndexerSync + resolver-overview refetch
        |
        v
Panoptes omits trusted zero-bitmap role
  attacker shifts to id = "0"
        |
        v
victim opens attacker row
  selectedAccount = attacker
  row.id = "0" == prevRowId
  permission reset skipped
        |
        v
victim changes only SetABI false -> true
        |
        v
computeRoleChanges(
  attacker actual SetText,
  stale edited SetText|Upgrade|SetABI
)
        |
        v
pendingAction = {
  account: attacker,
  rolesToGrant: [SetABI, Upgrade]
}
        |
        v
prepareGrantResolverRolesTransaction()
        |
        v
Portal modal says only "Update roles for <attacker>"
        |
        v
connected wallet signs grantRootRoles(SetABI|Upgrade, attacker)
        |
        v
attacker calls upgradeToAndCall()
        |
        v
shared resolver implementation changes
        |
        v
two separately owned names resolve to attacker
```

## Verified Result

I reproduced the complete component-to-browser-to-chain consequence using the frozen competition source, authentic local Panoptes, the production Portal role route and components, the production transaction modal and transaction manager, a connected throwaway wallet, a real fork-deployed PermissionedResolver proxy, and ENSjs resolution after exploitation.

| Item                                         | Verified result                                                              |
| -------------------------------------------- | ---------------------------------------------------------------------------- |
| Competition commit                           | `cda79acaad59711b943fc68207ebb3f1d0ff8596`                                   |
| Imported upstream commit                     | `63772fd872af472ced58b009499355f3430c2a86`                                   |
| Environment                                  | Private disposable Sepolia fork at block `11577942`                          |
| Node / pnpm                                  | `v22.23.2` / `10.27.0`                                                       |
| Shared resolver                              | `0xABEfD53a03de274dD1b26dEE9D9a7bfec2cF91c4`                                 |
| Authentic initial account order              | `[trusted, attacker]`                                                        |
| Authentic post-removal order                 | `[attacker]`                                                                 |
| Trusted row index                            | `0`                                                                          |
| Attacker row index                           | `1 -> 0`                                                                     |
| Trusted initial root roles                   | `ROLE_SET_TEXT` and `ROLE_UPGRADE`                                           |
| Attacker initial complete root bitmap        | `16` (`ROLE_SET_TEXT`)                                                       |
| Attacker initial `ROLE_SET_ABI`              | `false`                                                                      |
| Attacker initial `ROLE_UPGRADE`              | `false`                                                                      |
| Attacker upgrade before Portal grant         | Rejected                                                                     |
| Stale Upgrade in attacker editor             | Visible and checked                                                          |
| Only deliberate permission change            | `ROLE_SET_ABI false -> true`                                                 |
| Modal text                                   | `Update roles for 0x70997970c51812dc3a010c7d01b50e0d17dc79c8`                |
| Modal lists Upgrade                          | `false`                                                                      |
| Modal lists Set ABI                          | `false`                                                                      |
| Browser-signed function                      | `grantRootRoles`                                                             |
| Browser-signed bitmap                        | `21267647932558653966460912964485578752` (`ROLE_SET_ABI` and `ROLE_UPGRADE`) |
| Browser-signed account                       | Attacker-controlled collaborator                                             |
| Browser root-grant transaction               | `0x14537e2b063ee98d495fba1dd40722b67751fa8d1886ad8659de5432c6e21b98`         |
| Attacker root `ROLE_SET_ABI` after           | `true`                                                                       |
| Attacker root `ROLE_UPGRADE` after           | `true`                                                                       |
| Resolver implementation before               | `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`                                 |
| Resolver implementation after                | `0x28A15edC0DaAf81eb684C2B59EdC2680c1Aed971`                                 |
| Resolver upgrade transaction                 | `0x2f3a5dab53608a1d8e087b889e077152b5a2cd173e194f012a4ee1891a603b5f`         |
| Proxy bytecode unchanged                     | `true`                                                                       |
| First affected name owner                    | Resolver administrator                                                       |
| Second affected name owner                   | A separate victim account                                                    |
| Both names resolve through ENSjs to attacker | `true`                                                                       |
| Production component proof                   | `1 passed`                                                                   |
| Production browser proof                     | `1 passed (53.5s)`                                                           |
| Joined result                                | `PASS`                                                                       |
| Cleanup                                      | `servicesOwnedByRunnerStopped=true`                                          |

The transaction hashes and generated names are private-fork evidence and will differ on rerun. The row-index transition, role bitmap, and authority consequence are deterministic under the asserted role order.

### Negative controls

* Both original role rows use the genuine non-ambiguous root resource, resource `0`.
* Authentic Panoptes returns the trusted collaborator first and the attacker-controlled collaborator second before removal.
* Authentic Panoptes returns only the attacker-controlled collaborator after removal.
* The trusted role's timestamp is greater than the attacker role's timestamp, and the proof asserts all three row positions.
* The attacker-controlled collaborator's complete root bitmap is exactly `ROLE_SET_TEXT` before the vulnerable transaction.
* The collaborator has neither `ROLE_SET_ABI` nor `ROLE_UPGRADE` before the vulnerable transaction.
* A direct collaborator resolver upgrade is rejected before the vulnerable transaction.
* The role table correctly shows only `Set Text` on the collaborator before and after the row shift.
* The victim changes only `Set ABI` in the attacker editor.
* The exact mined browser transaction decodes to root `ROLE_SET_ABI | ROLE_UPGRADE` for the intended collaborator account.
* The implementation slot changes only after the unintended root Upgrade grant.
* The proxy address and proxy bytecode remain unchanged.
* The two affected names have different owners.
* No RPC response, Panoptes response, transaction payload, or wallet result is mocked in the browser-to-chain proof.
* No public network receives a write.

## Preconditions and Boundaries

The following preconditions are explicit:

* The victim is a legitimate root role administrator for the PermissionedResolver.
* The resolver has at least two collaborator rows.
* The attacker-controlled collaborator follows a row that the victim removes in the current table order. The proof makes this deterministic by separating role-event timestamps and verifying the authentic indexed order before starting the UI sequence.
* The removed row has a dangerous checked permission that the attacker-controlled collaborator lacks.
* The victim completes two deliberate, normal Portal workflows without navigating away or otherwise remounting the table: first removing the trusted collaborator, then editing the shifted collaborator.
* In the second workflow, the victim changes an ordinary permission and approves the connected-wallet transaction.

The following boundaries are also explicit:

* The stale `Upgrade` checkbox is visibly checked in the shifted collaborator's editor. The victim can notice the unexpected state and abort.
* Portal's final transaction modal names only the collaborator account and does not enumerate `Upgrade` or `Set ABI`.
* A wallet that independently decodes the function and bitmap and explains the permissions can expose the extra `Upgrade` bit; the victim can reject the request.
* The attacker cannot force the victim to remove the first collaborator, edit the second collaborator, or approve either transaction.
* The account and root resource are not substituted. Portal grants the stale permission to the account the victim is editing. The substituted value is the role bitmap.
* The verified consequence is resolver implementation control and cross-name resolution control. It does not claim ENS name/NFT ownership transfer.
* The proof does not claim stolen keys, wallet/session compromise, malicious RPC behavior, forged indexer data, backend compromise, public-chain transactions, or completed public-network asset theft.
* The proof implementation is deployed only on the private fork to demonstrate the authority conferred by `ROLE_UPGRADE`.

## Impact Details

This maps directly to the Critical malicious connected-wallet interaction impact. The victim deliberately adds ordinary `Set ABI` permission to the selected collaborator. Portal also includes the removed collaborator's root `Upgrade` permission in the same grant bitmap, presenting it in the editor as if it already belonged to the selected account; the connected wallet then signs that authorization-bearing modification.

The impact is not limited to a stale checkbox or inaccurate preview. The exact browser-signed call succeeds and creates new root authority:

```
before: attacker root roles = Set Text
signed: grantRootRoles(Set ABI | Upgrade, attacker)
after:  attacker root roles = Set Text | Set ABI | Upgrade
```

`ROLE_UPGRADE` is implementation authority over the shared PermissionedResolver. The proof shows the attacker-controlled collaborator using it to replace the implementation after an identical upgrade attempt was rejected before the Portal grant. The proxy remains at the same address, so every name still pointing to it is affected.

The completed consequence spans multiple owners. Two names owned by different accounts use the shared resolver. After the implementation replacement, ENSjs resolves both names to the attacker address. Thus, a role edit aimed at one known collaborator can grant that account control over resolution for unrelated users of the shared resolver.

From the attacker's perspective, an existing benign root collaborator row and favorable adjacency to a removed row are sufficient. The attacker begins without resolver upgrade authority. One later victim-approved edit of that row can carry the removed account's dangerous role into the attacker's bitmap.

From the administrator's perspective, the table correctly identifies the second collaborator and shows only its real `Set Text` role. The sidebar targets that same account, but its checkboxes originate from the removed account. The victim's only deliberate checkbox change is `Set ABI`; the final Portal modal does not enumerate the resulting role delta.

From other name owners' perspective, no interaction is required after they use the shared resolver. Once the attacker receives root Upgrade and replaces the implementation, their names can resolve according to attacker-controlled code.

## Recommended Remediation

{% stepper %}
{% step %}

## Use immutable row identities

Give every resolver role row an immutable security identity, for example:

```ts
{
  resolverAddress,
  resource,
  account,
}
```

Supply a stable `getRowId` derived from that identity instead of using the positional index.
{% endstep %}

{% step %}

## Reset editor state on identity changes

Reset or remount the editor whenever any part of the selected security identity changes. Do not use a table index to determine whether permission state belongs to the selected account.
{% endstep %}

{% step %}

## Clear state when a selected row disappears

When a selected row disappears from fresh query data, immediately close the editor and clear `editingRow`, `prevRowId`, `editedPermissions`, and pending transaction state.
{% endstep %}

{% step %}

## Store immutable selection identity

Store only the immutable identity in selection state. Resolve the current row from fresh table data rather than retaining a TanStack `Row` object across refetches.
{% endstep %}

{% step %}

## Recompute from fresh authoritative roles

Immediately before preparing calldata, re-read the target account's authoritative role bitmap and recompute the delta from that fresh value. Abort and require a new review if it differs from the baseline the victim edited.
{% endstep %}

{% step %}

## Bind review and transaction state immutably

Bind the selected account, resource, original bitmap, edited bitmap, role delta, modal text, prepared intent, and submitted intent to one immutable review object.
{% endstep %}

{% step %}

## Show explicit role deltas in final review

Enumerate every role grant and revoke in the final Portal transaction review. Dangerous permissions such as `Upgrade` should require explicit role-level confirmation and should never be represented only as an opaque bitmap behind `Update roles for <account>`.
{% endstep %}

{% step %}

## Add regression coverage

Add regression coverage for:

* Removing row `0` while row `1` remains.
* Opening the shifted row without remounting the table.
* Changes in sorting and filtering that reuse an index.
* Removal followed by query invalidation and refetch.
* Ensuring no permission from the removed row appears in the shifted row's intent.
* Checking the exact prepared and mined bitmap, not only visible checkbox state.
  {% endstep %}
  {% endstepper %}

## References

* [The resolver role route fetches indexed roles and exposes editing only after its connected-account management check](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/resolver/$address/roles/index.tsx#L30-L91)
* [The resolver overview query returns role account, resource, bitmap, block, transaction, and timestamp fields](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L60-L118)
* [Role grouping preserves first-seen account insertion order](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220)
* [ResolverRolesTable retains the selected TanStack Row and passes grouped data to the generic DataTable](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L90-L138)
* [DataTable calls useReactTable without a stable getRowId](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/components/DataTable.tsx#L33-L47)
* [The sidebar derives the new account but resets edited permissions only when row.id changes](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L193)
* [The removal success path refetches resolver-overview without resetting editor identity or permission state](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L293)
* [The removal workflow snapshots the selected account and roles](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323)
* [The transaction Done handler closes the sheet but preserves the stale editor state](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L467-L508)
* [The role-diff helper grants every permission present in edited state but absent from the current row](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/rolesToPermissions.ts#L66-L105)
* [Save binds the computed role delta to the currently selected account](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L295-L305)
* [The Portal transaction metadata identifies only the account](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L338-L349)
* [The production transaction state displays transactionName](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transaction-manager/components/TransactionStateContent.tsx#L87-L114)
* [The pending role delta is forwarded into the grant transaction preparation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts#L32-L76)
* [The grant helper encodes root grants and passes them to the production transaction manager](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L28-L118)
* [Portal defines ROLE\_SET\_ABI at bit 16 and ROLE\_UPGRADE at bit 124](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L24-L45)
* [The permission catalogue describes Upgrade as resolver-contract upgrade authority](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L57-L108)

## Proof of Concept

### Safety and environment

This proof runs entirely against a private disposable Anvil fork using standard throwaway development accounts. It does not send transactions to public Sepolia, Ethereum mainnet, or any other public network. It does not use production names, funds, wallets, sessions, credentials, or services.

The required Sepolia endpoint is used only by local Anvil to read the initial state at block `11577942`. Every name registration, resolver attachment, role grant, role removal, wallet approval, implementation deployment, resolver upgrade, and resolution assertion is mined inside the local fork.

The runner binds its services only to loopback:

```
Anvil:             127.0.0.1:30545
Panoptes GraphQL:  127.0.0.1:30655
Portal:            127.0.0.1:30701
```

It requires an explicit fork endpoint, verifies the frozen commit and tool versions, uses digest-pinned Anvil and Panoptes images, refuses occupied ports or pre-existing Compose resources, takes an atomic local run lock, and removes only resources belonging to its dedicated project. The final cleanup marker is printed only after its listeners, containers, network, and volume are gone.

### PoC evidence bundle

Download the complete runnable overlay and captured evidence here:

[Download ens-portal-resolver-row-state-bleed-poc.zip](https://drive.google.com/file/d/15BBS5uMYZfirOEKnsr68PqPGeiTsuIZY/view?usp=sharing)

Save it with this exact filename:

```
ens-portal-resolver-row-state-bleed-poc.zip
```

ZIP SHA-256:

```
fbdaa3682f32074451a71bef1587d445231192f9a28d46665e15869d0e0d9619  ens-portal-resolver-row-state-bleed-poc.zip
```

The archive contains the fail-closed joined runner, digest-pinned Compose and Panoptes configurations, the production component regression, the production Playwright browser proof, the minimal private-fork consequence implementation, the complete clean log, structured browser evidence, local service logs, and two production Portal screenshots. `POC-RESOLVER-ROW-STATE-BLEED-SHA256SUMS.txt` authenticates every included file except the manifest itself.

### Prerequisites

* Linux with Bash.
* Node.js v22. The captured verification used v22.23.2.
* Corepack and pnpm v10. The captured verification used pnpm 10.27.0.
* Docker with Docker Compose v2 and permission to start containers.
* Foundry `forge` and `cast`.
* Chromium at `/usr/bin/chromium`, or another executable supplied through `CHROMIUM_PATH`.
* Git, curl, jq, flock, grep, nice, setsid, ss, and timeout.
* A researcher-controlled Sepolia RPC URL with historical read and fork support for block `11577942`, reachable from the Docker container.
* Local ports `30545`, `30655`, and `30701` available. Alternate valid ports may be supplied through the documented environment variables.
* Network access for the initial clone, frozen dependency installation, pinned image download, and read-only fork initialization.

{% stepper %}
{% step %}

## Obtain the frozen competition repository

```bash
mkdir ens-portal-row-state-reproduction
cd ens-portal-row-state-reproduction

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git \
  audit-comp-ens

cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596

test "$(git rev-parse HEAD)" = \
  "cda79acaad59711b943fc68207ebb3f1d0ff8596"

git show -s --format='%H%n%B' HEAD
```

The commit message identifies imported upstream commit:

```
63772fd872af472ced58b009499355f3430c2a86
```

{% endstep %}

{% step %}

## Install frozen dependencies

```bash
node --version
corepack pnpm --version
corepack pnpm install --frozen-lockfile
```

Expected major versions:

```
Node.js: v22.x
pnpm:    10.x
```

{% endstep %}

{% step %}

## Download, verify, and overlay the PoC

Place `ens-portal-resolver-row-state-bleed-poc.zip` inside the cloned repository root. Verify the outer checksum before extraction:

```bash
printf '%s  %s\n' \
  'fbdaa3682f32074451a71bef1587d445231192f9a28d46665e15869d0e0d9619' \
  'ens-portal-resolver-row-state-bleed-poc.zip' | sha256sum -c -
```

Expected output:

```
ens-portal-resolver-row-state-bleed-poc.zip: OK
```

Extract the overlay, restore executable bits, and verify every internal file:

```bash
unzip ens-portal-resolver-row-state-bleed-poc.zip

chmod +x \
  poc/resolver-row-state-bleed/run-clean.sh \
  poc/resolver-row-state-bleed/test-runner-preflight.sh

sha256sum -c POC-RESOLVER-ROW-STATE-BLEED-SHA256SUMS.txt
```

Every path in the internal manifest must print `OK`.

Relevant bundle layout:

```
audit-comp-ens/
|-- POC-RESOLVER-ROW-STATE-BLEED-SHA256SUMS.txt
|-- apps/portal/poc/
|   |-- resolver-row-index-state-bleed.test.tsx
|   `-- resolver-row-state-bleed.vite.config.ts
|-- e2e/poc/
|   `-- portal-resolver-row-state-bleed.playwright.config.ts
|-- e2e/projects/portal/tests/
|   `-- portal-resolver-row-state-bleed.poc.spec.ts
|-- poc/resolver-row-state-bleed/
|   |-- README.md
|   |-- docker-compose.yml
|   |-- foundry.toml
|   |-- panoptes.json
|   |-- run-clean.sh
|   |-- test-runner-preflight.sh
|   |-- src/AttackerResolverImplementation.sol
|   `-- out/AttackerResolverImplementation.sol/
|       `-- AttackerResolverImplementation.json
`-- scans/
    |-- resolver-row-state-bleed-clean.log
    |-- portal-resolver-row-state-bleed-browser.json
    |-- portal-resolver-row-state-bleed-editor.png
    |-- portal-resolver-row-state-bleed-modal.png
    |-- resolver-row-state-bleed-indexer.log
    |-- resolver-row-state-bleed-api.log
    `-- resolver-row-state-bleed-vite.log
```

{% endstep %}

{% step %}

## Run the fail-closed preflight

```bash
poc/resolver-row-state-bleed/test-runner-preflight.sh
```

Expected output:

```
runnerPreflightTests=PASS
```

The preflight checks shell syntax, executable bits, the fixed competition commit, pinned images, fixed chain and block, loopback bindings, proxy configuration, endpoint rejection, and static browser-test safety properties.
{% endstep %}

{% step %}

## Supply the private-fork endpoint

Supply a researcher-controlled endpoint without placing it in source files or the report:

```bash
read -rsp 'Researcher-controlled Sepolia RPC URL: ' \
  ROW_STATE_BLEED_SEPOLIA_FORK_URL
export ROW_STATE_BLEED_SEPOLIA_FORK_URL
printf '\n'
```

The runner exits before Docker starts if this variable is absent.

If Chromium is not at `/usr/bin/chromium`, also set:

```bash
export CHROMIUM_PATH='/absolute/path/to/chromium'
```

{% endstep %}

{% step %}

## Execute the joined proof

From the frozen repository root:

```bash
poc/resolver-row-state-bleed/run-clean.sh
```

The runner writes the complete terminal record to:

```
scans/resolver-row-state-bleed-clean.log
```

It performs this sequence:

1. Verifies Node v22, pnpm v10, the exact competition commit, required commands, Chromium, the run lock, free loopback ports, and absence of pre-existing project resources.
2. Compiles the minimal private-fork consequence implementation.
3. Starts digest-pinned Anvil at Sepolia block `11577942` and verifies the expected local chain.
4. Starts digest-pinned Panoptes against the private fork.
5. Runs the production component proof for row removal, row shift, retained editor state, and exact root-grant intent.
6. Starts the frozen Portal app with its RPC and indexer access pointed at the isolated local services.
7. Runs one production Playwright test with one worker and a connected throwaway wallet.
8. Creates two differently owned names, attaches one shared PermissionedResolver, and establishes the trusted and low-privilege collaborator rows.
9. Waits for authentic Panoptes to return the deterministic `[trusted, attacker]` order.
10. Removes the trusted collaborator through the normal Portal modal and wallet flow.
11. Waits for authentic Panoptes and Portal to show the attacker-controlled collaborator shifted from index `1` to index `0`.
12. Opens the shifted collaborator, captures the visibly checked stale `Upgrade`, changes only `Set ABI`, and starts Portal's transaction flow.
13. Captures the final modal, approves through the connected wallet, locates the exact emitted role event, and decodes the mined transaction.
14. Proves the newly authorized collaborator can replace the shared resolver implementation and that two differently owned names resolve to the collaborator.
15. Removes only the PoC's local services and verifies cleanup.
    {% endstep %}

{% step %}

## Observe deterministic indexed row movement

The structured evidence must show:

```
initialRowOrder       = [trusted, attacker]
rowOrderAfterRemoval = [attacker]

trustedRowIndexBefore  = 0
attackerRowIndexBefore = 1
attackerRowIndexAfter  = 0
```

The initial Panoptes rows must both have resource zero. Their initial bitmaps must be:

```
trusted = ROLE_SET_TEXT | ROLE_UPGRADE
attacker = ROLE_SET_TEXT
```

The post-removal Panoptes response must contain only the attacker-controlled collaborator. The runner aborts if any of these order, resource, bitmap, timestamp, or account assertions fail.
{% endstep %}

{% step %}

## Observe the UI state bleed and honest visual boundary

After the row shift, the collaborator's table row must still show `Set Text` and must not show `Upgrade`.

When that row is opened, the editor must show:

```
Set Text = checked
Set ABI  = unchecked
Upgrade  = checked  (stale state from removed row)
```

The production screenshot is:

```
scans/portal-resolver-row-state-bleed-editor.png
```

The proof then clicks only `Set ABI`, leaves the already-checked Upgrade checkbox untouched, and activates Save.

The final Portal transaction modal must say:

```
Save resolver role changes
Update roles for 0x70997970c51812dc3a010c7d01b50e0d17dc79c8
```

It must contain neither the permission name `Upgrade` nor `Set ABI`. The captured modal is:

```
scans/portal-resolver-row-state-bleed-modal.png
```

{% endstep %}

{% step %}

## Observe the exact connected-wallet transaction

Before the Portal grant, the assertions are:

```
attacker complete root bitmap = 16 (ROLE_SET_TEXT)
attacker root Set ABI          = false
attacker root Upgrade          = false
attacker upgrade attempt       = rejected
```

The exact browser transaction must decode as:

```
grantRootRoles(
  21267647932558653966460912964485578752,
  0x70997970C51812dc3A010C7d01b50e0d17dc79C8
)
```

The proof verifies that the signed bitmap equals exactly:

```
ROLE_SET_ABI | ROLE_UPGRADE
```

After inclusion, it verifies that the collaborator's complete root bitmap is exactly:

```
ROLE_SET_TEXT | ROLE_SET_ABI | ROLE_UPGRADE
```

The included private-fork transaction is:

```
0x14537e2b063ee98d495fba1dd40722b67751fa8d1886ad8659de5432c6e21b98
```

Transaction hashes will differ on rerun.
{% endstep %}

{% step %}

## Observe the resolver implementation takeover

The collaborator then calls `upgradeToAndCall` with the locally deployed proof implementation.

The captured run verifies:

```
implementation before = 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
implementation after  = 0x28A15edC0DaAf81eb684C2B59EdC2680c1Aed971
implementation changed = true
proxy bytecode unchanged = true
```

The included private-fork upgrade transaction is:

```
0x2f3a5dab53608a1d8e087b889e077152b5a2cd173e194f012a4ee1891a603b5f
```

Two names owned by distinct accounts must then resolve through ENSjs to:

```
0x70997970C51812dc3A010C7d01b50e0d17dc79C8
```

The structured marker `twoSeparatelyOwnedNamesResolveToAttacker` must be `true`.
{% endstep %}

{% step %}

## Observe final pass and cleanup markers

The proof must finish with all of these markers:

```
Test Files  1 passed (1)
Tests       1 passed (1)
componentResult=PASS
portalLocalProxies=PASS
1 passed
browserResult=PASS
joinedResult=PASS
servicesOwnedByRunnerStopped=true
```

The cleanup marker is printed only after no listener remains on the three isolated ports and no container, network, or volume remains under the dedicated Compose project.
{% endstep %}
{% endstepper %}

### Captured evidence files

* `scans/resolver-row-state-bleed-clean.log`: the complete joined execution record.
* `scans/portal-resolver-row-state-bleed-browser.json`: structured authentic role order, row transition, modal, exact transaction, implementation, and final-resolution evidence.
* `scans/portal-resolver-row-state-bleed-editor.png`: the shifted collaborator editor before the victim changes `Set ABI`, with stale `Upgrade` visibly checked.
* `scans/portal-resolver-row-state-bleed-modal.png`: the production transaction modal identifying the target account while omitting the permission names.
* `scans/resolver-row-state-bleed-indexer.log`: local Panoptes indexer output.
* `scans/resolver-row-state-bleed-api.log`: local Panoptes GraphQL service output.
* `scans/resolver-row-state-bleed-vite.log`: frozen Portal dev-server output.

### Proof boundaries

The component regression and browser-to-chain proof are complementary. The component test isolates the exact production state transition and prepared intent. The browser proof performs the real connected-wallet sequence against authentic Panoptes and verifies the resulting fork state.

The stale `Upgrade` checkbox is visible in the editor and is deliberately captured in the evidence. The proof does not claim it is hidden. The critical mismatch is that this permission belongs to the removed account's editor state, is carried into a different collaborator's role delta without being selected for that collaborator, and is not enumerated by Portal's final transaction modal.

The victim intentionally performs both role-management workflows and approves both wallet requests. The proof does not claim attacker-forced interaction.

The locally compiled resolver implementation is a deterministic consequence demonstrator. It is used only on the disposable private fork. The application vulnerability is the cross-account editor-state reuse that creates the root Upgrade grant.
