# 92596 w a medium manager v1 v2 migration restores a removed v1 manager as persistent v2 resolver authority hijacking the name s resolver

**Submitted on Sep 10th 2026 at 20:30:32 UTC by @fuyuki0 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92596
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief / Intro

When a name is migrated from V1 to V2, the Manager takes the name's manager and resolver from a cached subgraph snapshot and does not verify either value against the live V1 Registry before performing the migration.

This creates a dangerous stale-state condition. For example, suppose an address previously controlled a name, but the current owner later recovered the name, removed that party, and changed the resolver to a trusted one. If the Manager still has the older snapshot cached, migrating the name can silently restore the previous resolver and grant the removed manager permanent on-chain `ROLE_SET_RESOLVER` authority.

The issue does not stop at the migration itself. The granted role is never revoked, so the removed manager can continue changing the resolver even after the name is transferred or sold to another owner.

This issue is within the scoped `apps/manager` / `packages/migration` code. The subgraph itself can return correct information; the stale state comes from the Manager's own client-side cache and the fact that the migration flow never revalidates these values on-chain.

## Vulnerability Details

Affected commit:

`1c9b47f18fcddd2e864dfe385c4171061c9811ae`

In `useV1Names.ts:17-22`, the V1 name data, including `owner.id` and `resolver.address`, is cached for five minutes without polling.

That cached snapshot is later consumed by `classifyNames.ts:137-185`, which maps:

* `owner.id` → `managerAddress`
* `resolver.address` → `v1ResolverAddress`

Neither value is subsequently revalidated against the live V1 Registry.

`preflightChecks.ts:47-101` does perform an on-chain ownership check, but only through `BaseRegistrar.ownerOf`. It never reads the current V1 Registry `owner(node)` or `resolver(node)`.

Similarly, `migrationService.ts:1161-1204` refreshes approval state before migration, but does not refresh or revalidate the manager or resolver.

As a result, the cached values directly influence the final on-chain migration.

In `encodeMigration.ts:24-50`, the cached `v1ResolverAddress` is included in the migration data, meaning an outdated resolver can be restored during migration.

More importantly, `buildRoleGrantCalls.ts:11-26` constructs:

`grantRoles(resource, 1 << 24, managerAddress)`

where `managerAddress` comes from the same cached V1 owner value.

The resulting role grant is included and executed through `buildAtomicMigrationBatches.ts:608-624,774-780`.

From the user's perspective, there is no meaningful warning that this stale address is about to regain authority. `WalletConfirmationStepsDialog.tsx:81-108` only describes the operation as:

> "Restore the existing managers for your names"

It does not show the actual manager address being restored or the resolver that will be used.

The cleanup path does not correct the problem either. `migrationService.ts:446-478` removes the HCA approval, but there is no corresponding `revokeRoles` operation for the restored manager.

The role therefore remains active until it is explicitly revoked. The PermissionedRegistry stores roles per account, so the previously removed manager retains `ROLE_SET_RESOLVER` after migration and even after a later transfer of the name.

The PermissionedRegistry behavior is part of `ens-contracts` and is not itself the reported vulnerability; it is referenced only to explain why the impact persists. I also verified this behavior against the deployed contract.

I reviewed the on-chain reads used throughout the migration path and found no point where the Manager reads the live V1 Registry's `owner(node)` or `resolver(node)` before using these values.

This is therefore not simply a case where a refresh happens too late. The migration flow never consults the authoritative on-chain state for these two values at all.

A correct subgraph response cannot prevent the issue once an older response has already been cached by the client.

## Impact Details

This matches the program's Critical impact:

> "a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker."

In this case, migration can restore a resolver associated with a previously removed manager and, more seriously, grant that address durable permission to change the resolver again through `ROLE_SET_RESOLVER`.

After migration, the removed manager can point the ENS name to attacker-controlled addresses. I verified this behavior for the ETH address record as well as one non-ETH coin type.

This also directly matches the listed impact of:

> "resolving an ENS name to an attacker-controlled address"

The victim does not need to approve an obviously malicious transaction. They only perform the expected V1 → V2 migration flow, while the interface does not expose the stale manager address or resolver being restored.

The vulnerability should also not be confused with QA-01. QA-01 concerns combinations of roles interfering with later actions. This issue is different: an address that the current owner had already removed can regain resolver control as a direct result of migration.

## References

* Asset: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager`
* Commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* Relevant files:
  * `useV1Names.ts`
  * `classifyNames.ts`
  * `preflightChecks.ts`
  * `encodeMigration.ts`
  * `buildRoleGrantCalls.ts`
  * `buildAtomicMigrationBatches.ts`
  * `migrationService.ts`
  * `WalletConfirmationStepsDialog.tsx`

## Link to Proof of Concept

<https://gist.github.com/Fuyuki0/79d20bdcf1450745ecb6fda5a4bd6a21>

## Proof of Concept

## Proof of Concept

### Setup

The victim owns the `.eth` token. A previously removed party was the V1 manager, and the Manager application has a cached V1 snapshot that still identifies that address as the manager.

At the time of migration, the live V1 Registry state has already been corrected: both the current owner and resolver point to the victim's intended safe values.

The victim then performs a normal V1 → V2 migration while the application is still using the stale cached snapshot.

### 1. Application-Level Migration PoC

`stale-manager-poc.test.ts` uses the real migration logic from the pinned repository, including `classifyNames`, the resolver-selection logic, and `buildRoleGrantCall`. Only the address lookup is mocked so the test can deterministically reproduce the stale-cache condition.

The test confirms that the stale manager address is propagated into the actual migration flow and produces the following decoded role grant:

`grantRoles(resource, 0x1000000, attacker)`

It then confirms that the restored manager can construct the expected follow-up resolver change:

`setResolver(resource, attackerResolver)`

The cleanup sequence is also inspected and contains only the HCA approval revocation; no corresponding role revocation is performed.

Run with:

`npx vitest run stale-manager-poc.test.ts --reporter verbose`

The test passes.

### 2. Role Persistence Across Transfer

`role-survives-transfer.mjs` validates the persistence behavior against the real deployed V2 `PermissionedRegistry` using a Sepolia fork.

The script:

* grants the attacker `ROLE_SET_RESOLVER`;
* transfers the name token to a new owner;
* verifies that the attacker's role remains assigned after the transfer; and
* confirms that the attacker can still successfully call `setResolver` against the name now owned by the new owner.

Run with:

`anvil -p 8548 --fork-url https://ethereum-sepolia-rpc.publicnode.com --silent &`

then:

`node role-survives-transfer.mjs`

This demonstrates that the authority granted during migration is not tied to the current token owner and survives a subsequent ownership transfer.

### 3. End-to-End Persistence PoC

`combined-persistence-local-evm.mjs` compiles the pinned registry implementation and reproduces the complete persistence sequence on a local EVM.

The script performs the following sequence:

1. the attacker receives `ROLE_SET_RESOLVER`;
2. the victim revokes the HCA approval;
3. the attacker's resolver role remains active;
4. the victim transfers the name;
5. the attacker's role still remains active after transfer;
6. the attacker successfully changes the resolver; and
7. an explicit role revocation finally removes the attacker's authority.

The final receipt reports:

`exploitConfirmed: true`

Run with:

`node combined-persistence-local-evm.mjs`

The explicit revocation at the end serves as a negative control and confirms that the persistent authority is specifically caused by the unrevoked role, rather than by unrelated ownership or approval state.

Full PoC scripts and preserved command outputs are included in the attached archive / linked secret Gist: <https://gist.github.com/Fuyuki0/79d20bdcf1450745ecb6fda5a4bd6a21>.

No mainnet transactions, real wallets, or funds were used.
