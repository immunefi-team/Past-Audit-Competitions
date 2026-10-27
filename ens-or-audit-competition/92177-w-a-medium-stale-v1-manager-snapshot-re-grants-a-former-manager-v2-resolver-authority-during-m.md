# 92177 w a medium stale v1 manager snapshot re grants a former manager v2 resolver authority during migration

**Submitted on Sep 7th 2026 at 21:26:04 UTC by @LucineT for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92177
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Impact

The ENS Manager migration flow can construct and submit a transaction that grants `ROLE_SET_RESOLVER` to an address that is no longer the name's current V1 Registry manager. The former manager can then change the migrated name's resolver.

This is a direct match for the competition's Critical Manager-flow impact class: the migration assigns resolver control to an attacker/former manager. The proof does not rely on compromising the subgraph, wallet, Hybrid Custody Account, or MigrationHelper. It relies on a stale authority value retained by the Manager app and later materialized as a fresh V2 role grant.

Once the former manager can set the resolver, they can redirect name resolution away from the owner's intended resolver. Any wallet, application, or user relying on that ENS resolution can consequently receive attacker-selected resolution data. The local proof stops after demonstrating the unauthorized role and resolver change; it does not transfer funds or touch a third-party account.

The vulnerable authority decision is inside the in-scope Manager application and its transaction-construction path. The contract fixtures are used only as the poststate oracle that proves the generated Manager batch has a real authority effect; this report does not claim a vulnerability in the out-of-scope ENS smart contracts or subgraph. It also requires no browser-extension defect. The owner's action is the ordinary, prompted Manager migration workflow.

## Summary

The migration page obtains V1 name data through `useV1Names`. A previously current V1 manager, `B`, is retained in the mounted query result. After V1 Registry ownership changes to the registrant/current owner `A`, `B` is no longer the current manager. However, the migration path does not re-read the authoritative V1 Registry owner before constructing the final migration batch.

The stale value crosses the complete transaction-construction path:

1. `useV1Names` supplies the mounted domain snapshot containing `B`.
2. `classifyNames` copies `domain.owner.id` into `managerAddress`.
3. `computeMigrationPreflight` refreshes other migration prerequisites but does not freshly bind `managerAddress` to current V1 Registry state.
4. `buildAtomicMigrationBatches` emits `grantRoles(resource, ROLE_SET_RESOLVER, B)`.
5. The owner-authorized migration executes the batch.
6. V2 poststate contains `ROLE_SET_RESOLVER` for `B`.
7. `B` successfully changes the migrated name's resolver.

The decisive negative control changes only the stale/fresh manager read. After query invalidation and a fresh read of `A`, the same path grants no role to `B` and rejects `B`'s resolver write.

## Root cause in the frozen competition source

The program resources name audited revision `63772fd872af472ced58b009499355f3430c2a86`. The official public competition repository's `audit-comp-ready` branch resolved to `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; its public fork base declares that audited upstream revision. The C-ENS-007 mechanism-bearing files are byte-stable from the public fork base through `1c9b47f`.

* [`useV1Names.ts` lines 12–29](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29) obtains the V1 domain snapshot. Its requested `staleTime` is removed by the shared query-options wrapper, but the mounted result still remains the consumed value until refetch/invalidation.
* [`query.ts` lines 120–147](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/utils/src/tanstack-query/neverthrow/query.ts#L120-L147) destructures and does not return `staleTime`.
* [`classifyNames.ts` lines 148–185](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L148-L185) promotes the snapshot owner to `managerAddress`.
* [`computeMigrationPreflight.ts` lines 192–259](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/computeMigrationPreflight.ts#L192-L259) reuses the classified domains without a fresh authoritative manager read.
* [`buildRoleGrantCalls.ts` lines 8–26](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26) encodes the new V2 role grant to `managerAddress`.
* [`buildAtomicMigrationBatches.ts` lines 608–624](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624) includes the grant in the atomic owner batch and expects it in poststate.

The security invariant that fails is:

```
manager used for a new V2 authority grant
    == current authoritative V1 Registry manager at final plan construction
```

The application instead binds the recipient to an earlier mounted snapshot.

## Preconditions

1. `B` is the V1 Registry manager when the migration page obtains its mounted snapshot.
2. Before final migration-plan construction, current V1 Registry ownership changes from `B` to `A`. This state can arise through a genuine registrant reclaim; the local source also shows that a current manager can call `setOwner(node, A)`.
3. The mounted query is not invalidated/refetched before the plan captures `managerAddress`.
4. `A` proceeds with the Manager-generated migration batch.

No malicious wallet, compromised dependency, private key theft, subgraph write, or privileged test hook is required.

## Expected result

Immediately before final batch construction, Manager must bind every authority recipient to current authoritative V1 chain state. If the current manager differs from the earlier snapshot, the plan must be rebuilt or rejected. A former manager must not receive a new V2 role.

## Actual result

The stale snapshot value is converted into new V2 `ROLE_SET_RESOLVER` authority. The former manager can exercise that authority and change the migrated name's resolver. A fresh-read control prevents both effects.

## Why the owner's signature does not remove the defect

The security-sensitive recipient is chosen by the application before the wallet prompt. The transaction is owner-authorized, but the application has constructed the role recipient from obsolete authority state. The issue is therefore the wrong transaction argument and authority recipient, not signature bypass. A wallet signature does not make stale application-selected authority current.

## Distinction from public known issues

* Public QA-01 covers transfer-role combinations that cause later functional errors and explicitly leaves a demonstrated unintended authority consequence as a new finding. C-ENS-007 demonstrates a new V2 role grant and successful resolver write during migration.
* ENS contracts-v2 issue #418 concerns a V2 delegate role that already exists and survives a later transfer. C-ENS-007 has a different root and transition: a revoked V1 manager is used by stale Manager state to create fresh V2 authority during migration.

A finite search of the competition's published known-issue list and official ENS issues found no exact public match for this stale V1 snapshot -> fresh V2 authority grant root cause.

## Source and deployment corroboration

* The mechanism-bearing files are byte-stable from the public fork base through `audit-comp-ready` commit `1c9b47f...`.
* The served `app.ens.dev` migration chunks observed on `2026-09-02` contained the `managerAddress` path, `grantRoles` surface, pinned helper address, and expected helper runtime hash.
* At Sepolia block `11621448`, the pinned helper runtime hash matched the exact runtime invariant embedded in the competition source and served bundle.
* The app-pinned helper is expressly not source-verified in the competition code; therefore the local PoC binds the published/frozen app source and exact PR #402 contract artifacts rather than claiming a vendor-signed reproducible production build that the project has not published.

Current source/version receipt SHA-256: `4aea78825b0f5bb51265289f2b1fd64a7dbf4a0fa44fe9d8c3f161de83845b76`.

## Recommended remediation

1. Immediately before building the migration batch, read the current V1 Registry owner for every name from authoritative chain state.
2. Derive `managerAddress` only from that fresh result.
3. Bind the recipient, role, block reference, and name into the preflight/plan digest.
4. Display the exact role recipient before signature.
5. Reject and rebuild if current ownership changes before submission.
6. Verify poststate against the freshly bound manager, never the earlier mounted snapshot.
7. Add the positive stale-read test and fresh-read negative control as permanent regression tests.

## Proof of Concept

## Step-by-step proof of concept

### PoC A — mounted Manager path through TransactionManager and local EVM

Attachment: `c-ens-007.continuous-cache-manager-evm.security-repro.test.tsx`

SHA-256: `1af8d6cafcfb78d79fe6351a8c14a9cf06128fb9596223d21762946f420976b7`

Place the attachment at:

```
apps/manager/src/features/migration/service/
```

From `apps/manager`, run:

```
pnpm exec vitest run src/features/migration/service/c-ens-007.continuous-cache-manager-evm.security-repro.test.tsx --reporter=verbose
```

The first test performs both the positive and fresh-read control:

1. Mount the real `useV1Names` hook while `B` is the V1 manager.
2. Observe cached manager `B`.
3. Perform a genuine V1 Registry state transition so the current manager becomes `A`.
4. Without invalidating the mounted query, execute exact `classifyNames` and `buildAtomicMigrationBatches`.
5. Submit the generated owner batch through the real app TransactionManager/XState path to the local EVM.
6. Assert that `B` receives `ROLE_SET_RESOLVER`.
7. Execute a resolver change as `B` and assert success.
8. Invalidate/refetch and repeat with current manager `A`.
9. Assert that `B` receives no role and that `B`'s resolver write is rejected.

The second test preserves the same stale recipient through the exact PR #402 `StandaloneHCAFactory`, `StandaloneSingleOwnerHCA.executeByOwner`, `MigrationHelper`, `UnlockedMigrationController`, and local `PermissionedRegistry` artifacts in the same mounted-app process.

Fresh run on `2026-09-07`:

```
Test Files  1 passed (1)
Tests       2 passed (2)
Exit code   0
Duration    25.81s
```

Observed transaction states:

```
c-ens-007-stale-manager-positive       pending -> success
c-ens-007-fresh-manager-negative       pending -> success
c-ens-007-all-real-pr402-positive      pending -> success
```

The Node test emits an expected `IndexedDB unavailable` warning and uses the transaction manager's fallback storage path. Both poststate oracles complete and the suite passes.

### PoC B — exact PR #402 HCA/MigrationHelper discriminator

Attachment: `c-ens-007-real-hca-pr402.t.sol`

SHA-256: `f171113262e889de05bf3bad72a776e8ee8579acd386f836f86eb258a595dc9c`

This preserved network-disabled Foundry discriminator uses ENS contracts commit `0717b0b6f1867f43078f19115dd5dfa4f35426ab` and varies only the manager read:

```
[PASS] test_staleRead_realFactoryAndNexusHCA_regrantsBAndPermitsResolverWrite()
[PASS] test_freshRead_realFactoryAndNexusHCA_doesNotGrantB()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The production migration route used by this source is the wallet-owner `executeByOwner` path. Adding a session/default-executor signature would test a different path and is therefore intentionally excluded.
