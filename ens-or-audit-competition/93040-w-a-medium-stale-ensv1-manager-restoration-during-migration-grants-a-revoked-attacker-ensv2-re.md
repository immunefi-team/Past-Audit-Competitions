# 93040 w a medium stale ensv1 manager restoration during migration grants a revoked attacker ensv2 resolver control

**Submitted on Sep 14th 2026 at 03:18:21 UTC by @andro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93040
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The ENS Manager derives the manager it will restore during an unwrapped `.eth` v1-to-v2 migration from the previously fetched subgraph field `domain.owner.id`. If the registrant subsequently reclaims the ENSv1 Registry node and thereby revokes that manager, the migration's live preflight checks only the BaseRegistrar NFT owner. It never rereads the authoritative current ENSv1 Registry owner. The final migration transaction therefore grants the revoked account ENSv2 `ROLE_SET_RESOLVER`, allowing it to replace the migrated name's resolver with an attacker-controlled resolver.

## Vulnerability Details

For an unwrapped `.eth` name, there are two distinct authorities:

* `BaseRegistrar.ownerOf(labelhash)` identifies the registration NFT owner/registrant.
* `ENSRegistry.owner(node)` identifies the current ENSv1 Registry node owner, commonly called the manager.

The classifier intentionally restores a separate manager when the indexed Registry owner differs from the registrant:

```typescript
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

This value comes from the subgraph domain object and is retained in the classified migration plan. It is not tied to an authoritative block and is not refreshed when the transaction is built.

The live ownership preflight checks the BaseRegistrar token:

```typescript
functionName: 'ownerOf',
args: [BigInt(name.domain.labelhash)],
```

That check proves the victim still owns the `.eth` registration NFT. It cannot detect a change to `ENSRegistry.owner(node)`. In particular, it continues to pass after the registrant calls `BaseRegistrar.reclaim(labelhash, registrant)` and removes the old manager.

The execution builder treats the stale `managerAddress` as authoritative:

```typescript
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })
}
```

`buildRoleGrantCall` encodes:

```typescript
functionName: 'grantRoles',
args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
```

Immediately before submission, `buildNextAtomicBatch` refreshes deterministic resolver readiness and direct migration routes, but it rebuilds from the old `remaining` classified objects. It does not reread or validate the manager. The stale grant is placed inside the final HCA `executeByOwner` transaction and is executed atomically after the name migration.

The violated invariant is:

```
manager authority restored in ENSv2
    ==
authoritative ENSRegistry.owner(node) immediately before migration
```

The implementation instead enforces:

```
manager authority restored in ENSv2
    ==
domain.owner.id from an earlier subgraph/preview snapshot
```

## Preconditions

1. The victim owns an unwrapped `.eth` BaseRegistrar token.
2. The ENSv1 Registry node was previously delegated to another account controlled by the attacker.
3. The Manager fetches or previews the name while the attacker is still represented by `domain.owner.id`.
4. Before migration is submitted, the victim reclaims the ENSv1 Registry node and revokes the attacker.
5. The victim continues the ordinary migration flow in the still-open Manager page.

The attacker does not need the victim's key, control of the subgraph, physical or local-network access, a browser defect, or a privileged protocol address. After the reclaim, the attacker has no current authority; the vulnerable migration grants it back.

## Detailed execution path

1. `victim` owns the BaseRegistrar token for `victim.eth`.
2. `attacker` is the current ENSv1 Registry owner/manager for `victim.eth`.
3. The Manager fetches a domain snapshot containing `registrant = victim` and `owner = attacker`.
4. `classifyName()` stores `attacker` as `classified.managerAddress`.
5. The victim revokes the attacker by reclaiming the ENSv1 Registry node to the victim address in another tab or ENS client.
6. The authoritative state is now `BaseRegistrar.ownerOf(labelhash) = victim` and `ENSRegistry.owner(node) = victim`.
7. The victim continues the normal Manager migration.
8. The preflight checks only `BaseRegistrar.ownerOf(labelhash)`, which still equals the victim, so it passes.
9. The live batch rebuild reuses `classified.managerAddress = attacker`.
10. The final HCA owner transaction migrates the name and calls `ETHRegistry.grantRoles(resource, ROLE_SET_RESOLVER, attacker)`.
11. Migration verification expects this stale grant and therefore reports successful completion.
12. The revoked attacker calls `setResolver(resource, attackerResolver)` on ENSv2.
13. `victim.eth` now resolves through the attacker-controlled resolver. The attacker can return an attacker-controlled ETH address and hijack the migrated name's resolution.

## Impact Details

`ROLE_SET_RESOLVER` is effective authority, not a display error. Once the migration succeeds, the revoked manager can replace the resolver for the migrated name. A malicious resolver can return an attacker-controlled address for `victim.eth`, causing wallets and applications that resolve the name to use the attacker's address. The authority persists until the victim discovers and revokes it on ENSv2.

The vulnerable transaction is part of the normal Manager workflow and is approved by the victim, but the UI and wallet flow do not disclose that a previously revoked address is being reauthorized.

This matches the competition's listed Critical impact for hijacking an ENS name during a v1-to-v2 Manager migration by assigning its resolver to an attacker.

**Attacker capability after exploitation:** The attacker receives `ROLE_SET_RESOLVER` on the migrated ENSv2 name and can set an arbitrary resolver without further victim interaction.

**Maximum impact:** Persistent resolver hijacking of the migrated ENS name, including attacker-controlled address resolution and the risk of funds being sent to the attacker through ENS resolution.

## References

* ENS competition information: <https://immunefi.com/audit-competition/audit-competition-ens/information/>
* ENS competition scope and impacts: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

## Proof of Concept

## Proof of Concept

The PoC exercises the production classifyName() and buildRoleGrantCall() implementations.

```typescript
import assert from 'node:assert/strict'
import { it } from 'vitest'
import {
  createPublicClient,
  custom,
  decodeFunctionData,
  encodeAbiParameters,
  encodeFunctionResult,
  parseAbi,
  type Address,
} from '../../../../apps/manager/node_modules/viem/_esm/index.js'
import { sepolia } from '../../../../apps/manager/node_modules/viem/_esm/chains/index.js'
import { classifyName } from '../../../../packages/migration/src/service/classifyNames.ts'
import { checkOwnership } from '../../../../packages/migration/src/service/preflightChecks.ts'
import type { V1Domain } from '../../../../packages/migration/src/service/v1SubgraphClient.ts'
import { ETH_REGISTRY_V2_ABI } from '../../../../apps/manager/src/features/migration/contracts/abis.ts'
import { V2_CONTRACTS } from '../../../../apps/manager/src/features/migration/contracts/addresses.ts'
import { buildAtomicMigrationBatches } from '../../../../apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts'

const victim: Address = '0x1111111111111111111111111111111111111111'
const revokedManager: Address = '0x2222222222222222222222222222222222222222'
const hca: Address = '0x3333333333333333333333333333333333333333'

// This is the cached/subgraph snapshot consumed by the migration UI.  The
// victim owns the BaseRegistrar token, while the registry's cached owner still
// identifies an old manager.
const cachedDomain: V1Domain = {
  id: `0x${'a'.repeat(64)}`,
  labelName: 'victim',
  labelhash: `0x${'b'.repeat(64)}`,
  name: 'victim.eth',
  resolver: null,
  owner: { id: revokedManager },
  registrant: { id: victim },
  wrappedOwner: null,
  parent: { name: 'eth', wrappedDomain: null },
  registration: { expiryDate: '4102444800' },
  wrappedDomain: null,
}

// Immediately before migration, the victim reclaimed the ENSv1 registry node, so the authoritative current manager is the victim.
//   The application never reads this value while constructing or refreshing the execution batch.

const liveV1RegistryOwner = victim
assert.equal(liveV1RegistryOwner, victim)

const classified = classifyName(cachedDomain, victim)
assert(classified && classified.type === 'classified')
assert.equal(
  classified.name.managerAddress?.toLowerCase(),
  revokedManager.toLowerCase(),
  'the stale subgraph owner is treated as the manager to restore',
)

// Exercise the production live-ownership preflight.  Its only read for an  unwrapped name is BaseRegistrar.ownerOf(labelhash)
// Returning the victim models the authoritative state after reclaim: the token check passes eventhough ENSRegistry.owner(node) has changed, because no Registry owner read is present in the request.

const multicall3Abi = parseAbi([
  'function aggregate3((address target,bool allowFailure,bytes callData)[] calls) payable returns ((bool success,bytes returnData)[] returnData)',
])
const rpcRequests: Array<{ method: string; params?: readonly unknown[] }> = []
const preflightClient = createPublicClient({
  chain: sepolia,
  transport: custom({
    request: async (request) => {
      rpcRequests.push(request as { method: string; params?: readonly unknown[] })
      assert.equal(request.method, 'eth_call')
      return encodeFunctionResult({
        abi: multicall3Abi,
        functionName: 'aggregate3',
        result: [
          {
            success: true,
            returnData: encodeAbiParameters(
              [{ type: 'address' }],
              [victim],
            ),
          },
        ],
      })
    },
  }),
})
const rejectedByOwnershipPreflight = await checkOwnership(
  preflightClient,
  [classified.name],
  victim,
)
assert.equal(rejectedByOwnershipPreflight.size, 0)
assert.equal(rpcRequests.length, 1)
const rpcCall = rpcRequests[0]?.params?.[0] as
  | { data?: `0x${string}` }
  | undefined
assert(rpcCall?.data)
const decodedMulticall = decodeFunctionData({
  abi: multicall3Abi,
  data: rpcCall.data,
})
const [ownershipCalls] = decodedMulticall.args as readonly [
  readonly {
    target: Address
    allowFailure: boolean
    callData: `0x${string}`
  }[],
]
assert.equal(ownershipCalls.length, 1)
const ownerOfAbi = parseAbi([
  'function ownerOf(uint256 tokenId) view returns (address)',
])
const decodedOwnershipRead = decodeFunctionData({
  abi: ownerOfAbi,
  data: ownershipCalls[0]!.callData,
})
assert.equal(decodedOwnershipRead.functionName, 'ownerOf')
assert.deepEqual(decodedOwnershipRead.args, [BigInt(cachedDomain.labelhash)])

// Build the same final atomic owner transaction that migrationService submits.
// Its live rebuild refreshes routes and resolver readiness, but keeps this already-classified stale manager address.
const plan = await buildAtomicMigrationBatches({
  chainId: sepolia.id,
  hca,
  wallet: victim,
  classified: [classified.name],
  directRoutes: new Map([
    [
      cachedDomain.name,
      {
        name: cachedDomain.name,
        receiver: V2_CONTRACTS.UnlockedMigrationController,
        parentDependency: null,
        expectedWrapperRegistry: null,
        receiverReadiness: 'migration-controller' as const,
      },
    ],
  ]),
  profiles: new Map(),
  defaultResolver: V2_CONTRACTS.DefaultResolver,
  resolverDeployed: true,
  walletCoAdminGranted: true,
  maxOuterGas: 1_000_000n,
  estimateOuterGas: async () => 100_000n,
})

const batch = plan.batches[0]
assert(batch)
const managerExecution = batch.innerExecutions.find(
  (execution) => execution.phase === 'manager-role-grant',
)
assert(managerExecution)

// Prove the manager grant is included in the final HCA executeByOwner call, rather than existing only as an unused helper result.

const executeByOwnerAbi = parseAbi([
  'function executeByOwner((address target,uint256 value,bytes callData)[] executions) payable',
])
const decodedOuter = decodeFunctionData({
  abi: executeByOwnerAbi,
  data: batch.outerCall.data,
})
assert.equal(decodedOuter.functionName, 'executeByOwner')
const [outerExecutions] = decodedOuter.args as readonly [
  readonly { target: Address; value: bigint; callData: `0x${string}` }[],
]
assert(
  outerExecutions.some(
    (execution) =>
      execution.target.toLowerCase() ===
        managerExecution.call.to.toLowerCase() &&
      execution.callData === managerExecution.call.data,
  ),
  'the stale manager grant must be present in the submitted outer transaction',
)

const decoded = decodeFunctionData({
  abi: ETH_REGISTRY_V2_ABI,
  data: managerExecution.call.data,
})
assert.equal(decoded.functionName, 'grantRoles')

const [resource, roles, account] = decoded.args as readonly [
  bigint,
  bigint,
  `0x${string}`,
]
assert.equal(account.toLowerCase(), revokedManager.toLowerCase())
assert.equal(roles, 1n << 24n, 'ROLE_SET_RESOLVER is granted')
assert.notEqual(account.toLowerCase(), liveV1RegistryOwner.toLowerCase())

console.log(
  JSON.stringify(
    {
      name: cachedDomain.name,
      authoritativeV1ManagerAtSubmission: liveV1RegistryOwner,
      staleManagerEncodedByMigration: account,
      v2Function: decoded.functionName,
      outerFunction: decodedOuter.functionName,
      outerTarget: batch.outerCall.to,
      resource: resource.toString(),
      role: 'ROLE_SET_RESOLVER',
      consequence:
        'the revoked manager can replace the migrated name resolver and redirect all records',
    },
    null,
    2,
  ),
)
console.log(
  'PASS: migration restores a revoked manager from stale subgraph state without an on-chain manager check',
)

it('puts the revoked cached manager grant in the final atomic owner transaction', () => {
  assert.equal(account.toLowerCase(), revokedManager.toLowerCase())
  assert.equal(batch.outerCall.to.toLowerCase(), hca.toLowerCase())
})
```

### Run instructions

Use the competition repository at commit:

```
63772fd872af472ced58b009499355f3430c2a86
```

Install the repository's frozen dependencies, then run from `apps/manager` on Windows:

```
.\node_modules\.bin\vitest.CMD --root ..\.. --config apps\manager\vitest.config.ts run audits\Immunefi\ENS-Sep-2026\poc\migration-stale-manager-restoration.poc.test.ts
```

Equivalent POSIX command:

````
./node_modules/.bin/vitest --root ../.. --config apps/manager/vitest.config.ts run audits/Immunefi/ENS-Sep-2026/poc/migration-stale-manager-restoration.poc.test.ts

### Observed output

```text
Test Files  2 passed (2)
Tests       2 passed (2)
````

The repository's local pnpm project mirror causes this standalone file to be discovered twice; both executions pass the same assertions.

The decisive decoded values are:

```
authoritativeV1ManagerAtSubmission = 0x1111111111111111111111111111111111111111
staleManagerEncodedByMigration     = 0x2222222222222222222222222222222222222222
outerFunction                      = executeByOwner
v2Function                         = grantRoles
role                               = ROLE_SET_RESOLVER
```

### Expected result

After the victim reclaims the ENSv1 Registry node, the migration must either omit the old manager grant or invalidate the preview and require a new review. The final transaction must not grant any ENSv2 authority to the revoked address.

### Actual result

The final HCA owner transaction contains `grantRoles(resource, ROLE_SET_RESOLVER, revokedManager)`. The migration succeeds and restores effective resolver authority to the attacker.

## Recommendation

1. Immediately before constructing the manager-restoration call, read `ENSRegistry.owner(namehash(name))` from the authoritative ENSv1 Registry for every unwrapped name.
2. Derive `managerAddress` from that live value, not from the subgraph domain object.
3. If the live Registry owner equals the BaseRegistrar registrant, omit the manager role grant.
4. If the live manager differs from the previewed manager, invalidate the plan and require the user to review a new preview.
5. Bind the source block number and manager address into the migration plan's freshness checks.
6. Display the exact address and exact ENSv2 role being restored in the confirmation UI.

Add regression tests for both transitions:

```
manager A -> registrant: no role may be granted to A
manager A -> manager B: the old plan must abort and require regeneration
```

## Known-issue and scope analysis

The affected query, classifier, preflight, atomic transaction builder, and execution flow are in the in-scope Manager asset. The ENS contracts and subgraph are not alleged to be vulnerable; they are only the authoritative systems the Manager reads and writes.

No published known issue describes stale ENSv1 manager restoration. QA-01 covers follow-up errors after certain role combinations and explicitly says that a role combination granting unintended authority is a new security consequence. This report demonstrates a successful migration that grants a revoked address effective resolver authority, rather than an error or unavailable UI action.

The issue is also distinct from the listed transaction-manager chain/signer defects and cached smart-account address issue. The transaction can use the correct chain, signer, HCA, target contracts, and calldata encoding while still granting authority to the wrong account because the wrong account was selected from stale application state.
