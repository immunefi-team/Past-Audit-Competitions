# 91909 w a critical grouped resolver role removal signs a single resource revoke leaving a removed delegate able to replace the ens address

**Submitted on Sep 5th 2026 at 12:57:43 UTC by @dxhod for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91909
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

When one address has Permissioned Resolver roles on more than one resource, Portal combines them into a single user row. Clicking **Remove user** says the account will be removed from all roles, but Portal builds a revoke for only one name. A root role or a role on another name remains active, so the supposedly removed delegate can still change ENS records.

I reproduced this with a root `ROLE_SET_ADDR` grant and an `example.eth` `ROLE_SET_TEXT` grant. After the owner completed the removal shown by Portal, the delegate's root address role was still effective and the delegate successfully replaced the address record for `example.eth`.

## Vulnerability Details

The issue starts in `groupRolesByAccount`:

```typescript
const existing = grouped.get(account)
if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)
  if (resolvedName) existing.resolvedNames.add(resolvedName)
}
```

This deliberately produces one row per account even when the underlying grants belong to different resources. The row therefore contains a union of roles and multiple resolved names.

The sidebar then reduces that multi-resource state to a single string:

```typescript
const decodedRoles = row?.original.decodedRoles ?? []
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The removal action combines that one name with every decoded role:

```typescript
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

Finally, `prepareRevokeResolverRolesTransaction` treats any non-empty name as one name-scoped revoke:

```typescript
name === ''
  ? { scope: 'root', roles, ... }
  : { scope: 'name', name, roles, ... }
```

For the root-plus-`example.eth` fixture, the resulting calldata is one `authorizeNameRoles(example.eth, ROLE_SET_ADDR | ROLE_SET_TEXT, delegate, false)` call. It never calls the root-role revoke function.

This is important because ENS EAC stores root and name permissions separately. Root roles apply to every name, but revoking roles for one name does not remove the stored root grant. The Permissioned Resolver behaves as designed; the incorrect transaction is constructed in the scoped Portal code.

The confirmation shown to the owner is unambiguous:

> Are you sure you want to remove this user from all roles?

There is no indication that only one resource will be affected.

Relevant source locations:

* `apps/portal/src/lib/roles/resolverRoles.ts:179-220`
* `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:69-81`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-170`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:307-323`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:441-463`
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:28-74`

## Impact Details

An owner can sign what Portal presents as a complete removal while leaving the former delegate with usable authority. In the PoC, that surviving authority is used to change the address record of `example.eth` to an attacker-controlled address.

This matches the selected impact, **malicious interaction with an already-connected wallet / modifying transaction arguments or parameters**. The requested operation covers every role and resource represented by the row, while the transaction sent to the connected wallet covers one resource only. The ENS competition information also explicitly prioritizes transactions built or signed with incorrect arguments.

The deterministic result is unauthorized resolver-record control, not automatic theft. A later payment resolved through the modified ENS address could be redirected, but I do not rely on that downstream event or claim a fixed monetary loss. The owner can recover after noticing the issue by separately revoking the residual root role and restoring the record. Until then, the removed delegate remains able to modify the affected record.

The delegate is an ordinary user-created resolver delegate, not a protocol administrator, operator, keeper, or project-trusted role. The exploit becomes possible specifically after the owner uses the normal flow intended to revoke that access.

## References

* ENS competition scope and impacts: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* ENS competition information: <https://immunefi.com/audit-competition/audit-competition-ens/information/>
* ENS Enhanced Access Control documentation: <https://docs.ens.domains/ensv2/enhanced-access-control/>
* ENS Sepolia deployments: <https://docs.ens.domains/learn/deployments/>
* Full PoC files: <https://gist.github.com/dxhod/b4fa92e4e598d71fff3ec24b0c7121d1>

## Link to Proof of Concept

<https://gist.github.com/dxhod/b4fa92e4e598d71fff3ec24b0c7121d1>

## Proof of Concept

## Proof of Concept

I used the frozen Portal revision `cda79acaad59711b943fc68207ebb3f1d0ff8596`. The secret Gist contains the complete Portal test, complete Solidity test, and a README:

<https://gist.github.com/dxhod/b4fa92e4e598d71fff3ec24b0c7121d1>

### Part 1: prove the transaction Portal prepares

1. Clone and check out the scoped repository:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
```

2. From the Gist, save `ResolverRolesSidebar.grouped-removal.poc.test.tsx` to:

```
apps/portal/src/features/resolver/components/ResolverRolesSidebar.grouped-removal.poc.test.tsx
```

3. Install the competition dependencies, then run:

```bash
corepack pnpm --filter portal exec vitest run \
  src/features/resolver/components/ResolverRolesSidebar.grouped-removal.poc.test.tsx
```

Expected result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

The test uses the real `groupRolesByAccount`, real `ResolverRolesSidebar`, and real ENSJS encoder. Presentation and wallet-query dependencies are mocked only so the test can capture the transaction intent without sending anything.

It creates two grants for the same delegate:

* root resource: `ROLE_SET_ADDR`
* `example.eth`: `ROLE_SET_TEXT`

It then clicks the actual Remove user flow and checks that:

* Portal displays one grouped row containing both resources and both roles;
* the dialog says the user will be removed from all roles;
* only one transaction is prepared;
* decoding its calldata yields `authorizeNameRoles` for `example.eth`, with the unioned role bitmap and `false`.

### Part 2: prove the residual permission is usable

1. Clone the ENSv2 contracts. The commit below is the source linked by the official Sepolia `PermissionedResolverImpl` deployment:

```bash
git clone --recursive https://github.com/ensdomains/contracts-v2.git
cd contracts-v2
git checkout 97a57293f3b4279d94b571e678edb53ce62638f4
git submodule update --init --recursive
```

2. From the Gist, save `PortalGroupedResolverRoleRemoval.t.sol` to:

```
contracts/test/poc/PortalGroupedResolverRoleRemoval.t.sol
```

3. Run:

```bash
cd contracts
forge test --match-path test/poc/PortalGroupedResolverRoleRemoval.t.sol -vv
```

Expected result:

```
[PASS] test_portalSingleNameRevokeLeavesRootAuthorityUsable()
[PASS] test_singleNameNegativeControlIsFullyRevoked()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The positive test applies the exact operation constructed by Portal. It proves that the name-scoped text role is removed, the root `ROLE_SET_ADDR` remains stored and effective for `example.eth`, and the former delegate can call `setAddr` to replace the record.

The negative control grants only a name-scoped `ROLE_SET_ADDR`, revokes it, and confirms that the same `setAddr` call reverts. This rules out the possibility that the mutation succeeds for a reason unrelated to the surviving root grant.

I repeated the combined tests three times. Each run produced the same result:

```
SCA_EVM_POC portal_grouped_revoke=1 root_role_survives=1 post_removal_addr_mutation=1 negative_control=1
```

No production transaction, private key, or third-party service is used. The contract test is included only to demonstrate the consequence of the scoped Portal calldata.
