# 93042 w a medium migration resurrects a revoked v1 manager from stale indexed state

**Submitted on Sep 14th 2026 at 03:27:54 UTC by @kl4r10n for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93042
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

Manager intentionally preserves the controller of an unwrapped V1 `.eth` registration during migration. It obtains that controller from `Domain.owner` returned by the V1 subgraph and stores it as `managerAddress` in the migration plan.

This value is security-sensitive: after migrating the name, the HCA executes:

```solidity
ETHRegistryV2.grantRoles(
    canonicalNameResource,
    ROLE_SET_RESOLVER,
    managerAddress
);
```

The V1 names query is cached as fresh for five minutes, and the subgraph can also lag behind the chain. Manager performs live preflight checks, but those checks validate only the current BaseRegistrar NFT owner or NameWrapper owner. They never read the current V1 ENS Registry owner/controller.

Therefore, the following state transition is possible:

1. An attacker was previously the V1 manager/controller of `victim.eth`.
2. The victim, who owns the BaseRegistrar NFT, revokes that authority by reclaiming the V1 ENS Registry node to the victim.
3. The Manager app still has a cached/indexed `Domain.owner = attacker` value.
4. The victim migrates `victim.eth` to V2.
5. Migration succeeds and grants the attacker V2 `ROLE_SET_RESOLVER`.
6. The attacker replaces the V2 resolver and redirects resolution.

This is a stale-state race, not a durable backdoor in every migration. It is reachable when the victim migrates after reclaiming or rotating the V1 controller while the client cache and/or V1 subgraph still reports the old manager. A first-time migration performed after the subgraph has caught up does not hit this issue.

The migration confirmation says only “Restore the existing managers for your names.” It does not display the address that will be restored. The post-migration verifier then treats the stale attacker address possessing the role as a successful migration invariant.

## Root cause

### 1. Manager obtains the security-sensitive V1 controller from the subgraph

`apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-47` requests both the registry `owner` and registrar `registrant` from the V1 subgraph:

```ts
const GET_NAMES_QUERY = `
query getNamesForAddress(...) {
  domains(...) {
    id
    labelName
    labelhash
    name
    resolver { address }
    owner { id }
    registrant { id }
    wrappedOwner { id }
    // ...
  }
}
`
```

For an unwrapped `.eth` 2LD, `registrant` is the BaseRegistrar NFT owner and `owner` is the V1 ENS Registry owner/controller. These can legitimately differ.

### 2. The off-chain value is cached as fresh for five minutes

`apps/manager/src/features/migration/hooks/useV1Names.ts:12-23` gives the indexed response a five-minute `staleTime`:

```ts
const v1NamesQueryOptions = (...) =>
  resultQueryOptions({
    queryKey: qk('migration', 'v1_names', {
      address: address?.toLowerCase(),
    }),
    queryFn:
      enabled && address ? () => getV1NamesForAddress(address) : skipToken,
    staleTime: 5 * 60 * 1000,
  })
```

Accordingly, the race does not require a malformed subgraph response. A previously correct response can remain trusted after the victim changes the controller on-chain. Normal subgraph indexing delay creates the same condition even without the client cache.

### 3. Classification converts the indexed owner directly into `managerAddress`

`packages/migration/src/service/classifyNames.ts:148-185` verifies only that the indexed registrant equals the connected owner, then copies a differing indexed registry owner into `managerAddress`:

```ts
const registrant = domain.registrant
if (registrant?.id.toLowerCase() !== addr) return null

const tokenHolder = toAddress(registrant.id)
if (!tokenHolder) return null

const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null

return {
  type: 'classified',
  name: {
    // ...
    tokenHolder,
    managerAddress,
  },
}
```

There is no provenance or block number attached to `managerAddress`, so later stages cannot determine whether it still reflects live authorization.

### 4. Live eligibility checks validate token ownership but not controller authority

`packages/migration/src/service/preflightChecks.ts:50-101` performs an on-chain ownership multicall. For an unwrapped name, it reads only `BaseRegistrar.ownerOf(labelhash)`:

```ts
const contracts = names.map((name) =>
  name.tokenType === 'unwrapped'
    ? {
        address: BASE_REGISTRAR,
        abi: BASE_REGISTRAR_ABI,
        functionName: 'ownerOf' as const,
        args: [BigInt(name.domain.labelhash)] as const,
      }
    : {
        address: NAME_WRAPPER,
        abi: NAME_WRAPPER_ABI,
        functionName: 'getData' as const,
        args: [BigInt(name.domain.id)] as const,
      },
)
```

This correctly proves that the victim can migrate the registrar NFT. It does not prove that the indexed `managerAddress` is still the V1 ENS Registry owner.

`apps/manager/src/features/migration/service/computeMigrationPreflight.ts:201-249` performs additional live checks for approvals, resolver readiness, migration contracts and routes, but none reads the current V1 ENS Registry owner.

### 5. The gas-estimate plan carries the original indexed domain objects forward

`apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts:62-109` selects domain objects from the cached `v1Names` array and passes them unchanged into both preflight and `buildMigrationPlan()`:

```ts
const domains = selectDomainsFromNames(v1Names, selectedNames)

const preflight = await ensurePreflight(domains, { staleTime: 0 })
const plan = await buildMigrationPlan({
  domains,
  hcaAddress,
  migrationOwner: ownerAddress,
  publicClient,
  preflight,
})
```

Setting the gas-estimate query's own `staleTime` to zero does not refresh the underlying subgraph domains. It merely rebuilds a plan from the same cached objects.

### 6. The atomic migration grants the stale address resolver authority that survives cleanup

`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624` adds a manager-role grant whenever the classified snapshot contains a manager:

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })
  verificationExpectations.push({
    type: 'manager-role',
    account: classified.managerAddress,
    roleBitmap: ROLE_SET_RESOLVER,
    // ...
  })
}
```

`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-26` encodes the actual grant:

```ts
const ROLE_SET_RESOLVER = 1n << 24n

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

The migration and role grant are placed in the same outer HCA transaction. `buildAtomicMigrationInnerExecutions()` orders the role grant immediately after migration:

```ts
return [
  ...executionsForPhase('resolver-deployment'),
  ...executionsForPhase('wallet-co-admin-grant'),
  ...helperExecutions,
  ...executionsForPhase('manager-role-grant'),
  ...executionsForPhase('profile-replay'),
]
```

The V2 registry checks whether the HCA is currently authorized to make the grant. It has no reason to know or validate the previous V1 controller. Therefore, a live gas estimate or actual execution does not reject the stale manager.

### 7. Execution-time checks still do not refresh the manager

Before the first wallet prompt, `executeMigration()` calls `assertMigrationApprovalPlanCurrent()`. Despite the comment that permission state is mutable, `apps/manager/src/features/migration/service/migrationService.ts:338-390` only refreshes the approvals required by the HCA.

When each live batch is rebuilt, `migrationService.ts:886-927` refreshes resolver readiness and migration routes but passes the original `remaining: plan.classified` values—including stale `managerAddress`—back into `buildAtomicMigrationBatches()`:

```ts
const atomicPlan = await buildAtomicMigrationBatches({
  // ...
  classified: remaining,
  directRoutes,
  profiles: plan.profiles,
  // ...
})
```

Thus neither plan creation nor final transaction construction performs the missing authority read.

### 8. The UI hides which address receives authority

`apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:81-89` displays only generic copy:

```tsx
<Trans>Approve manager restoration</Trans>
<Trans>Restore the existing managers for your names.</Trans>
```

It does not show the manager address or identify that the value came from an indexed snapshot. A user who just revoked a compromised manager is specifically likely to understand “existing manager” as the current on-chain manager, not the revoked address.

### 9. Verification treats the attacker's role as successful completion

`apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-353` verifies the stale address rather than recomputing the expected manager from V1 chain state:

```ts
case 'manager-role': {
  const registry = await resolveRegistryForName(...)
  return context.publicClient.readContract({
    address: registry,
    abi: permissionedRegistryReadAbi,
    functionName: 'hasRoles',
    args: [
      expectation.resource,
      expectation.roleBitmap,
      expectation.account,
    ],
  })
}
```

Because `expectation.account` is the stale manager, successfully granting the attacker is interpreted as proof that migration completed correctly.

## Attack scenario

1. The victim owns the BaseRegistrar NFT for `victim.eth` but has delegated its V1 ENS Registry manager/controller to the attacker. This can arise from a previously legitimate delegate, a compromised service, or an address the victim now distrusts.
2. Manager fetches the victim's V1 names. The response correctly contains:
   * `registrant = victim`; and
   * `owner = attacker`.
3. The victim removes the attacker by reclaiming the V1 registry node to the victim.
4. The old response remains cached as fresh for up to five minutes, or the V1 subgraph has not indexed the reclaim yet.
5. The victim selects `victim.eth` for migration. The live ownership check reads `BaseRegistrar.ownerOf()` and correctly sees the victim, so the name remains eligible.
6. No check reads the live V1 ENS Registry owner. The migration plan retains `managerAddress = attacker`.
7. The confirmation UI says only that existing managers will be restored. It does not disclose the attacker address.
8. The victim approves and submits the atomic migration.
9. The HCA migrates the registration to the victim, then calls V2 `grantRoles(resource, ROLE_SET_RESOLVER, attacker)`.
10. Cleanup revokes the HCA's temporary operator approval, but it does not revoke the role just granted to the attacker.
11. The attacker observes migration on-chain and calls the V2 registry's resolver-setting operation for the granted resource.
12. The attacker points `victim.eth` at an attacker-controlled resolver. The resolver can return the attacker's address for ETH and other coin types, redirecting payments and identity-based interactions.

The attack does not require control of the victim's wallet, HCA, session key, or browser. The attacker needs only to have been the previously authorized manager and for the victim to migrate while stale indexed state is trusted. A closely related variant occurs when the victim rotates management to a new trusted address: stale state restores the old attacker instead of the new current manager.

If the victim waits until both the client query and V1 subgraph reflect the reclaim or rotation before opening the migration, the stale address is not restored. Exploitation is specifically the interval in which Manager's authorization source disagrees with the chain.

## Impact

* A manager explicitly revoked by the registrar owner is resurrected during the V1-to-V2 security migration.
* The attacker receives V2 authority that survives the migration flow's temporary-approval cleanup and remains usable until the victim notices and revokes it.
* `ROLE_SET_RESOLVER` lets the attacker replace the resolver for the migrated name.
* An attacker-controlled resolver can resolve the victim's ENS name to attacker-controlled addresses.
* The UI does not disclose the restored address, and the verification layer declares the malicious authorization a success.

The registration/NFT stays owned by the victim, no migration payment is diverted, and the role is revocable by the V2 owner. The Critical impact is the resolver-authority assignment performed during the Manager migration and the resulting hijack window—not theft of the 2LD token or an unrevokable loss of ownership.

This is not merely stale display data. Off-chain stale data is converted into a new on-chain authorization that did not exist immediately before migration.

## Why this is not a known issue

No disclosed known issue describes stale V1 manager/controller data being converted into persistent V2 authorization.

| Known issue                                                           | Difference                                                                                                                                                 |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `WEB-806` — migrated subname records not visible                      | This report concerns an unwrapped `.eth` 2LD and a persistent attacker authorization, not record visibility.                                               |
| `WEB-1249` — creating a subname can detach an existing child registry | This report uses the V1-to-V2 migration manager-restoration path and grants `ROLE_SET_RESOLVER`; it does not create a subname or replace a child registry. |
| `EXP-INPUT-005` — ENS input validation/homoglyphs                     | No unusual name input is required. The root cause is failure to revalidate an authorization source.                                                        |
| `R2-03` / session-key findings                                        | No session key is involved. The victim signs the migration with the legitimate wallet/HCA flow.                                                            |
| QA migration/display defects                                          | Those concern presentation or functionality. Here the application creates a new persistent on-chain permission for an address already revoked on V1.       |

The root cause is wholly within the in-scope application: Manager treats an off-chain indexer field as authorization truth when constructing a state-modifying migration transaction. It is not a defect in the subgraph or an imported contract. Indexer lag is an expected property that authorization-sensitive clients must handle.

## Why “manager restoration is intended” does not make this safe

Preserving a currently authorized V1 manager may be intended product behavior. Restoring an address that is no longer authorized is not.

The correct security invariant is:

```
V2 manager restored by migration == live V1 manager at final confirmation
```

The implemented invariant is:

```
V2 manager restored by migration == manager in an earlier subgraph/cache snapshot
```

The PoC demonstrates the second invariant. The migration verifier reinforces it by checking that the stale address received the role.

## Recommended remediation

1. **Read the live V1 controller before constructing the final batch.** For every unwrapped `.eth` 2LD, read the current V1 ENS Registry owner for `namehash(name)` at a recent block.
2. **Fail closed on disagreement.** If the live value differs from the indexed `managerAddress`, invalidate the plan and return the user to confirmation.
3. **Rebuild from live values.** Do not pass the old `plan.classified` manager fields into `buildNextAtomicBatch()`.
4. **Use one consistent block snapshot.** Read source-token ownership and registry controller at the same block where practical, and bind that snapshot to the final plan.
5. **Show restored addresses.** Display each manager address and corresponding name in the confirmation UI.
6. **Fail closed on RPC errors.** An unavailable live manager read must never fall back to indexed authorization data.
7. **Invalidate V1-name queries after manager/reclaim operations.** This reduces exposure but does not replace the final on-chain check.
8. **Add a regression test** in which the indexed manager is attacker A, the live controller is owner B, and transaction construction is rejected or rebuilt for B.

Example precondition before adding any manager grant:

```ts
const liveManager = await publicClient.readContract({
  address: V1_CONTRACTS.Registry,
  abi: ENS_REGISTRY_ABI,
  functionName: 'owner',
  args: [namehash(classified.domain.name)],
  blockNumber: sourceStateBlock,
})

if (!isAddressEqual(liveManager, classified.managerAddress ?? migrationOwner)) {
  throw new MigrationPlanChangedError({
    message: `The manager for ${classified.domain.name} changed. Review the migration again.`,
  })
}
```

The live value—not the indexed value—should then be used to decide whether a V2 manager grant is necessary and which address receives it.

## Proof of Concept

## Proof of concept

### What the PoC proves

It drives the production:

1. `classifyName()` implementation that turns indexed `Domain.owner` into `managerAddress`;
2. complete `buildAtomicMigrationBatches()` implementation;
3. HCA `executeByOwner` wrapping; and
4. V2 `grantRoles` ABI.

It models the unavoidable input to any indexer-race test: the live V1 controller is the victim, while the cached/indexed domain object still contains the revoked attacker. It then decodes the actual outer batch and proves that:

* the phases are `migrate` followed by `manager-role-grant`;
* the outer call is sent to the HCA;
* the inner grant targets the deployed V2 ETH Registry;
* the account is the revoked manager; and
* the granted bitmap is exactly `ROLE_SET_RESOLVER` (`1 << 24`).

The only supplied callback is the gas-partitioning estimate. It does not participate in classification, authorization, target selection, argument construction, or calldata encoding.

### PoC code

Create:

`apps/manager/src/features/migration/service/F07StaleManagerRestoration.poc.test.ts`

```ts
import {
  type Address,
  decodeFunctionData,
  type Hex,
  parseAbi,
} from 'viem'
import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'

import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { buildAtomicMigrationBatches } from './buildAtomicMigrationBatches'
import { classifyName } from './classifyNames'
import type { DirectMigrationRoute } from './directMigrationRoutes'
import type { V1Domain } from './v1SubgraphClient'

const VICTIM: Address = '0x00000000000000000000000000000000000000a1'
const REVOKED_MANAGER: Address =
  '0x00000000000000000000000000000000000000b2'
const HCA: Address = '0x00000000000000000000000000000000000000c3'
const ROLE_SET_RESOLVER = 1n << 24n
const hcaOwnerExecutionAbi = parseAbi([
  'function executeByOwner((address target, uint256 value, bytes callData)[] executions) payable',
])

/**
 * Regression/PoC for a real indexer race:
 *
 * 1. REVOKED_MANAGER was the V1 ENS Registry owner/controller.
 * 2. VICTIM, who owns the BaseRegistrar NFT, reclaims the registry node to
 *    VICTIM before starting migration.
 * 3. The V1 subgraph (and the five-minute app query cache) still returns the
 *    old Domain.owner value below.
 *
 * Migration checks BaseRegistrar.ownerOf on-chain, but never reads the live
 * ENS Registry owner for manager restoration. The stale address therefore
 * reaches the persistent V2 grantRoles call.
 */
const staleIndexerDomain: V1Domain = {
  id: `0x${'11'.repeat(32)}`,
  labelName: 'revoked-manager',
  labelhash: `0x${'22'.repeat(32)}`,
  name: 'revoked-manager.eth',
  resolver: null,
  owner: { id: REVOKED_MANAGER },
  registrant: { id: VICTIM },
  wrappedOwner: null,
  parent: { name: 'eth', wrappedDomain: null },
  registration: { expiryDate: '99999999999' },
  wrappedDomain: null,
}

describe('F-07: migration restores a revoked manager from stale indexer data', () => {
  it('puts persistent V2 set-resolver authority for the revoked address in the signed atomic batch', async () => {
    const result = classifyName(staleIndexerDomain, VICTIM)
    if (result?.type !== 'classified') {
      throw new Error('Expected the victim-owned registration to be migratable')
    }

    expect(result.name.managerAddress?.toLowerCase()).toBe(
      REVOKED_MANAGER.toLowerCase(),
    )

    const directRoute: DirectMigrationRoute = {
      name: staleIndexerDomain.name,
      receiver: V2_CONTRACTS.UnlockedMigrationController,
      parentDependency: null,
      expectedWrapperRegistry: null,
      receiverReadiness: 'migration-controller',
    }
    const plan = await buildAtomicMigrationBatches({
      chainId: sepolia.id,
      hca: HCA,
      wallet: VICTIM,
      classified: [result.name],
      directRoutes: new Map([[staleIndexerDomain.name, directRoute]]),
      profiles: new Map(),
      resolverDeployed: true,
      walletCoAdminGranted: true,
      maxOuterGas: 1_000_000n,
      // Gas partitioning is orthogonal to calldata construction.
      estimateOuterGas: () => 100_000n,
    })
    const batch = plan.batches[0]
    if (!batch) throw new Error('Expected one atomic migration batch')
    expect(batch.innerExecutions.map(({ phase }) => phase)).toEqual([
      'migrate',
      'manager-role-grant',
    ])

    const outer = decodeFunctionData({
      abi: hcaOwnerExecutionAbi,
      data: batch.outerCall.data,
    })
    if (outer.functionName !== 'executeByOwner') {
      throw new Error('Expected executeByOwner calldata')
    }
    const [executions] = outer.args as [
      readonly { target: Address; value: bigint; callData: Hex }[],
    ]
    const managerGrant = executions.find(
      ({ target }) =>
        target.toLowerCase() === V2_CONTRACTS.ETHRegistry.toLowerCase(),
    )
    if (!managerGrant) throw new Error('Expected V2 manager restoration call')

    const decoded = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: managerGrant.callData,
    })
    if (decoded.functionName !== 'grantRoles') {
      throw new Error('Expected grantRoles calldata')
    }
    const [resource, roles, account] = decoded.args as [
      bigint,
      bigint,
      Address,
    ]

    expect(batch.outerCall.to.toLowerCase()).toBe(HCA.toLowerCase())
    expect(managerGrant.target.toLowerCase()).toBe(
      V2_CONTRACTS.ETHRegistry.toLowerCase(),
    )
    expect(managerGrant.value).toBe(0n)
    expect(roles).toBe(ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(REVOKED_MANAGER.toLowerCase())
    expect(account.toLowerCase()).not.toBe(VICTIM.toLowerCase())

    console.log('live V1 manager after victim reclaim:', VICTIM)
    console.log('stale subgraph manager restored in V2:', account)
    console.log('outer signer target (HCA):', batch.outerCall.to)
    console.log('atomic phases:', batch.innerExecutions.map(({ phase }) => phase))
    console.log('V2 registry target:', managerGrant.target)
    console.log('canonical name resource:', resource.toString())
    console.log('persistent role bitmap:', roles.toString())
    console.log('ROLE_SET_RESOLVER:', ROLE_SET_RESOLVER.toString())
  })
})
```

### Reproduction command

From the repository root:

```bash
pnpm --filter manager exec vitest run \
  src/features/migration/service/F07StaleManagerRestoration.poc.test.ts \
  --reporter=verbose
```

### Execution result

```
stdout | src/features/migration/service/F07StaleManagerRestoration.poc.test.ts >
F-07: migration restores a revoked manager from stale indexer data >
puts persistent V2 set-resolver authority for the revoked address in the signed atomic batch

live V1 manager after victim reclaim: 0x00000000000000000000000000000000000000a1
stale subgraph manager restored in V2: 0x00000000000000000000000000000000000000b2
outer signer target (HCA): 0x00000000000000000000000000000000000000c3
atomic phases: [ 'migrate', 'manager-role-grant' ]
V2 registry target: 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
canonical name resource: 75983775976435430337691443377804941843144178885074054523214498846978043019264
persistent role bitmap: 16777216
ROLE_SET_RESOLVER: 16777216

✓ F07StaleManagerRestoration.poc.test.ts (1 test)
Test Files  1 passed (1)
Tests       1 passed (1)
```
