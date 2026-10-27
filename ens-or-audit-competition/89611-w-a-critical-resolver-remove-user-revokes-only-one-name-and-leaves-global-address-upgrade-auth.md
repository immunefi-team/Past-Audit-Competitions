# 89611 w a critical resolver remove user revokes only one name and leaves global address upgrade authority active

**Submitted on Aug 20th 2026 at 06:17:08 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89611
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

The resolver roles page merges an account's root and per-name role records into one row. The sidebar labels a mixed row as **“Global roles (all names)”** and the confirmation says the user will be removed **“from all roles.”** However, Save and Remove select only the first non-root name and submit one name-scoped revoke.

As a result, a transaction can succeed while the target keeps every root-scoped role and every role on sibling names. In particular, a target can retain `ROLE_SET_ADDR` and `ROLE_UPGRADE` after the manager removes it, then redirect names to an attacker address or replace the resolver implementation.

The competition classifies connected-wallet interactions that modify transaction arguments as Critical. This flow changes an intended all-resource removal into one name-scoped revoke, while the PoC proves the removed account retains address-changing and upgrade authority. The report does not assume that a victim has already transferred funds to the redirected address.

## Vulnerability details

`groupRolesByAccount` groups records only by account. It appends role bits from every EAC resource into one `decodedRoles` array and collects all root/name labels into `resolvedNames`:

```ts
if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)
  if (resolvedName) existing.resolvedNames.add(resolvedName)
}
```

The sidebar then throws away the resource boundary when selecting the transaction target:

```ts
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Any mixed row containing `'(root)'` and `alice.eth` therefore uses `alice.eth`, even though the UI labels it as global. The Remove path passes the entire union of roles with that one `roleName`:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

`prepareRevokeResolverRolesTransaction` uses root scope only for an empty name. A non-empty `roleName` encodes `authorizeNameRoles(name, bitmap, account, false)`:

```ts
name === ''
  ? { scope: 'root', roles, targetAccount: account, resolverAddress }
  : { scope: 'name', name, roles, targetAccount: account, resolverAddress }
```

The resulting on-chain call changes only the resource for `alice.eth`. It cannot clear the root resource or any sibling-name resource. The transaction nevertheless succeeds because revoking absent bits on that one resource is not an error.

The UI copy makes the mismatch explicit:

```tsx
{resolvedNames.includes('(root)')
  ? 'Global roles (all names)'
  : `Roles scoped to ...`}

Are you sure you want to remove this user from all roles?
```

## Impact

A manager can complete the documented removal flow and receive a successful transaction while the removed account retains global resolver authority.

The attached contract PoC demonstrates both concrete consequences:

1. The target retains root `ROLE_SET_ADDR` and changes the ETH address returned for the name to itself after removal.
2. The target retains root `ROLE_UPGRADE` and replaces the resolver implementation after removal.

This is not a report about the original trusted grant. The security boundary is the successful revocation transaction: the target acts after the manager has removed it, using authority that the signed transaction and UI said would be cleared.

The same root cause also leaves permissions on sibling names active because the sidebar never iterates `row.original.roles`.

## Recommendation

Do not edit a union of resource-scoped roles through one `roleName`.

* Preserve the per-resource records in the edit state.
* For Remove, emit a revoke for every `row.original.roles` entry against that entry's exact resource.
* For Save, compute grants and revokes separately for each resource.
* Encode root records with root scope and name records with their corresponding name.
* If the UI remains grouped by account, show a multi-transaction confirmation that lists every affected resource.

A simpler safe UI is one row per account/resource pair, so one row can never mix root and per-name permissions.

## Target

Scoped repository commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

Affected asset: `apps/portal`

## Link to Proof of Concept

<https://gist.github.com/krutftw/13c2ca99c383abe35295929dac305ea8>

## Proof of Concept

## Proof of concept

The package contains two small tests:

* `poc-resolver-role-collapse.test.ts` exercises the exact in-scope Explorer grouping and transaction builder.
* `ResolverRoleCollapsePoC.t.sol` demonstrates the resulting authorization and resolver consequences in the ENS v2 contracts used by the app. The contracts are context for impact; the vulnerable target is the in-scope Explorer code.

### 1. Explorer transaction PoC

Copy `poc-resolver-role-collapse.test.ts` to:

```
apps/portal/src/features/resolver/components/poc-resolver-role-collapse.test.ts
```

From the repository root, run:

```bash
pnpm --filter portal exec vitest run src/features/resolver/components/poc-resolver-role-collapse.test.ts
```

Expected result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

The test constructs one root `ROLE_UPGRADE` record and one `alice.eth` `ROLE_SET_TEXT` record for the same account. It asserts that the production code:

* groups them into one row;
* selects `alice.eth` as `roleName`; and
* encodes `authorizeNameRoles(packetToBytes('alice.eth'), ROLE_UPGRADE | ROLE_SET_TEXT, target, false)` instead of `revokeRootRoles`.

### 2. Resolver consequence PoC

In `ensdomains/contracts-v2`, copy `ResolverRoleCollapsePoC.t.sol` to:

```
contracts/test/unit/resolver/ResolverRoleCollapsePoC.t.sol
```

Then run:

```bash
cd contracts
forge test --match-path test/unit/resolver/ResolverRoleCollapsePoC.t.sol \
  --match-test test_nameScopedUiRevokeLeavesRootUpgradeUsable -vvv
```

Expected result:

```
[PASS] test_nameScopedUiRevokeLeavesRootUpgradeUsable()
1 passed; 0 failed; 0 skipped
```

The test grants a target root `ROLE_SET_ADDR | ROLE_UPGRADE` and name-scoped `ROLE_SET_TEXT`, executes the exact name-scoped union revoke generated by the Explorer, and proves that the target can still redirect the name and upgrade the resolver.

Secret gist containing both executable tests: <https://gist.github.com/krutftw/13c2ca99c383abe35295929dac305ea8>
