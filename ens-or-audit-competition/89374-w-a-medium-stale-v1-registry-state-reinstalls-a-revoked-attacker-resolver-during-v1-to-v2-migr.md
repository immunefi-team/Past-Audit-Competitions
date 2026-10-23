# 89374 w a medium stale v1 registry state reinstalls a revoked attacker resolver during v1 to v2 migration enabling ens name hijacking

**Submitted on Aug 18th 2026 at 21:30:08 UTC by @calbebop for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89374
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Manager gets a V1 name's Registry owner and resolver from the V1 subgraph and caches the response for five minutes. It later copies those values into the V1-to-V2 migration without reading the live V1 Registry. If the owner revokes a former manager or replaces a compromised resolver after the page loads, the migration can restore both values. The transaction succeeds and Manager reports success, leaving the V2 name with an attacker-controlled resolver or a revoked account holding `ROLE_SET_RESOLVER`. This is the competition's Critical Manager-flow impact where a migration assigns the resolver to an attacker.

## Vulnerability Details

### Root cause and data flow

The bug is in the in-scope Manager app. The subgraph returns indexed state, and the contracts accept the migration data supplied by Manager. Manager treats the indexed owner and resolver as current when it builds the transaction.

{% stepper %}
{% step %}

## Manager requests V1 data from the indexer

Manager requests the V1 `resolver`, Registry `owner`, and BaseRegistrar `registrant` from the indexer:

* [`v1SubgraphClient.ts#L22-L48`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L48)
  {% endstep %}

{% step %}

## The indexed result is cached

That result is considered fresh for five minutes:

* [`useV1Names.ts#L12-L23`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23)
  {% endstep %}

{% step %}

## Eligibility checks do not read the V1 Registry

Eligibility is separately cached for 30 minutes using only the wallet and domain IDs. For an unwrapped `.eth` name, the live checks call BaseRegistrar `ownerOf(labelhash)` and V2 `getStatus(labelhash)`. They do not call V1 Registry `owner(node)` or `resolver(node)`:

* [`useMigrationEligibility.ts#L8-L28`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationEligibility.ts#L8-L28)
* [`preflightChecks.ts#L47-L101`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/preflightChecks.ts#L47-L101)
* [`preflightChecks.ts#L146-L179`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/preflightChecks.ts#L146-L179)
  {% endstep %}

{% step %}

## Classification copies stale data

Classification copies the indexed custom resolver into `v1ResolverAddress`. For an unwrapped name whose indexed Registry owner differs from the registrant, it also copies that owner into `managerAddress`. A custom resolver selects the `keep-v1` strategy:

* [`classifyNames.ts#L65-L89`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L65-L89)
* [`classifyNames.ts#L125-L185`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L125-L185)
  {% endstep %}

{% step %}

## The stale resolver is added to the migration

`resolverFor` returns that cached custom resolver. The atomic batch builder places it in `Migration.Data.resolver` and records it as the expected post-migration resolver:

* [`encodeMigration.ts#L24-L50`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/encodeMigration.ts#L24-L50)
* [`buildAtomicMigrationBatches.ts#L479-L565`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L479-L565)
  {% endstep %}

{% step %}

## The stale manager receives a role grant

When the cached Registry owner differs from the registrant, the same batch appends `grantRoles(resource, 1 << 24, cachedManager)`. Bit `1 << 24` is V2 `ROLE_SET_RESOLVER`:

* [`buildRoleGrantCalls.ts#L9-L24`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L9-L24)
* [`buildAtomicMigrationBatches.ts#L608-L624`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624)
  {% endstep %}

{% step %}

## Submission does not refresh V1 Registry state

Immediately before submission, execution rechecks HCA/operator approvals and estimates the transaction. It rebuilds the call from the same classified snapshot and does not refresh the V1 Registry owner or resolver:

* [`migrationService.ts#L367-L390`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L367-L390)
* [`migrationService.ts#L886-L960`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L886-L960)
  {% endstep %}

{% step %}

## Post-state verification validates the stale plan

Post-state verification compares V2 with the resolver and manager role stored in the plan. Because those expected values came from the stale snapshot, the unsafe result passes verification:

* [`verifyAtomicMigrationBatch.ts#L249-L292`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L249-L292)
* [`verifyAtomicMigrationBatch.ts#L337-L353`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353)
  {% endstep %}
  {% endstepper %}

### Why simulation and the contracts do not reject the transaction

The transaction is valid even though its intent is stale. `MigrationHelper.migrate` verifies source-token ownership/approval, while `UnlockedMigrationController._inject` passes the caller-supplied `md.resolver` directly to V2 registration. Neither contract compares it with the live V1 Registry resolver or owner:

* [`MigrationHelper.sol#L101-L122`](https://github.com/ensdomains/contracts-v2/blob/892311a7268bf6051d5fe65740f1eebfaf8db431/contracts/src/migration/MigrationHelper.sol#L101-L122)
* [`UnlockedMigrationController.sol#L107-L120`](https://github.com/ensdomains/contracts-v2/blob/892311a7268bf6051d5fe65740f1eebfaf8db431/contracts/src/migration/UnlockedMigrationController.sol#L107-L120)
* [`UnlockedMigrationController.sol#L152-L165`](https://github.com/ensdomains/contracts-v2/blob/892311a7268bf6051d5fe65740f1eebfaf8db431/contracts/src/migration/UnlockedMigrationController.sol#L152-L165)

V2 `PermissionedRegistry.setResolver` requires `ROLE_SET_RESOLVER`, which is the role Manager grants to the former manager:

* [`PermissionedRegistry.sol#L149-L155`](https://github.com/ensdomains/contracts-v2/blob/892311a7268bf6051d5fe65740f1eebfaf8db431/contracts/src/registry/PermissionedRegistry.sol#L149-L155)

The calldata is valid, so the live gas estimate and transaction both succeed. The application's verifier then accepts the unsafe post-state.

### Exploit sequence

{% stepper %}
{% step %}

## Stale state is available

An unwrapped V1 `.eth` name is owned by the victim at the BaseRegistrar, while a former seller/manager controls its V1 Registry node or a compromised custom resolver is configured.
{% endstep %}

{% step %}

## The migration page caches it

The victim opens Manager's migration page, which caches the owner and resolver.
{% endstep %}

{% step %}

## The victim revokes the authority

The victim uses BaseRegistrar `reclaim(tokenId, victim)` and replaces or clears the compromised resolver. The attacker now has no live V1 authority.
{% endstep %}

{% step %}

## The victim completes migration

Within the five-minute cache window, the victim returns to the loaded migration page and completes the normal V1-to-V2 flow.
{% endstep %}

{% step %}

## Eligibility passes

Live eligibility still passes because the victim remains the BaseRegistrar token owner and the V2 slot remains reserved.
{% endstep %}

{% step %}

## Stale state is restored in V2

Manager installs the cached attacker resolver in V2 and, in the former-manager variant, grants the revoked manager `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

## The attacker can redirect resolution

The attacker-controlled resolver returns an attacker payment address, or the former manager uses the restored role to set such a resolver later.
{% endstep %}
{% endstepper %}

This path requires no phishing, leaked key, or attacker transaction during migration. The owner completes the normal migration flow. The five-minute client cache provides the exploit window, and subgraph lag can lengthen it.

The affected path is an unwrapped V1 `.eth` name. Resolver restoration requires a cached nonzero custom resolver, while manager restoration requires the cached Registry owner to differ from the registrant. Known public resolvers map to the V2 default, and a name locked with `CANNOT_SET_RESOLVER` is not affected by the resolver-restoration path.

## Impact Details

**Selected in-scope impact (Critical):** "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer)," specifically "a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker."

After migration, V2 points to the attacker resolver. The Foundry PoC resolves the name through `ENSV2Resolver.resolve` and returns the attacker's payment address. In the manager variant, the former manager also receives `ROLE_SET_RESOLVER`.

Payments and other resolver-dependent actions can be redirected until the owner repairs the V2 state. I am not claiming a completed payment or permanent takeover. The selected impact is Critical because the competition includes assignment of a resolver to an attacker during migration.

## Recommended Remediation

{% stepper %}
{% step %}

## Read V1 Registry state directly

Immediately before preview construction and again before the first irreversible migration submission, directly read V1 Registry `owner(node)` and `resolver(node)` for every selected name.
{% endstep %}

{% step %}

## Derive migration values from live reads

Derive `managerAddress`, `resolverStrategy`, and `Migration.Data.resolver` only from those live reads.
{% endstep %}

{% step %}

## Bind the plan to observed state

Bind the approved plan to the observed live owner/resolver and abort if either changes before submission.
{% endstep %}

{% step %}

## Display restored values

Display every restored manager and non-default resolver address explicitly.
{% endstep %}

{% step %}

## Verify against user-visible values

Verify postconditions against the owner and resolver values shown to the user, not values copied from the original indexer snapshot.
{% endstep %}
{% endstepper %}

{% hint style="warning" %}
Setting the query cache duration to zero is insufficient because the indexer itself can lag. The final check must read the Registry directly.
{% endhint %}

## References

* [Audit Competition | ENS scope and impacts](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Competition resources and audited revision](https://immunefi.com/audit-competition/audit-competition-ens/resources/)
* [Audited Manager source snapshot](https://github.com/immunefi-team/audit-comp-ens/tree/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager)
* [ENS subgraph ownership terminology](https://docs.ens.domains/web/subgraph/)
* [ENS ETH Registrar and `reclaim`](https://docs.ens.domains/registry/eth/)
* [ENSv2 contracts revision used by the Foundry PoC](https://github.com/ensdomains/contracts-v2/tree/892311a7268bf6051d5fe65740f1eebfaf8db431)

## Link to Proof of Concept

<https://gist.github.com/calbebop/265efaa55fe6c3353e0ee30a4f3dd3a2>

## Proof of Concept

Supporting PoC source files (secret Gist):\
<https://gist.github.com/calbebop/265efaa55fe6c3353e0ee30a4f3dd3a2>

I used three tests for different parts of the bug:

1. an Anvil test builds and submits the HCA batch with Manager's production code, then checks the V2 resolver and former-manager role;
2. a digest-pinned Foundry test runs the official ENS migration fixtures and resolves the migrated name through `ENSV2Resolver.resolve`; and
3. a focused unit test decodes the migration and role calldata and runs Manager's post-state verifier.

All transactions run against disposable local chains. No transaction is sent to public Sepolia or mainnet.

### Prerequisites

* Checkout the audited competition repository and install dependencies with its pinned Node/pnpm versions.
* Docker Desktop must be running.
* Port `8545` must be free for the local Anvil fork.

### PoC 1: Manager-built transaction on Anvil

PoC source file:

`apps/manager/src/features/migration/service/staleV1RegistryMetadata.onchain.poc.test.ts`

From the repository root, start the repository's pinned disposable Sepolia fork:

```powershell
docker compose -f e2e/infra/docker-compose.yml up -d --wait anvil
```

Run the opt-in PoC:

```powershell
$env:TZ='UTC'
$env:RUN_STALE_V1_METADATA_ONCHAIN_POC='true'
corepack pnpm --filter manager exec vitest run src/features/migration/service/staleV1RegistryMetadata.onchain.poc.test.ts --reporter=verbose
```

Expected result:

```
✓ stale V1 Registry metadata migration PoC (Anvil fork)
  ✓ reinstalls a revoked resolver and grants the revoked manager live V2 resolver control

Test Files  1 passed (1)
Tests       1 passed (1)
```

The test snapshots the fork and reverts it in `finally`. It then:

{% stepper %}
{% step %}

## Create and reserve the name

Creates an unwrapped V1 `.eth` BaseRegistrar token and reserves the corresponding V2 slot.
{% endstep %}

{% step %}

## Configure attacker-controlled V1 state

Makes the attacker the V1 Registry manager and custom resolver.
{% endstep %}

{% step %}

## Capture the state

Captures that state in the same `V1Domain` shape supplied to Manager by the subgraph.
{% endstep %}

{% step %}

## Revoke the attacker

Has the victim call BaseRegistrar `reclaim` and clear the live V1 resolver.
{% endstep %}

{% step %}

## Confirm revocation

Reads V1 Registry `owner(node)` and `resolver(node)` and asserts the attacker is fully revoked.
{% endstep %}

{% step %}

## Run classification and eligibility

Calls the production `classifyName` and `runEligibilityChecks`; eligibility succeeds because those checks never read current V1 Registry authority.
{% endstep %}

{% step %}

## Deploy and approve the HCA

Deploys the HCA at its deterministic address and supplies the BaseRegistrar and V2 Registry approvals used by the application.
{% endstep %}

{% step %}

## Build the migration batch

Calls `resolveDirectMigrationRoutes` and `buildAtomicMigrationBatches` from Manager.
{% endstep %}

{% step %}

## Confirm stale values in the batch

Checks that the migration data contains the stale attacker resolver and the batch contains the stale manager-role grant.
{% endstep %}

{% step %}

## Estimate gas

Runs `eth_estimateGas` over the app-built HCA `executeByOwner` call. Estimation succeeds.
{% endstep %}

{% step %}

## Submit the transaction

Submits that outer call to the local fork.
{% endstep %}

{% step %}

## Validate the V2 state

Reads V2 and asserts that its resolver is the attacker and the former manager has `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

## Exercise restored resolver authority

Sends an attacker-signed V2 `setResolver` transaction and confirms that the call succeeds.
{% endstep %}
{% endstepper %}

Stop the disposable fork after validation:

```powershell
docker compose -f e2e/infra/docker-compose.yml down
```

Result on a newly created fork: **1/1 passed**.

### PoC 2: resolver takeover in the ENS contract fixtures

PoC files:

* `poc/StaleResolverMigration.e2e.t.sol`
* `poc/run-e2e.ps1`
* `poc/README.md`

From the competition repository root:

```powershell
.\poc\run-e2e.ps1
```

The runner uses a digest-pinned Foundry image. It checks out `ensdomains/contracts-v2` at `892311a7268bf6051d5fe65740f1eebfaf8db431`, initializes its submodules, checks formatting, and runs the PoC.

Expected and observed result:

```
[PASS] test_staleResolverIsReinstalledAfterVictimRevokesIt() (gas: 695863)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

This test uses the official migration fixtures and a resolver that returns the attacker's address. It:

{% stepper %}
{% step %}

## Register a V1 name

Registers an unwrapped V1 name.
{% endstep %}

{% step %}

## Configure and cache the attacker resolver

Configures and caches the attacker resolver.
{% endstep %}

{% step %}

## Replace it with a safe resolver

Has the victim replace it with the safe V1 resolver.
{% endstep %}

{% step %}

## Supply stale migration data

Supplies Manager's stale value in `Migration.Data`.
{% endstep %}

{% step %}

## Execute migration

Executes the `UnlockedMigrationController` migration.
{% endstep %}

{% step %}

## Confirm stale resolver installation

Asserts V2 stored the attacker resolver.
{% endstep %}

{% step %}

## Resolve the attacker payment address

Calls `ENSV2Resolver.resolve` and decodes the attacker payment address.
{% endstep %}
{% endstepper %}

Result from an empty Docker volume: **1/1 passed**.

### PoC 3: calldata and post-verifier check

PoC source file:

`apps/manager/src/features/migration/service/staleManagerRestoration.poc.test.ts`

Run:

```powershell
$env:TZ='UTC'
corepack pnpm --filter manager exec vitest run src/features/migration/service/staleManagerRestoration.poc.test.ts --reporter=verbose
```

Expected and observed result:

```
✓ POC: revoked V1 authorization state is restored during migration
  ✓ passes live eligibility without reading the current V1 Registry manager
  ✓ encodes and certifies a V2 SET_RESOLVER grant to the removed manager
  ✓ reinstalls a stale malicious resolver without any manager restoration

Test Files  1 passed (1)
Tests       3 passed (3)
```

This test uses production classification, eligibility, batch construction, ABI decoding, and post-verification. It checks that:

* the only live eligibility reads are `ownerOf` and `getStatus`;
* decoded `MigrationHelper.migrate` calldata contains the stale resolver;
* decoded `grantRoles` calldata gives the revoked manager `1 << 24`;
* Manager's production verifier accepts the stale resolver and manager role; and
* the resolver-only variant requires no manager-role restoration step.
