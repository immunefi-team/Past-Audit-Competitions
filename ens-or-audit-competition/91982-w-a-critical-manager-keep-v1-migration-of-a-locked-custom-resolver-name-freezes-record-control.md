# 91982 w a critical manager keep v1 migration of a locked custom resolver name freezes record control

**Submitted on Sep 6th 2026 at 00:23:56 UTC by @dxhod for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91982
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Manager lets a locked ENSv1 name through migration when `CANNOT_SET_RESOLVER` is burned and the resolver is a custom owner-aware contract. It marks the name ready, keeps that resolver, and asks the wallet to do a one-way upgrade.

After migration the user still owns the ENSv2 name and the old records still resolve. They cannot update those records or replace the resolver. The V1 wrapped token that authorized writes goes to the Graveyard, and the V2 owner never gets `ROLE_SET_RESOLVER`.

The bug is in the scoped Manager app. The ENSv2 tests below only show what the Manager-built migration actually does.

## Vulnerability Details

`resolverStrategyFor()` treats every transferable locked name with `CANNOT_SET_RESOLVER` as `keep-v1`. A custom resolver is not an ineligibility reason:

```ts
if (cannotSetResolverLocked) {
  return 'keep-v1'
}
```

* `packages/migration/src/service/classifyNames.ts:72-88`
* `packages/migration/src/service/classifyNames.ts:253-280`

Manager is the app that runs this classifier, shows the name as eligible, and builds the wallet transaction. `apps/manager/src/features/migration/service/classifyNames.ts` re-exports it.

The only resolver-membership check runs when `isKnownPublicResolver()` is true. A custom resolver is skipped and `PublicResolverSet.includes()` is never called:

```ts
isKnownPublicResolver(name.v1ResolverAddress)
```

* `apps/manager/src/features/migration/service/migrationInvariants.ts:154-175`
* `apps/manager/src/features/migration/service/migrationInvariants.ts:177-208`

Record-safety does nothing here either. It only inspects a locked name when the resolver is going to be replaced. Because the custom address is preserved, there is no candidate:

* `apps/manager/src/features/migration/service/buildMigrationPlan.ts:160-193`

The owner bitmap then omits `ROLE_SET_RESOLVER` whenever that fuse is burned:

```ts
if (!hasFuse(fuses, FUSES.CANNOT_SET_RESOLVER)) {
  roleBitmap |= ROLE_SET_RESOLVER
}
```

* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:367-381`

That matches `LockedWrapperReceiver`, and it is safe for a known public resolver that still accepts the new owner. It is not safe when Manager also preserves an owner-aware V1 resolver. Before migration, that resolver authorizes writes through V1 ownership. After migration the Graveyard holds the V1 token, so the new V2 owner cannot write and cannot replace the resolver.

The UI does not warn. Selection says "Your names are ready to upgrade". The wallet step says "Migrate the selected names and restore their records":

* `apps/manager/src/features/migration/components/SelectNamesStep.tsx:98-100`
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:92-108`

### Deployment Parity

Manager uses the Sepolia `LockedMigrationController` at `0x5c39E36a69A9897F08954c71aCB1F36E0Bd4f409`. Its runtime and immutable configuration match the pinned artifact used by the PoC. This was verified passively; no transaction was sent.

This issue is distinct from WEB-806, WEB-1304, and QA-01: none covers Manager admitting a locked custom-resolver name and migrating it without `ROLE_SET_RESOLVER`. The official CLI rejects the same `CANNOT_SET_RESOLVER` configuration:

`https://github.com/ensdomains/ens-cli/commit/30a5b4329954c238fe2ee758f90bf5e54562a204`

## Impact Details

A normal owner can trigger the issue through the supported Manager migration flow; no attacker-controlled transaction is required.

The affected name is a transferable locked ENSv1 `.eth` name using a custom owner-aware resolver, with `CANNOT_UNWRAP` and `CANNOT_SET_RESOLVER` burned. Manager marks it as ready and preserves its resolver during migration, while the V2 owner receives no `ROLE_SET_RESOLVER`.

After migration, the owner retains the V2 token and existing records remain readable, but record updates and resolver replacement revert. A stale address or contenthash can therefore continue resolving indefinitely.

The owner cannot recover resolver control while retaining the name. Renewal preserves the broken state, while expiry makes the name publicly registerable rather than restoring control. This matches the competition’s ENS-name bricking impact during a Manager V1-to-V2 migration.

## References

* Manager asset: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager`
* Competition scope and known issues: `https://immunefi.com/audit-competition/audit-competition-ens/scope/`

## Link to Proof of Concept

<https://gist.github.com/dxhod/9835562b3f6c1cd8ff540a0c95a69393>

## Proof of Concept

## Proof of Concept

Local only. No public-network transaction.

Pinned Manager revision: `cda79acaad59711b943fc68207ebb3f1d0ff8596`.

Full patch files are provided in the secret Gist field below.

### 1. Manager admits the name

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
corepack pnpm install --frozen-lockfile
git apply 2026-09-06-ens-manager-h01-manager-poc.patch
corepack pnpm --filter manager exec vitest run src/features/migration/service/lockedCustomResolver.brick.poc.test.ts --reporter=dot
```

Expected:

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

The test shows Manager classifies the name as eligible `keep-v1`, the resolver-membership invariant returns without reading `PublicResolverSet`, and the owner bitmap has no `ROLE_SET_RESOLVER`.

Patch: `2026-09-06-ens-manager-h01-manager-poc.patch`.

### 2. Resulting state

This uses the ENSv2 migration contracts only to show the state Manager's transaction leaves behind. The contracts are not the reported asset.

```bash
git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git
cd contracts-v2
git checkout 48b3e2d39513b9dd32ef1850877a29009bc807b9
git submodule update --init --recursive
git apply 2026-09-06-ens-manager-h01-contract-evidence.patch
cd contracts
forge test --match-contract LockedMigrationControllerTest --match-test test_migrate_lockedOwnerAwareCustomResolverLosesWriteAuthority -vv
forge test --match-contract ETHRegistrarTest --match-test test_renew_thirdPartyCanExtendAnotherOwnersName -vv
```

Expected for each:

```
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The first test shows:

* V1 wrapped token moves from the user to the Graveyard
* V2 token stays with the user
* custom resolver and old text record still read
* updating the record reverts
* replacing the resolver reverts (`ROLE_SET_RESOLVER` missing)
* renewal keeps the same resolver and the same missing role past the original expiry

The second test shows an unrelated funded address can extend someone else's registration without becoming owner.

Foundry may print the repo's existing non-fatal `unresolved symbol _exists` for a vendored ENSv1 dependency. The selected tests still pass and exit 0.

Existing Manager migration tests still pass: 4 files, 61 tests.
