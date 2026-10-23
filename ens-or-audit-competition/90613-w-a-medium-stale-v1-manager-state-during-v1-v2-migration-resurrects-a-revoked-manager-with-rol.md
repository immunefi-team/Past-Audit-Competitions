# 90613 w a medium stale v1 manager state during v1 v2 migration resurrects a revoked manager with role set resolver enabling resolver hijacking

**Submitted on Aug 25th 2026 at 23:52:54 UTC by @volpasec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90613
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

The ENS Manager v1→v2 migration flow does not revalidate the current V1 manager before execution. If a V1 manager is revoked after the migration plan is built but before the owner signs, Manager still reuses the manager address stored in the stale plan and grants that address fresh V2 `ROLE_SET_RESOLVER`. The revoked account can then change the migrated name's resolver independently, allowing it to hijack or brick resolution while the legitimate owner still owns the name.

## Vulnerability Details

For an unwrapped `.eth` name, `classifyName()` derives the legacy manager from the V1 Registry owner. If the Registry owner differs from the BaseRegistrar registrant, that address is stored as `managerAddress` in the resulting `ClassifiedName`.

`packages/migration/src/service/classifyNames.ts`:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null

return {
  type: 'classified',
  name: {
    domain,
    tokenType: 'unwrapped',
    label,
    parentName,
    fuses: 0n,
    tokenHolder,
    v1ResolverAddress,
    resolverStrategy: resolverStrategyFor({
      tokenType: 'unwrapped',
      fuses: 0n,
      v1ResolverAddress,
    }),
    managerAddress,
  },
}
```

This classification is correct when it is created. The issue is that Manager treats `managerAddress` as stable even though the underlying V1 Registry owner can change before the migration is signed.

In an ordinary session, domains is not re-fetched per plan build: MigrationPage.tsx reads it from useV1Names(), whose query (useV1Names.ts) has a 5-minute staleTime. A user who revokes a V1 manager and continues an already-open migration within that window — for example, cleaning up an old manager grant immediately before migrating — reaches this exact path without any unusual action.

Immediately before the first wallet prompt, `assertMigrationApprovalPlanCurrent()` attempts to detect a stale permission preview. However, the manager-restoration requirement is derived from the already-created migration plan:

`apps/manager/src/features/migration/service/migrationService.ts`:

```ts
const getMissingMigrationApprovals = async (params: {
  readonly ctx: MigrationCtx
  readonly plan: MigrationPlan
}): Promise<readonly MigrationApproval[]> => {
  const { ctx, plan } = params
  const basicNeeds = approvalNeedsFor(plan.groups)
  const needs = {
    ...basicNeeds,
    requiresManagerRestoration: plan.classified.some(
      (name) => name.managerAddress !== null,
    ),
  }

  const status = await checkMigrationApprovals({
    eoa: ctx.walletAddress,
    hcaAddress: ctx.hcaAddress,
    needs,
    wagmiConfig: ctx.wagmiConfig,
  })

  return planMigrationApprovals({
    hcaAddress: ctx.hcaAddress,
    needs,
    status,
  })
}
```

`assertMigrationApprovalPlanCurrent()` then compares the planned and currently-missing approval keys:

```ts
const missing = await getMissingMigrationApprovals(params)

const plannedApprovalKeys = sortedApprovalKeys(
  params.plan.preflight.migrationApprovals ?? [],
)
const currentApprovalKeys = sortedApprovalKeys(missing)
```

This detects changes to the approval topology, but it does not verify that the account for which manager authority will be restored is still the current V1 Registry manager.

For example, suppose the plan contains `managerAddress = A2`. The owner then revokes A2 by calling `BaseRegistrar.reclaim(tokenId, owner)`. The manager-restoration requirement remains `true` because `plan.classified` still contains A2, so the approval-key comparison can still succeed even though the identity that should receive the role has changed.

The stale `ClassifiedName` is also reused when the actual migration transaction is rebuilt. `executeRemainingAtomicBatches()` starts from the original classification:

```ts
let remaining = [...params.plan.classified]
```

and `buildNextAtomicBatch()` passes it directly into the production batch builder:

```ts
const atomicPlan = await buildAtomicMigrationBatches({
  chainId: ctx.publicClient.chain?.id ?? 11155111,
  hca: ctx.hcaAddress,
  wallet: ctx.walletAddress,
  classified: remaining,
  directRoutes,
  profiles: plan.profiles,
  // ...
})
```

There is no re-classification from the current V1 Registry owner before the role grant is constructed.

`buildAtomicMigrationBatches()` adds a `manager-role-grant` whenever the stale classification contains a manager:

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })
}
```

Finally, `buildRoleGrantCall()` creates fresh V2 resolver authority for that stored address:

`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`:

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

A concrete sequence is:

{% stepper %}
{% step %}

## V1 ownership and manager state

`V` owns the BaseRegistrar ERC-721 for `alice.eth`, while `A2` is its legitimate V1 Registry manager.
{% endstep %}

{% step %}

## Migration plan creation

Manager creates a migration plan and stores `managerAddress = A2`.
{% endstep %}

{% step %}

## Manager revocation

Before migration is signed, `V` calls `BaseRegistrar.reclaim(tokenId, V)`. The live V1 Registry owner is now `V`; `A2` has been revoked.
{% endstep %}

{% step %}

## Fresh classification

A fresh `classifyName()` on that state returns `managerAddress = null`.
{% endstep %}

{% step %}

## Stale plan remains

The existing migration plan still contains `managerAddress = A2`.
{% endstep %}

{% step %}

## Approval recheck misses manager change

The pre-signature approval recheck does not re-read the V1 Registry manager, so the stale plan survives.
{% endstep %}

{% step %}

## Migration grants stale manager role

The production migration batch executes `migrate` followed by `manager-role-grant`, granting `ROLE_SET_RESOLVER` to A2.
{% endstep %}

{% step %}

## Post-state verification accepts stale expectation

Manager's own post-state verifier accepts the role as expected state because its expectation was also built from the stale classification.
{% endstep %}

{% step %}

## HCA cleanup does not revoke A2 role

The normal migration cleanup revokes the temporary HCA operator approval, but A2's newly granted role remains.
{% endstep %}

{% step %}

## Revoked manager changes resolver

A2 can then call `setResolver()` independently.
{% endstep %}
{% endstepper %}

The linked PoC (Secret Gist) reproduces this entire sequence against the canonical local contracts. Immediately before the migration transaction, it verifies that the live V1 manager is the victim, the stale plan still points to A2, and A2 has no V2 resolver role. After the migration, the victim remains the V2 owner but A2 has `ROLE_SET_RESOLVER`. After the official HCA cleanup, that role is still present, and A2 successfully changes the resolver without any further victim transaction.

The documented behavior also suggests this state transition is unintended. `apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md` states that permission state is rechecked before the first wallet prompt and that changed state should cause the owner to review a refreshed preview. In this case, the manager identity changes, but the recheck does not observe it.

## Impact Details

The applicable in-scope Critical impact is:

> "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker."

The PoC demonstrates this impact directly.

A2 has been revoked in V1 before the owner signs the migration and has no V2 resolver role immediately before execution. The Manager migration then creates new `ROLE_SET_RESOLVER` authority for A2. After migration and after the temporary HCA authority has been cleaned up, A2 can submit its own `setResolver()` transaction successfully.

The legitimate owner is not replaced: the PoC verifies that the victim remains the V2 owner. The unauthorized state change is the resolver authority itself.

Because `ROLE_SET_RESOLVER` permits A2 to select an arbitrary resolver address, A2 can point the name to an attacker-controlled resolver or set an unusable resolver and brick resolution. The PoC sets the resolver to A2's chosen address and verifies the state change on-chain.

The fact that the legitimate owner may later revoke A2 or repair the resolver does not prevent the impact. Between migration and recovery, an account that was explicitly revoked before migration has independent authority over the name's resolver.

## References

* `packages/migration/src/service/classifyNames.ts` — `classifyName()`
* `apps/manager/src/features/migration/service/migrationService.ts` — `getMissingMigrationApprovals()`, `assertMigrationApprovalPlanCurrent()`, `buildNextAtomicBatch()`, `executeRemainingAtomicBatches()`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts`
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`
* `apps/manager/src/features/migration/service/migrationApprovals.ts`
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts`
* `apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md`
* Competition Known Issues, QA-01. QA-01 covers functional errors caused by role combinations after transfers. The Known Issues document explicitly states that a demonstrated security consequence that grants unintended authority goes beyond that known breakage and is a new finding.
* Runnable PoC: `mig01-onchain.integration.test.ts` (attached in gist)
* Secret Gist mirror: `https://gist.github.com/volpa32/0d8c05a50452b13bf2179a04149f8d98`

## Link to Proof of Concept

<https://gist.github.com/volpa32/0d8c05a50452b13bf2179a04149f8d98>

## Proof of Concept

The full test is included as a Secret Gist (`mig01-onchain.integration.test.ts`), reproducing the issue end-to-end against a local Anvil fork using the same canonical V1/V2 contract addresses and production Manager migration builders.

The PoC does not modify production application logic. Local Anvil impersonation/setup is only used to recreate the premigration state required by the repository's own migration development tooling. The security-relevant sequence — granting A2 legitimate V1 manager access, revoking A2, building the stale production migration, executing it, cleaning up temporary HCA access, and A2 changing the resolver — is performed and verified on-chain.

Secret Gist mirror: `https://gist.github.com/volpa32/0d8c05a50452b13bf2179a04149f8d98`

From the repository root, with the local ENS test environment running:

```bash
npx -y pnpm@10.27.0 --filter manager exec vitest run \
  src/features/migration/service/mig01-onchain.integration.test.ts \
  --reporter=verbose
```

The test performs the following steps:

{% stepper %}
{% step %}

## Verify canonical contracts

Verify the exact canonical contracts used by the tested Manager deployment:

```
V1 BaseRegistrar = 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
V1 Registry      = 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e
V2 ETHRegistry   = 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
```

{% endstep %}

{% step %}

## Create V1 name and reserve V2 entry

Create a real unwrapped V1 `.eth` ERC-721 for the victim and create the corresponding `RESERVED` entry in the V2 registry.

```
[MIG-01] register canonical V1 ERC721: ... status=success
[MIG-01] reserve name in CURRENT Manager V2: ... status=success
[MIG-01] V2 status after reservation = 1
```

{% endstep %}

{% step %}

## Establish A2 as V1 manager

Make A2 the legitimate V1 manager while the victim remains the BaseRegistrar ERC-721 holder. Run the production classifier and record the resulting migration snapshot:

```
[MIG-01] ===== t0 SNAPSHOT =====
[MIG-01] BaseRegistrar holder = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[MIG-01] V1 manager           = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[MIG-01] cached manager       = 0x70997970c51812dc3a010c7d01b50e0d17dc79c8
```

{% endstep %}

{% step %}

## Revoke A2 in V1

Before migration, the victim revokes A2 on-chain by reclaiming the V1 Registry ownership.

A fresh production classification now returns no manager, while the already-built migration plan still contains A2:

```
[MIG-01] REVOKE A2 in V1: ... status=success

[MIG-01] ===== t1 AFTER REVOKE =====
[MIG-01] LIVE V1 manager       = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[MIG-01] fresh classification  = null
[MIG-01] stale classification  = 0x70997970c51812dc3a010c7d01b50e0d17dc79c8
```

{% endstep %}

{% step %}

## Deploy and approve HCA

Deploy/verify the real Standalone HCA and submit the same approvals used by Manager:

```
[MIG-01] REAL HCA = 0x48B9c6898baFc8A3D3a495BF7c44CF3351486628
[MIG-01] approve REAL ERC721 to MigrationHelper: ... status=success
[MIG-01] temporarily approve HCA for manager restoration: ... status=success
[MIG-01] APPROVALS CONFIRMED ON CHAIN
```

{% endstep %}

{% step %}

## Build production migration batch

Build the migration with the production batch builder. The resulting batch contains the stale manager restoration after the migration:

```
[MIG-01] ===== PRODUCTION BATCH =====
[MIG-01] phases = resolver-deployment -> wallet-co-admin-grant -> migrate -> manager-role-grant
```

{% endstep %}

{% step %}

## Verify state immediately before signature

Immediately before the owner signs the migration transaction, verify the relevant live state:

```
[MIG-01] ===== IMMEDIATELY BEFORE SIGNATURE =====
[MIG-01] LIVE V1 manager   = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[MIG-01] STALE plan manager = 0x70997970c51812dc3a010c7d01b50e0d17dc79c8
[MIG-01] A2 V2 role before = false
```

At this point A2 is no longer the V1 manager and has no V2 resolver authority.
{% endstep %}

{% step %}

## Execute migration batch

Execute the actual production migration batch:

```
[MIG-01] EXECUTE production migration batch: ... status=success
```

The migration completes, Manager's own post-state verifier accepts it, and A2 now has fresh V2 resolver authority:

```
[MIG-01] ===== AFTER MIGRATION =====
[MIG-01] V2 owner = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[MIG-01] REVOKED A2 ROLE_SET_RESOLVER = true
[MIG-01] Manager verifier = CONFIRMED
```

{% endstep %}

{% step %}

## Perform migration cleanup

Perform the normal migration cleanup. The HCA approval is removed, but A2's newly-created role survives:

```
[MIG-01] cleanup temporary HCA approval: ... status=success

[MIG-01] ===== AFTER CLEANUP =====
[MIG-01] HCA approved = false
[MIG-01] stale A2 role survives = true
```

{% endstep %}

{% step %}

## Change resolver as revoked manager

Without any further victim transaction or signature, A2 submits `setResolver()`:

```
[MIG-01] REVOKED A2 changes resolver alone: ... status=success
```

{% endstep %}
{% endstepper %}

Final state:

```
[MIG-01] ================================================
[MIG-01] EXPLOIT CONFIRMED ON REAL LOCAL CONTRACT STATE
[MIG-01] ------------------------------------------------
[MIG-01] A2 legitimate V1 manager at t0       = true
[MIG-01] A2 revoked BEFORE migration          = true
[MIG-01] fresh classification manager         = null
[MIG-01] stale production classification      = A2
[MIG-01] real production migration executed   = true
[MIG-01] revoked A2 received SET_RESOLVER     = true
[MIG-01] official HCA cleanup completed       = true
[MIG-01] A2 role survived cleanup             = true
[MIG-01] A2 changed resolver independently    = true
[MIG-01] final owner    = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[MIG-01] final resolver = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[MIG-01] ================================================

Test Files  1 passed (1)
Tests       1 passed (1)
```

This demonstrates the relevant state transition directly:

```
A2 is legitimate V1 manager
        ↓
Manager caches A2
        ↓
victim revokes A2 on-chain
        ↓
fresh classification = no manager
        ↓
stale migration plan still contains A2
        ↓
production migration succeeds
        ↓
A2 receives fresh ROLE_SET_RESOLVER
        ↓
temporary HCA permission is cleaned up
        ↓
A2 role remains
        ↓
A2 changes the resolver independently
```
