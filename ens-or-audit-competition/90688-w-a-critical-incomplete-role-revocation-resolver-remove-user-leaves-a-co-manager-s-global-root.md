# 90688 w a critical incomplete role revocation resolver remove user leaves a co manager s global root role in place so a removed user can still set any name s address record and redirect funds

**Submitted on Aug 26th 2026 at 12:15:51 UTC by @ParallelPossum2463 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90688
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Unauthorized takeover of a user's smart account (Hybrid Custody Account) or authenticated session, letting an attacker transfer the victim's names or funds or act as them, such as:

    * adding, replacing, or escalating an account owner or co-owner
    * extending or widening a session key's authority beyond its intended scope or lifetime
    * stealing an authenticated session or bearer token

    (Exploiting the 7-day session-key design as described in the Known Issues is not eligible; a weakness that grants authority beyond its stated lifetime or permissions is in scope.)
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes

## Description

## Summary

The Explorer's PermissionedResolver role-management UI (the "Resolver Roles" page) builds an incomplete on-chain revocation. When an owner clicks "Remove user" for a co-manager who holds roles at more than one EAC resource, for example a global (root) role together with a name-scoped role, the app emits a single name-scoped revoke that carries the union of the account's role bits across all resources. The account's roles at every other resource, including a global root role, are never touched. The confirmation dialog says "remove this user from all roles ... cannot be undone", but the removal is silently partial: the removed co-manager keeps live authority to set records (notably the addr record, which controls where funds are sent) for names on that resolver.

## Root cause

The write scope is decoupled from the roles being written. `groupRolesByAccount` (`apps/portal/src/lib/roles/resolverRoles.ts:179-221`) merges every EAC resource an account holds roles on into one table row, so `decodedRoles` becomes the union of role bits across resources (the root resource `0x00..00` plus each per-name resource `keccak(namehash(name),0)`) and `resolvedNames` becomes the union of names.

The sidebar then collapses that to a single write scope at `ResolverRolesSidebar.tsx:169` with:

```typescript
roleName = resolvedNames.find(n => n !== '(root)') ?? ''
```

The `handleRemoveUser` function (`ResolverRolesSidebar.tsx:307-323`) issues one:

```typescript
revokeResolverRoles({ name: roleName, roles: decodedRoles })
```

This writes the union bitmap at that single scope, and the edit path `handleSaveChanges` (`:295-305`) has the same defect.

`revokeResolverRoles.ts:45-61` maps scope `name` to:

```typescript
authorizeNameRoles(dnsEncode(name), bitmap, account, false)
```

It only uses scope `root` (`revokeRootRoles`) when `roleName` is empty. So for a holder of a root role plus a name role, `roleName` is the name, the scope is `name`, and `revokeRootRoles` is never called: the on-chain EAC clears bits only at that one name resource and the root role stays active.

This is not a contract-level bypass, since the EAC enforces per-resource roles correctly. It is client-side incomplete transaction construction in an in-scope asset (`apps/portal`). For contrast, the sibling registry-roles feature is correct: `revokeRegistryRoles.ts` hard-codes `ROOT_RESOURCE` and `useRegistryRoles.ts:72` skips non-root resources.

## Impact

A co-manager the owner believes they fully removed keeps unauthorized authority. In the worst case they retain a root `ROLE_SET_ADDR`, which is authority to set the ETH addr record of every name the shared resolver serves, so they can redirect funds sent to any of those names to an attacker address.

Because root scope spans all names on a shared PermissionedResolver, one instance can affect several distinct users. The same single-scope defect on the save/edit path can also mis-scope a grant to root/global (an over-grant) when `resolvedNames` has no non-root entry.

This matches the program's primary concern ("direct users to transfer funds to an address other than the intended one") and the in-scope impact "a weakness that grants authority beyond its stated permissions".

## Suggested fix

Write grants and revokes per resource: iterate the account's actual role entries (each carries its own resource) and emit one grant or revoke per resource with that resource's bitmap.

For "Remove user", revoke every resource the account holds, including the root resource and each name. Do not derive a single scope from:

```typescript
resolvedNames.find(...)
```

Do not fall back to root scope for a name-scoped edit. The registry-roles code is a correct model.

## Link to Proof of Concept

<https://gist.github.com/liorger53509/10d28677ef977584f5b1b205986be024>

## Proof of Concept

## Step-by-step PoC

This walks through the exact transaction the "Remove user" button constructs and the resulting on-chain role state, with code references. No public-network testing was performed. A runnable reproduction against a local fork is provided in the linked secret Gist.

## Preconditions

A PermissionedResolver serves several names and Alice is its owner/admin. Over time Alice granted co-manager Bob roles at two distinct EAC resources:

* A root (global) role, `ROLE_SET_ADDR`, which covers every name on the resolver.
* A name-scoped role, `ROLE_SET_TEXT`, on `blog.alice.eth`.

The UI supports and labels both cases: a "Global roles (all names)" label for a root grant, and a "Roles scoped to ..." label that lists the specific name(s) for a name grant (`ResolverRolesSidebar.tsx:391-397`).

{% stepper %}
{% step %}

## Alice opens the Resolver Roles page

Bob appears as one row. `groupRolesByAccount` (`resolverRoles.ts:179-221`) merges both resources into a single row:

```typescript
decodedRoles
```

is the union of role bits, and:

```typescript
resolvedNames = ['(root)', 'blog.alice.eth']
```

{% endstep %}

{% step %}

## Alice clicks "Remove user"

The confirmation dialog reads:

> Are you sure you want to remove this user from all roles? This action cannot be undone.

(`ResolverRolesSidebar.tsx:446`)
{% endstep %}

{% step %}

## The app builds one revoke transaction at one scope

```typescript
roleName = resolvedNames.find(n => n !== '(root)') ?? ''
```

(`ResolverRolesSidebar.tsx:169`) resolves to `blog.alice.eth`.

`handleRemoveUser` (`:307-323`) then calls:

```typescript
revokeResolverRoles({
  name: 'blog.alice.eth',
  roles: decodedRoles
})
```

This uses the union bitmap at scope `name`. `revokeResolverRoles.ts:45-61` maps that to:

```typescript
authorizeNameRoles(
  dnsEncode('blog.alice.eth'),
  ROLE_SET_ADDR | ROLE_SET_TEXT,
  Bob,
  false
)
```

Because `roleName` is not empty, scope `root` is never selected, so `revokeRootRoles` is never called.
{% endstep %}

{% step %}

## Resulting on-chain state

The PermissionedResolver clears the given role bits only at the `blog.alice.eth` resource.

Bob's root `ROLE_SET_ADDR` lives at the root resource (`0x00..00`), which the transaction never references, so it stays active:

```typescript
hasRootRoles(ROLE_SET_ADDR, Bob)
```

still returns `true`, while:

```typescript
roles(nameResource('blog.alice.eth'), Bob)
```

becomes `0`.
{% endstep %}
{% endstepper %}

## Result

Alice believes Bob is fully removed, because the action completed and the dialog promised removal "from all roles". In reality Bob keeps the root `ROLE_SET_ADDR`, which is global authority to set the ETH addr record of every name the resolver serves, so he can redirect funds sent to any of those names.

The same single-scope defect on the save/edit path (`handleSaveChanges`, `:295-305`) can write a grant at root/global scope (an over-grant) when `resolvedNames` has no non-root entry.

## Notes

The sibling registry-roles feature is correct (`revokeRegistryRoles.ts` hard-codes `ROOT_RESOURCE`, and `useRegistryRoles.ts:72` skips non-root resources), which shows the intended per-resource behavior.

Bob only ever holds the co-manager role the owner intentionally granted and then tried to revoke, so no unintended pre-existing privilege is required. This is a code vulnerability that leaves an address with privileges it should not have after removal, which is valid per the competition's privileged-address exception.

A runnable reproduction (viem plus the pinned ENS-team ensjs build, run against a local fork rather than a public network) is provided in the linked secret Gist. It grants Bob root and name roles, issues the exact:

```typescript
authorizeNameRoles(name, union, Bob, false)
```

revoke the app builds, and reads:

```typescript
hasRootRoles(ROLE_SET_ADDR, Bob) == true
```

afterward.
