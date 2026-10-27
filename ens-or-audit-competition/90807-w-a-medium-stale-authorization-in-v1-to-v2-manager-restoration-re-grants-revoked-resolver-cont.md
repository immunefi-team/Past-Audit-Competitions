# 90807 w a medium stale authorization in v1 to v2 manager restoration re grants revoked resolver control enabling ens name hijack

**Submitted on Aug 27th 2026 at 11:40:30 UTC by @Ilkka for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90807
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

### Brief/Intro

The Manager builds V1-to-V2 migration calldata from an ENSv1 snapshot and does not revalidate the current V1 manager before execution. If the registrant removes a manager after the migration plan is built, the retained plan still grants that address persistent V2 `ROLE_SET_RESOLVER`. The revoked manager can then replace the migrated name's resolver and make canonical ENS resolution return an attacker-controlled payment address.

### Vulnerability Details

For an unwrapped `.eth` name, the Manager restores the V1 registry owner as a V2 manager when it differs from the BaseRegistrar registrant. `classifyNames.ts` derives that recipient from the V1 subgraph result:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

`useV1Names.ts` keeps the result fresh for five minutes:

```ts
staleTime: 5 * 60 * 1000,
```

The application uses those `V1Domain` objects to build the migration plan while calculating the gas estimate. Clicking **Upgrade** passes the retained `gasEstimate.plan` directly to the migration state machine. It does not perform an authoritative manager refetch.

The planned manager becomes the recipient of a security-sensitive V2 role:

```ts
args: [resource, ROLE_SET_RESOLVER, name.managerAddress]
```

Immediately before each live batch, `buildNextAtomicBatch()` refreshes resolver readiness and migration routing but reuses the plan's classified names:

```ts
const atomicPlan = await buildAtomicMigrationBatches({
  // ...
  classified: remaining,
  profiles: plan.profiles,
})
```

No execution-time call reads `ENSRegistry.owner(node)` or compares it with the planned `managerAddress`. Refocusing the page during the five-minute V1-name freshness window does not fix the input: the V1 query still considers its subgraph result fresh, so a gas-estimate refetch uses the same cached `V1Domain`.

The safeguards validate the retained plan rather than the current V1 authorization state:

* transaction simulation succeeds because the approved migration machinery is allowed to grant the planned role;
* the post-state verifier expects the snapshotted address to receive `ROLE_SET_RESOLVER`; and
* cleanup removes the HCA's temporary operator approval, not the manager's independent per-name role.

The confirmation UI does not display the manager address. It says only **“Approve manager restoration,” “Restore the existing managers for your names,”** and **“Migrate the selected names and restore their records.”** The user cannot see that the transaction restores an address that is no longer the current V1 manager.

#### Exploit sequence

{% stepper %}
{% step %}

## Initial V1 manager assignment

The victim owns an unwrapped V1 `.eth` BaseRegistrar token. An attacker-controlled address is initially its legitimate V1 registry manager.
{% endstep %}

{% step %}

## Migration plan creation

The Manager builds a migration plan containing the attacker as `managerAddress`.
{% endstep %}

{% step %}

## Manager revocation

The victim removes that manager through canonical `BaseRegistrar.reclaim()`. The victim retains the registration, and a fresh classification would now produce `managerAddress: null`.
{% endstep %}

{% step %}

## Migration execution

The victim clicks **Upgrade** while the previously calculated plan remains active.
{% endstep %}

{% step %}

## Revoked authority restoration

The migration grants the revoked attacker V2 `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

## Cleanup

Migration cleanup removes temporary HCA access but leaves the attacker's role intact.
{% endstep %}

{% step %}

## Resolver replacement

The attacker calls `ETHRegistry.setResolver()` and points the name at an attacker-controlled resolver.
{% endstep %}

{% step %}

## Attacker-controlled resolution

ENS.js resolution through the deployed Universal Resolver returns the attacker's address.
{% endstep %}
{% endstepper %}

No attacker message, crafted link, or phishing is required. The attacker does not induce the victim to sign any transaction outside the intended workflow. After independently revoking the former manager, the victim proceeds through the normal migration flow, which constructs the unintended authority grant.

Subgraph lag can create the same stale input when revocation occurs before the page is opened. The PoC uses the deterministic after-plan change above and therefore does not rely on measuring indexer latency.

This is not public `contracts-v2` issue #418. That issue concerns a pre-existing V2 delegated role surviving a later V2 ownership transfer. Here, no V2 transfer is required: the in-scope Manager creates a new V2 role during initial migration from revoked V1 authority. The fixes are also different—invalidating roles on V2 transfer would not prevent this migration-time grant.

The ENS contracts and subgraph behave according to their interfaces. The scoped application defect is using mutable discovery data as an authorization recipient without validating its authoritative on-chain source when constructing the live transaction.

### Impact Details

The selected in-scope impact is **hijacking an ENS name during a Manager migration**. The migration gives a revoked account persistent authority. It can subsequently replace the name's resolver in its own transaction, without any further signature or approval from the current owner.

An attacker-controlled resolver can return the attacker's address for the name's ETH record. Wallets and applications resolving the name may therefore direct payments to the attacker until the owner detects the change, restores the resolver, and revokes the surviving V2 role. The PoC demonstrates the complete authority change and canonical resolution result; it does not merely infer impact from stale UI state.

The attacker must previously have been a legitimate V1 manager, and the victim remains able to repair the name. Those constraints reduce likelihood and persistence, but the revoked permission is nevertheless recreated during a victim-approved migration and produces the program's explicit Critical Manager-flow hijack impact.

No leaked key, credential theft, access to the victim's wallet, or project-level privileged address is required. The attacker controls an ordinary user-appointed manager address whose authority the registrant subsequently revokes.

### References

* [In-scope Manager asset](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager)
* [V1 owner query](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L47) and [five-minute cache](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23)
* [Manager classification](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L159-L184) and [role-grant encoding](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26)
* [Plan construction](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L44-L109) and [Upgrade handoff](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx#L253-L286)
* [Live batch rebuild from retained state](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L886-L927)
* [Stale expectation construction](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624) and [post-state verification](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353)
* [Confirmation UI](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L108)
* [Competition scope and known issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/) and [distinct public issue #418](https://github.com/ensdomains/contracts-v2/issues/418)

Target commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

## Link to Proof of Concept

<https://gist.github.com/pnupu/f7f244fd40ac305a0f9682b9bb92b706/523aa370b5e1383d1111656eb62e4f39a8e11139>

## Proof of Concept

The [`stale-manager-migration.poc.ts`](https://gist.github.com/pnupu/f7f244fd40ac305a0f9682b9bb92b706/523aa370b5e1383d1111656eb62e4f39a8e11139) PoC drives the production `computeMigrationPreflight`, `buildMigrationPlan`, and `executeMigration` services against the canonical deployed ENS V1/V2 contracts on a local Anvil Sepolia fork.

It uses Anvil impersonation only to create a fresh V1 registration and matching zero-role V2 `RESERVED` entry. Classification, plan construction, migration calldata, HCA execution, V2 role storage, cleanup, the attacker's resolver change, and canonical resolution all use the production code and deployed contracts.

### Reproduction

{% stepper %}
{% step %}

## Check out the target repository

Check out the target repository at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
{% endstep %}

{% step %}

## Install dependencies

```sh
corepack pnpm install --frozen-lockfile
```

{% endstep %}

{% step %}

## Download the PoC

Download the linked PoC and save it as `apps/manager/scripts/stale-manager-migration.poc.ts`.
{% endstep %}

{% step %}

## Start a clean local Sepolia fork

```sh
docker compose -f e2e/infra/docker-compose.yml down
docker compose -f e2e/infra/docker-compose.yml up -d --wait anvil
```

{% endstep %}

{% step %}

## Run the PoC

Run the PoC without modifying `package.json`:

```sh
corepack pnpm --dir e2e exec tsx \
  --tsconfig ../apps/manager/tsconfig.json \
  ../apps/manager/scripts/stale-manager-migration.poc.ts
```

{% endstep %}
{% endstepper %}

### What the PoC demonstrates

{% stepper %}
{% step %}

## Initial authority

The victim owns the V1 registration while the attacker is its legitimate V1 manager.
{% endstep %}

{% step %}

## Plan construction

Production code builds a plan that will grant the attacker `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

## Pre-migration role check

Before migration, the attacker has no V2 role and cannot call `setResolver`.
{% endstep %}

{% step %}

## V1 manager revocation

The victim calls `reclaim()`. Live V1 state now identifies the victim as manager, and fresh production classification returns `managerAddress: null`.
{% endstep %}

{% step %}

## Retained plan execution

Production `executeMigration()` processes the retained plan, including the real approvals, HCA batch, verification, and cleanup.
{% endstep %}

{% step %}

## Revoked role grant

The migration emits a zero-to-`ROLE_SET_RESOLVER` role change for the revoked attacker.
{% endstep %}

{% step %}

## Cleanup outcome

Cleanup removes temporary HCA access while the attacker's per-name role remains.
{% endstep %}

{% step %}

## Resolver hijack

In a separate attacker-signed transaction, the revoked manager changes the resolver. ENS.js through the deployed Universal Resolver returns the attacker address.
{% endstep %}
{% endstepper %}

Successful execution ends with:

```
[PASS] stale V1 manager restored as V2 resolver authority
victim/current owner: <victim>
revoked attacker:     <attacker>
canonical ETH result: <attacker>
```

### Expected secure behavior

Immediately before encoding a manager-restoration grant, the Manager should read the current V1 `ENSRegistry.owner(node)` and compare it with the planned `managerAddress`. If it changed, execution should fail closed and require a new confirmation showing the exact address that will receive V2 authority.
