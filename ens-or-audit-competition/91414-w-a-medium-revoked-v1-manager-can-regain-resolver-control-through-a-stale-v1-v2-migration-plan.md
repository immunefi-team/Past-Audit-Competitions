# 91414 w a medium revoked v1 manager can regain resolver control through a stale v1 v2 migration plan

**Submitted on Sep 1st 2026 at 09:09:12 UTC by @AGalx for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91414
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The ENS Manager application captures a V1 manager address into a migration plan at classification time but never rebinds that address to current canonical V1 authority before constructing the irreversible V2 role grant. If Alice revokes Charlie as her V1 manager after the plan is built but before migration executes, the Manager can still construct and submit `grantRoles(resource, ROLE_SET_RESOLVER, Charlie)` during Alice's legitimate V1→V2 migration. Charlie, whose V1 authority was canonically removed and whose protected V1 actions revert, receives fresh V2 resolver-control authority and can independently redirect Alice's ENS name to a Charlie-controlled target without any fresh, address-specific authorization from Alice.

## Vulnerability Details

The finding affects the competition's audited revision `63772fd872af472ced58b009499355f3430c2a86`. The PoC was executed against the scope-locked local checkout verified as source-equivalent to that audited revision.

The production source-to-sink path is:

```
V1 Domain.owner.id
  → TanStack Query last-success data
  → ClassifiedName.managerAddress
  → MigrationPlan.classified
  → atomic migration batch
  → ETHRegistry.grantRoles(resource, ROLE_SET_RESOLVER, Charlie)
  → verifier checks the role expected by that same plan
```

`apps/manager/src/features/migration/hooks/useV1Names.ts:12-29` queries V1 names using only the connected owner address. An upstream `staleTime` option is removed by `packages/utils/src/tanstack-query/neverthrow/query.ts:120-147`, so the finding does not rely on any guaranteed cache interval.

The reproduced primitive is that TanStack Query keeps its last successful data available during a background refetch. The Manager consumers gate on pending state and can continue to expose a ready migration plan while authority changes across the operation lifecycle.

For an unwrapped `.eth` name, `packages/migration/src/service/classifyNames.ts:148-185` copies `domain.owner.id` into `managerAddress` when it differs from Alice's registrant address:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)

const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

`apps/manager/src/features/migration/service/buildMigrationPlan.ts:355-380` passes that classified object into the atomic batch builder.

`apps/manager/src/features/migration/pages/MigrationPage.tsx:259-282` starts migration with the already-built `gasEstimate.plan`, while `apps/manager/src/features/migration/state/migrationUi.machine.ts:150-163,243-267` captures and forwards that same plan.

Execution refreshes route/readiness information but not the V1 manager identity. `apps/manager/src/features/migration/service/migrationService.ts:886-927` rebuilds the next atomic batch from `remaining`, which originates from `plan.classified`.

The approval check at `migrationService.ts:367-390` compares approval keys only; it does not read `V1Registry.owner(namehash(name))` or compare the current canonical V1 manager with the planned V2 grantee.

At the irreversible authority sink, `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-26` directly encodes the captured manager address:

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

`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624` creates both the manager-role execution and its verification expectation from the same captured `classified.managerAddress`.

Lines 742-780 order the MigrationHelper call before the separate manager-role grant, and lines 783-816 wrap those calls in the owner-authorized HCA execution.

The MigrationHelper calldata builder at `apps/manager/src/features/migration/service/buildMigrationHelperCall.ts:60-86` receives migration data grouped by token type and does not receive or select `managerAddress`.

The stale `grantRoles(..., Charlie)` call is therefore constructed separately by the Manager application and is not rebound by the helper to the current V1 manager.

Finally, `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-353` checks whether the destination registry contains the role for the account already stored in the migration plan:

```ts
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
```

This checks execution consistency against the existing migration plan, not authority freshness against current canonical V1 state.

Plan integrity and authority freshness are different properties: the verifier detects divergence from the plan, but a stale plan that executes exactly as constructed is accepted.

The reproduced authority sequence is:

```
AUTHORIZED → REVOKED → ACTION REVERTS → STALE MIGRATION
→ NEW V2 AUTHORITY → ACTION SUCCEEDS → RESOLUTION REDIRECTED
```

Alice initially owns the BaseRegistrar token while Charlie is legitimately configured as the V1 Registry manager.

The Manager captures Charlie into the migration representation and migration plan. Alice then canonically revokes Charlie using `reclaim` while retaining ownership of the registration.

After the reclaim, Charlie's protected V1 resolver-changing action reverts, demonstrating that his previous V1 authority has actually been removed.

If the retained migration plan is consumed before the captured manager authority is rebound to current canonical V1 state, Alice can proceed with the normal V1→V2 migration while the application still encodes Charlie as the destination `ROLE_SET_RESOLVER` grantee.

The Bob differential demonstrates the same authority divergence independently: when Bob is made the current canonical V1 manager after Charlie's revocation, the retained migration plan still grants Charlie the V2 resolver role rather than Bob.

The root cause is therefore the absence of a binding between mutable canonical V1 manager authority and the irreversible V2 settlement input.

This is an ENS Manager application transaction-construction issue. The ENS smart contracts themselves are not claimed to be vulnerable.

Alice does sign the owner-authorized migration transaction, but that signature does not constitute a fresh, address-specific authorization restoring Charlie.

`apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:81-108` presents generic steps such as:

* `"Approve manager restoration"`
* `"Restore the existing managers for your names"`

After Alice has canonically revoked Charlie, Charlie is no longer an existing manager.

Additionally, `apps/manager/src/features/migration/service/buildStepDescriptors.ts:13-31` contains no field exposing:

* the manager address being restored;
* the exact role being granted; or
* the stale-versus-current authority difference.

Alice therefore authorizes the migration flow as presented; she is not explicitly authorizing a previously revoked Charlie address to receive fresh resolver-control authority.

The reviewed competition known-issue material contains adjacent migration, cache, transfer-role, and UI-state issues, but none documents the same combination of:

1. mutable V1 manager authority captured before revocation;
2. absence of an execution-time freshness binding at the V2 role-grant boundary; and
3. a canonically revoked actor receiving a fresh `ROLE_SET_RESOLVER` grant and subsequently redirecting resolution.

No trusted protocol actor is required to make a discretionary mistake for the stale grant to be constructed.

The failure is structural: the Manager application never validates current canonical V1 manager authority at the V2 role-grant construction boundary.

## Impact Details

A former V1 manager who has been canonically revoked by Alice can regain V2 resolver-control authority during Alice's legitimate V1→V2 migration and can then independently redirect the ENS name to a Charlie-controlled resolution target.

Alice's explicit revocation decision is silently undone without any fresh, address-specific authorization restoring Charlie. Alice loses exclusive resolver control until she detects and manually removes the recreated role.

This directly maps to the competition's selected Critical impact:

> "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer)."

The selected impact explicitly includes migration scenarios in which a name, its ownership, registry, or resolver is assigned to an attacker.

The demonstrated consequence here is resolver hijacking during the V1→V2 Manager migration flow.

At exploitation time, Charlie requires no:

* Alice private key;
* Alice authenticated session;
* Manager browser access;
* HCA or session key;
* backend credentials;
* governance authority;
* ENS administrator role;
* protocol privilege; or
* arbitrary calldata injection.

Charlie does not need to initiate, manipulate, or co-sign the migration.

After Alice revokes him, Charlie can remain passive until Alice performs the legitimate migration. Once the stale migration plan recreates `ROLE_SET_RESOLVER` for Charlie, he can independently exercise that newly created V2 authority and replace the resolver.

The impact does not rely on hypothetical fund theft, permanent ownership loss, or an additional payment-redirection scenario.

The demonstrated security consequence is the restoration of resolver authority to a canonically revoked actor, followed by successful resolver replacement and redirection of the ENS name's resolution to an attacker-controlled target.

## References

* `apps/manager/src/features/migration/hooks/useV1Names.ts:12-29`
* `packages/utils/src/tanstack-query/neverthrow/query.ts:120-147`
* `packages/migration/src/service/classifyNames.ts:148-185`
* `apps/manager/src/features/migration/service/buildMigrationPlan.ts:355-380`
* `apps/manager/src/features/migration/pages/MigrationPage.tsx:259-282`
* `apps/manager/src/features/migration/state/migrationUi.machine.ts:150-163,243-267`
* `apps/manager/src/features/migration/service/migrationService.ts:367-390,886-927`
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-26`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624,742-780,783-816`
* `apps/manager/src/features/migration/service/buildMigrationHelperCall.ts:60-86`
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-353`
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:81-108`
* `apps/manager/src/features/migration/service/buildStepDescriptors.ts:13-31`
* ENS contracts-v2 access-control documentation: <https://github.com/ensdomains/contracts-v2/blob/main/contracts/README.md>
* ENS Immunefi competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

## Proof of Concept

## PoC Overview

The final PoC is local and deterministic. It combines two focused tests in one command:

* `ENS_H01.cache.test.tsx` imports the production `useV1Names` hook and demonstrates that Charlie remains readable and the query remains non-pending while a background refresh whose eventual manager is Bob is unresolved.
* `ENS_H01.settlement.test.ts` imports the production Manager classifier/plan builder, atomic batch construction, HCA outer-call encoder, V2 resource derivation and post-state verifier. It demonstrates the revoke→stale grant→new authority→resolver redirection sequence and a Bob-current/Charlie-destination differential.

The unavailable ENS-v2 settlement runtime is represented by minimal local contracts implementing only the access-control semantics consumed by the production Manager output. This boundary is deliberate: validation was restricted to the frozen competition repository and local/offline assets; no external image, public fork, live system or third party was contacted.

Keeping the cache and settlement checks as two focused tests is more reviewable than hiding both boundaries inside one large browser mock; `run-final-poc.sh` still executes them as one coherent command.

## Preconditions

* Use the scope-locked local checkout verified as source-equivalent to the competition's audited revision `63772fd872af472ced58b009499355f3430c2a86` as `target/` beside `v3.2/`.
* Node.js 22 and the repository's pinned pnpm toolchain/dependencies.
* Foundry tools `forge`, `anvil`, and `cast` in `PATH` (or the standard Foundry user-bin location used by the runner).
* TCP port `8547` available locally.
* No public RPC or network access is required.

## Reproduction Steps

{% stepper %}
{% step %}

### Save the PoC files

Save the files under **FULL POC SOURCE** at their indicated repository-relative paths.
{% endstep %}

{% step %}

### Run the PoC

From the audit package root, run:

```bash
bash v3.2/tests/h01/run-final-poc.sh
```

{% endstep %}

{% step %}

### Observe runner behavior

The runner builds only the local Solidity equivalents, starts Anvil on chain ID `11155111`, runs both focused Vitest files against the frozen Manager checkout, then stops Anvil.
{% endstep %}

{% step %}

### Inspect settlement assertions

Inspect the settlement assertions showing canonical reclaim, the rejected pre-migration Charlie action, the decoded production grant, the Bob differential, the recreated role and the changed resolver result.
{% endstep %}
{% endstepper %}

## Expected Vulnerable Result

Before migration, current V1 authority is Alice or Bob rather than Charlie, Alice still owns the registration token, and Charlie's protected V1 resolver action reverts. Nevertheless, the retained production Manager plan encodes `grantRoles(resource, 1 << 24, Charlie)`. After local settlement of that production-built batch, Charlie has the V2 resolver role, Charlie's resolver action succeeds, and resolution changes from `ALICE_TARGET` to `CHARLIE_TARGET`.

The third settlement test is a remediation control: inserting an atomic current-manager check immediately before the role grant preserves an unchanged legitimate Charlie but aborts the Alice/Bob/replayed stale-plan cases.

## Command

From the audit package root:

```bash
bash v3.2/tests/h01/run-final-poc.sh
```

## Observed Result

The final local run under Node.js 22 completed successfully:

```
Test Files  2 passed (2)
Tests       4 passed (4)
```

The run comprised one production-hook cache/refetch test and three isolated-chain settlement/patch tests. It contacted no live ENS deployment, public testnet, subgraph, Rhinestone service, PostHog service, or other third party.

## Security Assertions Proven

* Alice owns the BaseRegistrar token while Charlie begins as the legitimate V1 manager.
* The production Manager representation and migration plan capture Charlie.
* Alice's canonical reclaim changes current V1 manager to Alice or Bob without transferring the registration.
* Charlie's old protected V1 action fails after reclaim.
* The production-built atomic batch still contains `grantRoles(resource, ROLE_SET_RESOLVER, Charlie)`.
* With Bob current in V1, V2 grants Charlie and does not grant Bob in the local settlement semantics.
* The production verifier accepts the destination role expected by its stale plan and does not consult V1.
* Charlie's newly granted V2 authority permits the resolver change in the local access-control equivalent.
* Resolution changes from Alice's target to Charlie's target.
* An atomic execution-time current-manager guard blocks the tested stale Alice/Bob/retry variants.

## PoC Fidelity / Limitations

**Production Manager code proven:** both tests import the frozen production hook or transaction-building path unchanged. The settlement test uses production classification, migration-plan construction, atomic call ordering, HCA outer-call encoding and verification logic. The actual generated calldata is decoded and asserted before execution.

**Local equivalent contract semantics proven:** `H01V1Registry`, `H01BaseRegistrar`, `H01HCA`, `H01MigrationHelperBoundary`, `H01PermissionedRegistry` and `H01ControlledResolver` are minimal audit-local contracts. They demonstrate canonical V1 reclaim, V1 action rejection, HCA atomic execution, exact helper-call reachability, V2 role enforcement, resolver replacement and resulting address resolution. They are not represented as production ENS contracts.

**Published ENSv2 semantic corroboration:** ENS's public contracts-v2 documentation defines `ROLE_SET_RESOLVER` at bit 24 and describes it as the permission to change a resolver address. This matches the exact role bitmap emitted by the production Manager and the access-control effect exercised by the local equivalent. The competition scope independently identifies a V1→V2 migration that assigns the resolver to an attacker as an example of Critical Manager-flow hijacking.

Official references:

* <https://github.com/ensdomains/contracts-v2/blob/main/contracts/README.md>
* <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

**Official ENS-v2 runtime not executed:** the frozen local workspace used for this PoC did not contain the competition ENS-v2 snapshot or runtime artifacts required to execute the complete settlement runtime locally. The report therefore does not claim execution of the official MigrationHelper, ETHRegistry, StandaloneHCA, or resolver runtime.

This limitation does not affect the demonstrated Manager-side grantee-selection bug: the production MigrationHelper calldata builder does not receive `managerAddress`, while the stale `grantRoles(..., Charlie)` is a separate application-built call. The local contracts are used only to demonstrate the access-control consequence of the exact authority grant produced by the in-scope Manager code.

## FULL POC SOURCE

### `v3.2/tests/h01/ENS_H01.cache.test.tsx`

```tsx
// ENS-H01 query-cache proof. This drives the production useV1Names hook with
// one persistent QueryClient and changes only the subgraph's next response.
import {
  focusManager,
  QueryClient,
  QueryClientProvider,
} from '@tanstack/react-query'
import { renderHook, waitFor } from '@testing-library/react'
import { ok, ResultAsync } from 'neverthrow'
import type { ReactNode } from 'react'
import type { Address } from 'viem'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import type { V1Domain } from '@/features/migration/service/v1SubgraphClient'

const smartAccountMock = vi.hoisted(() => ({
  useSmartAccountContext: vi.fn(),
}))

const wagmiMock = vi.hoisted(() => ({
  useConnection: vi.fn(),
}))

const v1SubgraphMock = vi.hoisted(() => ({
  getV1NamesForAddress: vi.fn(),
}))

vi.mock('@/lib/smart-account', () => smartAccountMock)
vi.mock('wagmi', () => wagmiMock)

vi.mock('@/features/migration/service/v1SubgraphClient', async () => {
  const actual = await vi.importActual<
    typeof import('@/features/migration/service/v1SubgraphClient')
  >('@/features/migration/service/v1SubgraphClient')

  return { ...actual, ...v1SubgraphMock }
})

import { useV1Names } from '@/features/migration/hooks/useV1Names'

const ALICE =
  '0x1111111111111111111111111111111111111111' as Address
const CHARLIE =
  '0x3333333333333333333333333333333333333333' as Address
const BOB =
  '0x2222222222222222222222222222222222222222' as Address

const domainWithManager = (manager: Address): V1Domain => ({
  id: '0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa',
  labelName: 'h01-cache',
  labelhash:
    '0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb',
  name: 'h01-cache.eth',
  resolver: null,
  owner: { id: manager },
  registrant: { id: ALICE },
  wrappedOwner: null,
  parent: { name: 'eth', wrappedDomain: null },
  registration: { expiryDate: '99999999999' },
  wrappedDomain: null,
})

const deferred = <T,>() => {
  let resolve!: (value: T) => void

  const promise = new Promise<T>((done) => {
    resolve = done
  })

  return { promise, resolve }
}

describe('ENS-H01 retained source result during refetch', () => {
  beforeEach(() => {
    vi.clearAllMocks()

    smartAccountMock.useSmartAccountContext.mockReturnValue({
      ownerAddress: ALICE,
    })

    wagmiMock.useConnection.mockReturnValue({ address: ALICE })

    focusManager.setFocused(undefined)
  })

  it('keeps Charlie usable while a focus refetch of Bob is unresolved', async () => {
    const queryClient = new QueryClient({
      defaultOptions: { queries: { retry: false } },
    })

    const Wrapper = ({
      children,
    }: {
      children: ReactNode
    }) => (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    )

    v1SubgraphMock.getV1NamesForAddress.mockReturnValueOnce(
      ok([domainWithManager(CHARLIE)]),
    )

    const first = renderHook(() => useV1Names(), {
      wrapper: Wrapper,
    })

    await waitFor(() => {
      expect(first.result.current.data?.[0]?.owner.id).toBe(
        CHARLIE,
      )
    })

    expect(
      v1SubgraphMock.getV1NamesForAddress,
    ).toHaveBeenCalledOnce()

    // The resultQueryOptions wrapper drops useV1Names' declared staleTime, so
    // focus starts a background refetch. Existing data remains Charlie and the
    // UI uses isPending (not isFetching), leaving the old plan actionable while
    // this canonical/indexed refresh is unresolved.
    const next = deferred<V1Domain[]>()

    v1SubgraphMock.getV1NamesForAddress.mockReturnValue(
      ResultAsync.fromPromise(
        next.promise,
        (cause) => cause as Error,
      ),
    )

    focusManager.setFocused(false)
    focusManager.setFocused(true)

    await waitFor(() => {
      expect(
        v1SubgraphMock.getV1NamesForAddress,
      ).toHaveBeenCalledTimes(2)
    })

    expect(first.result.current.data?.[0]?.owner.id).toBe(
      CHARLIE,
    )

    expect(first.result.current.isPending).toBe(false)

    next.resolve([domainWithManager(BOB)])

    await waitFor(() => {
      expect(first.result.current.data?.[0]?.owner.id).toBe(BOB)
    })
  })
})
```

### `v3.2/tests/h01/ENS_H01.settlement.test.ts`

```ts
// ENS-H01 isolated-chain authority round-trip and atomic patch proof.
// Production classification, plan, atomic batch, HCA wrapping, manager grant,
// and resource derivation are imported unchanged from the locked checkout.
import { readFileSync } from 'node:fs'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

import { buildHcaOwnerExecutionCall } from '@ens-apps/smart-account'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'

import {
  createPublicClient,
  createTestClient,
  createWalletClient,
  decodeFunctionData,
  encodeFunctionData,
  http,
  keccak256,
  namehash,
  parseAbi,
  stringToBytes,
  type Address,
  type Hash,
  type Hex,
} from 'viem'

import { sepolia } from 'viem/chains'
import {
  beforeAll,
  beforeEach,
  describe,
  expect,
  it,
} from 'vitest'

import { V2_CONTRACTS } from '@/features/migration/contracts/addresses'
import { buildMigrationPlan } from '@/features/migration/service/buildMigrationPlan'
import type { MigrationPreflight } from '@/features/migration/service/computeMigrationPreflight'
import type { DirectMigrationRoute } from '@/features/migration/service/directMigrationRoutes'
import type { V1Domain } from '@/features/migration/service/v1SubgraphClient'
import { verifyAtomicMigrationBatch } from '@/features/migration/service/verifyAtomicMigrationBatch'

const RPC_URL =
  process.env.H01_RPC_URL ?? 'http://127.0.0.1:8547'

const here = path.dirname(fileURLToPath(import.meta.url))

const ROLE_SET_RESOLVER = 1n << 24n

const LABEL = 'h01-stale-manager'
const NAME = `${LABEL}.eth`
const NODE = namehash(NAME)
const LABELHASH = keccak256(stringToBytes(LABEL))
const TOKEN_ID = BigInt(LABELHASH)
const RESOURCE = labelToCanonicalId(LABEL)

// Default Anvil accounts. Only local JSON-RPC signing is used.
const ALICE =
  '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as Address

const CHARLIE =
  '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC' as Address

const BOB =
  '0x90F79bf6EB2c4f870365E785982E1f101E93b906' as Address

const ALICE_TARGET =
  '0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65' as Address

const CHARLIE_TARGET =
  '0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc' as Address

const HCA =
  '0x00000000000000000000000000000000000000A1' as Address

const publicClient = createPublicClient({
  chain: sepolia,
  transport: http(RPC_URL),
})

const testClient = createTestClient({
  chain: sepolia,
  mode: 'anvil',
  transport: http(RPC_URL),
})

const aliceWallet = createWalletClient({
  account: ALICE,
  chain: sepolia,
  transport: http(RPC_URL),
})

const charlieWallet = createWalletClient({
  account: CHARLIE,
  chain: sepolia,
  transport: http(RPC_URL),
})

type Artifact = {
  abi: readonly unknown[]
  bytecode: { object: Hex }
  deployedBytecode: { object: Hex }
}

const artifact = (contract: string): Artifact =>
  JSON.parse(
    readFileSync(
      path.join(
        here,
        'settlement/out/H01LocalAuthority.sol',
        `${contract}.json`,
      ),
      'utf8',
    ),
  ) as Artifact

const rpc = async (
  method: string,
  params: unknown[] = [],
) => testClient.request({ method, params } as never)

const waitReceipt = (hash: Hash) =>
  publicClient.waitForTransactionReceipt({ hash })

const deploy = async (
  contract: string,
  args: readonly unknown[] = [],
) => {
  const a = artifact(contract)

  const hash = await aliceWallet.deployContract({
    abi: a.abi as never,
    bytecode: a.bytecode.object,
    args: args as never,
  })

  const receipt = await waitReceipt(hash)

  if (!receipt.contractAddress) {
    throw new Error(`${contract} deployment failed`)
  }

  return receipt.contractAddress
}

const installRuntime = async (
  address: Address,
  contract: string,
) => {
  await rpc('anvil_setCode', [
    address,
    artifact(contract).deployedBytecode.object,
  ])
}

const write = async (params: {
  account?: Address
  address: Address
  abi: readonly unknown[]
  functionName: string
  args?: readonly unknown[]
}) => {
  const wallet =
    params.account === CHARLIE
      ? charlieWallet
      : aliceWallet

  const hash = await wallet.writeContract({
    address: params.address,
    abi: params.abi as never,
    functionName: params.functionName as never,
    args: (params.args ?? []) as never,
  })

  const receipt = await waitReceipt(hash)

  if (receipt.status !== 'success') {
    throw new Error('local write reverted')
  }

  return receipt
}

const sendCall = async (
  wallet:
    | typeof aliceWallet
    | typeof charlieWallet,
  call: {
    to: Address
    data: Hex
    value: bigint
  },
) => {
  const hash = await wallet.sendTransaction(call)

  const receipt = await waitReceipt(hash)

  if (receipt.status !== 'success') {
    throw new Error('local call reverted')
  }

  return receipt
}

const v2Abi = parseAbi([
  'function seed(uint256 resource, address owner, address resolver)',
  'function setApprovalForAll(address operator, bool approved)',
  'function isApprovedForAll(address owner, address operator) view returns (bool)',
  'function grantRoles(uint256 resource, uint256 roles, address account)',
  'function hasRoles(uint256 resource, uint256 roles, address account) view returns (bool)',
  'function setResolver(uint256 resource, address resolver)',
  'function resolver(uint256 resource) view returns (address)',
])

const v1RegistryAbi = parseAbi([
  'function initialize(address registrar)',
  'function owner(bytes32 node) view returns (address)',
  'function resolver(bytes32 node) view returns (address)',
  'function setResolver(bytes32 node, address resolver)',
])

const baseRegistrarAbi = parseAbi([
  'function seed(uint256 tokenId, bytes32 node, address registrant, address manager, address resolver)',
  'function ownerOf(uint256 tokenId) view returns (address)',
  'function reclaim(uint256 tokenId, address manager)',
])

const hcaAbi = parseAbi([
  'function initialize(address owner)',
])

const helperBoundaryAbi = parseAbi([
  'function called() view returns (bool)',
])

const resolverAbi = parseAbi([
  'function addr(bytes32 node) view returns (address)',
])

const guardAbi = parseAbi([
  'function assertCurrentManager(address registry, bytes32 node, address expectedManager) view',
])

const grantAbi = parseAbi([
  'function grantRoles(uint256 resource, uint256 roleBitmap, address account)',
])

type LocalState = {
  registry: Address
  registrar: Address
  aliceResolver: Address
  charlieResolver: Address
  guard: Address
  staleDomain: V1Domain
  plan: Awaited<
    ReturnType<typeof buildMigrationPlan>
  >
}

const currentResolution =
  async (): Promise<Address> => {
    const resolver =
      await publicClient.readContract({
        address: V2_CONTRACTS.ETHRegistry,
        abi: v2Abi,
        functionName: 'resolver',
        args: [RESOURCE],
      })

    return publicClient.readContract({
      address: resolver,
      abi: resolverAbi,
      functionName: 'addr',
      args: [NODE],
    })
  }

const buildState =
  async (): Promise<LocalState> => {
    await installRuntime(
      V2_CONTRACTS.ETHRegistry,
      'H01PermissionedRegistry',
    )

    await installRuntime(
      V2_CONTRACTS.MigrationHelper,
      'H01MigrationHelperBoundary',
    )

    await installRuntime(
      HCA,
      'H01HCA',
    )

    const registry =
      await deploy('H01V1Registry')

    const registrar =
      await deploy(
        'H01BaseRegistrar',
        [registry],
      )

    const aliceResolver =
      await deploy(
        'H01ControlledResolver',
        [ALICE_TARGET],
      )

    const charlieResolver =
      await deploy(
        'H01ControlledResolver',
        [CHARLIE_TARGET],
      )

    const guard =
      await deploy(
        'H01SourceAuthorityGuard',
      )

    await write({
      address: registry,
      abi: v1RegistryAbi,
      functionName: 'initialize',
      args: [registrar],
    })

    await write({
      address: registrar,
      abi: baseRegistrarAbi,
      functionName: 'seed',
      args: [
        TOKEN_ID,
        NODE,
        ALICE,
        CHARLIE,
        aliceResolver,
      ],
    })

    await write({
      address: HCA,
      abi: hcaAbi,
      functionName: 'initialize',
      args: [ALICE],
    })

    await write({
      address:
        V2_CONTRACTS.ETHRegistry,
      abi: v2Abi,
      functionName: 'seed',
      args: [
        RESOURCE,
        ALICE,
        aliceResolver,
      ],
    })

    await write({
      address:
        V2_CONTRACTS.ETHRegistry,
      abi: v2Abi,
      functionName: 'setApprovalForAll',
      args: [HCA, true],
    })

    // This is the exact pre-revocation representation returned by the V1
    // subgraph and retained by useV1Names/MigrationPlan.
    const staleDomain: V1Domain = {
      id: NODE,
      labelName: LABEL,
      labelhash: LABELHASH,
      name: NAME,
      resolver: {
        address: aliceResolver,
      },
      owner: {
        id: CHARLIE,
      },
      registrant: {
        id: ALICE,
      },
      wrappedOwner: null,
      parent: {
        name: 'eth',
        wrappedDomain: null,
      },
      registration: {
        expiryDate: '99999999999',
      },
      wrappedDomain: null,
    }

    const directRoute:
      DirectMigrationRoute = {
        name: NAME,
        receiver:
          V2_CONTRACTS.UnlockedMigrationController,
        parentDependency: null,
        expectedWrapperRegistry: null,
        receiverReadiness:
          'migration-controller',
      }

    const preflight = {
      preExistingOwnedPermRes: null,
      skipApprovalPhase: true,
      skipFetchProfilesPhase: true,
      baseRegistrarApproved: true,
      nameWrapperApproved: true,
      requiresManagerRestoration: true,
      migrationApprovals: [],
      directMigrationRoutes:
        new Map([
          [NAME, directRoute],
        ]),
    } as unknown as MigrationPreflight

    const plan =
      await buildMigrationPlan({
        domains: [staleDomain],
        hcaAddress: HCA,
        migrationOwner: ALICE,
        publicClient: {
          chain: {
            id: sepolia.id,
          },
        } as never,
        preflight,
      })

    return {
      registry,
      registrar,
      aliceResolver,
      charlieResolver,
      guard,
      staleDomain,
      plan,
    }
  }

const reclaim = (
  state: LocalState,
  manager: Address,
) =>
  write({
    address: state.registrar,
    abi: baseRegistrarAbi,
    functionName: 'reclaim',
    args: [
      TOKEN_ID,
      manager,
    ],
  })

const executeCurrentPlan =
  async (
    state: LocalState,
  ) => {
    const batch =
      state.plan.atomicBatches[0]

    if (!batch) {
      throw new Error(
        'missing production atomic batch',
      )
    }

    return sendCall(
      aliceWallet,
      batch.outerCall,
    )
  }

const patchedOuterCall = (
  state: LocalState,
) => {
  const batch =
    state.plan.atomicBatches[0]

  if (!batch) {
    throw new Error(
      'missing production atomic batch',
    )
  }

  const calls =
    batch.innerExecutions.flatMap(
      (execution) => {
        if (
          execution.phase !==
          'manager-role-grant'
        ) {
          return [execution.call]
        }

        return [
          {
            to: state.guard,
            value: 0n,
            data:
              encodeFunctionData({
                abi: guardAbi,
                functionName:
                  'assertCurrentManager',
                args: [
                  state.registry,
                  NODE,
                  CHARLIE,
                ],
              }),
          },
          execution.call,
        ]
      },
    )

  return buildHcaOwnerExecutionCall({
    hca: HCA,
    calls,
  })
}

beforeAll(async () => {
  const block =
    await publicClient.getBlockNumber()

  expect(block).toBeGreaterThanOrEqual(0n)
})

beforeEach(async () => {
  await rpc('anvil_reset')
})

describe(
  'ENS-H01 isolated authority round trip',
  () => {
    it(
      'restores revoked Charlie through the production plan and lets Charlie redirect resolution',
      async () => {
        const state =
          await buildState()

        const classified =
          state.plan.classified[0]

        expect(
          classified?.managerAddress.toLowerCase(),
        ).toBe(
          CHARLIE.toLowerCase(),
        )

        const managerExecution =
          state.plan
            .atomicBatches[0]
            ?.innerExecutions.find(
              ({ phase }) =>
                phase ===
                'manager-role-grant',
            )

        expect(
          managerExecution,
        ).toBeDefined()

        const decoded =
          decodeFunctionData({
            abi: grantAbi,
            data:
              managerExecution!
                .call.data,
          })

        expect(
          decoded.functionName,
        ).toBe('grantRoles')

        expect(
          decoded.args,
        ).toEqual([
          RESOURCE,
          ROLE_SET_RESOLVER,
          CHARLIE,
        ])

        expect(
          managerExecution!
            .call.to.toLowerCase(),
        ).toBe(
          V2_CONTRACTS
            .ETHRegistry
            .toLowerCase(),
        )

        // STATE A: Charlie is the legitimate V1 manager and can act.
        await write({
          account: CHARLIE,
          address:
            state.registry,
          abi: v1RegistryAbi,
          functionName:
            'setResolver',
          args: [
            NODE,
            state.charlieResolver,
          ],
        })

        await write({
          account: CHARLIE,
          address:
            state.registry,
          abi: v1RegistryAbi,
          functionName:
            'setResolver',
          args: [
            NODE,
            state.aliceResolver,
          ],
        })

        // STATE B: Alice's canonical reclaim revokes Charlie but leaves the
        // registrar NFT with Alice, so migration eligibility survives.
        await reclaim(
          state,
          ALICE,
        )

        expect(
          await publicClient.readContract({
            address:
              state.registry,
            abi: v1RegistryAbi,
            functionName:
              'owner',
            args: [NODE],
          }),
        ).toBe(ALICE)

        expect(
          await publicClient.readContract({
            address:
              state.registrar,
            abi: baseRegistrarAbi,
            functionName:
              'ownerOf',
            args: [
              TOKEN_ID,
            ],
          }),
        ).toBe(ALICE)

        await expect(
          write({
            account:
              CHARLIE,
            address:
              state.registry,
            abi:
              v1RegistryAbi,
            functionName:
              'setResolver',
            args: [
              NODE,
              state.charlieResolver,
            ],
          }),
        ).rejects.toThrow()

        // STATE C/D: Alice sends the exact immutable production outer batch.
        // It calls the helper boundary, then grants stale Charlie SET_RESOLVER.
        expect(
          await currentResolution(),
        ).toBe(
          ALICE_TARGET,
        )

        await executeCurrentPlan(
          state,
        )

        expect(
          await publicClient.readContract({
            address:
              V2_CONTRACTS
                .MigrationHelper,
            abi:
              helperBoundaryAbi,
            functionName:
              'called',
          }),
        ).toBe(true)

        expect(
          await publicClient.readContract({
            address:
              V2_CONTRACTS
                .ETHRegistry,
            abi: v2Abi,
            functionName:
              'hasRoles',
            args: [
              RESOURCE,
              ROLE_SET_RESOLVER,
              CHARLIE,
            ],
          }),
        ).toBe(true)

        // The production verifier accepts the post-state because its manager
        // expectation is the same stale plan; it never consults V1 here.
        await expect(
          verifyAtomicMigrationBatch({
            publicClient:
              publicClient as never,
            batch:
              state.plan
                .atomicBatches[0]!,
          }),
        ).resolves.toMatchObject({
          status: 'confirmed',
        })

        await write({
          account: CHARLIE,
          address:
            V2_CONTRACTS
              .ETHRegistry,
          abi: v2Abi,
          functionName:
            'setResolver',
          args: [
            RESOURCE,
            state.charlieResolver,
          ],
        })

        expect(
          await currentResolution(),
        ).toBe(
          CHARLIE_TARGET,
        )
      },
    )

    it(
      'produces the Bob-current / Charlie-destination cross-consumer differential',
      async () => {
        const state =
          await buildState()

        await reclaim(
          state,
          BOB,
        )

        expect(
          await publicClient.readContract({
            address:
              state.registry,
            abi: v1RegistryAbi,
            functionName:
              'owner',
            args: [NODE],
          }),
        ).toBe(BOB)

        await executeCurrentPlan(
          state,
        )

        expect(
          await publicClient.readContract({
            address:
              V2_CONTRACTS
                .ETHRegistry,
            abi: v2Abi,
            functionName:
              'hasRoles',
            args: [
              RESOURCE,
              ROLE_SET_RESOLVER,
              CHARLIE,
            ],
          }),
        ).toBe(true)

        expect(
          await publicClient.readContract({
            address:
              V2_CONTRACTS
                .ETHRegistry,
            abi: v2Abi,
            functionName:
              'hasRoles',
            args: [
              RESOURCE,
              ROLE_SET_RESOLVER,
              BOB,
            ],
          }),
        ).toBe(false)
      },
    )

    it(
      'A/B: an atomic execution-time source guard blocks Alice/Bob/retry stale plans and preserves unchanged Charlie',
      async () => {
        // A: unchanged legitimate manager still migrates and is restored.
        let state =
          await buildState()

        await sendCall(
          aliceWallet,
          patchedOuterCall(state),
        )

        expect(
          await publicClient.readContract({
            address:
              V2_CONTRACTS
                .ETHRegistry,
            abi: v2Abi,
            functionName:
              'hasRoles',
            args: [
              RESOURCE,
              ROLE_SET_RESOLVER,
              CHARLIE,
            ],
          }),
        ).toBe(true)

        // B1: manager changed to Alice after preview; old plan aborts atomically.
        await rpc(
          'anvil_reset',
        )

        state =
          await buildState()

        await reclaim(
          state,
          ALICE,
        )

        await expect(
          sendCall(
            aliceWallet,
            patchedOuterCall(
              state,
            ),
          ),
        ).rejects.toThrow()

        expect(
          await publicClient.readContract({
            address:
              V2_CONTRACTS
                .ETHRegistry,
            abi: v2Abi,
            functionName:
              'hasRoles',
            args: [
              RESOURCE,
              ROLE_SET_RESOLVER,
              CHARLIE,
            ],
          }),
        ).toBe(false)

        // B2 / retry-old-plan: Bob becomes current after the plan exists. Reusing
        // that old plan is still rejected by the settlement-time guard.
        await rpc(
          'anvil_reset',
        )

        state =
          await buildState()

        await reclaim(
          state,
          BOB,
        )

        const oldPatchedPlan =
          patchedOuterCall(
            state,
          )

        await expect(
          sendCall(
            aliceWallet,
            oldPatchedPlan,
          ),
        ).rejects.toThrow()

        await expect(
          sendCall(
            aliceWallet,
            oldPatchedPlan,
          ),
        ).rejects.toThrow()

        expect(
          await publicClient.readContract({
            address:
              V2_CONTRACTS
                .ETHRegistry,
            abi: v2Abi,
            functionName:
              'hasRoles',
            args: [
              RESOURCE,
              ROLE_SET_RESOLVER,
              CHARLIE,
            ],
          }),
        ).toBe(false)
      },
    )
  },
)
```

### `v3.2/tests/h01/settlement/src/H01LocalAuthority.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// Minimal local replica of the V1 registry authority needed by ENS-H01.
contract H01V1Registry {
    mapping(bytes32 => address) public owner;
    mapping(bytes32 => address) public resolver;

    address public registrar;

    function initialize(
        address registrar_
    ) external {
        require(
            registrar == address(0),
            "already initialized"
        );

        registrar = registrar_;
    }

    function seed(
        bytes32 node,
        address manager,
        address resolver_
    ) external {
        require(
            msg.sender == registrar,
            "registrar only"
        );

        owner[node] = manager;
        resolver[node] = resolver_;
    }

    function reclaim(
        bytes32 node,
        address manager
    ) external {
        require(
            msg.sender == registrar,
            "registrar only"
        );

        owner[node] = manager;
    }

    function setResolver(
        bytes32 node,
        address resolver_
    ) external {
        require(
            msg.sender == owner[node],
            "not manager"
        );

        resolver[node] = resolver_;
    }
}

/// Minimal BaseRegistrar reclaim semantics: the ERC-721 holder can reset the
/// V1 registry owner without transferring the registration token.
contract H01BaseRegistrar {
    H01V1Registry public registry;

    mapping(uint256 => address) public ownerOf;
    mapping(uint256 => bytes32) public nodeOf;

    constructor(
        H01V1Registry registry_
    ) {
        registry = registry_;
    }

    function seed(
        uint256 tokenId,
        bytes32 node,
        address registrant,
        address manager,
        address resolver_
    ) external {
        require(
            ownerOf[tokenId] == address(0),
            "already seeded"
        );

        ownerOf[tokenId] = registrant;
        nodeOf[tokenId] = node;

        registry.seed(
            node,
            manager,
            resolver_
        );
    }

    function reclaim(
        uint256 tokenId,
        address manager
    ) external {
        require(
            msg.sender == ownerOf[tokenId],
            "not registrant"
        );

        registry.reclaim(
            nodeOf[tokenId],
            manager
        );
    }
}

/// Runtime installed at the production-plan ETHRegistry address on Anvil.
contract H01PermissionedRegistry {
    uint256 internal constant ROLE_SET_RESOLVER =
        1 << 24;

    mapping(uint256 => address) public owner;

    mapping(
        address =>
            mapping(address => bool)
    ) public isApprovedForAll;

    mapping(
        uint256 =>
            mapping(address => uint256)
    ) public roles;

    mapping(uint256 => address) public resolver;

    function seed(
        uint256 resource,
        address owner_,
        address resolver_
    ) external {
        owner[resource] = owner_;
        resolver[resource] = resolver_;
    }

    function setApprovalForAll(
        address operator,
        bool approved
    ) external {
        isApprovedForAll[msg.sender][operator] =
            approved;
    }

    function grantRoles(
        uint256 resource,
        uint256 roleBitmap,
        address account
    ) external {
        address resourceOwner =
            owner[resource];

        require(
            msg.sender == resourceOwner ||
                isApprovedForAll[
                    resourceOwner
                ][msg.sender],
            "grant denied"
        );

        roles[resource][account] |=
            roleBitmap;
    }

    function hasRoles(
        uint256 resource,
        uint256 roleBitmap,
        address account
    ) external view returns (bool) {
        return
            roles[resource][account] &
                roleBitmap ==
            roleBitmap;
    }

    function getOwner(
        uint256 resource
    ) external view returns (address) {
        return owner[resource];
    }

    function getResolver(
        string calldata label
    ) external view returns (address) {
        uint256 labelId =
            uint256(
                keccak256(
                    bytes(label)
                )
            );

        uint256 canonicalId =
            labelId ^
            (labelId & 0xffffffff);

        return resolver[canonicalId];
    }

    function setResolver(
        uint256 resource,
        address resolver_
    ) external {
        require(
            msg.sender ==
                owner[resource] ||
                roles[resource][msg.sender] &
                    ROLE_SET_RESOLVER !=
                0,
            "resolver denied"
        );

        resolver[resource] = resolver_;
    }
}

/// Local StandaloneHCA execution surface matching the production outer call.
contract H01HCA {
    struct Execution {
        address target;
        uint256 value;
        bytes callData;
    }

    address public owner;

    function initialize(
        address owner_
    ) external {
        require(
            owner == address(0),
            "already initialized"
        );

        owner = owner_;
    }

    function executeByOwner(
        Execution[] calldata executions
    ) external payable {
        require(
            msg.sender == owner,
            "owner only"
        );

        for (
            uint256 i;
            i < executions.length;
            ++i
        ) {
            (
                bool ok,
                bytes memory data
            ) = executions[i]
                .target
                .call{
                    value:
                        executions[i]
                            .value
                }(
                    executions[i]
                        .callData
                );

            if (!ok) {
                assembly {
                    revert(
                        add(data, 32),
                        mload(data)
                    )
                }
            }
        }
    }
}

/// The contracts-v2 helper is out of competition scope. This local boundary
/// records that the exact production helper calldata was executed; the test
/// seeds the post-helper V2 owner separately before exercising the app-owned
/// manager-restoration sink.
contract H01MigrationHelperBoundary {
    bool public called;

    fallback() external {
        called = true;
    }
}

contract H01ControlledResolver {
    address public immutable answer;

    constructor(
        address answer_
    ) {
        answer = answer_;
    }

    function addr(
        bytes32
    ) external view returns (address) {
        return answer;
    }
}

/// Reference atomic patch: execution revalidates the V1 manager inside the
/// same transaction immediately before the V2 grant. No external transaction
/// can interleave between this guard and settlement.
contract H01SourceAuthorityGuard {
    function assertCurrentManager(
        H01V1Registry registry,
        bytes32 node,
        address expectedManager
    ) external view {
        require(
            registry.owner(node) ==
                expectedManager,
            "stale manager"
        );
    }
}
```

### `v3.2/tests/h01/settlement/foundry.toml`

```toml
[profile.default]
src = "src"
out = "out"
solc_version = "0.8.24"
optimizer = true
optimizer_runs = 200
via_ir = true
```

### `v3.2/tests/h01/final.vitest.config.ts`

```ts
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const here = path.dirname(
  fileURLToPath(import.meta.url),
)

const targetRoot = path.resolve(
  here,
  '../../../target',
)

export default {
  root: targetRoot,

  resolve: {
    alias: {
      '@': path.join(
        targetRoot,
        'apps/manager/src',
      ),

      '@ens-apps/migration':
        path.join(
          targetRoot,
          'packages/migration/src',
        ),

      '@ens-apps/smart-account':
        path.join(
          targetRoot,
          'packages/smart-account/src/index.ts',
        ),

      '@ens-apps/transaction-manager':
        path.join(
          targetRoot,
          'packages/transaction-manager/src/index.ts',
        ),

      '@ens-apps/utils':
        path.join(
          targetRoot,
          'packages/utils/src',
        ),

      '@ensdomains/ensjs/utils/v2':
        path.join(
          targetRoot,
          'apps/manager/node_modules/@ensdomains/ensjs/dist/utils/v2/index.js',
        ),

      '@tanstack/react-query':
        path.join(
          targetRoot,
          'apps/manager/node_modules/@tanstack/react-query',
        ),

      '@testing-library/react':
        path.join(
          targetRoot,
          'apps/manager/node_modules/@testing-library/react',
        ),

      neverthrow:
        path.join(
          targetRoot,
          'apps/manager/node_modules/neverthrow',
        ),

      react:
        path.join(
          targetRoot,
          'apps/manager/node_modules/react',
        ),

      wagmi:
        path.join(
          targetRoot,
          'apps/manager/node_modules/wagmi',
        ),

      viem:
        path.join(
          targetRoot,
          'packages/transaction-manager/node_modules/viem',
        ),
    },

    dedupe: [
      '@rhinestone/sdk',
    ],
  },

  test: {
    environment: 'happy-dom',

    include: [
      path.join(
        here,
        'ENS_H01.cache.test.tsx',
      ),

      path.join(
        here,
        'ENS_H01.settlement.test.ts',
      ),
    ],

    testTimeout: 30_000,
    hookTimeout: 30_000,

    sequence: {
      concurrent: false,
    },
  },

  define: {
    global: 'globalThis',
    'process.env': {},
    'import.meta.env': {},
  },
}
```

### `v3.2/tests/h01/run-final-poc.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Foundry's default installer places its binaries here. Existing PATH entries
# retain precedence so managed installations continue to work unchanged.
export PATH="$PATH:$HOME/.foundry/bin"

script_dir="$(
  cd "$(dirname "${BASH_SOURCE[0]}")" &&
    pwd
)"

audit_root="$(
  cd "$script_dir/../../.." &&
    pwd
)"

rpc_url="http://127.0.0.1:8547"

forge build \
  --root "$script_dir/settlement" \
  >/dev/null

anvil \
  --silent \
  --port 8547 \
  --chain-id 11155111 \
  >/dev/null 2>&1 &

anvil_pid=$!

trap \
  'kill "$anvil_pid" 2>/dev/null || true' \
  EXIT

for _ in {1..50}; do
  if cast block-number \
    --rpc-url "$rpc_url" \
    >/dev/null 2>&1
  then
    break
  fi

  sleep 0.1
done

cast block-number \
  --rpc-url "$rpc_url" \
  >/dev/null

cd "$audit_root/target"

H01_RPC_URL="$rpc_url" \
  corepack pnpm \
  --filter manager \
  exec vitest run \
  --config \
  ../../v3.2/tests/h01/final.vitest.config.ts
```
