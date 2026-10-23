# 92509 w a medium revoked manager regains resolver control during v1 to v2 migration

**Submitted on Sep 10th 2026 at 11:29:59 UTC by @PlasticGinnie98802 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92509
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief / Summary

Manager can restore resolver-setting authority for a V1 manager whom the owner already revoked. The app trusts the V1 subgraph's Registry-owner value when building a V1-to-V2 migration and never checks the live V1 Registry owner before turning that value into a new on-chain V2 permission. If the victim migrates during the indexer's stale-data window, the former manager receives ROLE\_SET\_RESOLVER and can replace the name's resolver after migration and cleanup.

This matches the in-scope Critical impact: hijacking an ENS name during a Manager V1-to-V2 migration.

## Affected versions

* Manager application: audit-comp-ready\@1c9b47f18fcddd2e864dfe385c4171061c9811ae
* HCA-aware MigrationHelper: contracts-v2 PR #402 at 5bbe6add1192c635215031e0b3f20d1e517959e5
* The decisive HCA, factory, Registry and migration-controller files are identical in the PR #402 and PR #409 sources used by the proof.

## Vulnerability details

For an unwrapped V1 name, packages/migration/src/service/classifyNames.ts:125-174 compares the subgraph Registry owner with the registrant. If they differ, it stores the indexed Registry owner as managerAddress. That address is security-sensitive because it decides which manager will be restored after migration.

The following preflight in apps/manager/src/features/migration/service/computeMigrationPreflight.ts:196-235 checks registrant ownership, approvals, resolver setup, migration routes and HCA readiness, but never reads the current V1 Registry owner and compares it with managerAddress.

When managerAddress is present, apps/manager/src/features/migration/service/migrationApprovals.ts:116-123 and :320-350 prepare a temporary V2 operator approval for the HCA. apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9-27 and buildAtomicMigrationBatches.ts:608-628 then append a post-migration grant of ROLE\_SET\_RESOLVER to the indexed address.

Cleanup in migrationApprovals.ts:69-81 and :354-367 and migrationService.ts:455-464 removes only the temporary HCA operator approval. It does not revoke the manager role that the HCA granted during migration. Therefore the old manager's authority persists after the UI reports success.

The subgraph delay is only the trigger. The in-scope security failure is in Manager: an eventually consistent UI read is converted into fresh on-chain authority without a live-state check.

## Attack scenario

1. The victim owns a V1 .eth name and previously assigned an attacker-controlled address as Registry manager.
2. The victim revokes that manager on-chain with BaseRegistrar.reclaim, making the victim the live V1 Registry owner again.
3. Before the subgraph reflects the reclaim, the victim starts the normal V1-to-V2 migration in Manager.
4. Manager sees the victim as registrant, but copies the stale former manager into managerAddress.
5. The victim approves and completes the normal migration.
6. The HCA migrates the name and grants the former manager ROLE\_SET\_RESOLVER.
7. Cleanup removes the temporary HCA operator approval, but the restored manager role remains.
8. The former manager calls the V2 Registry and replaces the name's resolver with an attacker-controlled resolver.

The attacker needs no victim key or HCA key. The victim only performs the intended first-party migration.

## Impact

After a successful migration, an address that was already revoked can redirect the migrated ENS name by replacing its resolver. This can change address records, content resolution and other resolver-backed behavior. This is not a stale display or failed migration: the unwanted authority exists on-chain and survives cleanup.

The report does not claim that every revocation produces a usable window; production subgraph delay has not been measured. The demonstrated impact is conditional on migration during a stale window. When that condition is met, the scoped result is ENS name hijacking during a Manager migration.

## Scope rationale

This is not a report against the ENS subgraph or contracts. Indexers are eventually consistent. The bug is that the in-scope Manager app treats an unverified indexer result as authorization to create a persistent V2 role. A stale read should make the UI briefly inaccurate; it must not resurrect a revoked address.

## Recommended fix

Read the live V1 Registry owner immediately before building or executing the migration role grant. For each unwrapped name, compare that value with the indexed manager candidate. If they differ, discard managerAddress and rebuild the migration plan. To reduce the remaining race window, bind the verified manager and block number to the plan and recheck before HCA execution. The confirmation UI should also list every authority that will survive migration.

## Proof of Concept

## Proof of Concept

The proof has an application-level harness and an exact contract test. Together they reproduce the prior-manager, live-reclaim, stale-subgraph and victim-migration sequence. No public chain, real ENS name, real wallet or real funds are used.

### 1. Application test

From the repository root, run:

```bash
pnpm --filter manager exec vitest run \
  harness/migration-stale-manager-regrant.test.ts \
  --root <REPO_ROOT>/.codex \
  --config <REPO_ROOT>/.codex/harness/vitest.manager-harness.config.ts \
  --reporter=verbose
```

Expected: 1/1 passed.

The unchanged application harness proves that:

* classification copies the stale indexed Registry owner into managerAddress;
* preflight performs no live V1 Registry-manager read;
* the V1 token approval targets the separate MigrationHelper;
* the V2 operator approval targets the HCA;
* the atomic batch migrates first and grants ROLE\_SET\_RESOLVER to the stale manager afterward.

### 2. Exact contract test

The retained fixture uses the HCA-aware MigrationHelper from contracts-v2 PR #402 and the HCA/factory/registry/controller code shared with PR #409.

Run:

```bash
docker run --rm \
  -v <REPO_ROOT>/.codex/external/contracts-v2-pr402:/workspace \
  -w /workspace/contracts \
  ghcr.io/foundry-rs/foundry:v1.3.2 \
  'forge test --match-path test/unit/migration/MigrationHelper.t.sol --match-test test_migrate_unwrappedFromHcaAndRestoreRevokedManager -vvv'
```

Expected: the test passes.

Observed sequence:

1. The victim registers a V1 name and assigns priorManager.
2. The victim calls reclaim, so the live Registry no longer recognizes priorManager.
3. The factory deploys and certifies a real StandaloneSingleOwnerHCA for the victim.
4. The V1 token is approved only to MigrationHelper, and the V2 Registry operator approval only to the HCA.
5. HCA.executeByOwner calls the separate MigrationHelper and then grants the stale manager role, matching Manager's generated batch.
6. Cleanup removes the HCA operator approval.
7. priorManager still has ROLE\_SET\_RESOLVER and successfully replaces the resolver.

The exact replay was repeated unchanged. Evidence is retained under:

* .codex/ens-migration-stale-manager-exact-v4/attempts/A0001/raw/evidence.txt
* .codex/ens-migration-stale-manager-exact-v4/attempts/A0002/raw/evidence.txt
* .codex/ens-migration-stale-manager-exact-v4/reviews/D001.json

The independent reviewer accepted the narrowed conditional claim.

### Preconditions

* The attacker was a legitimate V1 manager before revocation.
* The victim starts migration while the subgraph still reports that former manager.
* The local proof models the stale interval; it does not measure production lag.
* The vulnerability claimed is Manager's recreation of authority, not a bug in the contracts or subgraph.
