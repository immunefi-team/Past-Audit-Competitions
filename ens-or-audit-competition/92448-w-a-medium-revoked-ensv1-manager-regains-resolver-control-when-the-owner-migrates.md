# 92448 w a medium revoked ensv1 manager regains resolver control when the owner migrates

**Submitted on Sep 9th 2026 at 21:56:17 UTC by @xylz for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92448
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Manager app preserves a separate ENSv1 manager during migration by reading the manager from the V1 subgraph and later granting that address ENSv2 `ROLE_SET_RESOLVER`. It does not revalidate the address against the live V1 ENS Registry before the owner confirms. If the registrant revokes a manager after the migration page loads, the prepared plan silently restores that former manager's resolver authority. The former manager can then replace the name's resolver while the victim remains the ENSv2 owner.

## Vulnerability Details

At audited commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`:

* [classifyNames.ts lines 148-184](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L148-L184) derives `managerAddress` from the subgraph's `domain.owner.id`.
* [useV1Names.ts line 22](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useV1Names.ts#L22) gives that response a five-minute `staleTime`.
* [preflightChecks.ts lines 47-85](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/preflightChecks.ts#L47-L85) checks the live BaseRegistrar token owner but never the current `ENSRegistry.owner(node)`.
* [buildRoleGrantCalls.ts lines 9-23](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L9-L23) converts the cached address into `grantRoles(resource, 1 << 24, managerAddress)`.
* [buildAtomicMigrationBatches.ts lines 608-623](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L623) includes and verifies that grant.
* [MigrationPage.tsx lines 259-279](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L279) submits the already prepared `gasEstimate.plan`.

The live registrar-owner check and cached registry-manager value represent different authorities. Passing the first does not show that the second remains authorized.

Attack sequence:

1. The victim owns the BaseRegistrar token and has legitimately delegated V1 registry management to a second address.
2. The victim opens the migration page, which reads that manager and prepares a plan preserving it.
3. Before confirming, the victim removes the manager through `BaseRegistrar.reclaim(id, victim)`. The removed manager does not participate.
4. The victim confirms migration from the already open page.
5. Migration completes with the victim as V2 owner, but grants the former manager `ROLE_SET_RESOLVER`.
6. The former manager calls `setResolver` and points the name to an attacker-controlled resolver.

The PoC uses data that was correct when fetched, then changes the live registry state and ensures every later synthetic subgraph response is also corrected. It therefore does not rely on a false or compromised indexer response.

## Impact Details

This matches the in-scope Critical impact: hijacking an ENS name during the v1-to-v2 Manager migration by assigning its resolver to an attacker. The victim retains ownership, so the migration appears successful, while resolution can be redirected by an address the victim explicitly revoked.

## Recommendation

Read live `ENSRegistry.owner(namehash(name))` before finalizing the migration plan and use that value as the only source for any preserved manager. Repeat the check immediately before submission. If it differs from the manager in the plan, discard the plan, refetch, and require a new confirmation. Keep the BaseRegistrar ownership check because it protects a separate invariant.

## Proof of Concept

## Proof of Concept

The reproduction adds one Playwright test and the minimum fork fixtures/configuration needed to run it against the Manager app and the real V1/V2 contracts on a local Sepolia fork.

Run from the competition repository after starting its normal Anvil fork and Manager dev server:

```bash
MANAGER_APP_URL=http://localhost:3001 pnpm --filter @ens-apps/e2e exec playwright test --config=projects/manager/playwright.stale-manager.config.ts
```

The test:

1. Registers a real unwrapped V1 .eth name to the victim.
2. Delegates V1 registry management to a second account.
3. Loads the migration page while that delegation is current.
4. Waits until the Upgrade confirmation is visible.
5. Has the victim reclaim the registry node and verifies the live V1 owner is now the victim.
6. Changes the synthetic subgraph route so every future fetch also reports the victim.
7. Confirms migration through the normal UI and authorizes all four transactions.
8. Verifies bit 24 was granted to the removed manager.
9. Sends `setResolver` from the removed manager.
10. Verifies the resolver changed while the victim remained the V2 owner.

Latest run:

```
1 passed (20.2s)

liveV1ManagerAtMigration: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
staleIndexedManager:       0x70997970C51812dc3A010C7d01b50e0d17dc79C8
restoredRoleBitmap:        0x1000000
v2OwnerBefore:             0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
v2OwnerAfter:              0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
resolverBefore:            0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0
resolverAfter:             0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
hijackTransaction:         0x4d3d9b9d6c91e544938e79cd59429e3319c2888793399c673fe00beaac016cf8
```

The transaction hash is from the local Anvil fork. The test exits unsuccessfully if the revoked manager does not receive bit 24, cannot change the resolver, or if the victim loses V2 ownership.
