# 89507 w a critical remove user on the resolver roles panel revokes only one resource scope leaving listed authority in place

**Submitted on Aug 19th 2026 at 17:04:06 UTC by @UptightChipmunk990 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89507
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Unauthorized takeover of a user's smart account (Hybrid Custody Account) or authenticated session, letting an attacker transfer the victim's names or funds or act as them, such as:

    * adding, replacing, or escalating an account owner or co-owner
    * extending or widening a session key's authority beyond its intended scope or lifetime
    * stealing an authenticated session or bearer token

    (Exploiting the 7-day session-key design as described in the Known Issues is not eligible; a weakness that grants authority beyond its stated lifetime or permissions is in scope.)
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer's resolver-roles panel merges every resource a delegate holds roles on into a single table row, then builds the "Remove user" transaction at exactly **one** scope — the first non-root name in that row — while the confirmation dialog tells the user they are removing that delegate "from all roles" and that the action "cannot be undone". Every other resource in the row is left untouched — and because roles granted at the root resource are ORed into every resource by `EnhancedAccessControl._effectiveRoles`, the grant most likely to survive is the one that confers authority over *all* names on the resolver. In production this means an owner who revokes a delegate through the UI is shown a completed, irreversible removal while the delegate silently retains resolver write authority, including `ROLE_SET_ADDR`, the role controlling which ETH address a name resolves to. In the root-grant case the removal reduces the delegate's authority over the displayed name by nothing at all: it strips the narrow name-scoped role and preserves the broad global one. A party the owner believes is gone can still repoint any of their names and capture funds that third parties send to them.

## Vulnerability Details

Four in-scope TypeScript files. The merged row is not an assumption about indexer behaviour — it is forced by the query shape, so step 0 establishes the input before the logic is traced.

{% stepper %}
{% step %}

## The input really is one row per `(account, resource)`

`apps/portal/src/features/resolver/hooks/useResolverOverview.ts:90-98`

```graphql
roles {
  account
  resource      # <-- per-grant, distinct for root vs a name
  roleBitmap
  blockNumber
  transactionHash
  timestamp
  name          # <-- per-grant resource label, returned and never used
}
```

`roles` is a flat list off the resolver entity with no grouping, no dedup and no `distinct`. An account holding a root grant *and* a name-scoped grant is returned as **two entries with different `resource` values**. Nothing between the indexer and the table filters or collapses that.
{% endstep %}

{% step %}

## The per-resource linkage is destroyed

`apps/portal/src/lib/roles/resolverRoles.ts:193-215` — `groupRolesByAccount` keys strictly on `account`:

```ts
for (const role of roles) {
  const account = role.account.toLowerCase()
  const resource = role.resource.toLowerCase()
  const decoded = decodeResolverRoleBitmap(role.roleBitmap)
  const resolvedName = resourceToName?.get(resource) ?? null

  const existing = grouped.get(account)

  if (existing) {
    existing.roles.push(role)
    existing.decodedRoles.push(...decoded)                      // flat union of roles
    if (resolvedName) existing.resolvedNames.add(resolvedName)  // flat set of resources
  }
```

Roles become a flat `decodedRoles` array and resources a flat `resolvedNames` Set, with nothing tying one to the other. The per-role resource label the indexer already returns — `ResolverRole.name: string | null`, `useResolverOverview.ts:21` — is never read.
{% endstep %}

{% step %}

## The row is collapsed back to a single scope

`ResolverRolesSidebar.tsx:169`:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Note that this expression *deliberately skips* `(root)` and takes the first name. It then becomes **the** scope for the write, carrying the full merged role list with it — `:316-321`:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

{% endstep %}

{% step %}

## `name` is the scope selector

`revokeResolverRoles.ts:47-60`:

```ts
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

Empty string means the root resource (all names); anything else means that one name. Exactly one resource is ever written.
{% endstep %}
{% endstepper %}

### What the user is shown, on the same screen

Sidebar header, `ResolverRolesSidebar.tsx:391-396` — twenty-five lines *after* the expression that discards root:

```tsx
{resolvedNames.includes('(root)')
  ? 'Global roles (all names)'
  : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
```

Confirm dialog, `ResolverRolesSidebar.tsx:445-448`:

```tsx
<DialogDescription>
  Are you sure you want to remove this user from all roles? This
  action cannot be undone.
</DialogDescription>
```

The header enumerates every resource in the merged row. The dialog promises all of them will go. One is written. The same `resolvedNames` array is the source for both the header that says "all names" and the expression that selects a single name.

### Reachability

The multi-resource state is what the app's own defaults produce, not an exotic configuration.

The Add-user sheet defaults its node picker to the **root** resource: `ResolverAddUserSheet.tsx:38-39` (`const ROOT_NODE_VALUE = ''`, commented "Empty string = root resource (roles apply to all names)"), `:127` (`useState<string>(ROOT_NODE_VALUE)`), `:253` (the only affordance is grey placeholder text, `<ComboboxInput placeholder="Root (all nodes)" />`). A user who grants a role without touching the picker grants it globally. Any later name-scoped grant to the same account produces the merged row.

Granting the same role on two different names produces the merged row directly, without relying on the default at all.

## Recommended Fix

Keep the resource attached to the role through both display and write.

The minimal change is to stop merging across resources: key `groupRolesByAccount` on `(account, resource)` rather than `account` alone, so each row carries exactly one scope and "Remove user" writes that scope. The per-role `name` needed to label those rows is already returned by the indexer (`useResolverOverview.ts:21`) and currently discarded.

If the merged, account-per-row presentation is wanted for the table, then the sidebar must emit **one `revokeRoles` call per resource** in the row rather than picking `resolvedNames.find(...)`, and the confirm dialog should enumerate the scopes it is about to write. Until it does, the copy should not promise removal "from all roles".

## Impact Details

**What the application displays that is incorrect.** The user is shown a transaction detail — the scope of the revocation — that does not match the transaction they approve. The sidebar header states "Global roles (all names)" and the confirm dialog states "remove this user from all roles… cannot be undone", while the emitted call carries `scope: 'name'` for a single name. Both strings and the scope selector are derived from the same `resolvedNames` array in the same component, so the divergence is between what that component renders and what it signs.

*One honest note on category fit:* the selected impact's harm clause reads "to an unintended **recipient**". Here the divergence is in the resource **scope** argument rather than a recipient address. The rest of the description — the application displaying an incorrect transaction detail which the user then approves — matches precisely, and this is the closest of the listed in-scope impacts. I would rather flag the seam than have it read as a stretch.

**Loss path.** `ROLE_SET_ADDR` controls which ETH address a name resolves to. A delegate who retains it after a UI-confirmed removal can repoint the name at an address they control at any later time. Anyone who then pays that ENS name — a counterparty, a donor, an exchange withdrawal, an automated payout — sends funds to the attacker instead of the name's owner. The loss is unbounded by the protocol: it is whatever is sent to the affected name between the repoint and its discovery.

**In the root-grant case the revocation achieves nothing at all.** This is confirmed against the deployed access-control contract, not inferred. `EnhancedAccessControl._revokeRoles` masks unheld bits (`currentRoles & ~roleBitmap`) instead of reverting, so the transaction succeeds; and `_effectiveRoles` ORs `_roles[ROOT_RESOURCE][account]` into every resource, so the surviving root grant still authorizes writes on **every name the resolver serves — including the one the user was looking at when they clicked Remove**. The call revokes the narrow name-scoped role and leaves the broad global one, which inverts the user's intent. The blast radius is therefore not one name but the whole resolver: for an HCA-deployed resolver that is every name that account has registered through it.

**Why it goes unnoticed.** The owner has no signal that anything survived. The transaction succeeded. The dialog said the removal was complete and irreversible. On refetch the row disappears from the table for the resource that *was* revoked. Nothing in the UI distinguishes a full removal from a partial one, so there is no natural moment at which the owner would re-check. In the root-grant case (BUG 2) the surviving authority is the broadest one on the resolver — every name it serves, not just the one that was displayed.

**Who bears it.** The immediate victim is the resolver owner, who has lost control of resolution for names they still believe they administer. The party who actually loses funds is any third party paying the name, who has no visibility into the resolver's role table at all and no way to detect the condition.

**Second affected path.** The same defect reaches the Save flow. `computeRoleChanges(decodedRoles, editedPermissions)` (`ResolverRolesSidebar.tsx:190-193`) derives `rolesToRevoke` from the merged role list and writes it at the same single `roleName`. Unchecking a permission the account holds at another resource therefore produces a call that never touches it. The checkbox reappears checked after the indexer refetch, with no error shown — so a targeted permission reduction fails as silently as a full removal.

**Evidence this is not the intended design.** The registry roles page handles the identical problem correctly: it filters to the root resource before display (`useRegistryRoles.ts:36`, `:72`) and hard-codes `resource: ROOT_RESOURCE` on write (`grantRegistryRoles.ts:45`, `:67`), so its displayed scope and its written scope cannot diverge. Only the resolver path loses the linkage.

**Attacker model.** No privilege is required to *trigger* this — the resolver owner triggers it themselves by clicking "Remove user", and the PoC needs no attacker action at all. The party who benefits is a delegate the owner previously granted roles to and has now attempted to remove. That is the standard threat model for a revocation feature: it must work against a party who *was* trusted and no longer is. The finding is not that a privileged actor can do something; it is that a de-privileging action does not de-privilege.

### Relation to the published known issues

Not covered by any of the 60 listed entries.

* **QA-01** concerns role combinations *when transferring a name*, and the symptom it describes is later errors updating records or setting a primary name. This is the resolver roles panel's revoke path, involves no transfer, and produces a silent success rather than an error. QA-01's own text expressly reserves "a role combination that grants unintended authority" as a new finding.
* **R2-03** concerns the ephemeral HCA session key's scope and browser-storage exposure. Different subsystem and different mechanism — this is a PermissionedResolver role grant, not a session key.
* **R2-06** concerns session revocation clearing only browser storage with no on-chain revocation. This revocation *is* on-chain; it simply writes the wrong resource.
* **WEB-1249** concerns subname creation replacing a child registry; **WEB-716** concerns the Registry page's deploy-subregistry affordance. Neither touches resolver role scoping.

## References

Repository: `https://github.com/immunefi-team/audit-comp-ens` · branch `audit-comp-ready` · commit `1c9b47f`

### The defect

* Scope collapsed to one name, discarding root — [`ResolverRolesSidebar.tsx:169`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169)
* Merged role list written at that single scope — [`ResolverRolesSidebar.tsx:316-321`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L316-L321)
* `name` as the root-vs-name scope selector — [`revokeResolverRoles.ts:47-60`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L47-L60)
* Grouping keyed on account alone — [`resolverRoles.ts:193-215`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/resolverRoles.ts#L193-L215)

### The contradicting UI copy

* "Global roles (all names)" header — [`ResolverRolesSidebar.tsx:391-396`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L396)
* "remove this user from all roles… cannot be undone" — [`ResolverRolesSidebar.tsx:445-448`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L445-L448)

### Input shape and reachability

* Indexer returns one row per `(account, resource)` — [`useResolverOverview.ts:90-98`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L90-L98)
* Per-role resource label, returned and unused — [`useResolverOverview.ts:21`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L21)
* Add-user node picker defaults to root — [`ResolverAddUserSheet.tsx:38-39`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L38-L39), [`:127`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L127), [`:253`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L253)

### Second affected path, and the correct comparison

* Save path reuses the merged list at one scope — [`ResolverRolesSidebar.tsx:190-193`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L190-L193)
* Registry path filters to root before display — [`useRegistryRoles.ts:36`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/hooks/useRegistryRoles.ts#L36), [`:72`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/hooks/useRegistryRoles.ts#L72)
* Registry path hard-codes the written resource — [`grantRegistryRoles.ts:45`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/helpers/grantRegistryRoles.ts#L45), [`:67`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/helpers/grantRegistryRoles.ts#L67)

### On-chain behaviour relied on for impact

`ensdomains/contracts-v2` @ `main` — out of scope as an asset, cited only to establish what the emitted transaction does.

* `_revokeRoles` masks unheld bits, never reverts on them — [`EnhancedAccessControl.sol:290-315`](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/access-control/EnhancedAccessControl.sol#L290-L315)
* `_checkRoleBitmap` rejects only bits outside `ALL_ROLES` — [`EnhancedAccessControl.sol`](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/access-control/EnhancedAccessControl.sol)
* `_checkCanRevokeRoles` checks the caller's admin rights, not the target's held roles — [`EnhancedAccessControl.sol`](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/access-control/EnhancedAccessControl.sol)
* `_effectiveRoles` ORs root roles into every resource; `hasRoles` reads through it — [`EnhancedAccessControl.sol`](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/access-control/EnhancedAccessControl.sol)
* `ROOT_RESOURCE = 0`, and the header comment "roles granted in `ROOT_RESOURCE` (0x0) automatically apply" — [`EnhancedAccessControl.sol:19`](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/access-control/EnhancedAccessControl.sol#L19), [`:54`](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/access-control/EnhancedAccessControl.sol#L54)
* Resolver accepts node-specific **or** node-`0` global authority — [`PermissionedResolver.sol:181-182`](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/resolver/PermissionedResolver.sol#L181-L182)
* App maps `ROOT_RESOURCE` → `'(root)'` — [`resolverRoles.ts:151`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/resolverRoles.ts#L151)

## Link to Proof of Concept

<https://gist.github.com/Manimal777/e615db649f8db33abec210bd7208f30d>

## Proof of Concept

`poc_resolver_revoke_scope.mjs` — plain Node, no dependencies: `node poc_resolver_revoke_scope.mjs`.

`groupRolesByAccount`, `decodeResolverRoleBitmap`, `buildResourceToNameMap` and the `resolverRoles` bitmap table are copied verbatim from source; they are pure (bigint, Map, Set only). The scope-selection expression and the revoke-scope ternary are likewise reproduced verbatim. The single substitution is `computeNameResource`, which in the real file is `keccak256(encodePacked(['bytes32','bytes32'],[namehash(name), 0]))` and needs viem; the stub returns a distinct deterministic hex per name. That output is used only as a Map key (`resolverRoles.ts:155`, `:197`), so distinctness is the only property the logic depends on.

Two controls establish that the panel behaves correctly when a delegate's roles live at a single resource. Two bug cases differ from those controls in exactly one variable — a second resource.

```
CONTROL A — one role on ONE name
  granted             : ROLE_SET_ADDR@alice.eth
  emitted revoke scope: name "alice.eth"
  STILL HELD AFTER    : (nothing — fully revoked)          PASS

CONTROL B — roles at ROOT only (the Add-user default)
  granted             : ROLE_SET_ADDR@(root), ROLE_SET_TEXT@(root)
  emitted revoke scope: root (all names)
  STILL HELD AFTER    : (nothing — fully revoked)          PASS

BUG 1 — the SAME role on TWO names
  granted             : ROLE_SET_ADDR@alice.eth, ROLE_SET_ADDR@bob.eth
  panel header shows  : "Roles scoped to alice.eth, bob.eth"
  emitted revoke scope: name "alice.eth"
  STILL HELD AFTER    : ROLE_SET_ADDR@bob.eth              RETAINED

BUG 2 — a ROOT role and a NAME-scoped role
  granted             : ROLE_SET_ADDR@(root), ROLE_SET_TEXT@alice.eth
  panel header shows  : "Global roles (all names)"
  emitted revoke scope: name "alice.eth"
  STILL HELD AFTER    : ROLE_SET_ADDR@(root)               RETAINED
```

**BUG 1** is the simplest case: the role bitmap is identical at both resources, so the emitted call revokes a role the account genuinely holds at `alice.eth` and succeeds cleanly, while `bob.eth` is never written.

**BUG 2 is the severe case, and the on-chain behaviour it depends on is confirmed.** The emitted call includes `ROLE_SET_ADDR`, which the delegate holds at root but not at `alice.eth`. `EnhancedAccessControl._revokeRoles` masks rather than validates:

```solidity
uint256 currentRoles = _roles[resource][account];
uint256 updatedRoles = currentRoles & ~roleBitmap;

if (currentRoles != updatedRoles) {
    _roles[resource][account] = updatedRoles;
    ...
    return true;
} else {
    return false;
}
```

Bits the account does not hold at `resource` are simply masked off. There is no revert path for them — `_checkRoleBitmap` rejects only bits outside `ALL_ROLES`, and `canRevokeRoles` checks the *caller's* admin rights, not the target's held roles. **The transaction succeeds and the root grant survives silently.**

The consequence is worse than partial revocation. Root roles are not one scope among many — they are ORed into every resource on every permission check:

```solidity
function _effectiveRoles(uint256 resource, address account) private view returns (uint256) {
    return _getRoles(ROOT_RESOURCE, account) | _getRoles(resource, account);
}

function hasRoles(uint256 resource, uint256 roleBitmap, address account) public view returns (bool) {
    return _effectiveRoles(resource, account) & roleBitmap == roleBitmap;
}
```

`PermissionedResolver` applies the same fallback a second time at its own layer, accepting a caller who holds the role at either the node's resource **or** the node-`0` (global) resource (`PermissionedResolver.sol:181-182`). The contract's own header comment states it plainly: *"roles granted in `ROOT_RESOURCE` (0x0) automatically apply"* (`EnhancedAccessControl.sol:19`).

So in BUG 2 the revocation removes the **narrow** name-scoped `ROLE_SET_TEXT` and preserves the **broad** global `ROLE_SET_ADDR` — the exact inverse of what the user asked for. The delegate's effective authority over `alice.eth`, the very name on screen, is unchanged after the transaction, because the surviving root grant still satisfies `hasRoles(alice.eth, ROLE_SET_ADDR, delegate)`. The revocation the user was told was complete and irreversible protected nothing.

The app's `(root)` label is exactly this resource: `resolverRoles.ts:151` maps `ROOT_RESOURCE` → `'(root)'`, and `ROOT_RESOURCE` is `0` (`EnhancedAccessControl.sol:54`).
