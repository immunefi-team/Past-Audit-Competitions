# 89593 w a critical incorrect scope inference in the resolver roles editor leaves a removed user s resolver wide grant live up to role upgrade

**Submitted on Aug 20th 2026 at 03:16:31 UTC by @sikoh for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89593
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Explorer's resolver roles page collapses every grant an account holds into one table row and then infers a single scope for it, so "Remove user" revokes the name-scoped grant and silently leaves the resolver-wide (root) one in place. The operator answers a dialog that says "remove this user from all roles", signs a transaction that succeeds, and the account keeps its root roles over every name on that resolver. With ROLE\_SET\_ADDR that means repointing the address record of any of those names; with ROLE\_UPGRADE, which is only ever checked at root, it means replacing the implementation behind the resolver proxy. The same inference fails in the opposite direction on the grant path, writing a permission the operator intended for one name at root scope instead.

## Vulnerability Details

`apps/portal/src/lib/roles/resolverRoles.ts:193-215` keys the grouping purely on the account and concatenates the decoded roles of every resource it holds:

```
for (const role of roles) {
  const account = role.account.toLowerCase()
  ...
  if (existing) {
    existing.roles.push(role)
    existing.decodedRoles.push(...decoded)
    if (resolvedName) existing.resolvedNames.add(resolvedName)
```

The sidebar reduces that set of names back to one, in `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169`:

```
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

and passes that single `roleName` as the scope for both directions, at `:299` (save) and `:318` (remove). The parameter is documented at `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:20` as "Dotted name (e.g. 'myname.eth') or empty string for ROOT\_RESOURCE (all names)", and `grantResolverRoles.ts:46-60` and `revokeResolverRoles.ts:45-60` branch on exactly that:

```
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

Those two scopes resolve to different contract functions: `scope: 'root'` to `grantRootRoles`/`revokeRootRoles` against ROOT\_RESOURCE, `scope: 'name'` to `authorizeNameRoles` against that name's resource. So the scope written is not tied to the scope read.

Nothing revalidates it in between. `roleName` goes to `setPendingAction` at `:299` and `:318`, the confirm button passes `pendingAction.name` to the mutation at `:488-493`, and `removeUserMutation`'s `mutationFn` hands it straight to `revokeResolverRoles` at `:266-276`. The operator therefore signs a transaction whose scope argument the interface inferred for them, never displayed, and got wrong. The confirmation they answer first is explicit about the intent: "Are you sure you want to remove this user from all roles? This action cannot be undone." (`ResolverRolesSidebar.tsx:444-446`).

That expression fails in both directions. When the account has a root grant and a name grant, `find` skips `(root)` and returns the name, so the revoke targets the name and the root grant remains. When the resource hash cannot be mapped to a name, `resolvedNames` is empty, `roleName` becomes the empty string, and any tick in the permission grid is written with `grantRootRoles` instead: a grant the operator intended for one name is applied to all of them.

That second case is not exotic. The map comes from `buildResourceToNameMap(nodes)`, and `nodes` is fetched by a query that passes no pagination arguments at all (`useResolverOverview.ts:75`), where the sibling registry query caps itself explicitly (`useRegistryRoles.ts:32`, `ROLES_LIMIT = 1000`). Whatever the indexer's default page size turns out to be, a resource missing from that list yields the empty scope rather than an error. And the operator is given nothing to go on, because the scope caption renders only `if resolvedNames.length > 0` (`ResolverRolesSidebar.tsx:390-396`): a row with no mapped name shows no scope text at all, while every checkbox in it writes to root.

The indexer already returns a `name` field per role (`useResolverOverview.ts:97`) which this code does not use, and the sibling registry implementation gets the scoping right by testing the resource explicitly (`apps/portal/src/features/registry/hooks/useRegistryRoles.ts:33-36`, `const isRootResource = (resource: string) => BigInt(resource) === 0n`).

One of the roles this grid writes makes the direction of the error matter more than it first looks. `resolverPermissions` (`apps/portal/src/lib/roles/resolverRoles.ts:57-107`) is a flat list of ten checkboxes, and the tenth is ROLE\_UPGRADE, described there as "Can upgrade the resolver contract", rendered at the same level as "Set Address" and "Set Text", with no separation or warning. On the contract side that role is only ever checked at root: PermissionedResolver is a UUPS proxy whose `_authorizeUpgrade` carries `onlyRootRoles(PermissionedResolverLib.ROLE_UPGRADE)` (`PermissionedResolver.sol:748-752`), and `onlyRootRoles` resolves against ROOT\_RESOURCE alone (`EnhancedAccessControl.sol:96-98`). So the scope this code silently defaults to, and the scope it silently fails to revoke, is exactly the one that authorises replacing the resolver's implementation.

The fix is to stop collapsing resources into an account row and carry the resource through to the write. The row should be per (account, resource), which is also what the indexer already returns:

```
// resolverRoles.ts - key the grouping on both
-const account = role.account.toLowerCase()
+const key = `${role.account.toLowerCase()}:${role.resource.toLowerCase()}`

// ResolverRolesSidebar.tsx:169 - no scope inference
-const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
+const scope = row.original.resource === ROOT_RESOURCE
+  ? { kind: 'root' as const }
+  : { kind: 'name' as const, name: row.original.name }
```

Until that lands, refuse to act rather than guess: if `resolvedNames` contains `(root)` alongside a name, or is empty for a non-root resource, disable the buttons instead of defaulting to a scope. Silent selection of root is the worst available default in both directions.

## Impact Details

An operator revoking a collaborator's or a compromised key's access answers a dialog that says "all roles", signs one transaction that succeeds, and the account keeps whatever it held at root over every name on that resolver. With ROLE\_SET\_ADDR that means repointing the address record of any of those names, so anyone paying those names pays the holder instead. The grant direction is the mirror image: a role meant for one name is written resolver-wide, handing the grantee authority over names that were never part of the flow.

Where it stops being about records is ROLE\_UPGRADE. Because that role is only ever checked at root, both halves of this defect point at it: an operator editing one name's permissions with an unmapped resource writes every ticked box, Upgrade included, at root scope, and an account already holding it at root keeps it through a removal that completes without error. What the holder then replaces is not a record but the resolver's code, which covers every name on it including ones added later, can drop the checks that would otherwise allow removing them, and unlike every other role here does not come back by revoking anything. The proof of concept below executes that step: the account the interface reported as removed calls `upgradeToAndCall` on the proxy.

Two limits worth stating rather than leaving for you to find. The interface does not stay silent forever: after the transaction the sidebar refetches `resolver-overview` (`:278-286`) and the table regroups, so the account reappears showing `(root)`, and an operator who reads that row and knows what `(root)` means can click "Remove user" again. That second attempt works, because with `(root)` as the only entry `find` returns undefined and the scope falls back to root. So the revoke half is recoverable by whoever notices; what it never is, is announced. Nothing errors, nothing warns, and the dialog has just said the action cannot be undone. The grant direction has no equivalent second chance: the wider scope is already written, and undoing it still starts with noticing it.

This is also not self-escalating, and I checked rather than assumed it. `_getSettableRoles` is `withAdminRolesApplied(_effectiveRoles(resource, account))`, and `withAdminRolesApplied` opens with `roleBitmap >>= 128` (`EACBaseRolesLib.sol:31-34`), which discards every regular role and keeps only the admin nybbles. An account holding ROLE\_SET\_ADDR at root has `settableRoles == 0` and can grant nothing; one holding ROLE\_SET\_ADDR\_ADMIN can grant ROLE\_SET\_ADDR and that same admin variant, and nothing outside that one role. Reaching ROLE\_UPGRADE requires having been given ROLE\_UPGRADE\_ADMIN. The per-role admin separation holds, so the upgrade path needs the operator to have granted that role, through the checkbox or through the defaulted-to-root write above, and is not something the grantee arranges alone.

## References

* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/resolverRoles.ts>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/hooks/useRegistryRoles.ts>
* <https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/resolver/PermissionedResolver.sol>
* <https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/access-control/EnhancedAccessControl.sol>
* <https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/access-control/libraries/EACBaseRolesLib.sol>

## Link to Proof of Concept

<https://gist.github.com/SiikoH/5fbdc51903b7b4b1a9b2c264ea03aef9>

## Proof of Concept

## Proof of Concept

Two tests. The first drives the production grouping function and the production calldata builders, so it measures what the interface concludes about an account and what transaction the buttons produce. The second sends that transaction to a real PermissionedResolver and reads the permission state afterwards, so the surviving grant is measured rather than argued. Both carry a positive control.

Step by step, against the app:

1. On a resolver you operate, grant account B `ROLE_SET_ADDR | ROLE_SET_TEXT` at root scope (`grantRootRoles`), and `ROLE_SET_TEXT` scoped to `blog.alice.eth` (`authorizeNameRoles`).
2. Open `/resolver/<resolver>/roles`. B appears as a single row. Open it: the sidebar reads "Global roles (all names)".
3. Click "Remove user". The dialog says "Are you sure you want to remove this user from all roles? This action cannot be undone." Confirm and sign.
4. The transaction succeeds. Read `hasRoles(ROOT_RESOURCE, ROLE_SET_ADDR, B)` on the resolver: still true. B can call `setAddr` on any name using that resolver.
5. Repeat step 1 granting ROLE\_UPGRADE at root instead. After the same removal, `hasRoles(ROOT_RESOURCE, ROLE_UPGRADE, B)` is still true, so B satisfies `_authorizeUpgrade` and can call `upgradeToAndCall` on the proxy.

TEST 1, the interface and the calldata it builds, in the in-scope repo:

```
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable && pnpm install --frozen-lockfile
cp resolver-role-scope.test.ts apps/portal/src/resolver-role-scope.test.ts
cd apps/portal && ./node_modules/.bin/vitest run src/resolver-role-scope.test.ts --reporter=verbose
```

Output as I ran it:

```
> groups two resources into ONE row and merges their roles
resolvedNames = [ '(root)', 'blog.alice.eth' ]
decodedRoles  = [ 'ROLE_SET_ADDR', 'ROLE_SET_TEXT', 'ROLE_SET_TEXT' ]
UI label      = Global roles (all names)
roleName used = "blog.alice.eth"

> BUG: "Remove user" emits authorizeNameRoles, NOT revokeRootRoles => the global grant survives
remove-user calldata -> authorizeNameRoles ["0x04626c6f6705616c6963650365746800","17","0x00000000000000000000000000000000000000bb",false]

> CONTROL: an account with ONLY root roles is correctly removed at root scope
control calldata -> revokeRootRoles

> BUG 2: unresolvable resource => empty resolvedNames => any grant/save silently goes to ROOT
resolvedNames (unmapped) = []
rolesToGrant = [ 'ROLE_SET_CONTENTHASH' ]
grant calldata -> grantRootRoles ["256","0x00000000000000000000000000000000000000bb"]

> BUG 3: roles on TWO names => "Remove user" only revokes on the FIRST name
resolvedNames = [ 'blog.alice.eth', 'pay.alice.eth' ]
single scope used for BOTH = blog.alice.eth

> BUG 4: with an unresolvable resource, ticking "Upgrade" grants it at ROOT scope
operator ticked -> [ 'ROLE_UPGRADE' ]
grant calldata -> grantRootRoles bitmap = 21267647932558653966460912964485513216
ROLE_UPGRADE (1n<<124n) set? -> true

> BUG 5: "Remove user" leaves a root-scoped ROLE_UPGRADE untouched
decodedRoles = [ 'ROLE_UPGRADE', 'ROLE_SET_TEXT' ]
scope the sidebar picked = "blog.alice.eth"
remove-user calldata -> authorizeNameRoles
=> ROLE_UPGRADE at ROOT survives the removal

Test Files  1 passed (1)
     Tests  7 passed (7)
```

The control is the load-bearing part: an account holding only root roles does produce `revokeRootRoles`, so the harness is measuring the scope decision and not a stub. `0x04626c6f6705616c6963650365746800` is the DNS encoding of `blog.alice.eth`, `17` is `ROLE_SET_ADDR | ROLE_SET_TEXT`, and `21267647932558653966460912964485513216` is 2\*\*124, i.e. ROLE\_UPGRADE alone.

TEST 2, what that calldata does to the permission state:

```
git clone --recursive https://github.com/ensdomains/contracts-v2 && cd contracts-v2
git checkout 48b3e2d39513b9dd32ef1850877a29009bc807b9 && git submodule update --init --recursive
cp R3ScopeCollapse.t.sol contracts/test/unit/resolver/R3ScopeCollapse.t.sol
cd contracts && forge test --match-path test/unit/resolver/R3ScopeCollapse.t.sol -vv
```

Output as I ran it:

```
[PASS] test_CONTROL_rootOnly_account_is_really_removed()
  CONTROL hasRoles(ROOT_RESOURCE, ROLE_SET_ADDR, grantee): false

[PASS] test_removeUser_leaves_the_root_grant_live()
  hasRoles(ROOT_RESOURCE, ROLE_SET_ADDR, grantee) after removal: true
  grantee re-pointed pay.alice.eth after being 'removed'

[PASS] test_removeUser_leaves_ROLE_UPGRADE_live_and_grantee_can_upgrade()
  hasRoles(ROOT_RESOURCE, ROLE_UPGRADE, grantee) after removal: true
  grantee replaced the resolver implementation after being 'removed'

Suite result: ok. 3 passed; 0 failed
```

The sequence in each case is the operator's: `grantRootRoles(ROLE_SET_ADDR|ROLE_SET_TEXT, B)` and `authorizeNameRoles(blog.alice.eth, ROLE_SET_TEXT, B, true)`, then the removal call the Explorer emits for that row, `authorizeNameRoles(blog.alice.eth, 17, B, false)`. Afterwards B still holds the role at ROOT\_RESOURCE, and the test spends it: `setAddr` on `pay.alice.eth`, a name that was never part of the removal, succeeds. The third case repeats it with ROLE\_UPGRADE and ends with B calling `upgradeToAndCall` on the proxy. The control is the account holding root roles only, where the removal does land on ROOT\_RESOURCE and the read comes back false.

`groupRolesByAccount`, the sidebar's scope expression as cited, and both calldata builders are the real modules, with no mocks or stubs anywhere. The one thing the first test does not do is render the sidebar component, which is why the second takes the calldata it produces and runs it against the contract. The defect is in `apps/portal`, not in the contracts; the Foundry test is here only to show what that calldata does to the permission state, which the front-end test cannot observe.

Both test files are in the linked secret Gist.

Bug found and verified by me. — Sikoh
