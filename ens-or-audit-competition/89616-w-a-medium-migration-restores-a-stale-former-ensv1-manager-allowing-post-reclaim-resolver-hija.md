# 89616 w a medium migration restores a stale former ensv1 manager allowing post reclaim resolver hijacking

**Submitted on Aug 20th 2026 at 06:24:20 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89616
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

The in-scope Manager app decides which ENSv1 manager to restore from a subgraph snapshot. For an unwrapped `.eth` name, it records `domain.owner.id` as `managerAddress` whenever that address differs from the registrant. The query is cached for five minutes.

If the registrant reclaims manager control on ENSv1 while that snapshot is stale, the migration plan still contains the former manager. Execution checks migration approvals and source-token ownership, but it does not reread the current ENSv1 registry owner before building the atomic batch. The batch therefore grants `ROLE_SET_RESOLVER` on the new ENSv2 name to the former manager.

After migration, that former manager can replace the name's resolver even though the registrant had already removed its ENSv1 authority. The registrant still owns the ENSv2 token, so ownership alone does not reveal that resolver control was silently restored.

## Root cause

`apps/manager/src/features/migration/service/classifyNames.ts:162-184` copies the subgraph's registry owner into the long-lived classification:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

`apps/manager/src/features/migration/hooks/useV1Names.ts:12-29` obtains this data through `getV1NamesForAddress()` and gives it a five-minute `staleTime`.

`apps/manager/src/features/migration/service/migrationService.ts:900-927` rebuilds the transaction using the same `classified` entries. Its fresh `directRoutes` and approval checks do not update `managerAddress`.

Finally, `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-26` unconditionally encodes:

```ts
grantRoles(resource, ROLE_SET_RESOLVER, name.managerAddress)
```

The post-state verifier treats that stale address receiving the role as the expected successful state (`buildAtomicMigrationBatches.ts:608-624`).

## Preconditions

1. An unwrapped `.eth` registration has a registrant and a separate ENSv1 manager.
2. The registrant reclaims manager control from that address.
3. The migration app still has the earlier subgraph/cache view when the registrant previews and executes migration.

The former manager does not need any role after step 2. The authority is restored by the migration transaction signed by the registrant.

## Attack path and impact

1. The victim is the `.eth` registrant; the attacker is its former ENSv1 manager.
2. The victim reclaims the ENSv1 manager field, intentionally removing the attacker's control.
3. Before the subgraph/cache reflects that change, the victim migrates the name through the scoped manager app.
4. The app's atomic batch grants the former manager `ROLE_SET_RESOLVER` on the ENSv2 resource.
5. The attacker calls `setResolver()` and points the migrated name at a resolver under the attacker's control.
6. Address records can then resolve to attacker-chosen destinations while the victim still appears as token owner.

This creates unauthorized post-reclaim authority and can redirect users interacting with the migrated name. The competition explicitly classifies a v1-to-v2 migration that assigns a name, registry, or resolver to an attacker as Critical. The timing and former-manager preconditions are stated above and do not change that exact impact mapping.

## Recommended fix

Immediately before building each atomic migration batch, read the name's current ENSv1 registry owner from chain and derive the manager grant from that value. If it differs from the previewed value, fail closed and require a new review.

The confirmation screen should also display the exact address receiving `ROLE_SET_RESOLVER`; however, display alone should not replace the execution-time on-chain binding.

## Known-issue distinction

The published ENS competition material acknowledges that some transfer/role combinations can later produce functional errors. This report is not an error-only variant: migration grants a former manager a live resolver-changing permission after the registrant removed that authority. The security consequence and required fix are both specific to binding the restoration grant to current on-chain state.

## Target

Scoped repository commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

Affected asset: `apps/manager`

## Link to Proof of Concept

<https://gist.github.com/krutftw/4734fb645d63b8e13db9b7259d1463de>

## Proof of Concept

## Proof of concept

The package contains two tests:

* `poc-stale-manager-grant.test.ts` imports the scoped manager app's real `buildRoleGrantCall()`. It proves the encoded grantee remains the former manager after the live manager has changed.
* `StaleManagerGrantImpactPoC.t.sol` uses the ENSv2 `PermissionedRegistry`. It grants the exact role/address pair emitted by the app and proves the former manager can replace the resolver while the registrant remains the current token owner.

### App test

Place `poc-stale-manager-grant.test.ts` at:

```
apps/manager/src/features/migration/service/poc-stale-manager-grant.test.ts
```

Run from the audit repository root:

```bash
pnpm --filter manager exec vitest run src/features/migration/service/poc-stale-manager-grant.test.ts
```

Expected result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

### Contract impact test

Place `StaleManagerGrantImpactPoC.t.sol` at:

```
contracts/test/unit/registry/StaleManagerGrantImpactPoC.t.sol
```

Run from the `contracts` directory of the ENSv2 contracts checkout:

```bash
forge test --match-path test/unit/registry/StaleManagerGrantImpactPoC.t.sol -vvv
```

Expected result:

```
[PASS] test_staleManagerGrantCanReplaceTheMigratedNamesResolver()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Secret gist containing both executable tests:

<https://gist.github.com/krutftw/4734fb645d63b8e13db9b7259d1463de>
