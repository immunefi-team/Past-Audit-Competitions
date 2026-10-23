# 92618 w a medium revoked v1 manager receives persistent ensv2 resolver control from an already prepared migration

**Submitted on Sep 10th 2026 at 23:52:08 UTC by @thudlow for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92618
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

ENS Manager can restore resolver authority to a V1 manager who was valid when a migration preview was prepared but was revoked before migration began. For an active, premigrated, unwrapped `.eth` 2LD, the page can retain a ready plan containing the former manager address after the registrant uses the ordinary `BaseRegistrar.reclaim` operation to become the current V1 registry manager. When the registrant starts that prepared migration, the victim-signed HCA batch grants the former manager the persistent ENSv2 `ROLE_SET_RESOLVER` role. The former manager can then redirect or zero the resolver of the migrated name.

## Vulnerability Details

Let `V` be the BaseRegistrar token owner and migration wallet, and let `A` be the V1 `ENSRegistry.owner(node)` when Manager prepares the migration.

The V1 subgraph response supplies both `registrant` and `owner`. For an unwrapped `.eth` name, `packages/migration/src/service/classifyNames.ts:148-186` verifies that `V` is the registrant and copies a distinct `domain.owner.id` into `managerAddress`. This correctly represents the existing manager at preview time, but later code treats that snapshot as authority without binding it to current chain state.

The migration gas-plan query in `apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts:62-110` is keyed by the wallet address, HCA address, selected domain IDs, and a selection revision that changes only when the IDs change. The key omits `managerAddress` and the selected row contents. If the same row changes from manager `A` to `V`, React Query can continue returning the already-ready `{ estimate, plan }`, and `MigrationPage.tsx:259-282` passes that captured plan directly into execution.

This creates the following ordinary sequence:

1. `V` owns the unwrapped `.eth` BaseRegistrar token while `A` manages the V1 registry node.
2. Manager builds a ready migration plan containing `managerAddress=A`.
3. From another context, `V` calls `BaseRegistrar.reclaim` and becomes the live V1 manager. The token owner and selected domain ID do not change.
4. Even if the same domain row refreshes from `A` to `V`, the ready gas-plan query identity is unchanged and the captured plan remains A-based unless an independent gas-query refetch occurs.
5. `V` clicks Upgrade and approves the expected MigrationHelper and HCA transactions.

The live guards do not repair the stale authority. For unwrapped names, `packages/migration/src/service/preflightChecks.ts:47-101` reads `BaseRegistrar.ownerOf(labelhash)` and compares it with `V`; it does not read `ENSRegistry.owner(node)`. Retry handling in `migrationService.ts:705-752` repeats the BaseRegistrar owner check, and batch rebuilding consumes entries from the captured plan with their saved `managerAddress`.

`buildRoleGrantCalls.ts:8-25` encodes:

```solidity
ETHRegistry.grantRoles(resource, 1 << 24, A);
```

The role bitmap is ENSv2 `ROLE_SET_RESOLVER`. `buildAtomicMigrationBatches.ts:608-624` adds this direct manager-role grant after MigrationHelper in the same outer HCA execution. `verifyAtomicMigrationBatch.ts:337-353` then requires `hasRoles(resource, 1 << 24, A)` to be true, so the stale grant is treated as successful migration state. Normal cleanup only revokes the HCA's temporary operator approval and does not revoke the direct role granted to `A`.

In the pinned contracts-v2 `PermissionedRegistry`, `setResolver` checks `ROLE_SET_RESOLVER` and writes the resolver. An approved operator inherits the owner's role-admin authority, so the victim-approved HCA can grant the regular role directly to `A`; removing HCA approval does not remove `A`'s role.

No guaranteed five-minute cache window is claimed. Although one hook requests a five-minute V1 `staleTime`, the shared result-query helper drops that option. The deterministic issue is reuse of an already-prepared same-domain plan.

## Impact Details

The selected in-scope Critical impact is: hijacking or bricking an ENS name during a Manager V1-to-V2 migration by assigning its resolver control to an attacker.

After the victim signs the stale migration, former manager `A` can change the migrated name's resolver to an attacker-selected or zero resolver. This can redirect resolution or make the name inaccessible until `V` notices, explicitly revokes `A`'s V2 role, and repairs the resolver.

The demonstrated authority is token-scoped `ROLE_SET_RESOLVER`. This report does not claim ownership, registry, HCA, transfer, arbitrary-role, irreversible-loss, or direct asset-theft authority. The attacker cannot force the victim's timing: the sequence requires a ready A-based preview, a later reclaim to V without remounting or independently refetching the gas plan, and the victim approving the normal migration transactions. The wallet copy says only “Restore the existing managers” and does not display the recipient address, making the stale grant difficult to notice.

## Suggested Fix

Treat every restored-manager address as security-sensitive state:

1. Include the normalized manager address, chain, and relevant source-state revision in the gas-plan identity and immutable plan digest.
2. Invalidate and rebuild the plan whenever any selected row's manager changes, and display every role recipient before approval.
3. Re-read V1 `ENSRegistry.owner(node)` during preflight and every retry.
4. Enforce `plannedManager == liveManager` atomically before migration and grant. A client-side last-moment read still has a TOCTOU window; MigrationHelper should validate and consume the expected manager before changing V1 state and grant the corresponding V2 role in the same guarded operation. Requiring a deliberate post-migration grant is another safe design.

## Deployment Provenance and References

The competition source revision tested is `cda79acaad59711b943fc68207ebb3f1d0ff8596`, which declares upstream revision `63772fd872af472ced58b009499355f3430c2a86`.

The Manager release pins contracts-v2 PR #388 head `8d1c89350729f87b3968cf41ef3d683951444a44`. The installed ensjs manifest resolves ETHRegistry to `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`, matching that checkout's Sepolia deployment. A read-only `eth_getCode` comparison found its 14,730-byte runtime byte-for-byte equal to the PR #388 deployment artifact after inserting the declared LabelStore immutable. Runtime Keccak-256: `0x99a6ba74173ac220fd9d7a2000a8142cf52d98c7a17ac6abc6d74fa17d8f086c`.

The deployed HCA-aware MigrationHelper is separately pinned at `0xddC597d937618849348E18Db5D631Ce747bCDeEF`. The exact model boundary and reproduction commands are provided in the PoC field.

## Link to Proof of Concept

<https://gist.github.com/majesticio/dc26547b45aa92a9feecd778e866d113>

## Proof of Concept

## Proof of Concept

All testing used synthetic local state or read-only public artifacts. No production transaction, third-party name, wallet session, scan, or denial of service was used.

### Preconditions

* `V` controls the migration wallet and owns an active, premigrated, unwrapped `.eth` 2LD BaseRegistrar token.
* `A` is the V1 `ENSRegistry.owner(node)` when Manager builds the ready migration plan.
* `V` changes the live manager from `A` to `V` with `BaseRegistrar.reclaim` while the migration page remains mounted and the same domain ID stays selected, from a separate context that does not remount, reconnect, or independently refetch the gas-plan query.
* `V` starts the already-ready plan and signs the ordinary MigrationHelper/HCA approvals and atomic migration.
* `A` later submits a V2 `setResolver` transaction.

### 1. Prepare the frozen Manager checkout

Use competition revision `cda79acaad59711b943fc68207ebb3f1d0ff8596`. The successful run used Node.js `v26.7.0`, pnpm `10.27.0`, and Vitest `4.1.11`.

```sh
pnpm install --frozen-lockfile --ignore-scripts
```

### 2. Prove that the ready A-based plan survives an A-to-V row change

A focused React test mounts the production `useMigrationGasEstimate` hook. It first returns the same selected domain ID with manager `A`, waits for a ready plan, then rerenders that ID with manager `V`. The assertions require that `buildMigrationPlan` was called exactly once and that the returned ready plan still contains `A`.

Run the test from `Targets/ens-audit/apps/manager` after placing it at `src/features/migration/hooks/E0002-stale-plan-harness.test.tsx`:

```sh
node node_modules/vitest/vitest.mjs run \
  src/features/migration/hooks/E0002-stale-plan-harness.test.tsx \
  --reporter=verbose
```

Observed:

```
Test Files  1 passed (1)
     Tests  1 passed (1)
  Duration  647ms
```

This isolates the query-identity flaw: a same-ID `managerAddress` change does not rebuild the prepared plan.

### 3. Prove that frozen migration code accepts the stale row and grants A

A second Vitest imports the production classifier, eligibility checks, `buildRoleGrantCalls`, and atomic migration builder. It supplies live `BaseRegistrar.ownerOf=V` and V2 `RESERVED` status for an A-based classified row, records every chain read, and decodes the built grant.

```sh
node node_modules/vitest/vitest.mjs run \
  --config ../../../../Research/campaigns/2026-09-10-ens-stale-manager-migration/harness/vitest.config.mts \
  --dir ../../../../Research/campaigns/2026-09-10-ens-stale-manager-migration/harness \
  stale-manager.test.ts --reporter=verbose
```

Observed:

```
Test Files  1 passed (1)
     Tests  2 passed (2)
  Duration  3.22s
```

The assertions establish:

1. The A-based unwrapped row is classified as eligible.
2. The only live authorization/status reads are `BaseRegistrar.ownerOf` and V2 `getStatus`; no V1 `ENSRegistry.owner` read occurs.
3. The atomic output decodes to `grantRoles(resource, 1 << 24, A)`.
4. A negative control implementing the missing guard rejects planned `A` / live `V` and accepts planned `A` / live `A`.

### 4. Prove the V1 transition and terminal resolver impact

Check out `ensdomains/contracts-v2` at `8d1c89350729f87b3968cf41ef3d683951444a44`, initialize its exact submodules, and use ens-contracts submodule `3b1cc225ccdf64581d5fdc81db574f51ba5c8c09`. The successful run used Foundry `1.7.1` and solc `0.8.27`.

The Solidity test deploys the actual V1 ENSRegistry/BaseRegistrar and the PR #388 PermissionedRegistry, then performs these assertions:

1. `V` owns the V1 token while `A` manages its registry node.
2. `V.reclaim` changes the live V1 manager to `V` without changing token ownership.
3. `A.setResolver` on V2 reverts before the stale grant.
4. A victim-approved HCA caller grants `A` exactly `ROLE_SET_RESOLVER` (`1 << 24`).
5. Removing the HCA's operator approval leaves `A`'s direct role intact.
6. The role survives registry token-ID regeneration through the canonical resource mapping.
7. `A.setResolver` then succeeds and stores the attacker-selected resolver.

Run from the contracts-v2 `contracts/` directory:

```sh
/tmp/foundry-v1.7.1/forge test --offline \
  --match-path test/StaleManagerAuthority.t.sol \
  --match-test test_staleGrantSurvivesHcaCleanupAndChangesResolver -vvv
```

Observed:

```
Ran 1 test for test/StaleManagerAuthority.t.sol:StaleManagerAuthorityTest
[PASS] test_staleGrantSurvivesHcaCleanupAndChangesResolver() (gas: 2567145)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

### Expected result

Before any restored-manager grant is signed or executed, Manager should bind the planned address to the current V1 `ENSRegistry.owner(node)`. If it differs from the preview snapshot, Manager should invalidate the plan, rebuild it, display the new address, and request fresh approval. A revoked manager must receive no V2 role.

### Actual result

The prepared plan retains `A`; every live guard accepts `V` because `V` still owns the BaseRegistrar token; and the victim's atomic migration grants `A` persistent V2 resolver authority. HCA cleanup leaves the role in place, after which `A` can change the resolver.

### Model boundary

* The React hook test mocks the plan builder, preflight, gas estimator, public client, and HCA to isolate query identity and ready-plan retention.
* The composition test imports production classification, eligibility, and atomic planning while modeling chain responses, destination addresses, resolver derivation, and HCA outer-call encoding.
* The Solidity test uses the actual V1 contracts and the exact app-pinned PermissionedRegistry source. It represents the HCA as an approved caller and does not execute the deployed HCA, MigrationHelper, or migration controller.
* The official browser/wallet/HCA/helper path was not executed end to end with a researcher-owned Sepolia name. This is a source-and-harness reproduction. Critical severity is conditional on accepting the demonstrated ordinary-flow reachability.

The manager grant is a separate production call after MigrationHelper, so the helper does not validate or rewrite its recipient. Full harness source and SHA-256 manifests are retained and can be supplied immediately if requested.
