# 90250 w a medium missing v1 registry owner revalidation re authorizes a revoked manager during v2 migration enabling resolver hijacking

**Submitted on Aug 24th 2026 at 01:44:08 UTC by @adshao for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90250
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

During V1 to V2 migration, ENS Manager treats the V1 subgraph's `owner.id` as the manager to restore. If a registrant has reclaimed the V1 Registry node from a former manager but the subgraph response or the Manager's five-minute query cache is still stale, the migration grants `ROLE_SET_RESOLVER` in V2 to that revoked former manager. The former manager can then change the migrated name's resolver and redirect its records. The victim signs the normal Manager-generated migration batch; the attacker does not need the victim's key or a malicious subgraph.

## Vulnerability Details

### Root cause

For unwrapped .eth names, the V1 subgraph response is used to classify a distinct Registry owner as `managerAddress`:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

This value is not bound to a fresh authoritative read of `V1 Registry.owner(node)`.

The vulnerable data flow is:

1. `v1SubgraphClient.ts` queries `owner { id }` and `registrant { id }`.
2. `packages/migration/src/service/classifyNames.ts` copies a distinct subgraph owner into `ClassifiedName.managerAddress`.
3. `computeMigrationPreflight.ts` reduces the security decision to `classified.some(name => name.managerAddress !== null)`.
4. The execution-time approval recheck in `migrationService.ts` repeats the same predicate over the existing plan; it does not re-read the V1 Registry owner.
5. `buildRoleGrantCalls.ts` directly encodes:

   ```ts
   grantRoles(resource, 1n << 24n, name.managerAddress)
   ```
6. `buildAtomicMigrationBatches.ts` adds both the grant and a verification expectation for that same address.
7. `verifyAtomicMigrationBatch.ts` calls `hasRoles(..., expectation.account)`, so it verifies that the stale, plan-selected address received the role rather than verifying it against the current V1 owner.

The browser-side query also uses:

```ts
staleTime: 5 * 60 * 1000
```

Therefore a response obtained before the registrant calls `reclaim` can remain reusable by the Manager for up to five minutes even without assuming any particular production subgraph indexing delay.

### Exploit sequence

Let:

* A be the legitimate .eth registrant.
* B be a former V1 manager whose authority A has revoked.
* HCA be the honest migration executor controlled through the normal Manager flow.

The sequence is:

1. Initially, A is the registrant and B is the V1 Registry owner/manager.
2. A calls `reclaim`, so the authoritative V1 Registry owner becomes A.
3. A stale cached/indexed response still says `owner.id == B`.
4. A starts the V1 to V2 migration while that response is still used.
5. Manager classifies B as `managerAddress`, requests the ETHRegistry HCA approval, and builds the migration batch.
6. The batch grants `ROLE_SET_RESOLVER` to B in V2.
7. Post-migration verification succeeds because it checks B, the same address selected by the stale plan.
8. B can set an attacker-controlled resolver for the migrated name.

This does not allow an arbitrary unrelated address to become manager. The attacker must control a former manager address that the registrant already revoked. The vulnerability resurrects that revoked authority.

## Impact Details

The selected in-scope impact is:

> Hijacking or bricking an ENS name during a Manager flow (v1 to v2 migration), including a migration that assigns the resolver to an attacker.

`ROLE_SET_RESOLVER` is the exact V2 permission needed to change the resolver. Once the revoked former manager receives that role, it can redirect the migrated ENS name's resolution and records to attacker-controlled destinations. The legitimate registrant may see the migration as successful because the verifier checks the stale grantee.

The preconditions are limited but realistic: the legitimate registrant must migrate after reclaiming the V1 node and before the stale cached/indexed owner is refreshed, and the migration feature must be enabled for that user. The attacker does not need to compromise the HCA, modify the subgraph, or obtain the victim's signing key.

## Recommended Fix

Use a fresh `V1 Registry.owner(node)` multicall as the only authority for manager restoration. Bind the returned manager value into the immutable migration plan, re-read and compare it before approval and immediately before every batch submission or retry, and abort if it changes. Pass the freshly proven value into the role encoder instead of accepting an unproven `ClassifiedName.managerAddress`. Verification should also compare the granted account with the authoritative value, not only the plan-selected account.

## References

* V1 subgraph owner query: `apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-47`
* Five-minute Manager cache: `apps/manager/src/features/migration/hooks/useV1Names.ts:12-23`
* Classification of subgraph owner as manager: `packages/migration/src/service/classifyNames.ts:148-186`
* Preflight predicate: `apps/manager/src/features/migration/service/computeMigrationPreflight.ts:192-207`
* Role grant sink: `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-26`
* Batch construction and self-referential expectation: `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624`
* Verification of the plan-selected account: `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-353`

## Proof of Concept

## Proof of Concept

This PoC is local-only. It does not broadcast a transaction to Sepolia or any other public network.

### Preconditions modeled

* `CURRENT_V1_OWNER` is the legitimate registrant after reclaiming the V1 Registry node.
* `STALE_FORMER_MANAGER` is the revoked former manager still present in the cached/subgraph-derived migration input.
* The HCA and all target contracts are honest.

### Steps to reproduce

1. Check out the competition repository and install dependencies.
2. Create `apps/manager/src/features/migration/service/staleManagerMigration.poc.test.ts` with the test below.
3. Run:

```bash
pnpm --filter manager test -- staleManagerMigration.poc.test.ts
```

4. Observe that the real approval planner requests `eth-registry:hca`, and the real role encoder creates `grantRoles(resource, ROLE_SET_RESOLVER, STALE_FORMER_MANAGER)` instead of rejecting the stale address or requiring the current V1 Registry owner.

```ts
import { type Address, decodeFunctionData } from 'viem'
import { describe, expect, it } from 'vitest'
import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { makeClassified } from './_fixtures'
import { buildRoleGrantCall } from './buildRoleGrantCalls'
import { planMigrationApprovals } from './migrationApprovals'

const CURRENT_V1_OWNER: Address =
  '0x1111111111111111111111111111111111111111'
const STALE_FORMER_MANAGER: Address =
  '0x2222222222222222222222222222222222222222'
const HCA: Address =
  '0x3333333333333333333333333333333333333333'
const ROLE_SET_RESOLVER = 1n << 24n

describe('stale former-manager restoration during migration', () => {
  it('re-authorizes the revoked former manager', () => {
    const approvals = planMigrationApprovals({
      hcaAddress: HCA,
      needs: {
        hasUnwrapped: true,
        unwrappedTokenIds: [123n],
        hasWrapped: false,
        requiresManagerRestoration: true,
      },
      status: {
        baseRegistrarHcaApproved: false,
        unwrappedTokenApprovals: [{ tokenId: 123n, approved: false }],
        nameWrapperHcaApproved: false,
        ethRegistryHcaApproved: false,
      },
    })

    expect(approvals.map(({ id }) => id)).toContain('eth-registry:hca')

    // This is the stale ClassifiedName retained after CURRENT_V1_OWNER
    // reclaimed the authoritative V1 Registry node from STALE_FORMER_MANAGER.
    const call = buildRoleGrantCall(
      makeClassified({
        label: 'victim',
        name: 'victim.eth',
        tokenHolder: CURRENT_V1_OWNER,
        managerAddress: STALE_FORMER_MANAGER,
      }),
    )

    const decoded = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: call.data,
    })

    expect(call.to).toBe(V2_CONTRACTS.ETHRegistry)
    expect(decoded.functionName).toBe('grantRoles')

    const [, roles, account] = decoded.args as [bigint, bigint, Address]
    expect(roles).toBe(ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(STALE_FORMER_MANAGER.toLowerCase())
    expect(account.toLowerCase()).not.toBe(CURRENT_V1_OWNER.toLowerCase())

    console.log({
      target: call.to,
      functionName: decoded.functionName,
      role: roles.toString(),
      grantedAccount: account,
      currentV1Owner: CURRENT_V1_OWNER,
    })
  })
})
```

### Expected result

The test passes and prints a grant to the stale former manager. On the pinned competition source, the locally confirmed call decoded as:

```
target: 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
functionName: grantRoles
role: 16777216
grantedAccount: 0x2222222222222222222222222222222222222222
currentV1Owner: 0x1111111111111111111111111111111111111111
```

The emitted calldata was:

```
0x7c30058678bc914bc3d5656a140207b49573f1c2a4d8c6de385818d435bec10e0000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000002222222222222222222222222222222222222222
```

A secure implementation should abort because the stale manager was not proven against the current V1 Registry owner. After the proposed fix, this same test should fail closed before a role-grant call is produced.
