# 90056 w a medium toctou in v1 to v2 migration restores revoked manager resolver authority

**Submitted on Aug 22nd 2026 at 19:13:04 UTC by @crypt0n1337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90056
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief / Introduction

The ENS Manager treats the V1 subgraph's cached `domain.owner.id` as proof that an account is still authorized to manage an unwrapped `.eth` name. It stores that address as `managerAddress` and never compares it with the canonical live ENSv1 Registry owner before constructing or submitting the atomic migration batch.

If the BaseRegistrar registrant revokes a delegated V1 manager with `reclaim(tokenId, registrant)` after the subgraph/UI snapshot was taken, the migration still grants the former manager ENSv2 `ROLE_SET_RESOLVER`. The grant is executed after the name has been migrated to the victim, passes the built-in post-migration verification, and remains in place after the temporary HCA approval is removed.

The revoked account can consequently replace the migrated name's resolver with an attacker-controlled resolver and redirect ENS address resolution after the registrant explicitly revoked that account.

This maps to the competition's Critical impact 6286: hijacking an ENS name during a V1-to-V2 Manager migration, including a migration that assigns the resolver to an attacker.

## Affected snapshot and asset

* Audited upstream revision: `63772fd872af472ced58b009499355f3430c2a86`
* Immunefi frozen snapshot: `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* In-scope asset: `apps/manager`
* Primary affected code: `apps/manager/src/features/migration/service`

All testing was performed locally. No public-chain transaction or live-user interaction was used.

## Vulnerability details

For an unwrapped `.eth` name, `classifyName()` copies the V1 subgraph owner into `managerAddress` whenever it differs from the BaseRegistrar registrant:

```ts
const managerAddress =
  registrantAddress &&
  ownerAddress &&
  !isAddressEqual(registrantAddress, ownerAddress)
    ? ownerAddress
    : null
```

`buildRoleGrantCalls()` then unconditionally creates a grant of bit 24 (`ROLE_SET_RESOLVER`) to that retained address:

```ts
buildGrantRoleCall({
  resource: rolesResource,
  roles: 1n << 24n,
  account: managerAddress,
})
```

`buildAtomicMigrationBatches()` puts the migration helper execution first and the manager role grant later in the same HCA `executeByOwner` batch. It also creates a verification expectation for the same stale account. Therefore, verification confirms the vulnerability instead of detecting it.

The preflight and execution-time refresh logic rechecks routes, resolver readiness, approvals and balances, but does not read `ENSv1 Registry.owner(node)` and does not compare the live owner with the cached `managerAddress`. Rebuilding the batch from `plan.classified` therefore preserves the obsolete address.

The role grant succeeds because the migration registers the ENSv2 name to the connected wallet with resolver-admin authority and Manager temporarily calls `ETHRegistry.setApprovalForAll(hca, true)`. ENSv2 combines an approved operator's effective roles with the token owner's roles, allowing the HCA to grant `ROLE_SET_RESOLVER`. Cleanup later removes only the HCA operator approval; it does not remove the direct role granted to the former manager.

This is an authorization TOCTOU: indexer/UI state captured at preview time is later used as an execution-time authorization fact.

## Attack scenario

1. Alice is the BaseRegistrar registrant of `alice.eth`; Mallory is its delegated ENSv1 Registry owner/manager.
2. The V1 subgraph or an already-open migration preview records `domain.owner.id = Mallory` and `registrant.id = Alice`.
3. Alice revokes Mallory by calling `BaseRegistrar.reclaim(tokenId, Alice)`. The canonical V1 Registry owner is now Alice.
4. Before the subgraph/client snapshot refreshes, Alice continues the Manager migration.
5. Manager migrates the name to Alice, then grants Mallory ENSv2 `ROLE_SET_RESOLVER` using the stale address.
6. The post-migration verifier expects Mallory to have the role and reports success.
7. Manager removes only the HCA's temporary operator approval. Mallory's direct resolver role persists.
8. Mallory sets an attacker-controlled resolver for `alice.eth`. Applications resolving the name can then receive attacker-controlled cryptocurrency addresses or profile data.

The confirmation UI only says “Approve manager restoration” / “Restore the existing managers for your names”; it does not display the address that will receive authority.

## Preconditions and feasibility

* The attacker was a legitimate delegated V1 manager at the stale snapshot.
* The registrant revoked that manager before migration.
* The victim migrates while the earlier subgraph/client snapshot is still used.

This can arise during ordinary subgraph indexing delay or from an already-open migration plan. The client also keeps the V1 name query fresh for five minutes. No attacker privilege, victim signature, or manager cooperation is required after revocation; the victim follows the normal migration UI.

## Impact

`ROLE_SET_RESOLVER` is security-sensitive. It permits the former manager to replace the resolver pointer after migration. A malicious resolver can return attacker-owned payment addresses and arbitrary identity data for the victim's ENS name. The result is a successfully verified Manager migration that silently restores a revoked third party's ability to hijack resolution.

## Known-issue differentiation

* `QA-01` concerns role combinations that cause later profile or primary-name operations to fail. This finding creates unintended authority by restoring a specifically revoked account from stale source state; it is not a functional role-combination failure.
* `WEB-1249` concerns child-registry replacement and orphaned subnames. No subregistry replacement occurs here.
* `WEB-806` / `WEB-390` concern migrated record visibility or missing subnames, not authorization freshness.
* Report `#89465` concerns normalized display versus raw renewal labels. This finding does not involve renewal, labels, or normalization.

## Recommended remediation

1. Immediately before the first migration wallet prompt, read the canonical ENSv1 Registry `owner(node)` for every selected unwrapped name.
2. Revalidate or bind that read when `buildNextAtomicBatch()` constructs the executable batch. Do not reuse `managerAddress` from a subgraph DTO as an authorization fact.
3. Restore a manager only if the live V1 owner is nonzero and differs from the current registrant/migration owner.
4. If the live value differs from the preview, abort with `MigrationPlanChangedError` and return the user to review.
5. Display the exact account and role being restored in the final confirmation.
6. Add a regression test that performs `reclaim` between plan creation and batch construction and verifies that the obsolete grant is rejected.

## References

* `packages/migration/src/service/classifyNames.ts:148-185`
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-24`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624, 742-780`
* `apps/manager/src/features/migration/service/computeMigrationPreflight.ts:192-259`
* `apps/manager/src/features/migration/service/migrationService.ts:886-963, 1169-1211`
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:84-87`

## Proof of Concept

## Setup

Use the frozen Immunefi repository snapshot and install dependencies as documented by the competition.

Save the following test as:

`apps/manager/src/features/migration/service/migrationStaleManagerAuthority.poc.test.ts`

```ts
import { type Address, decodeFunctionData, parseAbi } from 'viem'
import { sepolia } from 'viem/chains'
import { assert, describe, expect, it } from 'vitest'

import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { OWNER, makeDomain } from './_fixtures'
import { buildAtomicMigrationBatches } from './buildAtomicMigrationBatches'
import { classifyName } from './classifyNames'
import type { DirectMigrationRoute } from './directMigrationRoutes'

const HCA: Address = '0x00000000000000000000000000000000000000a1'
const REVOKED_MANAGER: Address =
  '0x00000000000000000000000000000000000000c1'
const DEFAULT_RESOLVER: Address =
  '0x00000000000000000000000000000000000000d1'
const ROLE_SET_RESOLVER = 1n << 24n

const hcaOwnerExecutionAbi = parseAbi([
  'function executeByOwner((address target, uint256 value, bytes callData)[] executions) payable',
])

type OwnerExecution = {
  readonly target: Address
  readonly value: bigint
  readonly callData: `0x${string}`
}

describe('PoC: stale V1 manager authority is restored during migration', () => {
  it('encodes grantRoles for a revoked manager without a live V1 owner check', async () => {
    const staleDomain = makeDomain({
      name: 'alice.eth',
      labelName: 'alice',
      registrantId: OWNER,
      ownerId: REVOKED_MANAGER,
      resolverAddress: DEFAULT_RESOLVER,
    })

    const classifiedResult = classifyName(staleDomain, OWNER)
    assert(classifiedResult?.type === 'classified')
    expect(classifiedResult.name.managerAddress).toBe(REVOKED_MANAGER)

    // This models BaseRegistrar.reclaim(tokenId, OWNER) occurring after the
    // subgraph/preview snapshot and before migration execution.
    const liveV1RegistryManagerAfterReclaim = OWNER
    expect(liveV1RegistryManagerAfterReclaim).not.toBe(REVOKED_MANAGER)

    const route: DirectMigrationRoute = {
      name: 'alice.eth',
      receiver: V2_CONTRACTS.UnlockedMigrationController,
      parentDependency: null,
      expectedWrapperRegistry: null,
      receiverReadiness: 'migration-controller',
    }

    const plan = await buildAtomicMigrationBatches({
      chainId: sepolia.id,
      hca: HCA,
      wallet: OWNER,
      classified: [classifiedResult.name],
      directRoutes: new Map([['alice.eth', route]]),
      profiles: new Map(),
      defaultResolver: DEFAULT_RESOLVER,
      resolverDeployed: true,
      walletCoAdminGranted: true,
      maxOuterGas: 1_000_000n,
      estimateOuterGas: () => 100_000n,
    })

    const batch = plan.batches[0]
    assert(batch)

    const decodedOuter = decodeFunctionData({
      abi: hcaOwnerExecutionAbi,
      data: batch.outerCall.data,
    })
    assert(decodedOuter.functionName === 'executeByOwner')

    const [executions] = decodedOuter.args as [readonly OwnerExecution[]]
    const migrationIndex = executions.findIndex(
      ({ target }) =>
        target.toLowerCase() === V2_CONTRACTS.MigrationHelper.toLowerCase(),
    )
    const grantIndex = executions.findIndex(
      ({ target }) =>
        target.toLowerCase() === V2_CONTRACTS.ETHRegistry.toLowerCase(),
    )

    expect(migrationIndex).toBeGreaterThanOrEqual(0)
    expect(grantIndex).toBeGreaterThan(migrationIndex)

    const grant = executions[grantIndex]
    assert(grant)
    const decodedGrant = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: grant.callData,
    })
    assert(decodedGrant.functionName === 'grantRoles')

    const [, roles, account] = decodedGrant.args as [bigint, bigint, Address]
    expect(roles).toBe(ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(REVOKED_MANAGER.toLowerCase())
  })
})
```

## Run

From `apps/manager`:

```bash
./node_modules/.bin/vitest run src/features/migration/service/migrationStaleManagerAuthority.poc.test.ts
```

On Windows:

```powershell
.\node_modules\.bin\vitest.CMD run src/features/migration/service/migrationStaleManagerAuthority.poc.test.ts
```

## Observed result

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

The test uses the production `classifyName()` and `buildAtomicMigrationBatches()` implementations. It proves that:

1. the stale subgraph owner becomes `managerAddress`;
2. the real batch builder places migration before the role grant; and
3. after a modeled live V1 reclaim, the encoded `grantRoles` call still assigns `ROLE_SET_RESOLVER` to the revoked manager because no live V1 owner check is performed.
