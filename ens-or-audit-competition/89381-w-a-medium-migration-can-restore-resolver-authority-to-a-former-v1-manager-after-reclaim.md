# 89381 w a medium migration can restore resolver authority to a former v1 manager after reclaim

**Submitted on Aug 18th 2026 at 23:04:00 UTC by @hmiya for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89381
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

Asset: `apps/manager`

Competition repository head: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

Program-declared upstream revision: `63772fd872af472ced58b009499355f3430c2a86`

## Summary

ENS Manager caches the V1 name row used for migration for five minutes. For an unwrapped `.eth` name, it treats `domain.owner` as the legacy manager when that address differs from the Base Registrar registrant.

If the registrant reclaims V1 registry ownership after the page loads, the former manager loses its current V1 authority. The already loaded Manager state nevertheless retains that address. The ordinary migration then creates a new V2 `ROLE_SET_RESOLVER` grant for the former manager without re-reading the current V1 registry owner.

The former manager can use that new role to install an attacker-selected resolver. This is not theft of manager data and does not work for an arbitrary stranger: the attacker was a real V1 manager before reclaim, and the victim must finish migration from the stale page state. The defect is that ended V1 authority is restored as fresh V2 authority.

## Root cause

The following production path converts a cached identity into an on-chain authorization grant:

{% stepper %}
{% step %}

## Cached V1 name query

[`useV1Names.ts` L12-L29](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29) gives the owner-keyed V1 query a five-minute `staleTime`.
{% endstep %}

{% step %}

## Legacy manager classification

[`classifyNames.ts` L148-L185](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L148-L185) copies a differing `domain.owner.id` into `managerAddress`.
{% endstep %}

{% step %}

## Migration gas estimate retains stale domain objects

[`useMigrationGasEstimate.ts` L49-L104](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L49-L104) selects the objects from that array once and passes the same `domains` to both `ensurePreflight` and `buildMigrationPlan`.

Its `staleTime: 0` applies to the separate preflight query; it does not refetch or replace the five-minute `v1_names` row.
{% endstep %}

{% step %}

## Preflight uses supplied domain objects

[`useMigrationPreflight.ts` L25-L52](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationPreflight.ts#L25-L52) keys preflight by the supplied domain IDs and invokes `computeMigrationPreflight` with those same objects.

[`computeMigrationPreflight.ts` L39-L58 and L192-L260](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/computeMigrationPreflight.ts#L39-L58) derives approvals, resolver readiness, profiles, and routes from that input.

`MigrationPreflight` has no current V1 owner value with which to replace `domain.owner`.
{% endstep %}

{% step %}

## Migration plan reclassifies original domains

[`buildMigrationPlan.ts` L317-L372](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L372) immediately reclassifies the original `domains`; it uses preflight only for routes, resolver readiness, approvals, and profile work.

No preflight or `publicClient` result overwrites the registry owner before classification.
{% endstep %}

{% step %}

## Cached manager becomes resolver role recipient

[`buildAtomicMigrationBatches.ts` L608-L624](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624) turns non-null `managerAddress` into `ROLE_SET_RESOLVER` for that recipient.
{% endstep %}

{% step %}

## Migration precedes role grant

[`buildAtomicMigrationBatches.ts` L742-L780](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L742-L780) orders migration before the role grant.
{% endstep %}

{% step %}

## Mutable inputs are rebuilt without current V1 owner validation

[`migrationService.ts` L886-L963 and L1119-L1204](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L886-L963) rebuilds and verifies other mutable inputs, but does not obtain a current V1 registry owner before the grant is built.
{% endstep %}
{% endstepper %}

Post-batch verification only confirms that the expected account received the expected role; it does not establish that the account is still the V1 manager ([`verifyAtomicMigrationBatch.ts` L337-L353](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353)).

## Authorization and impact

The grant succeeds because migration first makes the victim the V2 name owner. Registry owners hold the resolver-role admin capability, and the HCA approved by that owner can execute the batched role grant. The PoC applies the exact calldata emitted by the production classifier/builder against the pinned official `contracts-v2` implementation.

The realized impact is unauthorized resolver control for one migrated name. The attacker can replace the name's resolution logic until the victim detects the change, revokes the role, and restores the resolver. The included recovery control proves that recovery is possible; this report does not claim ownership theft, permanence, or unrecoverable loss.

The program lists as Critical a Manager registration or migration that assigns a name's resolver to an attacker. This path reaches that exact resolver-hijacking condition. The timing and prior-delegate requirements reduce likelihood, but they do not remove the new authority or the successful resolver write.

## Preconditions and boundaries

* The attacker was the legitimate V1 Registry manager when the victim loaded the page. An arbitrary address cannot inject itself.
* The victim remains the Base Registrar registrant and reclaims/receives V1 Registry ownership after page load.
* The victim completes the ordinary migration while the retained query result is still fresh. The former manager cannot force or observe the victim's off-chain page timing; it only needs to act after the on-chain V2 role grant exists.
* The victim approves the normal HCA manager-restoration step, not an address-specific grant shown in the UI.
* The victim retains V2 owner/admin recovery authority.
* No public testnet/mainnet action, compromised wallet, malformed input, backend compromise, or live indexer manipulation is required by the local proof.

## Known-issue and design separation

The current 59-row known-issue table does not contain this root/effect tuple. QA-01 concerns errors from role combinations and expressly treats demonstrated unintended authority as a new finding. SEC-TXM-002 concerns a cached smart-account address not checked against an SDK-derived address and has no confirmed attack path; it does not use a V1 owner or create this label-scoped resolver grant.

ENSv2 intentionally permits an already delegated regular resolver role to remain through a V2 ownership transfer. That documented baseline is different: here the application creates a new V2 role from cached V1 identity data after the identity's V1 authority has ended. Migration documentation describes restoring legacy managers, but does not state that a pre-reclaim former manager should be restored.

## Recommended fix

The safest fix is to omit automatic manager restoration and let the migrated owner delegate after migration.

If restoration remains, obtain the current V1 Registry owner from an authoritative source immediately before final batch construction and bind that value to execution with an expected-owner guard. Fail closed and rebuild the plan if it differs from `managerAddress`. UI refresh alone is insufficient because the authorization identity must be bound to the transaction that grants the V2 role.

## Proof of Concept

### Step-by-step proof of concept

Use three addresses:

* victim/registrant: `0x0000000000000000000000000000000000000011`
* former V1 manager: `0x0000000000000000000000000000000000000099`
* attacker resolver: `0x0000000000000000000000000000000000000Bad`

The name is an unwrapped `.eth` 2LD. The victim owns its Base Registrar token, while `0x...0099` is initially the V1 Registry owner. V1 deliberately separates those positions; the registrant can later call the Base Registrar's ordinary `reclaim(id, owner)` path.

{% stepper %}
{% step %}

## Load migration as the victim

Connect ENS Manager as `0x...0011` and open migration. The V1 query returns `registrant=0x...0011` and `domain.owner=0x...0099`.
{% endstep %}

{% step %}

## Cache the former manager

The classifier records `managerAddress=0x...0099`, and TanStack Query keeps that row fresh for five minutes.
{% endstep %}

{% step %}

## Reclaim V1 Registry ownership

After the row is loaded, have `0x...0011` call `reclaim(labelhash, 0x...0011)` in V1. Equivalently, the current manager can return Registry ownership with `setOwner`.

The canonical V1 owner is now the victim, so `0x...0099` no longer has V1 manager authority.
{% endstep %}

{% step %}

## Continue migration from stale state

Without invalidating/refetching the already loaded V1 query, continue the normal migration inside the fresh interval.
{% endstep %}

{% step %}

## Approve ordinary manager restoration

Because manager restoration is present, the UI asks the victim for the ordinary V2 ETHRegistry HCA approval. The confirmation says that existing managers will be restored, but does not display the cached recipient address ([`WalletConfirmationStepsDialog.tsx` L81-L90](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L90)).
{% endstep %}

{% step %}

## Execute the production role grant

The victim-approved HCA migrates the label to the victim and then emits the production call below:

```
grantRoles(
  0x8ba424dc5a878630f6f6f7f1ec434fc7878df3ed69dcf0440ade5f1d00000000,
  16777216,
  0x0000000000000000000000000000000000000099
)
```

{% endstep %}

{% step %}

## Verify the unauthorized V2 role

Verify that `0x...0099` now holds label-scoped `ROLE_SET_RESOLVER` in V2, even though the address had no V2 role before this batch and no longer held the corresponding V1 manager position.
{% endstep %}

{% step %}

## Set the attacker-selected resolver

From `0x...0099`, call `setResolver(resource, 0x...0Bad)`. The call succeeds and the migrated name now uses the attacker-selected resolver.
{% endstep %}
{% endstepper %}

The same-key control in the reproduction package confirms one query-function call before invalidation and retention of `0x...0099`; explicit invalidation causes a second fetch, classifies the current victim owner, and produces no manager grant. A separate initial-plan control calls the production `buildMigrationPlan` rather than the classifier in isolation: the stale arm emits the exact grant to `0x...0099`, the current-owner arm emits no manager grant, and the plan builder makes zero `publicClient` reads in this unwrapped-name path.

## Runnable validation (optional supporting evidence)

The attached twelve-file package is fully local. `README.md` contains the exact commands. Its four checks are:

{% stepper %}
{% step %}

## Cache control

`cache-control.mts` proves the same query key retains the old owner inside `staleTime` and that invalidation refreshes it.
{% endstep %}

{% step %}

## Production differential

`production-differential.mts` plus `export-production-calldata.mts` imports the target classifier and grant builder; changing only current versus retained `domain.owner` changes no-grant into the exact grant above.
{% endstep %}

{% step %}

## Initial plan build

`initial-plan-build.mts` imports the production `buildMigrationPlan`; changing only the supplied V1 registry owner changes an exact initial migration plan from no manager grant to the exact former-manager grant, while a throwing `publicClient` trace records no chain read.
{% endstep %}

{% step %}

## Resolver hijack test

`StaleManagerResolverHijack.t.sol`, against `contracts-v2` commit `8d1c89350729f87b3968cf41ef3d683951444a44`, proves the former manager cannot write the resolver before the grant, can write it after the exact production grant, and cannot write again after victim revocation.
{% endstep %}
{% endstepper %}

Expected Forge result:

```
[PASS] test_exactProductionGrantEnablesFormerManagerResolverWrite()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

All information required to understand and reproduce the vulnerability is above; the runnable files strengthen the step-by-step proof and do not depend on an external upload location.
