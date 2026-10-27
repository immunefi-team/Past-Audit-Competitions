# 92899 w a medium stale v1 resolver snapshot during migration reinstalls a removed attacker operated resolver

**Submitted on Sep 13th 2026 at 05:15:13 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92899
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

I found that Manager treats the V1 subgraph's cached resolver address as transaction-authoritative during V1-to-V2 migration. If a victim replaces an attacker-operated resolver after Manager caches it, the migration can reinstall the removed resolver and verify that state as successful. V2 resolution then returns attacker-controlled records. I assess this as a Medium instance of the impact "Hijacking or bricking an ENS name during a Manager flow" because it depends on historical trust and victim timing, and the V2 owner can recover.

## Vulnerability Details

The affected asset is `audit-comp-ens/apps/manager` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. It promotes an eventually consistent V1 resolver pointer into authoritative V2 calldata without comparing it with the current V1 Registry resolver.

### Root cause and check/use gap

`audit-comp-ens/apps/manager/src/features/migration/service/v1SubgraphClient.ts:getV1NamesForAddress()` requests `resolver { address }` from the V1 subgraph at lines 22-47 and returns the resulting domains at lines 114-137. `audit-comp-ens/apps/manager/src/features/migration/hooks/useV1Names.ts:v1NamesQueryOptions()` keys this inventory only by wallet address and marks it fresh for five minutes at lines 12-23.

`audit-comp-ens/apps/manager/src/features/migration/service/buildMigrationPlan.ts:buildMigrationPlan()` classifies the cached `V1Domain[]` at lines 317-329 and builds batches at lines 355-380. A non-public custom resolver selects `keep-v1`; `audit-comp-ens/apps/manager/src/features/migration/service/encodeMigration.ts:resolverFor()` returns `v1ResolverAddress` at lines 24-50.

`audit-comp-ens/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:buildNameExecution()` computes `expectedResolver` from the snapshot and copies it into `Migration.Data.resolver` at lines 479-491, then creates the `name-resolver` postcondition from it at lines 544-565. Lines 742-780 pass this data to `buildMigrationHelperCall()`, which encodes it in `MigrationHelper.migrate()` at `audit-comp-ens/apps/manager/src/features/migration/service/buildMigrationHelperCall.ts` lines 60-86.

The final rebuild still passes existing classifications to the batch builder at `migrationService.ts` lines 886-927. Retry validation at lines 705-752 reads token ownership, not V1 `resolver(node)`, and approval refresh at lines 1169-1204 also omits it.

After execution, `audit-comp-ens/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:checkExpectation()` reads the V2 `getResolver(label)` at lines 263-275 and compares it to the stale expected value. The exact state introduced by the bug passes verification.

Official `MigrationHelper.migrate()` transfers the real token but forwards app-supplied `Migration.Data`. `UnlockedMigrationController.onERC721Received()` clears V1 state and calls `_inject(md)`, whose `ETH_REGISTRY.register()` uses `md.resolver` without reading V1. My PoC executes these contracts and observes the removed resolver in V2.

### Preconditions and attack path

Victim preconditions:

* The victim owns an unmigrated, unwrapped V1 `.eth` name and previously chose an attacker-operated custom resolver.
* Manager has fetched the victim's V1 inventory while that resolver is still configured.
* The victim replaces it with a safe V1 resolver, then proceeds with the ordinary Manager migration before the five-minute client cache or ordinary indexer delay has refreshed the resolver field.

Attacker preconditions:

* The attacker still controls the previously trusted resolver, but needs no victim key, registry role, or HCA session.
* The attacker cannot force the resolver replacement, indexer delay, or victim migration. The attacker can only observe the public migration and begin returning attacker-selected records once the old resolver is reattached.

Attack sequence:

1. Manager obtains a valid snapshot in which `recipient.eth` uses the attacker's custom resolver `R_old`.
2. The victim changes authoritative V1 state to safe resolver `R_safe`. V1 resolution is now safe, while Manager's cached/indexed domain still contains `R_old`.
3. The victim starts or continues the normal migration. Manager classifies `R_old` as `keep-v1`, never reads the current V1 Registry resolver, and encodes `R_old` in `Migration.Data.resolver`.
4. The official migration contracts transfer the victim's actual V1 token and register the victim as V2 owner, but install `R_old` as the V2 resolver.
5. Manager reads `R_old` from V2 and marks its own stale `name-resolver` expectation satisfied.
6. The attacker makes `R_old` answer the victim name's ETH address record with the attacker address. My PoC resolves through the official V2 Universal Resolver and receives that attacker address.
7. The victim can recover future resolution by changing the V2 resolver, but resolution remains attacker-controlled until detection and repair.

### Distinction from stale former-manager restoration

The confirmed stale-manager issue snapshots V1 `owner.id`, derives `managerAddress`, and emits a separate `grantRoles(..., ROLE_SET_RESOLVER, formerManager)` call. My report instead covers `resolver.address` embedded directly in `Migration.Data.resolver`. My PoC keeps owner equal to registrant, so `managerAddress` is null and no manager grant exists.

The minimal fixes differ: a live V1 owner check does not detect a changed resolver, while a live `resolver(node)` check does not detect a changed manager. A broad reconciliation can fix both, but each field needs separate validation.

## Impact Details

I chose Medium under the project-authored impact "Hijacking or bricking an ENS name during a Manager flow". My PoC installs an attacker-operated resolver even though authoritative V1 state no longer points to it, and official V2 resolution returns the attacker's address.

The likelihood and recovery constraints are material. The victim previously trusted the custom resolver and must migrate during a cache or indexing window the attacker cannot force. The V2 owner can replace the resolver after detection, and the attacker receives neither ownership nor a V2 role. These constraints support Medium rather than Critical.

## Recommended Remediation

I recommend reading the current V1 Registry `resolver(namehash(name))` immediately before building each final migration batch. Compare it with the resolver shown in the reviewed plan. If it differs, fail closed, invalidate the inventory, rebuild the plan and calldata, show the change, and require fresh confirmation.

Generate the `name-resolver` expectation only from the freshly validated value. Repeat validation after approvals and on every retry. For stronger atomicity, add an expected-current-resolver guard to the migration contract or make the receiver read V1 during execution.

## References

* [Pinned V1 inventory query](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L47)
* [Pinned five-minute V1 inventory cache](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23)
* [Pinned migration plan construction](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L380)
* [Pinned resolver encoding and expectation](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L479-L565)
* [Pinned final batch rebuild](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L886-L963)
* [Pinned post-state verifier](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L263-L275)
* [Official MigrationHelper execution](https://github.com/ensdomains/contracts-v2/blob/068b9dab2bd1ac0dfd8159b18bc4fd9a890ca781/contracts/src/migration/MigrationHelper.sol#L94-L113)
* [Official unlocked migration receiver](https://github.com/ensdomains/contracts-v2/blob/068b9dab2bd1ac0dfd8159b18bc4fd9a890ca781/contracts/src/migration/UnlockedMigrationController.sol#L92-L165)

## Link to Proof of Concept

<https://gist.github.com/niafreu/cda0a2d0888dbc52b968bc5dcd03f6e1>

## Proof of Concept

## Proof of Concept

My PoC is at <https://gist.github.com/niafreu/cda0a2d0888dbc52b968bc5dcd03f6e1>.

### Prerequisites

* Node.js 22 and Corepack.
* Foundry with Solidity 0.8.27 available.
* Git and the attached `pocs/stale-v1-custom-resolver/` directory.

### Setup and command

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git audit-comp-ens
git -C audit-comp-ens checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack pnpm@10.27.0 --dir audit-comp-ens install --frozen-lockfile

git clone --recursive https://github.com/ensdomains/contracts-v2.git contracts-v2
git -C contracts-v2 checkout --detach 068b9dab2bd1ac0dfd8159b18bc4fd9a890ca781
git -C contracts-v2 submodule update --init --recursive

chmod +x pocs/stale-v1-custom-resolver/run-poc.sh
./pocs/stale-v1-custom-resolver/run-poc.sh "$PWD/audit-comp-ens" "$PWD/contracts-v2"
```

### What the PoC proves

My first stage imports Manager's real `buildMigrationPlan()` and `verifyAtomicMigrationBatch()` implementations. It creates a stale V1 object whose cached resolver is `R_old`, while the authoritative resolver spy returns `R_safe`.

The test proves that the live spy is never called. It decodes the production `MigrationHelper.migrate()` calldata and checks that `resolver` is `R_old` and `subregistry` is zero. It then gives the official post-state to the production verifier and proves that both expectations pass.

My second stage uses the official contracts-v2 `MigrationControllerFixture`, `MigrationHelper`, `UnlockedMigrationController`, `PermissionedRegistry`, and `UniversalResolverV2`. It performs these transitions locally:

1. Register and premigrate the victim's unwrapped `recipient.eth`.
2. Set V1 to attacker-operated `R_old` and retain that value as the Manager snapshot.
3. Change authoritative V1 state to `R_safe` and assert the change succeeded.
4. Approve the official helper and execute the exact zero-subregistry `MigrationHelper.migrate()` tuple generated by Manager. Both stages assert the same calldata hash.
5. Assert the official V1 token is retired, the V2 owner is the victim, and V2 `getResolver("recipient")` is the removed `R_old` rather than `R_safe`.
6. Resolve through the official V2 Universal Resolver and assert that it uses `R_old` and returns the attacker address. A fresh-snapshot control instead migrates `R_safe` with zero subregistry and resolves to the victim.

### Expected output

```
manager-batch.test.ts > production Manager stale custom-resolver migration batch > encodes the old resolver and declares that old resolver successful
Test Files  1 passed (1)
Tests       2 passed (2)

[PASS] test_ManagerCalldataReattachesRemovedResolverAndResolvesToAttacker()
[PASS] test_FreshSafeSnapshotKeepsSafeResolution()
2 tests passed, 0 failed, 0 skipped
```

My Manager stage also prints JSON with `authoritativeV1Resolver` equal to `R_safe`, `expectedV2Resolver` equal to `R_old`, the complete helper calldata and its hash, and `verifierAccepted: true`.
