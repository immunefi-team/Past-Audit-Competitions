# 89314 w a critical manager v1 to v2 migration silently keeps an attacker s resolver and grants the attacker role set resolver on the v2 registry for gifted secondary market names

**Submitted on Aug 18th 2026 at 14:46:07 UTC by @zkbm for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89314
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Transferring an unwrapped V1 `.eth` name moves the BaseRegistrar ERC-721. It does not update the ENS registry `owner` or resolver. Until the recipient calls `reclaim()`, the previous owner remains the name's manager and controls its resolver.

Manager treats that stale state as intentional during V1-to-V2 migration. The `executeByOwner` batch carries the old resolver into V2. It also grants the previous manager `ROLE_SET_RESOLVER` on the V2 ETHRegistry. The new owner signs this batch, but the confirmation UI identifies neither third-party address.

An attacker can register a name, configure an attacker-controlled resolver, and transfer the ERC-721 to a victim. If the victim later migrates the name, the attacker retains control of its resolution and can redirect address records. Funds sent to the name then resolve to an attacker-chosen address.

## Vulnerability Details

{% stepper %}
{% step %}

### A V1 transfer leaves the manager and resolver behind

V1 `BaseRegistrar.transferFrom` moves only the NFT. The ENS registry `owner(node)` stays with the previous holder until the new registrant calls `reclaim()`.

An attacker can create the vulnerable state through ordinary protocol operations. They register `gift.eth`, set an attacker-controlled resolver, and transfer the ERC-721 to the victim. The victim is now the `registrant`. The attacker remains the registry `owner` and resolver operator.
{% endstep %}

{% step %}

### Migration planning trusts the stale subgraph state

Classification reads both third-party addresses from the subgraph rows in `packages/migration/src/service/classifyNames.ts`:

```ts
// classifyNames.ts:137
const v1ResolverAddress = domain.resolver?.address ?? null
// classifyNames.ts:162-166: for unwrapped names whose registry owner differs
// from the registrant, that owner becomes "the manager to restore":
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

The on-chain preflight checks only ERC-721 ownership. `checkOwnership` in `preflightChecks.ts` calls `BaseRegistrar.ownerOf`. A transferred name therefore passes eligibility even when its registry owner is still the previous holder.
{% endstep %}

{% step %}

### The batch carries the attacker's resolver into V2

Because the attacker-controlled resolver is not in `KNOWN_PUBLIC_RESOLVERS`, classification selects `keep-v1`:

```ts
// classifyNames.ts:84-87
if (v1ResolverAddress && !isKnownPublicResolver(v1ResolverAddress)) {
  return 'keep-v1'
}
```

`resolverFor` in `apps/manager/src/features/migration/service/encodeMigration.ts:32-44` writes that address to `MigrationData.resolver`. `buildMigrationHelperCall.ts` then places it in the `MigrationHelper.migrate(...)` calldata. `buildAtomicMigrationBatches.ts:481-491` adds that calldata to the batch.
{% endstep %}

{% step %}

### The same batch grants the attacker a V2 role

The presence of `managerAddress` triggers “manager restoration.” `buildRoleGrantCalls.ts` grants the stale manager `ROLE_SET_RESOLVER` on the V2 registry:

```ts
// apps/manager/src/features/migration/service/buildRoleGrantCalls.ts
const ROLE_SET_RESOLVER = 1n << 24n
export const buildRoleGrantCall = (name: ClassifiedName): Call => {
  ...
  return {
    to: V2_CONTRACTS.ETHRegistry,
    data: encodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      functionName: 'grantRoles',
      args: [resource, ROLE_SET_RESOLVER, name.managerAddress], // attacker
    }),
    ...
```

`buildAtomicMigrationBatches.ts:608-614` adds this call to the same atomic batch.
{% endstep %}

{% step %}

### The UI does not identify the resolver or role recipient

The selection list (`NameRow.tsx` and `SelectNamesStepNameList.tsx`) renders only the name, avatar, and checkbox. The step-descriptor union in `buildStepDescriptors.ts` has no address field. `WalletConfirmationStepsDialog.tsx` says “Restore the existing managers for your names” without naming those managers.

No migration `.tsx` file references `managerAddress`, `v1ResolverAddress`, or `resolverStrategy`. The wallet prompt shows the encoded `executeByOwner` batch. Post-execution verification also accepts the result. `verifyAtomicMigrationBatch` derives its expectations from the same migration plan, including the attacker's resolver and role grant.

The PoC runs the production planning pipeline with a transferred-name fixture. It calls `buildMigrationPlan`, `classifyNames`, and `buildAtomicMigrationBatches`. It then decodes the victim-signed batch and finds:

1. `migrate(label: "gift", owner: VICTIM, resolver: ATTACKER_RESOLVER)`
2. `grantRoles(canonicalId("gift"), 1 << 24, ATTACKER)`

It also confirms that neither attacker address appears in the step descriptors rendered by the UI.
{% endstep %}
{% endstepper %}

### Attack sequence

{% stepper %}
{% step %}
The attacker registers `gift.eth`, sets an attacker-controlled resolver, and transfers the ERC-721 to the victim as a gift, giveaway, or secondary-market sale.
{% endstep %}

{% step %}
The victim selects the name for migration and signs the batch. Manager supports bulk selection in `selectNames.helpers.ts:22-33`.
{% endstep %}

{% step %}
Migration keeps the attacker's resolver and grants the attacker `ROLE_SET_RESOLVER` on the V2 registry entry.
{% endstep %}

{% step %}
The attacker changes the name's ETH address record or other records. If the resolver is replaced, the attacker can use the granted role to set it back.
{% endstep %}
{% endstepper %}

## Impact Details

**Selected impact (Critical):** *Hijacking or permanently bricking an ENS name during a Manager registration, V1-to-V2 migration, or transfer flow.*

The batch also qualifies as a malicious interaction with an already-connected wallet. Manager inserts an undisclosed `grantRoles` call to a third-party address into the transaction the victim signs.

* **Funds sent to the name can be stolen.** Wallets, dapps, and payment flows resolve `gift.eth` to the address chosen by the attacker.
* **Replacing the resolver does not remove the role.** The attacker can use `ROLE_SET_RESOLVER` to restore an attacker-controlled resolver later. Migration verification reports success because it expects the grant.
* **Any transferred V1 name that has not been reclaimed can enter this path.** Bulk selection can include it alongside the victim's other names.
* **Migration makes the stale V1 state harder to recover from.** Before migration, the victim can call `reclaim()` to reset the manager. Migration instead turns the previous manager into a V2 role holder while the UI says it is restoring “your” managers.

## Recommended fix

{% stepper %}
{% step %}
Read V1 `owner(node)` and `resolver(node)` on-chain during migration planning.
{% endstep %}

{% step %}
When the manager differs from the registrant or the resolver is unknown, default the name to an owner-managed configuration with a known public resolver.
{% endstep %}

{% step %}
If Manager preserves either third-party address, display it and require confirmation for that specific name.
{% endstep %}
{% endstepper %}

## Proof of Concept

This unit-level PoC exercises the production migration planning pipeline. It does not need a chain because the unwrapped 2LD path performs no chain reads. The only stub is a `{ chain: { id } }` `PublicClient`. The repository's `buildMigrationPlan.test.ts` uses the same seam.

Save as `apps/manager/src/features/migration/service/poc-gifted-name-attacker-resolver.test.ts` and run:

```bash
cd apps/manager && pnpm exec vitest run --disable-console-intercept \
  src/features/migration/service/poc-gifted-name-attacker-resolver.test.ts
```

Expected result: one passing test. The console output decodes the victim-signed `executeByOwner` batch and shows:

1. `MigrationHelper.migrate` with `resolver: 0x3333…`
2. `ETHRegistry.grantRoles(canonicalId("gift"), 16777216, 0x2222…)`
3. UI step descriptors containing neither attacker address

```ts
// LOCAL POC
//
// Claim: during Manager V1-to-V2 migration planning, a V1 .eth name whose ERC-721
// registrant is the VICTIM but whose V1 registry `owner` (manager) and resolver
// are still the original ATTACKER (the common "gifted/bought without reclaim()"
// state) produces a signed migration batch that:
//   (a) writes the ATTACKER's resolver contract as the migrated V2 name's
//       resolver (resolverStrategy 'keep-v1' because the attacker resolver is
//       not in KNOWN_PUBLIC_RESOLVERS), and
//   (b) grants the ATTACKER ROLE_SET_RESOLVER (1n << 24n) on the V2 ETHRegistry
//       via grantRoles(resource, roleBitmap, managerAddress),
// with no per-name disclosure of either address anywhere in the step
// descriptors the UI renders.
//
// This test drives the REAL production path with no mocks of any migration
// module. It calls makeDomain (subgraph fixture shape), classifyNames,
// buildMigrationPlan, resolveDirectMigrationRoutes, buildAtomicMigrationBatches,
// buildMigrationHelperCall, buildRoleGrantCall, and the executeByOwner outer call.
// The only stub is the PublicClient object (used solely for `chain.id`; the
// unwrapped-2LD path performs no chain reads).

import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import {
  type Address,
  decodeFunctionData,
  getAddress,
  type Hex,
  parseAbi,
  type PublicClient,
} from 'viem'
import { assert, describe, expect, it } from 'vitest'

import { ETH_REGISTRY_V2_ABI, MIGRATION_HELPER_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { makeDomain } from './_fixtures'
import { buildMigrationPlan } from './buildMigrationPlan'
import type { MigrationPreflight } from './computeMigrationPreflight'

// Subgraph returns lowercase addresses; keep the fixture realistic.
const VICTIM: Address = '0x1111111111111111111111111111111111111111'
const ATTACKER: Address = '0x2222222222222222222222222222222222222222'
const ATTACKER_RESOLVER: Address = '0x3333333333333333333333333333333333333333'
const HCA: Address = '0x00000000000000000000000000000000000000ca'

const ROLE_SET_RESOLVER = 1n << 24n

const hcaOwnerExecutionAbi = parseAbi([
  'function executeByOwner((address target, uint256 value, bytes callData)[] executions) payable',
])

type OwnerExecution = {
  readonly target: Address
  readonly value: bigint
  readonly callData: Hex
}

const stringifyWithBigints = (value: unknown): string =>
  JSON.stringify(
    value,
    (_key, v) => (typeof v === 'bigint' ? `${v.toString()}n` : v),
    2,
  )

describe('LOCAL POC: gifted/bought V1 name with stale attacker manager+resolver', () => {
  it('emits the attacker resolver and an attacker ROLE_SET_RESOLVER grant into the signed batch, undisclosed', async () => {
    // The canonical "gifted/bought without reclaim()" V1 state:
    //   - ERC-721 registrant (BaseRegistrar owner)   = VICTIM  (they own the NFT)
    //   - V1 registry owner (a.k.a. manager)         = ATTACKER (never reclaimed)
    //   - V1 resolver                                = ATTACKER-controlled contract
    const domain = makeDomain({
      id: '0xg1f7',
      name: 'gift.eth',
      labelName: 'gift',
      labelhash:
        '0x00000000000000000000000000000000000000000000000000000000000000a1',
      registrantId: VICTIM,
      ownerId: ATTACKER,
      resolverAddress: ATTACKER_RESOLVER,
      registrationExpiry: '99999999999',
    })

    const preflight: MigrationPreflight = {
      preExistingOwnedPermRes: null,
      skipApprovalPhase: true,
      skipFetchProfilesPhase: true,
      baseRegistrarApproved: true,
      nameWrapperApproved: true,
      hcaReadiness: { status: 'deployment-required', hca: HCA },
    }

    const plan = await buildMigrationPlan({
      domains: [domain],
      hcaAddress: HCA,
      migrationOwner: VICTIM, // the victim is the connected wallet
      publicClient: { chain: { id: 11155111 } } as PublicClient,
      preflight,
    })

    // --- Classification: the real classifyNames output for this state -------
    const classified = plan.classified[0]
    assert(classified, 'expected gift.eth to classify as migratable')
    expect(classified.tokenType).toBe('unwrapped')
    expect(classified.resolverStrategy).toBe('keep-v1')
    expect(classified.v1ResolverAddress).toBe(ATTACKER_RESOLVER)
    expect(getAddress(classified.managerAddress as Address)).toBe(
      getAddress(ATTACKER),
    )

    // --- The signed batch ----------------------------------------------------
    const batch = plan.atomicBatches[0]
    assert(batch, 'expected one atomic migration batch')
    expect(batch.outerCall.to).toBe(HCA)

    const decodedOuter = decodeFunctionData({
      abi: hcaOwnerExecutionAbi,
      data: batch.outerCall.data,
    })
    expect(decodedOuter.functionName).toBe('executeByOwner')
    const [executions] = decodedOuter.args as [readonly OwnerExecution[]]

    // (a) The migrate calldata carries the ATTACKER's resolver for gift.eth.
    const migrateExecution = executions.find(
      (execution) =>
        getAddress(execution.target) ===
        getAddress(V2_CONTRACTS.MigrationHelper),
    )
    assert(migrateExecution, 'expected a MigrationHelper.migrate execution')
    const decodedMigrate = decodeFunctionData({
      abi: MIGRATION_HELPER_ABI,
      data: migrateExecution.callData,
    })
    expect(decodedMigrate.functionName).toBe('migrate')
    const [unwrapped] = decodedMigrate.args as [
      readonly {
        label: string
        owner: Address
        subregistry: Address
        resolver: Address
      }[],
      unknown,
      unknown,
      unknown,
    ]
    const giftMigration = unwrapped.find((entry) => entry.label === 'gift')
    assert(giftMigration, 'expected gift.eth in the unwrapped migration group')
    expect(getAddress(giftMigration.resolver)).toBe(
      getAddress(ATTACKER_RESOLVER),
    )
    // Victim's wallet becomes the V2 owner, so the name looks migrated correctly.
    expect(getAddress(giftMigration.owner)).toBe(getAddress(VICTIM))

    // (b) The batch grants the ATTACKER ROLE_SET_RESOLVER on the V2 ETHRegistry.
    const grantExecution = executions.find(
      (execution) =>
        getAddress(execution.target) === getAddress(V2_CONTRACTS.ETHRegistry),
    )
    assert(grantExecution, 'expected an ETHRegistry.grantRoles execution')
    const decodedGrant = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: grantExecution.callData,
    })
    expect(decodedGrant.functionName).toBe('grantRoles')
    const [resource, roleBitmap, grantee] = decodedGrant.args as [
      bigint,
      bigint,
      Address,
    ]
    expect(resource).toBe(labelToCanonicalId('gift'))
    expect(roleBitmap).toBe(ROLE_SET_RESOLVER)
    expect(getAddress(grantee)).toBe(getAddress(ATTACKER))

    // --- No per-name disclosure ----------------------------------------------
    // The step descriptors are all the UI ever renders about the batch content
    // (see WalletConfirmationStepsDialog.tsx). Neither the attacker manager
    // address nor the attacker resolver address appears anywhere in them.
    const descriptorsJson = stringifyWithBigints(plan.stepDescriptors)
    expect(descriptorsJson.toLowerCase()).not.toContain(
      ATTACKER.toLowerCase().slice(2),
    )
    expect(descriptorsJson.toLowerCase()).not.toContain(
      ATTACKER_RESOLVER.toLowerCase().slice(2),
    )

    // --- Evidence dump: what the victim actually signs -----------------------
    console.log('\n===== LOCAL POC EVIDENCE: gift.eth migration batch =====')
    console.log('victim (registrant / connected wallet):', VICTIM)
    console.log('attacker (stale V1 manager):           ', ATTACKER)
    console.log('attacker resolver (stale V1 resolver): ', ATTACKER_RESOLVER)
    console.log('\nclassification:', {
      tokenType: classified.tokenType,
      resolverStrategy: classified.resolverStrategy,
      v1ResolverAddress: classified.v1ResolverAddress,
      managerAddress: classified.managerAddress,
    })
    console.log(
      '\nouter call (signed by victim): executeByOwner @',
      batch.outerCall.to,
    )
    for (const [index, execution] of executions.entries()) {
      const label =
        getAddress(execution.target) ===
        getAddress(V2_CONTRACTS.MigrationHelper)
          ? 'MigrationHelper.migrate'
          : getAddress(execution.target) ===
              getAddress(V2_CONTRACTS.ETHRegistry)
            ? 'ETHRegistry.grantRoles'
            : 'other'
      console.log(`\n  execution[${index}] at ${execution.target} (${label})`)
      if (label === 'MigrationHelper.migrate') {
        console.log(
          '    decoded:',
          stringifyWithBigints(
            decodeFunctionData({
              abi: MIGRATION_HELPER_ABI,
              data: execution.callData,
            }),
          ),
        )
      }
      if (label === 'ETHRegistry.grantRoles') {
        console.log(
          '    decoded:',
          stringifyWithBigints(
            decodeFunctionData({
              abi: ETH_REGISTRY_V2_ABI,
              data: execution.callData,
            }),
          ),
        )
      }
    }
    console.log('\nstep descriptors shown to the user:', descriptorsJson)
    console.log('===== END LOCAL POC EVIDENCE =====\n')
  })
})
```
