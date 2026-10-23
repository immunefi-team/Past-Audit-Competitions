# 91342 w a medium manager resurrects a relinquished v1 manager as a v2 resolver delegate after the pre wallet state check

**Submitted on Aug 31st 2026 at 18:56:19 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91342
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Summary

Manager intentionally restores a V1 name's existing manager as a V2 resolver delegate during migration. The authority decision is made from a V1 subgraph object captured while the Migration page is mounted. The resulting transaction plan stores `managerAddress` and later encodes:

```
ETHRegistry.grantRoles(resource, ROLE_SET_RESOLVER, managerAddress)
```

The application does not bind this decision to the current V1 Registry owner at execution. Its documented pre-wallet state check recomputes only which operator approvals are missing from the already-classified plan. It never calls the V1 Registry's `owner(node)` and never compares the planned manager with current authority.

This produces a real manager-offboarding race:

{% stepper %}
{% step %}

## Victim revokes global approval

Victim `V` revokes `A`'s separate global V1 operator approval through the real Registry. `A` remains only the direct manager of this node.
{% endstep %}

{% step %}

## Migration page builds a plan

`V` opens Manager's Migration page while `A` is the truthful current direct V1 manager.

The page automatically selects the eligible name and builds its normal preview/transaction plan. My PoC proves that `A` has no victim-wide V1/V2 approval and no preexisting V2 resolver role at this point.
{% endstep %}

{% step %}

## Former manager hands back control

`A` completes the ordinary on-chain handback by calling V1 `Registry.setOwner(node, V)`. A real post-handback V1 resolver write from `A` is rejected.
{% endstep %}

{% step %}

## Victim upgrades

`V` clicks the normal **Upgrade** button and receives every migration wallet prompt only after the handback.

Manager's promised pre-prompt check accepts the old plan because the required approval *types* did not change.
{% endstep %}

{% step %}

## Migration restores obsolete authority

The mined victim-to-HCA atomic transaction migrates the name and then executes `grantRoles(resource, ROLE_SET_RESOLVER, A)` from the frozen plan.

Cleanup removes the HCA's temporary operator approval but does not revoke `A`'s new per-name role.
{% endstep %}

{% step %}

## Attacker changes the resolver

`A` sets an attacker-controlled resolver, and the deployed Universal Resolver returns the attacker's address for the victim's name.
{% endstep %}
{% endstepper %}

The victim never selects `A`, never chooses an attacker-controlled recipient, and never constructs an unsafe call order. At the time the victim clicks Upgrade and signs every transaction, `A` is no longer authorized. Manager identifies the step only as “Restore the existing managers for your names” and does not display the address being restored.

## Finding Description

### Intended behavior

Restoring a manager that still holds current V1 manager authority is intentional. The migration release note says that migration restores legacy manager access.

The same release note also establishes the security boundary for mutable state:

> Permission state is rechecked before the first wallet prompt; if it changed, the flow asks the owner to review a refreshed preview.

The correct authority to restore must therefore be the manager still authorized at the execution boundary, not any address that appeared in an earlier page snapshot.

### Actual behavior

#### 1. The page obtains V1 authority data without a manager generation

[`useV1Names.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29) keys its query only by the connected owner address:

```ts
queryKey: qk('migration', 'v1_names', {
  address: address?.toLowerCase(),
})
```

There is no block number, V1 Registry manager value, version, subscription, or polling interval in this path. An already-mounted successful query keeps its last data until a refetch trigger or explicit invalidation. An external V1 `setOwner` transaction does not invalidate the Manager query.

The hook supplies a five-minute `staleTime`, but the shared query adapter destructures and omits that property before returning the options. The effective runtime stale time is therefore the default zero. This does **not** create polling or discard the successful data; it only allows event-triggered refetches. My reproduction does not rely on a five-minute TTL.

#### 2. Automatic selection freezes a plan keyed only by domain IDs

[`useNameSelection.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useNameSelection.ts#L33-L49) automatically selects eligible names after the mounted query completes.

[`useMigrationGasEstimate.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L110) reduces those objects to sorted domain IDs for its query identity:

```ts
const domainIds = domains
  .map((domain) => domain.id)
  .sort()
  .join(',')

queryKey: [
  'migration-gas-estimate',
  ownerAddress,
  hcaAddress,
  domainIds,
  selectionRevision,
]
```

A same-name manager change preserves `domain.id`, so the plan identity contains no manager generation. `buildMigrationPlan()` returns the classified objects, batches, and step descriptors as one ready plan. [`MigrationPage.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L282) later passes that exact `gasEstimate.plan` into `migration.start` when the victim clicks Upgrade.

#### 3. Classification converts the snapshot's owner into `managerAddress`

The imported classifier correctly interprets an unwrapped `.eth` domain whose registrant is the victim and whose registry owner differs from the registrant as having a legacy manager. The security defect is not that classification rule; it is using its historical input as authorization after the underlying state changes.

[`buildMigrationPlan.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L410) retains the classified name in the execution plan and builds its atomic batches from that value.

#### 4. The pre-wallet “current” check checks approval keys, not the manager

[`migrationService.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L338-L390) derives `requiresManagerRestoration` from frozen `plan.classified`:

```ts
requiresManagerRestoration: plan.classified.some(
  (name) => name.managerAddress !== null,
)
```

It then compares only sorted approval identifiers between preview and execution. If the HCA still requires the same `eth-registry:hca` approval, the check passes even though the identity being restored is now unauthorized.

[`executeMigration()`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1161-L1204) explicitly calls this check “against the latest chain state before opening the first wallet prompt,” but the check never reads V1 Registry `owner(node)`.

This is the decisive TOCTOU boundary. In my PoC, the victim first revokes `A`'s global V1 operator approval, the handback is mined in block `11565221`, and a real V1 resolver write from `A` then reverts. The service's first victim migration transaction is the HCA deployment in block `11565222`. All five Manager migration transactions are from the victim and occur after `A` has lost every demonstrated V1/V2 authority path, yet the pre-prompt check accepts the frozen plan.

#### 5. The batch migrates first and resurrects the former manager second

[`buildRoleGrantCalls.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26) encodes the persistent resolver role for the historical address:

```ts
args: [resource, 1n << 24n, name.managerAddress]
```

[`buildAtomicMigrationBatches.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624) adds both the call and a verification expectation for that same account. The phase ordering at [lines 742–780](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L742-L780) is:

```
migrate
manager-role-grant
```

The name becomes victim-owned in V2, and then the same victim-authorized atomic HCA batch grants the obsolete address fresh V2 authority.

#### 6. Verification blesses the unwanted role and cleanup preserves it

[`verifyAtomicMigrationBatch.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353) considers the batch correct only if `hasRoles(resource, ROLE_SET_RESOLVER, formerManager)` returns true.

Cleanup is intentionally limited to temporary operator approval. [`migrationApprovals.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationApprovals.ts#L354-L366) encodes only:

```
setApprovalForAll(HCA, false)
```

It never revokes the newly created per-name role. My PoC confirms both post-cleanup states simultaneously:

```
HCA operator approval:               false
former manager ROLE_SET_RESOLVER:    true
```

#### 7. The first-party confirmation does not identify the principal

[`WalletConfirmationStepsDialog.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L89) displays only:

```
Approve manager restoration
Restore the existing managers for your names.
```

No address appears in the Manager preview. The victim does not choose `A` or affirm that `A` remains authorized after the completed handback. A wallet may choose to decode nested calldata, but the vulnerability does not depend on claiming that every wallet is opaque: the first-party Manager flow itself constructs the obsolete grant and presents it as restoring current managers.

## Root Cause

Manager treats a page/query snapshot of V1 manager identity as durable authority across preview and execution.

Three in-scope omissions combine:

1. the migration inventory and gas-plan identities contain no V1 manager generation;
2. the pre-wallet “permission state” check revalidates approval categories but not the current manager address; and
3. the atomic grant and its verifier consume the frozen address without an on-chain equality assertion before migration.

The exact security invariant that is missing is:

```
plannedManager == V1Registry.owner(node)
```

That invariant must hold atomically, or as part of the same atomic batch, immediately before the V1 migration consumes the old authority state.

## Impact Explanation

The program expressly classifies name hijacking during V1-to-V2 Manager migration as Critical. My PoC reaches that exact terminal rather than merely showing stale UI or latent calldata:

{% stepper %}
{% step %}

## V2 ownership

The victim is the V2 owner of `turnkey.eth` after migration.
{% endstep %}

{% step %}

## Unauthorized V2 role

Former manager `A`, whose global V1 approval was revoked and whose direct V1 resolver write fails after handback, holds newly created V2 `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

## Resolver change

`A` calls the deployed V2 Registry's `setResolver` successfully.
{% endstep %}

{% step %}

## Attacker-controlled resolver

The resolver becomes an attacker-deployed, source-compiled ENSIP-10 resolver.
{% endstep %}

{% step %}

## Attacker-controlled resolution

The deployed Universal Resolver resolves `turnkey.eth` to `A`.
{% endstep %}
{% endstepper %}

This allows the attacker to redirect resolution-dependent traffic and payments until the victim detects the change and revokes the role. The separate recovery control proves that later revocation stops subsequent attacker writes; it does not undo the resolver hijack that already occurred.

## Likelihood Explanation

The prerequisite is an ordinary V1 split between registrant and direct manager. A malicious delegate, seller, operator, or former controller can be the current Registry manager while the registrant remains the BaseRegistrar owner.

The realistic handback flow is:

{% stepper %}
{% step %}

## Remove separate global approval

The registrant removes any separate global operator approval as the first normal offboarding step.
{% endstep %}

{% step %}

## Open the Migration page

The registrant opens the official Migration page while waiting for the still-direct manager to return node control.
{% endstep %}

{% step %}

## Preview builds from pre-handback state

The page truthfully loads the pre-handback state and automatically prepares the migration.
{% endstep %}

{% step %}

## Manager returns node control

The manager completes the requested `setOwner(node, registrant)` handback.
{% endstep %}

{% step %}

## Registrant upgrades

The registrant sees the handback confirmed and proceeds with the already-open Manager page.
{% endstep %}

{% step %}

## Wallet actions occur after handback

Every wallet action occurs after the handback.
{% endstep %}
{% endstepper %}

The attacker controls the security-relevant transition. The victim only keeps the same focused Manager tab open and follows its normal Upgrade flow. No precise mempool front-running, hidden key, admin action, unusual configuration, mock, or victim-selected attacker address is required.

Focus, reconnect, remount, or explicit invalidation can cause refetching. There is no time-based polling, however, and the normal same-tab handback-to-Upgrade sequence reproduced by my PoC requires none of those events. More importantly, the execution boundary still lacks an atomic current-manager assertion, so client refetch behavior does not bind the final grant to current authority.

## Attack Path / Reproduction Path

{% stepper %}
{% step %}

## Establish V1 authorities

Victim `V` is the BaseRegistrar registrant of an unwrapped `.eth` name; attacker `A` is its current direct V1 Registry manager.
{% endstep %}

{% step %}

## Revoke separate global approval

`V` revokes the fixture's separate V1 `setApprovalForAll(A, true)` permission through a real Registry transaction. The readback is false while `A` remains the direct node manager.
{% endstep %}

{% step %}

## Open Migration

`V` opens Manager's V1-to-V2 Migration page.
{% endstep %}

{% step %}

## Obtain the production V1 row

Manager's production V1 query returns a truthful domain row whose `owner.id` is `A` and `registrant.id` is `V`.
{% endstep %}

{% step %}

## Build the frozen plan

The page automatically selects the name and builds its production preflight, confirmation descriptors, and atomic plan with `managerAddress = A`.
{% endstep %}

{% step %}

## Hand back V1 node ownership

`A` calls V1 Registry `setOwner(node, V)`. The transaction confirms; a real simulated `setResolver` from `A` now reverts, and a fresh classification has `managerAddress = null`.
{% endstep %}

{% step %}

## Start the upgrade

`V` clicks Upgrade on the still-mounted page.
{% endstep %}

{% step %}

## Pre-wallet check accepts the frozen plan

`executeMigration()` recomputes only missing approval keys. The key set remains `base-registrar:hca-token` plus `eth-registry:hca`, so the frozen plan passes.
{% endstep %}

{% step %}

## Victim signs normal transactions

`V` signs the normal HCA deployment and approval transactions.
{% endstep %}

{% step %}

## Atomic migration grants stale authority

The mined transaction is decoded as a call from `V` to the production HCA's `executeByOwner`; its inner calls exactly match the frozen preview, first calling MigrationHelper and then granting `ROLE_SET_RESOLVER` to `A`.
{% endstep %}

{% step %}

## Cleanup preserves the role

Manager's verifier confirms the obsolete grant as expected, and cleanup removes only the HCA operator approval.
{% endstep %}

{% step %}

## Former manager changes resolver

`A` calls V2 Registry `setResolver(resource, attackerResolver)`.
{% endstep %}

{% step %}

## Universal Resolver returns attacker address

Universal Resolver queries for the victim's name return the attacker-controlled address.
{% endstep %}

{% step %}

## Victim recovery

`V` revokes the role and restores the resolver; a subsequent real V2 `setResolver` simulation from `A` reverts.
{% endstep %}
{% endstepper %}

## External Preconditions

* The frozen Manager migration feature is enabled in the competition's intended local/testnet configuration.
* The victim is the V1 `.eth` registrant and uses the normal Manager migration flow.
* The attacker is the current direct V1 Registry manager before a legitimate handback and, by preview time, controls only that ordinary per-name authority.
* The Manager page remains mounted without a focus/remount/reconnect/invalidation trigger between its automatic preview and the victim's Upgrade action.

## Internal Preconditions

* The name is eligible for unwrapped V1-to-V2 migration.
* The V1 subgraph row is truthful when first fetched.
* Any separate victim-to-attacker global V1 operator approval has been revoked before Manager builds the preview.
* The attacker calls V1 Registry `setOwner(node, victim)` after the automatic plan is built and before the victim's first migration wallet prompt.
* The V2 name has not already been migrated.
* The victim follows the normal approval and migration prompts.

## Affected Code

1. [`apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29) — caches V1 authority inventory by connected address without a manager/block generation or polling.
2. [`apps/manager/src/features/migration/hooks/useNameSelection.ts#L33-L49`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useNameSelection.ts#L33-L49) — automatically selects eligible names from that inventory.
3. [`apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L110`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L110) — keys the frozen preview/plan by domain IDs rather than current manager identity.
4. [`apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L282`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L282) — passes the previously built plan into execution when Upgrade is clicked.
5. [`apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L410`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L410) — persists snapshot-derived `managerAddress` in the execution plan.
6. [`apps/manager/src/features/migration/service/migrationService.ts#L338-L390`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L338-L390) and [`#L1161-L1204`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1161-L1204) — claim to recheck current state before the first prompt but compare only approval identifiers derived from the frozen plan.
7. [`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26) — grants `ROLE_SET_RESOLVER` to the historical address.
8. [`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624) and [`#L742-L780`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L742-L780) — place migration and the stale authority grant in one successful batch and expect that grant in verification.
9. [`apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353) — treats the former manager receiving the role as correct post-state.
10. [`apps/manager/src/features/migration/service/migrationApprovals.ts#L354-L366`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationApprovals.ts#L354-L366) — removes only temporary HCA approval and leaves the stale per-name resolver role.
11. [`apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L89`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L89) — describes generic manager restoration without identifying the address receiving permanent authority.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/6d7e79165517867598fb1b00d70720d4>

## Proof of Concept

## FIXED-FORK TRANSACTION E2E POC

### Description

My supplied PoC is a fixed-fork, no-mock, transaction-level E2E reproduction. Once the truthful V1 inventory row has been returned, every security-relevant chain transition, calldata assertion, and terminal resolution is deterministic at the pinned fork block. The production V1 inventory client itself queries the live, unpinned `https://v1-graphql.ens.dev/subgraph`; my PoC treats availability of that truthful row as an explicit prerequisite and verifies its security-relevant fields against the pinned on-chain state before continuing.

It imports and executes the competition source's production:

* `getV1NamesForAddress()` V1 query client
* classifier and migration preflight
* `buildMigrationPlan()` and atomic batch builder
* `executeMigration()` transaction service
* HCA deployment and approvals
* receipt/post-state verifier
* temporary-approval cleanup

It runs those components against deployed Sepolia contracts on a local fixed-block Anvil fork, then deploys a real source-compiled ENSIP-10 attacker resolver and queries the deployed Universal Resolver.

The fixture begins with two distinct V1 authorities held by the former manager: direct node management and a victim-wide operator approval. The victim first revokes the global approval through the deployed V1 Registry. Manager then obtains its truthful production V1 row and builds the automatic preview solely from the remaining direct-manager relationship. After the direct manager hands the node back, my PoC proves a real V1 resolver write by that address reverts before any migration prompt is signed. It also decodes the mined atomic HCA transaction and proves that the obsolete address is nevertheless the account receiving `ROLE_SET_RESOLVER`.

I supplied the complete seven-file PoC and clean-machine reproduction guide in the [**ENS #1 PoC Gist**](https://gist.github.com/aydarkhusaenov/6d7e79165517867598fb1b00d70720d4). The Gist contains only the PoC source, runner, manifest, Solidity resolver, and full README; it does not contain a copied source repository or generated output.

### Scenarios and controls

1. **Independent-authority cleanup:** the victim revokes the fixture's global V1 operator approval and proves it false before Manager queries or plans anything; the attacker has no preexisting V2 resolver role or victim-wide V2 approval.
2. **Default automatic-preview race:** production preview while `A` is only the direct V1 node manager; attacker handback; a real V1 resolver write from `A` now reverts; every victim prompt occurs afterward; the frozen manager grant survives.
3. **Fresh-authority negative control:** replace the query object's owner with the post-handoff on-chain owner and rebuild; classification returns `managerAddress = null`, and the batch contains no `manager-role-grant`.
4. **Mined-transaction binding:** the atomic transaction is proven to be sent by the victim to the production HCA; its decoded inner calls exactly match the frozen preview and its role grant names `A`.
5. **Production-service completion:** real HCA deployment, BaseRegistrar approval, ETHRegistry HCA approval, atomic migration, post-state verification, and cleanup all succeed.
6. **Cleanup persistence:** temporary HCA operator approval is false after cleanup while former manager resolver authority remains true.
7. **Resolver hijack:** former manager points the V2 name to the included ENSIP-10 resolver; Universal Resolver returns the attacker.
8. **Resolution-denial control:** former manager points the resolver to its EOA; Universal Resolver returns `ResolverNotContract`.
9. **Recovery control:** victim revokes the former manager and restores the original resolver; a subsequent V2 resolver write from the attacker reverts.

### Exact files and placement

Place the complete supplied directory at the repository root:

```
poc/smell1-stale-v1-manager/
├── .gitignore
├── README.md
├── foundry.toml
├── poc.ts
├── run.sh
├── SOURCE_SHA256SUMS
└── src/
    └── AttackerResolver.sol
```

### Requirements and clean setup

* Git
* Docker
* curl
* Node.js 22
* pnpm 10
* Foundry 1.7.1

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
pnpm install --frozen-lockfile
```

### Manifest verification

From the repository root, obtain and place the Gist files exactly as follows. GitHub Gists flatten subdirectories, so the final `mv` restores the Solidity source to its required `src/` path:

```bash
mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/6d7e79165517867598fb1b00d70720d4.git \
  poc/smell1-stale-v1-manager
cd poc/smell1-stale-v1-manager
mkdir -p src
mv AttackerResolver.sol src/AttackerResolver.sol
shasum -a 256 -c SOURCE_SHA256SUMS
```

If the PoC directory is supplied directly rather than cloned from the Gist, place it at `poc/smell1-stale-v1-manager/`, then run from the repository root:

```bash
cd poc/smell1-stale-v1-manager
shasum -a 256 -c SOURCE_SHA256SUMS
```

### Exact command

After either placement method, return to the frozen repository root and run:

```bash
cd ../..
bash poc/smell1-stale-v1-manager/run.sh
```

The runner uses this fixed-fork transaction environment:

```
Docker image: ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd
Anvil:       1.7.1
chain ID:    11155111
fork block:  11565219
block hash:  0xda564596b0ab93b11a215fe3113ec9c925b3b5a49a63be4349f2b26cdae4ee61
Anvil RPC:   loopback only
upstream:    read-only Sepolia archive endpoint; no public transaction is sent
```

The script starts the pinned Docker node, waits for JSON-RPC, runs the PoC, and stops/removes the temporary container. Full setup, alternate port/RPC options, assertion descriptions, and expected evidence are in the included `README.md`.

### Expected and passing logs

My clean run on 2026-08-26 exited successfully and produced:

```json
{
  "result": "PASS",
  "forkBlockNumber": "11565219",
  "forkBlockHash": "0xda564596b0ab93b11a215fe3113ec9c925b3b5a49a63be4349f2b26cdae4ee61",
  "name": "turnkey.eth",
  "victim": "0x1ba43becc3ea96c37343b7cb18de7386ba29445b",
  "attacker": "0x5f71c6074141d41d0e769689dc9830388a5c7e3e",
  "hca": "0x845EEe994e78032834f4fA07b941e0a7fE6635Fd",
  "staleTransitionActor": "attacker",
  "staleTransitionTiming": "after-plan-before-first-wallet-prompt",
  "initialV1Registrant": "0x1bA43Becc3Ea96c37343b7CB18de7386bA29445B",
  "v1ManagerAtTruthfulSnapshot": "0x5f71c6074141d41d0e769689dC9830388a5C7e3E",
  "v1ManagerWhenAutomaticPreviewBuilt": "0x5f71c6074141d41d0e769689dC9830388a5C7e3E",
  "initialV1Resolver": "0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5",
  "initialV2Status": 1,
  "initialV2Owner": "0x0000000000000000000000000000000000000000",
  "initialVictimV1ApprovalToAttacker": true,
  "initialAttackerV2ResolverRole": false,
  "initialVictimV2ApprovalToAttacker": false,
  "productionV1SubgraphRowMatchedOnchain": true,
  "hcaWasUndeployedWhenPreviewBuilt": true,
  "victimV1OperatorRevocationTx": "0xb6c15e15dc39185762004c38ba9d83120f24298d63f9d974e1c843c1fe9ea04d",
  "victimV1ApprovalToAttackerWhenPreviewBuilt": false,
  "staleTransitionTx": "0xaa5129a1609633f11bc54ec0be8ebbf30266612e160837ec43719795b23389c5",
  "staleTransitionBlock": "11565221",
  "v1ManagerAfterTransitionBeforeMigration": "0x1bA43Becc3Ea96c37343b7CB18de7386bA29445B",
  "formerManagerV1WriteRejectedAfterHandback": true,
  "frozenPreviewPhases": ["migrate", "manager-role-grant"],
  "freshCurrentClassificationManager": null,
  "managerBatchPhases": ["migrate", "manager-role-grant"],
  "productionPlannedApprovalIds": [
    "base-registrar:hca-token",
    "eth-registry:hca"
  ],
  "productionHcaDeploymentTx": "0xf9d05bd45eaf1b43b3df053d41ece846a9f1aac48b2d79460c3e247d42e01ee0",
  "productionTokenApprovalTx": "0x3b0703b3cfd469bb23ac3d843d2cec16e5419c2b2cffc5bdcfdf55a482215f2f",
  "productionManagerApprovalTx": "0x9fa141780e577157730a858c597942c263d41e113e4523fd6cb6b1bc010d7d3c",
  "productionMigrationTx": "0x75293fe0744f2c41f35ee7e1afdb5107f6464b5880b9eafe98d421a5bdcde244",
  "productionMigrationSender": "0x1ba43becc3ea96c37343b7cb18de7386ba29445b",
  "productionMigrationTarget": "0x845eee994e78032834f4fa07b941e0a7fe6635fd",
  "minedManagerGrantAccount": "0x5f71c6074141d41d0e769689dC9830388a5C7e3E",
  "productionCleanupTx": "0xbca623961cf42740cf26fa8af2bba9b6b2c488e2e9aa49d2dcb223e2dfb56dc2",
  "firstProductionWalletPromptBlock": "11565222",
  "allProductionWalletPromptsAfterFormerManagerUnauthorized": true,
  "productionServiceTxHashes": [
    "0xf9d05bd45eaf1b43b3df053d41ece846a9f1aac48b2d79460c3e247d42e01ee0",
    "0x3b0703b3cfd469bb23ac3d843d2cec16e5419c2b2cffc5bdcfdf55a482215f2f",
    "0x9fa141780e577157730a858c597942c263d41e113e4523fd6cb6b1bc010d7d3c",
    "0x75293fe0744f2c41f35ee7e1afdb5107f6464b5880b9eafe98d421a5bdcde244",
    "0xbca623961cf42740cf26fa8af2bba9b6b2c488e2e9aa49d2dcb223e2dfb56dc2"
  ],
  "productionServiceProgress": [
    "Setting up your HCA",
    "HCA ready",
    "Allowing the migration helper to migrate this registration",
    "Permission confirmed",
    "Allowing your HCA to restore managers",
    "Permission confirmed",
    "Atomically upgrading turnkey.eth",
    "Atomic batch verified",
    "Removing manager-restoration access from your HCA",
    "Temporary access removed"
  ],
  "productionVerifierExpectationCount": 3,
  "v2Owner": "0x1bA43Becc3Ea96c37343b7CB18de7386bA29445B",
  "cachedFormerManagerReceivedRole": true,
  "hcaOperatorApprovalAfterManagerCleanup": false,
  "formerManagerRoleAfterManagerCleanup": true,
  "resolverAfterMigration": "0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5",
  "attackerResolverContract": "0xa216741bb8589367952204f5c09ebda2f6dc3d7a",
  "attackerTakeoverTx": "0xf28303b220beb4162b79521fa5ff91ef5ed1b01269a66422d20cffedba062d90",
  "resolverAfterFormerManagerAttack": "0xa216741bb8589367952204f5c09ebdA2f6DC3d7a",
  "universalResolverResolvedAddress": "0x5f71c6074141d41d0e769689dC9830388a5C7e3E",
  "eoaBrickingTx": "0xdb035a52ad641135c05e60333867db9ede42bb72e8d0ee76e49b09f642c1cf6e",
  "universalResolverFailure": "ResolverNotContract",
  "victimCouldRevokeAndRestore": true,
  "formerManagerV2WriteRejectedAfterRecovery": true,
  "resolverAfterRecovery": "0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5"
}
```

The deterministic local transaction trace is:

| Action                                                   | Transaction                                                          |
| -------------------------------------------------------- | -------------------------------------------------------------------- |
| Victim revokes the separate global V1 operator approval  | `0xb6c15e15dc39185762004c38ba9d83120f24298d63f9d974e1c843c1fe9ea04d` |
| Attacker completes the direct V1 manager handback        | `0xaa5129a1609633f11bc54ec0be8ebbf30266612e160837ec43719795b23389c5` |
| Victim's first prompt: production HCA deployment         | `0xf9d05bd45eaf1b43b3df053d41ece846a9f1aac48b2d79460c3e247d42e01ee0` |
| Victim approves MigrationHelper for the V1 token         | `0x3b0703b3cfd469bb23ac3d843d2cec16e5419c2b2cffc5bdcfdf55a482215f2f` |
| Victim grants temporary HCA manager-restoration access   | `0x9fa141780e577157730a858c597942c263d41e113e4523fd6cb6b1bc010d7d3c` |
| HCA atomic migration and obsolete manager grant          | `0x75293fe0744f2c41f35ee7e1afdb5107f6464b5880b9eafe98d421a5bdcde244` |
| Victim's cleanup revokes temporary HCA access            | `0xbca623961cf42740cf26fa8af2bba9b6b2c488e2e9aa49d2dcb223e2dfb56dc2` |
| Former manager replaces the resolver                     | `0xf28303b220beb4162b79521fa5ff91ef5ed1b01269a66422d20cffedba062d90` |
| Former manager exercises the EOA-resolver denial control | `0xdb035a52ad641135c05e60333867db9ede42bb72e8d0ee76e49b09f642c1cf6e` |

The Manager service's own progress output confirms the normal path:

```
Setting up your HCA
HCA ready
Allowing the migration helper to migrate this registration
Permission confirmed
Allowing your HCA to restore managers
Permission confirmed
Atomically upgrading turnkey.eth
Atomic batch verified
Removing manager-restoration access from your HCA
Temporary access removed
```

### What each assertion proves

* `turnkey.eth`, its registrant, manager, resolver, expiry, and V2 reserved status come from the canonical fixed-block fork.
* The fixture's independent victim-wide V1 operator approval is identified and revoked through the deployed V1 Registry before Manager queries the name; the on-chain readback is false.
* Manager's unmodified production V1 subgraph client then obtains the row and asserts every relevant field against on-chain state while the attacker remains only the direct node manager.
* The attacker handback is a real V1 Registry transaction. After it confirms, a real V1 `setResolver` simulation by that address reverts and leaves the resolver unchanged.
* The previously absent deterministic HCA is deployed through the production factory by unmodified `executeMigration()` as the victim's first transaction after handback.
* Every approval, HCA execution, migration, V2 role grant, verifier read, and cleanup is a real contract call. The mined HCA calldata is decoded and its inner calls are byte-compared to the frozen preview.
* The attacker resolver is compiled from the included Solidity source and deployed normally.
* The deployed Universal Resolver provides the terminal result.
* Anvil impersonation supplies signatures and local gas only for the two ordinary fixture actors. It does not seed contract storage, replace code, impersonate a trusted protocol role, or bypass a guard.
* `MemoryLocalStorage` is only the browser-API shim used by the real transaction journal under Node. It carries no protocol or exploit state.

### Cleanup

The runner stops and removes only its uniquely named temporary Anvil container on success, failure, or interruption. It does not edit scoped production source or send a transaction to a public network.

### Complete source and clean-machine reproduction

The [**ENS #1 PoC Gist**](https://gist.github.com/aydarkhusaenov/6d7e79165517867598fb1b00d70720d4) contains all seven files needed to reproduce the result:

```
.gitignore
README.md
SOURCE_SHA256SUMS
foundry.toml
poc.ts
run.sh
AttackerResolver.sol   -> place at src/AttackerResolver.sol
```

After upload, I cloned the Gist and byte-compared it against the package I executed. All seven files matched. The Gist is complete: no source-repository files, captured output files, archives, or generated Foundry directories are required.

The source manifest verifies these exact hashes:

```
a670c00d8ce77a715d6cd5409543ed7404c2470bea762f57e742ac2553be9dd5  .gitignore
55541a4487b73bf0692c5e02caa730553f92a635405aa8ab08a75e472c394730  README.md
5ecd38d746eef404c37e9db4acff148cd4e797f095f8e2fbc24de059ac65d074  foundry.toml
88edb85b4b1dfd4c2706e8f56dbcb65b2ca27ce465bb7dfc60632178c4fae6e8  poc.ts
2712b64db87e6567c13cc25f91f7e1156b0b0a404c65e8de6f2cb0ea3e15c64c  run.sh
7f5143fb53e24bb499f11ef66c1c0220611ca2825150836cf5a6a547d53dd48d  src/AttackerResolver.sol
```
