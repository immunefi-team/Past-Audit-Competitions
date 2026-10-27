# 92976 w a medium stale v1 manager state during ensv2 migration resurrects former manager resolver authority and enables name hijacking

**Submitted on Sep 13th 2026 at 15:49:11 UTC by @xenomxbr for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92976
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

ENS Manager snapshots the per-name manager of an unwrapped V1 `.eth` name and later converts that historical address into a direct V2 `ROLE_SET_RESOLVER` grant. If the V1 manager changes after the migration plan is captured but before the final migration batch is constructed, Manager refreshes the required approvals but does not re-read or rebind the manager identity. Consequently, a former V1 manager can relinquish its live V1 authority and still receive new, durable V2 resolver authority through the victim's ordinary migration flow. That authority survives the intended cleanup and allows the former manager to assign an attacker-controlled resolver, enabling ENS name hijacking while the victim remains the V2 owner.

## Vulnerability Details

**Target:** Manager app (`apps/manager/**`)\
**Audited revision:** `63772fd872af472ced58b009499355f3430c2a86`

Let:

* `V` be the victim, `.eth` registrant, and eventual V2 owner.
* `A` be an ordinary per-name V1 manager previously authorized by `V`.

When Manager loads the unwrapped V1 name, the observed V1 Registry manager `A` is classified as `managerAddress`. The migration plan then includes a future call equivalent to:

```
ETHRegistry.grantRoles(resource, ROLE_SET_RESOLVER, A)
```

After this plan is captured and the first name-specific migration approval is mined, `A` can use its still-valid V1 manager authority to call:

```
V1Registry.setOwner(node, V)
```

Immediately before the dangerous migration transaction, the authoritative state is therefore:

```
live V1 manager = V
A's V2 role     = 0
```

At this point, `A` is no longer the live V1 manager and has no V2 authority.

However, the real `executeMigration` path retains `plan.classified` throughout the approval phase. After the required approvals are mined, it rebuilds the atomic migration batch from this retained plan without re-reading the canonical V1 manager. The newly rebuilt calldata still contains:

```
grantRoles(resource, ROLE_SET_RESOLVER, A)
```

The victim then signs and submits the ordinary Hybrid Custody Account (HCA) migration batch. Execution changes `A`'s V2 role bitmap from:

```
0 → 16777216
```

where `16777216` is `ROLE_SET_RESOLVER`.

The normal post-migration cleanup removes the temporary HCA operator approval, but it does not remove the direct role granted to `A`:

```
HCA operator approval = false
A's V2 role           = 16777216
V2 owner              = V
```

`A` can subsequently call the deployed ETHRegistry's `setResolver` function and assign a resolver it controls. The demonstrated resolver returns an attacker-controlled address for `addr(node)`.

The broken security invariant is:

> Any authority restored during migration must be derived from authoritative V1 state that is still valid when the final migration batch is constructed, or execution must prove that the previously reviewed authority state has not changed.

Instead, the vulnerable flow is:

```
V1 snapshot records manager A
→ classification stores managerAddress A
→ migration plan stores grantRoles(..., A)
→ live V1 manager changes A → V
→ approvals are refreshed, but manager identity is not
→ post-approval rebuild consumes retained plan.classified
→ migration grants direct V2 resolver authority to A
```

This is stale-authority resurrection / an authority TOCTOU issue, rather than merely a short-lived query-cache inconsistency. Once the stale migration executes, it creates durable on-chain authority.

### Root-cause locations

* `useV1Names.ts` caches the V1 name response containing the mutable per-name manager.
* `classifyNames.ts` copies the observed V1 manager into `managerAddress`.
* `useMigrationGasEstimate.ts` refreshes preflight information but builds the plan from the already supplied domain objects.
* `migrationUi.machine.ts` captures that plan for execution.
* `migrationService.ts` waits for approvals and later rebuilds the batch from retained `plan.classified` without re-reading the live V1 manager.
* `buildRoleGrantCalls.ts` converts the retained `managerAddress` into a direct `ROLE_SET_RESOLVER` grant.
* `verifyAtomicMigrationBatch.ts` considers the stale planned role to be the expected successful post-state instead of verifying that its recipient is still the authoritative V1 manager.

### Required conditions

1. `V` owns an unwrapped V1 `.eth` BaseRegistrar token.
2. V1 Registry manager `A` differs from registrant `V` when Manager observes the name.
3. Manager captures a migration plan that restores `A` as a manager.
4. The victim starts migration and mines the first name-specific approval.
5. Before the final migration batch, `A` relinquishes the V1 manager slot to `V`.
6. Manager continues using the retained plan without rebinding the manager identity.
7. The victim continues and signs the ordinary remaining migration transactions.

The attacker cannot force the victim to start or finish migration and cannot force any victim signature. Victim continuation is an explicit precondition. The attack uses separate, normally ordered transactions and does not require an impossible atomic race.

The demonstrated affected class is limited to unwrapped V1 `.eth` names for which Manager plans restoration of a separate per-name manager. Wrapped classifications set `managerAddress` to `null` and are not claimed to be affected.

This finding is also distinct from published QA-01. QA-01 describes functional errors after transfers involving certain role combinations. This report demonstrates a different migration-specific stale-authority root cause and a successful security consequence: creation of unintended V2 resolver authority followed by an operational resolver takeover.

## Impact Details

The selected in-scope impact is:

> **Critical — Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer).**

The program expressly includes as an example a migration that assigns the name's resolver to an attacker.

This report demonstrates that exact postcondition:

1. Immediately before final migration construction, `A` is no longer the live V1 manager and has V2 role `0`.
2. The victim-signed migration grants `A` direct `ROLE_SET_RESOLVER`.
3. The role survives the intended HCA approval cleanup.
4. `A` successfully assigns an attacker-controlled resolver.
5. The resolver returns an attacker-controlled address for the victim's ENS node.
6. The victim remains the V2 owner, so ownership alone does not reveal that resolver authority was retained by the former manager.

The impact is operational resolver control, not an incorrect internal value or an unexercised permission. An attacker controlling the resolver can redirect address resolution for the affected ENS name and thereby hijack how the name resolves.

This report does not claim direct NFT theft, V2 ownership transfer, forced victim signatures, or a separately demonstrated downstream funds transfer. The proposed Critical classification is based specifically on the competition's listed Manager-flow impact and its explicit example of a migration assigning resolver control to an attacker.

## References

* [ENS competition scope and impacts](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Audited ENS repository revision](https://github.com/immunefi-team/audit-comp-ens/tree/63772fd872af472ced58b009499355f3430c2a86)
* [`useV1Names.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/manager/src/features/migration/hooks/useV1Names.ts#L17-L22)
* [`classifyNames.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/packages/migration/src/service/classifyNames.ts#L148-L185)
* [`useMigrationGasEstimate.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L81-L104)
* [`migrationUi.machine.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/manager/src/features/migration/state/migrationUi.machine.ts#L150-L158)
* [`migrationService.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/manager/src/features/migration/service/migrationService.ts#L886-L925)
* [`buildAtomicMigrationBatches.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624)
* [`buildRoleGrantCalls.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L11-L26)
* [`verifyAtomicMigrationBatch.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L354)

## Link to Proof of Concept

<https://gist.github.com/NasserddineB/c0ee32d344faa78418ead597fc11e4f4>

## Proof of Concept

## Proof of Concept

Two complementary PoCs are supplied in the secret Gist linked with this report:

1. A mounted Manager lifecycle test proving that the stale V1 manager reaches the real migration plan and `migration.start`.
2. An exact local-EVM test proving that the real post-approval `executeMigration` rebuild grants the former manager `ROLE_SET_RESOLVER`, that the role survives cleanup, and that it can be exercised to install an attacker-controlled resolver.

## Safety

All state-changing transactions are executed against loopback Anvil.

The Sepolia RPC endpoint is used only by Anvil to read historical state and deployed bytecode. No transaction is sent to Sepolia or any other public network. The EVM PoC rejects a non-loopback `LOCAL_RPC_URL`.

## Tested environment

```
Official revision: 63772fd872af472ced58b009499355f3430c2a86
Node:             22.23.2
pnpm:             10.27.0
Foundry/Anvil:    1.7.1
Fork block:       11692735
Fork block hash:  0x84be4c0ff9f395b2a9d90263731933a7f44825334dbabc62153140101b848e83
```

A read-only Sepolia RPC endpoint with historical state at block `11692735` is required.

## Step 1 — Prepare the workspace

Clone and check out the official audited revision:

```
git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens.git ens
git -C ens checkout 63772fd872af472ced58b009499355f3430c2a86
```

Download the files from the linked secret Gist and arrange them so the official checkout and the supplied PoC directory are siblings. GitHub Gists store their files at the top level, so place the three `c16-*` reproduction files inside `C-16-submission/reproductions/` as shown below:

```
workspace/
├── ens/
└── C-16-submission/
    ├── C-16-stale-manager-migration.md
    ├── execution-provenance.md
    ├── README.md
    ├── C-16-MANIFEST.txt
    └── reproductions/
        ├── c16-mounted-lifecycle-bridge.test.tsx
        ├── c16-exact-local-evm.test.ts
        └── c16-vitest.config.mjs
```

Install the official repository dependencies from `ens/`:

```
cd ens
corepack pnpm install --frozen-lockfile
```

The supplied PoCs do not modify or patch the application source.

## Step 2 — Security-state sequence reproduced by the PoC

Let:

* `V` be the victim and `.eth` registrant.
* `A` be the separate V1 per-name manager observed by Manager when the migration is planned.

The reproduced sequence is:

```
1. Manager observes A as the live V1 manager.
2. Manager classifies A as managerAddress.
3. The captured migration plan contains grantRoles(resource, ROLE_SET_RESOLVER, A).
4. V mines BaseRegistrar.approve(MigrationHelper, tokenId).
5. Before the remaining approval and final batch, A calls V1Registry.setOwner(node, V).
6. The authoritative state is now:
      live V1 manager = V
      A V2 role       = 0
7. V mines the remaining ordinary migration approval.
8. Real executeMigration performs its post-approval rebuild from retained plan.classified.
9. The newly rebuilt HCA calldata still contains:
      grantRoles(resource, ROLE_SET_RESOLVER, A)
10. V signs and submits the ordinary migration batch.
11. A's V2 role changes from 0 to 16777216.
12. Normal cleanup removes the HCA operator approval.
13. A's direct ROLE_SET_RESOLVER survives cleanup.
14. V remains the V2 owner.
15. A assigns an attacker-controlled resolver.
16. The resolver's addr(node) returns the attacker-controlled address A.
```

The first BaseRegistrar approval is a name-specific, observable indication that migration has started while later transactions remain pending. The PoC does not assume that the attacker can force the victim to continue or sign.

## Step 3 — Run the mounted Manager lifecycle proof

From `ens/apps/manager`, run:

```
corepack pnpm run test C-16-submission/reproductions/c16-mounted-lifecycle-bridge.test.tsx --root ../../.. --config C-16-submission/reproductions/c16-vitest.config.mjs --reporter=verbose
```

Expected marker:

```
C16_MOUNTED_STALE_MANAGER_REACHES_MIGRATION_START
```

This test mounts the real `MigrationPage` under the real `MigrationUiProvider`. In one uninterrupted mount, it:

1. supplies a V1 domain whose observed manager is `A`;
2. changes the controlled authoritative response to `V` without refetching the mounted snapshot;
3. selects the name through the real Manager page callback;
4. proves that the same stale domain object reaches the real gas-plan construction;
5. decodes `grantRoles(resource, 16777216, A)` from that plan; and
6. proves that the same plan reaches the real `executeMigration` input boundary through `migration.start`.

This first PoC establishes application-level reachability. It intentionally stops at the `executeMigration` boundary; the second PoC exercises that real service and the resulting deployed-bytecode execution.

## Step 4 — Start the pinned local Sepolia fork

Open Terminal A at the workspace root and run:

```
anvil --host 127.0.0.1 --port 8545 --fork-url <READ_ONLY_SEPOLIA_RPC> --fork-block-number 11692735 --chain-id 11155111
```

Replace `<READ_ONLY_SEPOLIA_RPC>` with a read-only Sepolia endpoint that supports historical state at the pinned block. Leave Anvil running.

## Step 5 — Run the exact post-approval service/EVM proof

Open Terminal B and change to `ens/apps/manager`.

Set `SEPOLIA_FORK_URL` to the same read-only endpoint supplied to Anvil.

For a POSIX shell:

```
export SEPOLIA_FORK_URL=<READ_ONLY_SEPOLIA_RPC>
```

For PowerShell, use this instead:

```
$env:SEPOLIA_FORK_URL = "<READ_ONLY_SEPOLIA_RPC>"
```

Run:

```
corepack pnpm run test C-16-submission/reproductions/c16-exact-local-evm.test.ts --root ../../.. --config C-16-submission/reproductions/c16-vitest.config.mjs --reporter=verbose
```

Expected markers:

```
C16_POST_APPROVAL_REBUILD_RETAINS_STALE_MANAGER
C16_APPROVAL_SIGNAL_AUTHORITY_RESURRECTION_REPRODUCED
C16_FRESH_AUTHORITY_REBIND_SAFE_CONTROL
```

## Step 6 — What the exact EVM proof verifies

The second test invokes the frozen Manager `executeMigration` service with the real transaction manager against loopback Anvil. It verifies the pinned fork block and configured deployed runtime bytecode before executing the reproduction.

The service then:

1. mines the first BaseRegistrar approval;
2. pauses before the second approval;
3. executes `A → V` through the canonical V1 Registry;
4. confirms that `A` has V2 role `0`;
5. mines the second approval;
6. performs the real post-approval migration rebuild;
7. decodes the newly rebuilt HCA transaction before submission;
8. proves that its inner calls still grant `ROLE_SET_RESOLVER` to `A`;
9. submits those same decoded bytes;
10. completes normal migration cleanup;
11. reads the resulting ownership, approval, role, and resolver state; and
12. exercises the granted role to install the attacker-controlled resolver.

At the final pre-submission wallet boundary, the asserted state and calldata are:

```
live V1 manager = V
A V2 role       = 0
inner targets   = [MigrationHelper, ETHRegistry]
role grant      = grantRoles(resource, ROLE_SET_RESOLVER, A)
```

After execution and normal cleanup, the asserted state is:

```
V2 owner                     = V
A V2 role                    = 16777216
HCA operator approval        = false
ETHRegistry resolver         = attacker-deployed resolver
attacker resolver addr(node) = A
```

Therefore, the direct resolver role is created by migration after `A` has ceased to be the live V1 manager, and it is not removed when the temporary HCA migration approval is cleaned up.

## Step 7 — Safe control

The exact test also builds a control migration plan from the authoritative post-transition state where the observed manager is already `V`.

The real planner then produces:

```
managerAddress = null
manager-role-grant absent
```

This control proves that the unintended grant is caused by the retained stale manager identity. When the same planner receives the fresh authoritative state, it does not grant the former manager any resolver role.

## Expected result

The migration must re-read or otherwise bind the live V1 manager before constructing any authority-restoration call. Because the authoritative manager is `V`, the final batch must not grant any role to `A`.

## Actual result

The vulnerable migration grants the former V1 manager new and durable V2 `ROLE_SET_RESOLVER` authority even though, immediately before final migration construction:

```
canonical V1 manager = V
A's V2 role          = 0
```

The granted authority survives normal cleanup and is operational: `A` uses it to install an attacker-controlled resolver whose `addr(node)` returns `A`.

```
Victim retains V2 token ownership
Former V1 manager receives direct V2 resolver authority
Former manager changes the ENS resolver
The affected ENS node resolves to the attacker-controlled address
```

This reproduces the selected Critical impact: a Manager v1-to-v2 migration assigning resolver control to an attacker and thereby enabling ENS name hijacking.

## Notes

* Local Anvil transaction hashes may differ between runs and are not success criteria.
* If the exact test fails before execution, verify that the RPC endpoint provides historical Sepolia state at block `11692735` and that the checkout is exactly the audited revision.
* Do not change the pinned fork block or configured deployed addresses to make the test pass; those identities are evidence.
* An `IndexedDB unavailable` warning under Vitest is expected; the application transaction manager uses its tested fallback.
* The PoCs do not claim direct token theft, ownership transfer, forced victim signatures, or an impossible atomic interleaving.

## Included evidence

```
C-16-stale-manager-migration.md
execution-provenance.md
README.md
reproductions/c16-mounted-lifecycle-bridge.test.tsx
reproductions/c16-exact-local-evm.test.ts
reproductions/c16-vitest.config.mjs
C-16-MANIFEST.txt
```

The linked secret Gist is supporting evidence; the steps and expected security result are included here so that the report remains understandable and reproducible without opening the Gist.
