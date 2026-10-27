# 93096 w a medium missing manager revalidation during v1 v2 migration grants resolver control to a revoked manager

**Submitted on Sep 14th 2026 at 08:35:52 UTC by @kyan for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93096
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

ENS Manager builds a V1→V2 migration plan from a cached V1 Registry Manager and never revalidates that mutable authorization recipient before execution. If the registrant canonically reclaims the V1 Manager role to a new address after preview but continues the still-open upgrade, Manager grants the revoked former Manager a new V2 `ROLE_SET_RESOLVER` permission; the former Manager can then select a resolver and make Sepolia's deployed Universal Resolver return attacker-controlled records for the migrated name until the owner detects and revokes the role.

## Vulnerability Details

The affected asset is `apps/manager` at competition revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

`useV1Names` caches the owner-address query for five minutes. Its key is the connected registrant, not each name's mutable Registry Manager:

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

For an unwrapped name, `classifyNames` copies the Registry owner into `managerAddress` when it differs from the BaseRegistrar registrant:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

When the user begins the upgrade, `MigrationPage` passes the already prepared plan into the migration actor. The migration service refreshes approval categories, route state, and resolver readiness, but it neither re-reads the canonical V1 Registry owner nor replaces `managerAddress`. Its approval freshness check is derived from the existing plan, so a confirmed `BaseRegistrar.reclaim` can change the Manager while leaving the same approval categories.

The stale identity reaches the authorization sink:

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

Post-transaction verification checks the same snapshotted address, so it confirms that the erroneous grant executed rather than comparing the recipient with current canonical state. The final confirmation describes restoring “existing managers” but does not show the exact role recipient.

The strongest integrated proof runs the exact frozen `getV1NamesForAddress` operation against official `ensdomains/ens-subgraph` mappings backed by the complete official Rainbow name archive on the same pinned fork. The query first returns Mallory as the Manager and production classification builds the stale plan. Alice then completes `BaseRegistrar.reclaim(tokenId, Bob)`. After the official indexer advances, the same production query returns Bob and a fresh plan targets Bob, while the stale and fresh plans require the same approval categories. Alice's still-open stale plan executes normally and grants Mallory `ROLE_SET_RESOLVER` (`16777216`) while Bob receives zero.

That run executes this continuous authorization chain:

1. Alice owns an eligible unwrapped `.eth` registration. Mallory is the delegated V1 Registry Manager when the preview is built; Mallory has no V2 resolver role.
2. Alice canonically reclaims the V1 Registry Manager to Bob and waits for confirmation. Bob is now the live Manager. Mallory cannot use the tested V1 owner/resolver mutations and still has no V2 role.
3. A fresh production query and plan identify Bob. Alice nevertheless continues the already-open plan, whose approval categories remain valid.
4. Every migration transaction is sent by Alice's EOA. The frozen production migration path grants Mallory `ROLE_SET_RESOLVER`; Bob receives no equivalent role.
5. Mallory deploys a code-bearing resolver, selects it using the newly created role, and Sepolia's deployed Universal Resolver changes from `null` to Mallory's address.
6. Alice explicitly revokes Mallory and restores the resolver. Resolution returns to `null`, and Mallory's repeat mutation fails.

The cause-changing controls are decisive. A freshly rebuilt plan after reclaim targets Bob, not Mallory. Two separately restarted fork runs also reproduce the same stale grant when the Manager transition is performed through owner `BaseRegistrar.reclaim` and through the former Manager's canonical `Registry.setOwner`; the root is the plan's binding time, not a specific transition actor. The official-indexer owner-reclaim run is sequential and uses no watcher, interval mining, fee priority, wallet pause, or concurrent race.

Necessary conditions are explicit: Mallory must legitimately be the delegated Manager when the preview is created; the canonical Manager must change before migration; Alice must continue signing the open flow before the plan is replaced; and Mallory must act before recovery. One name is demonstrated, and recovery is available. This report does not claim ownership theft, autonomous fund theft, permanent bricking, common occurrence, or protocol-wide compromise.

Recommended remediation is to re-read the canonical V1 Registry owner for every selected name immediately before the first wallet prompt, rebuild every Manager-dependent call, and invalidate the plan when any recipient changes. The atomic migration should bind an expected current Manager and revert on divergence. The confirmation should display the exact `ROLE_SET_RESOLVER` recipient, and post-migration verification should compare against current canonical state rather than the plan's snapshot.

## Impact Details

The selected in-scope impact is: **“Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker.”**

The migration creates Mallory's first V2 resolver-control permission after Alice has already revoked Mallory's V1 Manager authority. Mallory then changes the migrated name's resolver and makes the deployed Universal Resolver return Mallory's address. This is the selected impact's enumerated V1→V2 migration case: resolver control is assigned to a revoked former Manager.

The demonstrated lower bound is unauthorized control of one migrated name's resolver and resolution answer until explicit owner recovery. The prior delegation, post-preview Manager change, continued owner signatures, finite plan window, one-name scope, and recoverability limit occurrence and duration. No unsupported monetary upper bound is claimed.

## References

* Affected asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* Cached V1-name query: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts>
* V1 name classification: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts>
* Migration page handoff: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx>
* Migration execution service: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts>
* Role-grant builder: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts>
* Atomic migration builder: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts>
* Confirmation dialog: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx>
* ENS subgraph source used by the integrated control: <https://github.com/ensdomains/ens-subgraph/tree/ad62221d8ebb6a0102a5caa41a23130e2e6251b2>

## Link to Proof of Concept

<https://gist.github.com/erenyeager1483/3e7d89042286b950896ea21e641afdd1>

## Proof of Concept

The attached secret Gist contains the supporting files. It supports, but does not replace, these complete reproduction steps.

**Step 1 — Check out the frozen revision.** Check out `immunefi-team/audit-comp-ens` at revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and install the locked dependencies.

**Step 2 — Place the primary PoC files.** Copy `W6MplPreclick.local-fork.test.ts` to `apps/manager/src/features/migration/service/W6MplPreclick.local-fork.test.ts` and `vitest.w6.preclick.config.ts` to the repository root.

**Step 3 — Start the pinned local fork.** Start a fresh localhost-only Anvil fork of Sepolia block `11549180`, expected hash `0xf122ea293f1371712bd27a5253f03dfd14884c931f7120832e8d2d9cd514ed79`, chain ID `11155111`:

```bash
docker run --rm --name ens-w6-anvil -p 127.0.0.1:8545:8545 \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd \
  "anvil --host 0.0.0.0 --fork-url <ARCHIVE_SEPOLIA_RPC_URL> --fork-block-number 11549180 --chain-id 11155111 --block-time 12"
```

`<ARCHIVE_SEPOLIA_RPC_URL>` must be the reviewer's own archive endpoint. Do not publish its credentials. It seeds Anvil only; every state-changing transaction remains local.

**Step 4 — Run the owner-revocation proof.** From the repository root, run:

```bash
ANVIL_RPC_URL=http://127.0.0.1:8545 \
W6_FORK_BLOCK=11549180 \
W6_FORK_BLOCK_HASH=0xf122ea293f1371712bd27a5253f03dfd14884c931f7120832e8d2d9cd514ed79 \
W6_MANAGER_TRANSITION_MODE=owner-revocation \
W6_ATTACKER_DELAY_MS=9000 \
W6_BLOCK_TIME_SECONDS=12 \
W6_OBSERVER_POLL_MS=50 \
pnpm --filter manager exec vitest run \
  --config ../../vitest.w6.preclick.config.ts \
  --reporter=verbose
```

**Step 5 — Verify the primary result.** Observe one file and one test passing. `W6_MPL_EXACT_TOPOLOGY_PROOF` records at least:

```
previewManager = Mallory
liveManagerBeforeAtomic = Bob
freshPlanManager = Bob
managerTransitionMethod = BaseRegistrar.reclaim
attackerSubmittedNoPreAtomicTransaction = true
attackerRoles = 16777216
currentManagerRoles = 0
everyMigrationTransactionSentByOwnerEoa = true
resolutionBeforeAttack = null
resolutionAfterAttack = Mallory
resolutionAfterRecovery = null
```

**Step 6 — Run the transition-actor control.** Restart Anvil from the same pinned block and repeat with `W6_MANAGER_TRANSITION_MODE=attacker-self-replacement`. The preserved pair passed `1/1` each and the causality matrix records the same erroneous role recipient under both canonical transition actors.

**Step 7 — Review the official-indexer control.** The Gist additionally preserves the source and evidence for the official-indexer integrated owner-reclaim control. That run used official `ensdomains/ens-subgraph` revision `ad62221d8ebb6a0102a5caa41a23130e2e6251b2`, graph-node `0.45.0`, and the complete official Rainbow archive with SHA-256 `9e7d1686cd9f3a139a4d1ceadc9004b26f8622b144d4f25c9a8724213ddbf136` and `133856895` copied rows. No mapping, ABI, schema, production query, variable, parser, or classification code was edited.

**Step 8 — Reproduce the official-indexer control.** After provisioning that stack, copy `W6MplOfficialIndexer.local-fork.test.ts` to the same service directory and `vitest.w6.official-indexer.config.ts` to the repository root, then run:

```bash
ANVIL_RPC_URL=http://127.0.0.1:8545 \
W6_LOCAL_V1_SUBGRAPH_URL=<LOCAL_OFFICIAL_V1_SUBGRAPH_URL> \
W6_PRODUCTION_V1_SUBGRAPH_URL=<V1_SUBGRAPH_URL_FROM_FROZEN_SOURCE> \
W6_INDEXED_LABEL=<LABEL_PRESENT_IN_RAINBOW> \
W6_FORK_BLOCK=11549180 \
W6_FORK_BLOCK_HASH=0xf122ea293f1371712bd27a5253f03dfd14884c931f7120832e8d2d9cd514ed79 \
pnpm --filter manager exec vitest run \
  --config ../../vitest.w6.official-indexer.config.ts \
  --reporter=verbose
```

The preserved integrated run passed `1/1`. The exact production operation obtained Mallory before reclaim and Bob after it; graph-node ended at block `11549194` with `hasIndexingErrors=false`; the stale plan granted Mallory `16777216`, Bob remained at zero, and the deployed Universal Resolver returned Mallory until recovery.

Initial registration and V2 reservation use setup-only impersonation of already-authorized deployed controllers. No migration participant is impersonated. The official-indexer fixture emits a same-value `eth` parent event solely so a late-start indexer can discover the unchanged BaseRegistrar parent. No `mockCall`, storage/code write, fabricated API/indexer response, direct indexed-entity database insertion, forced outcome, public-network write, or migration-participant impersonation supplies a necessary condition.
