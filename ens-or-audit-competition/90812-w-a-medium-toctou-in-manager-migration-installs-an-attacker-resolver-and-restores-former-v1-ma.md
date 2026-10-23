# 90812 w a medium toctou in manager migration installs an attacker resolver and restores former v1 manager authority

**Submitted on Aug 27th 2026 at 12:13:46 UTC by @sl1de for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90812
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

Manager can bind a V1 name's mutable manager and resolver into a cached migration plan, then reuse those values after both have changed on-chain. A former per-name V1 manager can place an attacker-controlled resolver on the name, let Manager preview that state, and then restore the safe resolver and relinquish V1 management before the victim approves the final migration. Manager does not re-read either authority value. Its owner-authorized HCA transaction installs the stale attacker resolver in V2 and grants the former manager `ROLE_SET_RESOLVER`. The migrated name immediately resolves to the attacker, and the former manager can later rotate the resolver even though its V1 authority was gone before final owner authorization. This directly matches the Critical impact for a Manager V1 to V2 migration that assigns a name's resolver to an attacker.

## Vulnerability Details

### Attack scenario

There are two actors:

* The victim is the V1 `.eth` registrant and uses Manager's normal V1 to V2 migration flow.
* The attacker is an ordinary per-name delegate who was previously made the V1 Registry owner, commonly called the manager. The attack requires no project privilege, HCA authority, victim key, wallet session, or existing V2 `ROLE_SET_RESOLVER` on the target name.

The attacker controls a custom resolver that returns an attacker address for the victim's name.

The complete attack is:

{% stepper %}
{% step %}

## Install an attacker-controlled resolver

While still the legitimate V1 manager, the attacker sets the name's resolver to an attacker-controlled custom resolver.
{% endstep %}

{% step %}

## Capture the V1 state in Manager

The victim opens Manager's migration flow. The captured V1 snapshot can be accurate at this moment: the victim is the registrant, the attacker is the manager, and the attacker resolver is installed.
{% endstep %}

{% step %}

## Create a migration plan

Manager classifies the name and creates a migration plan that preserves the custom resolver and restores the current manager in V2.
{% endstep %}

{% step %}

## Complete the name-specific approval

For a single unwrapped name that still needs token approval, the victim completes Manager's normal name-specific `BaseRegistrar.approve(MigrationHelper, tokenId)` step. This mined transaction gives the attacker an observable signal that the migration sequence is active.
{% endstep %}

{% step %}

## Restore safe V1 state

Before the victim approves the later atomic migration batch, the attacker restores the safe V1 resolver and transfers V1 Registry ownership back to the victim. The attacker can no longer call V1 `setResolver`.
{% endstep %}

{% step %}

## Retain the stale migration plan

The live V1 name is now victim-controlled and safe, but Manager retains the plan built from the earlier manager and resolver values.
{% endstep %}

{% step %}

## Build the final batch from stale data

Manager checks and submits the remaining approvals, then rebuilds the final atomic batch from the stale `classified` objects in that plan. Its live route, readiness, and gas checks do not re-read V1 Registry ownership or the V1 resolver.
{% endstep %}

{% step %}

## Approve the migration

The victim approves the HCA transaction described as upgrading the name and restoring its records.
{% endstep %}

{% step %}

## Migrate using the stale resolver

The transaction first migrates the name with the attacker's old resolver, then calls `grantRoles(resource, ROLE_SET_RESOLVER, attacker)`.
{% endstep %}

{% step %}

## Obtain standing V2 resolver authority

The V2 name still belongs to the victim, but it resolves to the attacker. The attacker also has standing V2 resolver authority and can replace the resolver again until the victim separately revokes that role.
{% endstep %}
{% endstepper %}

The security-sensitive state transition is:

```
Accurate preview snapshot
  manager  = attacker
  resolver = attacker resolver
          |
          v
Victim mines the normal name-specific migration approval
          |
          v
Former manager restores the safe resolver and relinquishes management
  live manager  = victim
  live resolver = safe resolver
          |
          v
Manager reuses the stale classified snapshot
          |
          v
Owner-authorized HCA batch contains
  migrate(... resolver = attacker resolver)
  grantRoles(... ROLE_SET_RESOLVER, attacker)
          |
          v
V2 name resolves to attacker and attacker can rotate the resolver
```

No malicious subgraph response or indexer lag is required. The response can be completely correct when first returned. The failure is that Manager treats mutable authority data as reusable after the on-chain state has changed.

### Root cause 1: Manager snapshots the mutable manager and resolver

The V1 query requests the fields that control this outcome:

```graphql
domains {
  id
  labelName
  labelhash
  name
  resolver { address }
  owner { id }
  registrant { id }
  ...
}
```

For an unwrapped `.eth` name, `registrant` identifies the ERC-721 registrant while `owner` identifies the current V1 Registry owner or manager. Both `owner` and `resolver.address` can change after the response is cached.

The V1 names query is considered fresh for five minutes:

```ts
queryKey: qk('migration', 'v1_names', {
  address: address?.toLowerCase(),
}),
queryFn: enabled && address
  ? () => getV1NamesForAddress(address)
  : skipToken,
staleTime: 5 * 60 * 1000,
```

The attack does not require waiting for an inaccurate response. The attacker changes the live V1 state only after Manager has correctly captured the earlier state.

### Root cause 2: the migration plan cache key ignores authority changes

`useMigrationGasEstimate` selects full domain objects, but its query identity is derived only from their stable IDs:

```ts
const domains = selectDomainsFromNames(v1Names, selectedNames)
const domainIds = domains
  .map((domain) => domain.id)
  .sort()
  .join(',')
```

Its revision increases only when `domainIds` changes. The query key contains the wallet, HCA, domain IDs, and that ID-only revision:

```ts
queryKey: [
  'migration-gas-estimate',
  ownerAddress?.toLowerCase() ?? '',
  hcaAddress?.toLowerCase() ?? '',
  domainIds,
  selectionRevisionRef.current.revision,
]
```

Changing `domain.owner.id` and `domain.resolver.address` does not change any of those values. The hook can therefore keep returning the old `plan`, even if a refreshed V1 domain object for the same ID is supplied. The attached regression also proves that an already-started plan refetch can capture the old domain object and complete after fresh same-ID data arrives.

The preflight query has the same identity weakness. It also keys only on the EOA, HCA, and sorted domain IDs. Neither cache key fingerprints the manager, resolver, registrant, wrapper owner, or other security-sensitive name state.

### Root cause 3: classification turns the snapshot into authorization-bearing fields

For an unwrapped name, classification confirms that the connected migration owner is the registrant, then derives `managerAddress` from the V1 Registry owner:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

It also copies the V1 resolver directly:

```ts
const v1ResolverAddress = domain.resolver?.address ?? null
```

An unknown custom resolver is deliberately classified as `keep-v1`:

```ts
if (v1ResolverAddress && !isKnownPublicResolver(v1ResolverAddress)) {
  return 'keep-v1'
}
```

The resulting `ClassifiedName` therefore carries both stale security-sensitive values:

```
managerAddress      = former manager
v1ResolverAddress   = attacker resolver
resolverStrategy    = keep-v1
```

This behavior is safe only if Manager proves those values are still current at the final authorization boundary.

### Root cause 4: the pre-submit checks never re-read V1 manager or resolver

Before the wallet sequence, `assertMigrationApprovalPlanCurrent` refreshes only the approvals required by the existing plan. It compares approval keys derived from that same plan:

```ts
const missing = await getMissingMigrationApprovals(params)
const plannedApprovalKeys = sortedApprovalKeys(
  params.plan.preflight.migrationApprovals ?? [],
)
const currentApprovalKeys = sortedApprovalKeys(missing)
```

It does not reload the name and does not compare the live V1 Registry owner or resolver with the values embedded in `plan.classified`.

Immediately before each atomic submission, `buildNextAtomicBatch` does perform live checks, but only for resolver deployment readiness, migration routes, and gas estimation. It passes the previously classified objects back into the batch builder unchanged:

```ts
const resolverReadiness =
  await checkDeterministicMigrationResolverReadiness(...)
const directRoutes = await resolveDirectMigrationRoutes({
  publicClient: ctx.publicClient,
  classified: remaining,
})

const atomicPlan = await buildAtomicMigrationBatches({
  classified: remaining,
  profiles: plan.profiles,
  ...
})
```

`remaining` comes directly from `params.plan.classified`. There is no call that reclassifies from current V1 Registry state.

### Root cause 5: the stale values become signed migration and role arguments

The atomic builder resolves `keep-v1` from `classified.v1ResolverAddress` and places it into `Migration.Data`:

```ts
const expectedResolver = resolverFor(
  classified,
  defaultResolver,
  resolver,
)
const migrationData = createMigrationData({
  label: classified.label,
  owner: wallet,
  subregistry: zeroAddress,
  resolver: expectedResolver,
})
```

If `classified.managerAddress` is non-null, the same batch adds a role grant:

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    call: buildRoleGrantCall(classified),
  })
}
```

`buildRoleGrantCall` encodes:

```ts
grantRoles(
  labelToCanonicalId(name.label),
  1n << 24n,
  name.managerAddress,
)
```

The `1n << 24n` bit is `ROLE_SET_RESOLVER`. The final phase ordering places migration before that grant, then wraps both calls in one `executeByOwner` request to the victim's HCA.

For the verified payload, the two inner calls decode as:

```
1. MigrationHelper.migrate([
     {
       label: "turnkey",
       owner: victim,
       subregistry: 0x0000000000000000000000000000000000000000,
       resolver: 0x2000000000000000000000000000000000000002
     }
   ])

2. ETHRegistry.grantRoles(
     resource,
     16777216,
     0x5f71c6074141d41d0e769689dC9830388a5C7e3E
   )
```

The post-state verifier does not detect the mismatch. Its expected resolver and manager-role recipient were created from the same stale classified object, so installing both stale values satisfies its checks.

### Why Manager confirmation does not expose the stale recipients

The Manager step model represents the operation only as an atomic batch count. Its visible copy says `Upgrade 1 name` and `Migrate the selected names and restore their records.` The manager approval copy says `Approve manager restoration` and `Restore the existing managers for your names.`

The step descriptors do not carry the manager address or resolver address. The regression confirms that the application preview model contains neither the attacker address nor the attacker resolver. A wallet may independently decode nested calldata, but Manager itself does not identify the authority that is being restored.

### Verified joined payload-to-state result

I reproduced the complete contract consequence on a private disposable Sepolia fork at block `11577300` using the competition import `cda79acaad59711b943fc68207ebb3f1d0ff8596`, which imports upstream commit `63772fd872af472ced58b009499355f3430c2a86`.

The proof has three connected evidence layers:

1. Production React Query hook regressions prove that same-ID manager and resolver changes do not invalidate the cached migration plan, including the in-flight refetch ordering case.
2. The production migration builders consume the captured snapshot, decode the stale migration and role-grant calls, and prove that a fresh snapshot would instead use the safe resolver and omit the manager grant.
3. After the live V1 state is made safe on the fork, the proof reproduces Manager's submission-mode batch rebuild with live route resolution, resolver-readiness checks, and live gas estimation. It proves that the rebuilt bytes equal the preview bytes, then mines those exact bytes through the real deployed StandaloneHCA implementation.

The application hook test uses the ordinary ASCII fixture `stale-manager.eth`. The private-fork execution uses the ordinary ASCII fixture `turnkey.eth`. Each evidence layer keeps one name identity internally; the layers test different invariants and do not depend on Unicode or display ambiguity.

| Item                                                    | Verified value                                                       |
| ------------------------------------------------------- | -------------------------------------------------------------------- |
| Competition commit                                      | `cda79acaad59711b943fc68207ebb3f1d0ff8596`                           |
| Fork block and chain                                    | Sepolia block `11577300`, chain ID `11155111`                        |
| Execution fixture                                       | `turnkey.eth`                                                        |
| Fresh local victim                                      | `0x1000000000000000000000000000000000000001`                         |
| Attacker and former V1 manager                          | `0x5f71c6074141d41d0e769689dC9830388a5C7e3E`                         |
| Cached attacker resolver                                | `0x2000000000000000000000000000000000000002`                         |
| Live safe resolver before payload generation and mining | `0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5`                         |
| Production HCA                                          | `0x7F1C9855cE6ea3BFd111b1ed783374F34fb14698`                         |
| Observable first approval                               | `0x1ebd6a3a729743236ce8ef67b1d7a83bb459159648e1614e7b5b0631b47d7d13` |
| SHA-256 of the `0x`-prefixed calldata text              | `c8a68628de44afc4da3e557647e714b7a4c5fe42956567097ec1c19cecbb5a59`   |
| Preview equals submission rebuild                       | `true`                                                               |
| Exact HCA transaction                                   | `0x7ada5509922824ad8d7e8a91741d7ba3633f1971f844e3c7841777d38685d032` |
| Mined input equals production payload                   | `true`                                                               |
| Former manager V1 call after relinquishment             | Rejected                                                             |
| V2 owner after migration                                | Victim                                                               |
| V2 resolver after migration                             | Attacker resolver                                                    |
| V2 ETH resolution after migration                       | Attacker                                                             |
| Former manager role before migration                    | `false`                                                              |
| Former manager role after migration                     | `true`                                                               |
| Resolver after former manager uses restored role        | `0x2100000000000000000000000000000000000012`                         |
| Final ETH resolution                                    | Attacker                                                             |
| App regression tests                                    | 3 passed                                                             |
| Production payload export test                          | 1 passed                                                             |
| Joined result                                           | `PASS`                                                               |
| Cleanup                                                 | `servicesOwnedByRunnerStopped=true`                                  |

The local proof deliberately moves the V1 registrar token to a fresh victim before the scenario. This removes any historical operator approval from the fixture. It then proves that the former manager is unable to call V1 `setResolver` after relinquishing V1 Registry ownership. The authority observed after migration is newly restored by the victim-authorized V2 batch.

### Fresh-state negative control

The same production builder is run with the authoritative live values instead of the captured snapshot:

```
fresh manager  = victim
fresh resolver = safe resolver
```

That control produces only the migration phase, installs the safe resolver, and emits no `manager-role-grant` phase. This isolates the exploit to stale authority reuse rather than a required property of V1 to V2 migration.

### Controls that work but do not stop the exploit

* Manager checks that the connected migration owner is the V1 registrant. The victim is the legitimate registrant.
* No inaccurate backend response is required because the cached manager and resolver values can be accurate when captured.
* The approval plan is refreshed before the wallet sequence. That refresh checks only approvals required by the existing stale plan.
* Submission mode refreshes migration routes, deterministic resolver readiness, and gas. It does not refresh V1 manager or resolver state.
* The former manager loses the ability to call V1 `setResolver` before the final batch. The proof verifies that direct call is rejected.
* The HCA enforces owner authorization and executes the exact submitted bytes. Those bytes already contain the stale resolver and grantee.
* Post-state verification succeeds because its expected values are derived from the stale plan.
* The `ROLE_SET_RESOLVER` grant is separate from temporary HCA operator approvals. The attached proof executes the atomic migration batch, not Manager's later cleanup phase, and directly proves that the former-manager role remains usable until separately revoked.

### Preconditions and boundaries

The following requirements are explicit:

* The attacker was previously delegated per-name V1 Registry manager authority.
* While authorized, the attacker set a custom resolver that it controls.
* The victim began migration while that state was accurately represented in Manager's snapshot.
* The selected unwrapped name needed at least one separate, observable approval, or the attacker otherwise learned that the cached migration was in progress.
* Before the final HCA batch, the attacker restored the safe resolver and relinquished V1 management to the victim.
* The cached plan remained active through the victim's final migration approval.
* The victim approved Manager's normal connected-wallet migration transaction.

The following are not required or claimed:

* No stolen private key, session, HCA authority, or browser storage access.
* No project-privileged role, compromised RPC, malicious subgraph response, or backend compromise.
* No mutation of calldata after the victim approves it.
* No Unicode, name-normalization, or display-name ambiguity.
* The proof does not send transactions to public Sepolia or mainnet.
* The proof does not claim a public-network funds transfer or NFT transfer.
* The proof does not claim that every wallet hides nested calldata. The defect is that Manager builds stale authorization arguments and does not disclose them itself.
* The attached proof does not automate the full browser and wallet-extension interface. It proves the hook retention separately, generates the exact production submission payload, mines that payload through the deployed HCA on the private fork, and verifies the resulting authority.

## Impact Details

This maps directly to the selected Critical Manager impact:

> Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker; redirecting the recipient of a name transfer; leaving a name permanently inaccessible after the flow.

The final victim-authorized migration installs an attacker-controlled resolver. The verified resolver immediately returns the attacker as the name's ETH address. This is a completed resolver-hijack state transition, not merely a misleading preview or a transaction that later reverts.

The final owner-authorized HCA transaction contains two security-sensitive stale arguments:

* `Migration.Data.resolver` is the attacker's old resolver rather than the safe live V1 resolver.
* The `grantRoles` recipient is the former manager even though that account no longer has V1 authority.

The victim approves a migration after those values have ceased to represent the live V1 name. The chain then executes the exact arguments supplied by Manager.

From the attacker's perspective, previously legitimate per-name manager authority is enough to seed the cached snapshot. After relinquishing that authority and restoring a safe live state, the attacker receives a malicious resolver installation and fresh V2 `ROLE_SET_RESOLVER` authority through the victim's migration.

From the victim's perspective, both the manager and resolver are safe immediately before final owner authorization, yet the completed V2 migration revives both stale values. The victim remains the nominal V2 owner, but name resolution is attacker-controlled and can be changed again by the attacker.

The direct verified consequence is control of the migrated name's resolver and ETH resolution. Applications that rely on the affected ETH resolution would receive the attacker address. The proof does not rely on assigning a speculative market value to the name or completing a downstream transfer.

## Recommended Remediation

{% stepper %}
{% step %}

## Treat mutable authority state as authorization-bearing input

Treat the V1 Registry owner, resolver, registrant, wrapper owner, token state, and relevant fuses as authorization-bearing inputs, not display metadata.
{% endstep %}

{% step %}

## Require explicit manager restoration approval

Do not restore an arbitrary prior manager automatically. Require an explicit, address-level user choice for every role recipient.
{% endstep %}

{% step %}

## Revalidate before final wallet authorization

Immediately before the final wallet request, reload every selected name from authoritative chain state, reclassify it, and compare a complete authority fingerprint with the reviewed plan. If anything changed, discard the plan and require a new review.
{% endstep %}

{% step %}

## Add an execution-time invariant

The migration transaction should either read the current V1 resolver and manager atomically or require that their current on-chain values equal the reviewed expected values. A frontend-only re-read still leaves a race before mining.
{% endstep %}

{% step %}

## Include authority values in cache identity

Include all security-sensitive fields in the migration plan and preflight cache keys, or invalidate those queries whenever any selected domain object changes.
{% endstep %}

{% step %}

## Display exact authority recipients

Show the exact resolver address, manager address, role bitmap, registry, and name resource in the final Manager confirmation.
{% endstep %}

{% step %}

## Build post-state expectations from final reviewed authority

Build post-state expectations from the final live, user-reviewed authority state rather than from an earlier cached plan.
{% endstep %}

{% step %}

## Add regression coverage

Add regression coverage for manager removal, resolver replacement, same-ID query updates, in-flight stale refetch completion, and state changes between separate approval transactions.
{% endstep %}
{% endstepper %}

## References

[V1 query captures mutable owner and resolver fields](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L48)

[V1 names are treated as fresh for five minutes](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29)

[Migration plan identity includes domain IDs but not manager or resolver values](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L110)

[The cached plan is returned as the ready migration plan](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L128-L140)

[Migration preflight is also keyed only by stable domain IDs](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationPreflight.ts#L25-L52)

[Custom V1 resolvers are preserved with the keep-v1 strategy](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L65-L88)

[Classification copies V1 resolver and derives managerAddress from the snapshot](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L125-L185)

[buildMigrationPlan classifies the supplied domain objects without reloading V1 authority](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L380)

[The keep-v1 strategy returns the captured custom resolver](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/encodeMigration.ts#L24-L50)

[Migration data receives the selected resolver](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L479-L491)

[A non-null cached manager adds a manager-role-grant and matching verification expectation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624)

[The manager call grants ROLE\_SET\_RESOLVER to the captured managerAddress](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L24)

[The atomic batch orders migration before the manager grant and creates the outer HCA call](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L742-L816)

[The pre-submit approval check compares approval keys rather than live V1 authority](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L367-L390)

[The name-specific ERC-721 approval targets MigrationHelper](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationApprovals.ts#L83-L108)

[Manager submits required approvals as separate calls before the atomic batch](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L393-L443)

[Submission rebuild refreshes routes and gas but reuses plan.classified](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L886-L964)

[The rebuilt outer HCA call is submitted and verified](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L1044-L1117)

[Execution performs approvals and then submits the remaining stale-plan batches](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L1161-L1204)

[Post-state resolver verification compares against the stale expected resolver](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L263-L275)

[Post-state manager verification checks the stale role recipient](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353)

[Manager displays generic manager-restoration and migration copy](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L108)

[The final calls are wrapped into owner-authorized StandaloneHCA execution](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/smart-account/src/providers/rhinestone/owner-execution.ts#L15-L39)

## Proof of Concept

### Safety and environment

This proof runs entirely against a private disposable Anvil fork using local-only state changes and impersonated fork accounts. It does not send transactions to public Sepolia, Ethereum mainnet, or any other public network. It does not use production funds, wallets, sessions, API credentials, or a public HCA service.

The supplied `STALE_MIGRATION_SEPOLIA_FORK_URL` is used only by Anvil to read the initial Sepolia state at block `11577300`. Every manager change, resolver change, approval, HCA deployment, migration, role grant, and authority assertion executes inside the local fork.

The runner uses a digest-pinned Foundry image and a dedicated Compose project. It binds Anvil to `127.0.0.1:28557`, refuses to reuse an occupied port or existing project resource, uses an atomic run lock, and removes only the resources it starts. It prints `servicesOwnedByRunnerStopped=true` only after the isolated port, containers, network, and volume are gone.

### PoC evidence bundle

Download the complete runnable overlay and captured final evidence here:

[Download ens-manager-stale-v1-authority-poc.zip from Google Drive](https://drive.google.com/file/d/199yJLVPC58nCpFTXi4nLOTu5U8aR9GBO/view?usp=sharing)

Save it with this exact filename:

```
ens-manager-stale-v1-authority-poc.zip
```

ZIP SHA-256:

```
5e88344aaab738ebb03806e87634b628be802f85fa9d354f932fb2ef0c949cc2  ens-manager-stale-v1-authority-poc.zip
```

The archive contains the fail-closed runner, fixed Compose definition, production hook regressions, production payload exporter, private-fork impact harness, captured clean log, structured payload evidence, structured execution evidence, and an internal checksum manifest.

### Prerequisites

* Linux with Bash.
* Node.js v22. The final verification used v22.23.2.
* Corepack and pnpm v10. The final verification used pnpm 10.27.0.
* Docker with Docker Compose v2 and permission to start containers.
* Foundry `cast`.
* Git, curl, jq, `flock`, `grep`, `sed`, `ss`, `sha256sum`, and unzip.
* A researcher-controlled Sepolia RPC URL with historical read and fork support for block `11577300`, reachable from the Docker container.
* Local port `28557` available. A different valid port can be provided through `STALE_MIGRATION_ANVIL_PORT`.
* Network access for the first frozen dependency installation, pinned image download, and read-only fork initialization.

### Reproduction

{% stepper %}
{% step %}

## Obtain the frozen competition repository

Create a clean working directory and clone the in-scope repository:

```bash
mkdir ens-manager-stale-v1-reproduction
cd ens-manager-stale-v1-reproduction

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git \
  audit-comp-ens

cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
```

Verify the frozen competition import:

```bash
test "$(git rev-parse HEAD)" = \
  "cda79acaad59711b943fc68207ebb3f1d0ff8596"

git show -s --format='%H%n%B' HEAD
```

The commit message identifies the imported upstream commit as:

```
63772fd872af472ced58b009499355f3430c2a86
```

{% endstep %}

{% step %}

## Install the frozen dependencies

```bash
node --version
corepack pnpm --version
corepack pnpm install --frozen-lockfile
```

Expected major versions:

```
Node.js: v22.x
pnpm:    10.x
```

{% endstep %}

{% step %}

## Download, verify, and overlay the PoC

Download the Google Drive archive from the link above and place it inside the cloned `audit-comp-ens` directory. Verify the outer checksum before extraction:

```bash
printf '%s  %s\n' \
  '5e88344aaab738ebb03806e87634b628be802f85fa9d354f932fb2ef0c949cc2' \
  'ens-manager-stale-v1-authority-poc.zip' | sha256sum -c -
```

Expected output:

```
ens-manager-stale-v1-authority-poc.zip: OK
```

Extract the overlay into the repository root and verify every internal file:

```bash
unzip ens-manager-stale-v1-authority-poc.zip

chmod +x \
  poc/manager-jawdrop-stale-manager/run-clean.sh \
  poc/manager-jawdrop-stale-manager/fork-role-impact.sh \
  poc/manager-jawdrop-stale-manager/test-runner-preflight.sh

sha256sum -c POC-SHA256SUMS.txt
```

Every path in `POC-SHA256SUMS.txt` must print `OK`.

The relevant layout is:

```
audit-comp-ens/
|-- POC-README.md
|-- POC-SHA256SUMS.txt
|-- poc/manager-jawdrop-stale-manager/
|   |-- docker-compose.yml
|   |-- fork-role-impact.sh
|   |-- payload-export.vitest.config.ts
|   |-- production-payload-export.poc.ts
|   |-- run-clean.sh
|   |-- stale-manager-grant.test.ts
|   |-- stale-plan-cache.test.tsx
|   |-- test-runner-preflight.sh
|   `-- vitest.config.ts
`-- scans/
    |-- manager-stale-migration-clean.log
    |-- manager-stale-migration-production-payload.json
    `-- manager-stale-migration-execution.json
```

{% endstep %}

{% step %}

## Run the fail-closed preflight

```bash
poc/manager-jawdrop-stale-manager/test-runner-preflight.sh
```

Expected output:

```
runnerPreflightTests=PASS
```

The preflight checks shell syntax, executable bits, the pinned image digest, fixed chain and block, loopback binding, missing endpoint rejection, invalid port rejection, non-local RPC rejection, and crafted URL userinfo rejection.
{% endstep %}

{% step %}

## Supply the private-fork endpoint

Supply a researcher-controlled endpoint without putting it in the source or shell command history:

```bash
read -rsp 'Researcher-controlled Sepolia RPC URL: ' \
  STALE_MIGRATION_SEPOLIA_FORK_URL
export STALE_MIGRATION_SEPOLIA_FORK_URL
printf '\n'
```

Running without this variable fails before Docker is started:

```
STALE_MIGRATION_SEPOLIA_FORK_URL must be set to a researcher-controlled Sepolia RPC URL.
```

{% endstep %}

{% step %}

## Execute the joined proof

From the frozen repository root, run:

```bash
poc/manager-jawdrop-stale-manager/run-clean.sh
```

The runner writes the complete terminal record to:

```
scans/manager-stale-migration-clean.log
```

It performs this sequence:

1. Verifies Node v22, pnpm v10, the exact competition commit, required commands, the run lock, free port, and absence of pre-existing project resources.
2. Runs the production hook and builder regressions.
3. Starts digest-pinned Anvil at Sepolia block `11577300` on the isolated loopback port.
4. Verifies chain ID `11155111`, the exact initial block, and an Anvil client identity.
5. Prepares a local V1 fixture using the real `turnkey.eth` registration and deployed contracts, while moving the registrar token to a fresh local victim to eliminate historical operator approvals.
6. Installs the attacker resolver while the former manager is still authorized.
7. Mines the same name-specific BaseRegistrar approval required by Manager, using the local victim account.
8. Has the former manager restore the safe resolver and relinquish V1 Registry ownership to the victim.
9. Proves that the former manager's direct V1 `setResolver` call is now rejected.
10. Generates the migration with Manager's production plan and atomic-batch builders after the live V1 state is safe.
11. Reproduces submission-mode route checks, resolver readiness, and live gas estimation, then proves the rebuilt calldata is byte-identical to the preview calldata.
12. Mines those exact bytes through the real deployed StandaloneHCA implementation.
13. Verifies the attacker resolver, attacker ETH resolution, restored former-manager role, and the former manager's successful later resolver rotation.
14. Removes the isolated Docker resources and verifies cleanup.
    {% endstep %}

{% step %}

## Observe stale-plan retention and payload construction

The first Vitest run must report:

```
Test Files  2 passed (2)
Tests       3 passed (3)
appRegressionTests=PASS
```

The two hook tests establish:

* A same-ID update from attacker manager and resolver to victim manager and safe resolver does not rebuild the cached plan.
* An already-started plan refetch can preserve the older authority values even if fresh same-ID V1 data arrives first.

The builder test decodes the actual outer `executeByOwner` payload and asserts this phase order:

```
migrate
manager-role-grant
```

The fresh-state control produces `migrate` only and has no manager grant.
{% endstep %}

{% step %}

## Observe the attacker losing live V1 authority

The output must include:

```
V1 manager before=0x5f71c6074141d41d0e769689dC9830388a5C7e3E after-attacker-relinquishment=0x1000000000000000000000000000000000000001
former manager V1 resolver call after-revocation rejected=true
pre-migration live V1 manager=0x1000000000000000000000000000000000000001 resolver=0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
```

This is the decisive authority boundary. The former manager cannot alter the live V1 resolver when the later migration payload is generated and mined.

The included run recorded this observable first approval:

```
0x1ebd6a3a729743236ce8ef67b1d7a83bb459159648e1614e7b5b0631b47d7d13
```

It is a local-fork `BaseRegistrar.approve(MigrationHelper, tokenId)` transaction.
{% endstep %}

{% step %}

## Observe the exact production submission payload

The production payload export must finish with:

```
productionPayloadBridge=PASS
```

The SHA-256 of the ASCII `0x`-prefixed calldata text is:

```
c8a68628de44afc4da3e557647e714b7a4c5fe42956567097ec1c19cecbb5a59
```

Inspect the structured evidence:

```bash
jq '{
  name,
  liveV1BeforeSigning,
  cachedPreviewSnapshot,
  freshSnapshotControl,
  outerCall,
  decodedInnerCalls
}' scans/manager-stale-migration-production-payload.json
```

The required comparisons are:

```
live manager                 = victim
cached manager               = former manager
live resolver                = safe resolver
cached resolver              = attacker resolver
submissionRebuildMatchesPreview = true
previewSha256                = outerCall.sha256
decoded phases               = migrate, manager-role-grant
```

The stale `migrate` call contains the attacker resolver. The `grantRoles` call contains role bitmap `16777216`, which is `ROLE_SET_RESOLVER`, and the former manager as recipient.
{% endstep %}

{% step %}

## Observe exact-byte mining and the V2 hijack

The included clean run mined the outer payload as:

```
0x7ada5509922824ad8d7e8a91741d7ba3633f1971f844e3c7841777d38685d032
```

The execution evidence requires:

```
minedTransaction.exactlyMatchesProductionPayload = true
v2AfterMigration.owner                         = victim
v2AfterMigration.resolver                      = attacker resolver
v2AfterMigration.resolvedEthAddress            = attacker
v2AfterMigration.formerManagerRoleBefore       = false
v2AfterMigration.formerManagerRoleAfter        = true
```

Inspect it with:

```bash
jq . scans/manager-stale-migration-execution.json
```

The former manager then uses the restored role to change the V2 resolver to:

```
0x2100000000000000000000000000000000000012
```

The final ETH resolution still returns the attacker:

```
0x5f71c6074141d41d0e769689dC9830388a5C7e3E
```

{% endstep %}

{% step %}

## Observe final result and cleanup

The proof must finish with all of these markers:

```
appRegressionTests=PASS
productionPayloadBridge=PASS
PASS: migration restored the stale resolver and former manager authority
joinedResult=PASS
servicesOwnedByRunnerStopped=true
```

Confirm no PoC listener or project resource remains:

```bash
ss -H -ltn 'sport = :28557'

docker ps -a \
  --filter label=com.docker.compose.project=managerstalemigrationpoc \
  --format '{{.ID}} {{.Names}}'

docker network ls \
  --filter label=com.docker.compose.project=managerstalemigrationpoc \
  --format '{{.ID}} {{.Name}}'

docker volume ls \
  --filter label=com.docker.compose.project=managerstalemigrationpoc \
  --format '{{.Name}}'
```

All four commands must print no matching resource.
{% endstep %}
{% endstepper %}

### Negative controls

* The cached snapshot supplied to the production builders matches the real fork state at the modeled preview point. The proof does not exercise the V1 subgraph, and the attack does not require a forged response.
* The local victim has not approved the former manager as a V1 operator.
* The live V1 manager is the victim before the final migration payload is built.
* The live V1 resolver is the safe resolver before the final migration payload is built.
* The former manager's direct V1 resolver change is rejected after relinquishment.
* The former manager has no V2 `ROLE_SET_RESOLVER` before migration.
* A fresh-state production build uses the safe resolver and omits the manager grant.
* The preview and submission-mode rebuilt outer payloads are byte-identical.
* The mined HCA input is byte-identical to the production-generated payload.
* The final V2 owner remains the victim, isolating the impact to resolver hijack and restored authority.
* The former manager successfully changes the V2 resolver only after the migration grants the new role.
* No public-chain write, stolen key, compromised RPC, malicious subgraph response, or public service is used.

### Proof boundaries

The hook regression, production payload bridge, and private-fork execution are separate evidence layers joined by asserted values and exact calldata. The proof does not claim that an actual React Query object is passed through a real browser and wallet extension in one process.

The attached execution uses local impersonation to authorize deterministic private-fork transactions. It does not claim a genuine victim wallet signature. The security result comes from the exact Manager-generated HCA bytes and their verified state transition.

The attacker-controlled resolver is implemented with local `anvil_setCode` for a minimal deterministic return value. This models a normally deployed attacker resolver without interacting with a public chain.

The direct verified consequence is attacker-controlled V2 resolution plus usable `ROLE_SET_RESOLVER`. No public-network fund transfer, NFT transfer, HCA takeover, or session theft is claimed.
