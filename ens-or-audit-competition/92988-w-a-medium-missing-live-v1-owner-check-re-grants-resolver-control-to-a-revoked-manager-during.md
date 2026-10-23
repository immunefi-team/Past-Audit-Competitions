# 92988 w a medium missing live v1 owner check re grants resolver control to a revoked manager during migration

**Submitted on Sep 13th 2026 at 17:33:30 UTC by @dxhod for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92988
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Manager derives the V1 manager of an unwrapped `.eth` name from subgraph data that remains fresh in the UI cache for five minutes. If the registrant reclaims the V1 Registry name after that snapshot was loaded, Manager does not re-read the live V1 Registry owner before migration. It therefore builds a V2 migration batch that grants `ROLE_SET_RESOLVER` to the former manager even though that account has already been revoked. The former manager can then replace the migrated name's resolver with an attacker-controlled resolver.

## Vulnerability Details

For an unwrapped `.eth` 2LD there are two relevant ownership values:

* the BaseRegistrar registrant, who owns the ERC-721 registration; and
* the ENS Registry owner, who acts as the current manager.

Manager loads both values through `useV1Names`. The query is deliberately considered fresh for five minutes:

```typescript
resultQueryOptions({
  queryKey: qk('migration', 'v1_names', {
    address: address?.toLowerCase(),
  }),
  queryFn:
    enabled && address ? () => getV1NamesForAddress(address) : skipToken,
  staleTime: 5 * 60 * 1000,
})
```

`classifyName` then treats any Registry owner different from the registrant as the manager that should be restored in V2:

```typescript
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

That is correct only while the snapshot still represents the live V1 Registry state. A registrant can revoke a manager at any time by reclaiming the Registry name. After that transaction, the cached `domain.owner.id` still contains the former manager.

Manager does perform a fresh preflight before migration, but the check is against the wrong ownership layer for this decision. For unwrapped names, `checkOwnership` calls only `BaseRegistrar.ownerOf(labelhash)`. The remaining reservation check calls `ETHRegistryV2.getStatus(labelhash)`. Neither check calls the V1 ENS Registry `owner(node)` function, so both pass after the victim has reclaimed the Registry name: the victim still owns the BaseRegistrar NFT and the V2 slot is still reserved.

The stale `managerAddress` is later used directly when constructing the migration calls:

```typescript
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

The resulting call is included in the atomic HCA migration batch. The confirmation screen only says **Approve manager restoration** and **Restore the existing managers for your names**. It does not show the account that will receive the role, so it gives the victim no indication that a manager they just removed is being restored.

A concrete sequence is:

1. The victim owns the BaseRegistrar NFT for `alice.eth`, while the attacker is its legitimate V1 Registry manager.
2. The victim opens Manager. The app loads a snapshot with the victim as registrant and the attacker as Registry owner/manager.
3. The victim uses the normal reclaim operation in another tab or tool, making the victim the live V1 Registry owner and revoking the attacker.
4. Before the five-minute cache window expires, the victim migrates `alice.eth` through Manager.
5. The live preflight sees that the victim still owns the registration and that the V2 slot is reserved, but never checks the live V1 Registry owner.
6. Manager submits `grantRoles(..., ROLE_SET_RESOLVER, attacker)` as part of the migration.
7. The attacker can now change the V2 resolver for `alice.eth` despite having been revoked before the migration.

This sequence does not require a malicious project role, a compromised browser, a special fuse combination, or a custom resolver. Ordinary subgraph indexing delay can also produce a stale snapshot, but the deterministic five-minute client cache is sufficient to reproduce the issue.

This is separate from my earlier report #91982. That report concerns a locked name with `CANNOT_SET_RESOLVER`, where Manager preserves the current custom resolver but gives the owner no resolver-management role. This report does not depend on that fuse or resolver strategy. Its root cause is the missing live V1 Registry-owner check, and its result is the creation of new V2 authority for an address that is no longer the manager.

## Impact Details

The selected impact is **Hijacking or bricking an ENS name during a Manager flow**. The migration assigns resolver control to an attacker/former manager even though the victim revoked that authority before signing the migration. `ROLE_SET_RESOLVER` lets the attacker replace the name's resolver and point resolution at attacker-controlled records.

The direct demonstrated impact is unauthorized resolver control and an unintended role recipient inside the signed migration batch. I am not claiming that the V2 token itself is stolen, or that a fixed amount of funds is immediately lost. The victim remains the V2 owner and can recover by detecting the issue, revoking the restored role, and repairing the resolver. Until that happens, however, the former manager controls where the migrated ENS name resolves. This matches the program's Critical example of a Manager migration assigning a name's resolver to an attacker.

The victim interaction is the normal migration approval. The important detail is that the UI describes the recipient as an "existing manager" even though the live V1 Registry no longer authorizes that account, and the actual address is not displayed.

## References

* `useV1Names` five-minute cache: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/hooks/useV1Names.ts>
* Manager classification logic: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/migration/src/service/classifyNames.ts>
* Live preflight checks: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/migration/src/service/preflightChecks.ts>
* V2 role-grant builder: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts>
* Wallet confirmation copy: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx>
* ENS V2 permissioned registry roles: <https://docs.ens.domains/ensv2/permissioned-registry/>

## Proof of Concept

## Proof of Concept

The PoC is local only and does not send any public-network transaction. It has two parts: the first proves that the production preflight accepts a stale former-manager snapshot without reading the V1 Registry owner; the second proves that the same snapshot is converted into a V2 `grantRoles` call for the former manager.

### 1. Prepare the repository

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
corepack pnpm install --frozen-lockfile
```

### 2. Add the preflight test

Create `packages/migration/src/service/stale-v1-manager-preflight.poc.test.ts`:

```typescript
import { multicall } from 'viem/actions'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { makeClassified, ok, OWNER, publicClient } from './_fixtures'
import { runEligibilityChecks } from './preflightChecks'

vi.mock('viem/actions', () => ({
  multicall: vi.fn(),
}))

const multicallMock = vi.mocked(multicall)
const FORMER_MANAGER = '0x0000000000000000000000000000000000000099'

beforeEach(() => {
  multicallMock.mockReset()
})

describe('PoC: eligibility preflight does not refresh the V1 manager', () => {
  it('keeps a stale former-manager snapshot eligible after live ownership checks pass', async () => {
    const staleName = makeClassified({
      managerAddress: FORMER_MANAGER,
    })

    multicallMock
      .mockResolvedValueOnce([ok(OWNER)]) // BaseRegistrar.ownerOf(labelhash)
      .mockResolvedValueOnce([ok(1)]) // ETHRegistryV2.getStatus(labelhash)

    const result = await runEligibilityChecks(
      publicClient,
      [staleName],
      OWNER,
    )

    expect(result.eligible).toEqual([staleName])
    expect(result.failed).toEqual([])

    const requestedFunctions = multicallMock.mock.calls.flatMap(
      ([, parameters]) =>
        parameters.contracts.map((contract) => contract.functionName),
    )
    expect(requestedFunctions).toEqual(['ownerOf', 'getStatus'])
    expect(requestedFunctions).not.toContain('owner')
  })
})
```

Run it from `packages/migration`:

```bash
cd packages/migration
corepack pnpm exec vitest run src/service/stale-v1-manager-preflight.poc.test.ts
```

Expected result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

The important assertion is the function list: the stale name remains eligible, while the only live calls are `ownerOf` and `getStatus`. There is no V1 Registry `owner(node)` read.

### 3. Add the calldata test

Create `apps/manager/src/features/migration/service/stale-v1-manager-restoration.poc.test.ts`:

```typescript
import { decodeFunctionData, type Address } from 'viem'
import { describe, expect, it } from 'vitest'
import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { makeDomain, OWNER } from './_fixtures'
import { buildRoleGrantCall } from './buildRoleGrantCalls'
import { classifyName } from './classifyNames'

const FORMER_MANAGER: Address =
  '0x0000000000000000000000000000000000000099'
const ROLE_SET_RESOLVER = 1n << 24n

describe('PoC: stale V1 manager snapshot restores revoked resolver authority', () => {
  it('encodes a V2 role grant to the former manager from stale indexer data', () => {
    const staleIndexerDomain = makeDomain({
      registrantId: OWNER,
      ownerId: FORMER_MANAGER,
      name: 'alice.eth',
      labelName: 'alice',
    })

    const result = classifyName(staleIndexerDomain, OWNER)
    if (result?.type !== 'classified') throw new Error('not classified')

    expect(result.name.tokenType).toBe('unwrapped')
    expect(result.name.tokenHolder.toLowerCase()).toBe(OWNER.toLowerCase())
    expect(result.name.managerAddress?.toLowerCase()).toBe(
      FORMER_MANAGER.toLowerCase(),
    )

    const call = buildRoleGrantCall(result.name)
    const decoded = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: call.data,
    })
    expect(decoded.functionName).toBe('grantRoles')

    const [, roles, account] = decoded.args as [bigint, bigint, Address]
    expect(roles).toBe(ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(FORMER_MANAGER.toLowerCase())
  })

  it('does not restore a manager when the indexer snapshot is current', () => {
    const currentIndexerDomain = makeDomain({
      registrantId: OWNER,
      ownerId: OWNER,
      name: 'alice.eth',
      labelName: 'alice',
    })

    const result = classifyName(currentIndexerDomain, OWNER)
    if (result?.type !== 'classified') throw new Error('not classified')
    expect(result.name.managerAddress).toBeNull()
  })
})
```

From the repository root, run:

```bash
cd apps/manager
corepack pnpm exec vitest run src/features/migration/service/stale-v1-manager-restoration.poc.test.ts
```

Expected result:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

The first test decodes the real calldata and proves that the stale former-manager address receives exactly `ROLE_SET_RESOLVER`. The second test is the negative control: when the snapshot matches the live post-reclaim state, no manager role is created.

On my clean local run both files passed. Together they reproduce the full application-side cause: stale V1 authority survives the live preflight and is encoded as fresh V2 resolver authority.
