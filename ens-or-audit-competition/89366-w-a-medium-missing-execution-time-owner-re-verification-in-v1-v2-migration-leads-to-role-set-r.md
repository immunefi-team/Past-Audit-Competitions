# 89366 w a medium missing execution time owner re verification in v1 v2 migration leads to role set resolver granted to a former manager

**Submitted on Aug 18th 2026 at 20:31:15 UTC by @pizzalover337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89366
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

During a v1→v2 migration in the Manager app, the `manager-role-grant` execution grants `ROLE_SET_RESOLVER` on the freshly migrated v2 name to the v1 manager address read from cached/subgraph data at plan time — and the execution path never re-reads the current v1 registry owner on-chain before submitting the grant. If the manager changed between preview and execution (a deterministic, app-side window of up to 5 minutes by construction — see Reachability), the live resolver-control role on the migrated name lands on the **former** manager. One follow-up `setResolver` call by that address gives them the name's resolution entirely: the migrated name then serves attacker-controlled records, including the ETH `addr` record, so payments and integrations route to the attacker.

## Impact

**Critical — "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a migration that assigns the name, its ownership, registry, or resolver to an attacker."**

The migration assigns a live `ROLE_SET_RESOLVER` grant — the power to assign the resolver — to an address the victim did not intend and cannot see (nothing in the UI surfaces the grant). This is the listed "resolver assigned to an attacker" example up to one trivial follow-up transaction by the grantee, after which the name resolves attacker-controlled records — the listed Critical impact "resolving or rendering an ENS name to an attacker-controlled address" (wrong-recipient funds). The "such as" list is illustrative; the governing phrase is *hijacking an ENS name during a Manager flow*, which is exactly what a live resolver-role grant to a former manager is. The victim owner can revoke the role only if they notice the invisible grant.

## Reachability — the window is deterministic and app-side

The grantee is read once at classification and frozen into the migration plan; execution re-resolves routes/readiness/gas but never the manager. Two compounding staleness layers make the frozen value stale in normal operation:

1. **App-side query cache (deterministic, dominant).** Classification input comes from `useV1Names`, a React-Query cache with `staleTime: 5 minutes` (`apps/manager/src/features/migration/hooks/useV1Names.ts:22`). Nothing invalidates it on v1 manager changes — `invalidateMigrationQueries` runs only after a *successful migration*. Any manager change (`setOwner`, transfer, sale settlement) is invisible to the migration planner for up to 5 minutes by construction, even with a fully synced subgraph.
2. **Subgraph indexing lag (amplifier).** Any indexing congestion or reorg widens the window further.

**Attacker-cued sequence (no lag required):** the attacker is the manager of record (e.g. a seller whose sale just settled, or a former operator) and cues the buyer/victim to open the Manager app → the victim's app populates the 5-minute cache with *attacker-as-manager* → the `setOwner` to the victim lands on-chain → the victim clicks **Upgrade** within the cache window → the plan classifies the attacker as manager → execution grants the attacker `ROLE_SET_RESOLVER` on the v2 name. Every step is a normal user action. The grant only fires for names whose manager differs from the registrant — i.e. precisely custodial/marketplace-held or recently-transferred names, the population where an incentivized former manager exists.

**Supporting variant (no race at all):** a seller transfers only the registrant NFT and retains managership — a stable, settled on-chain state (manager ≠ registrant, attacker is manager, no staleness anywhere). When the buyer migrates, the grant lands on the seller by construction, with zero verification against current chain state. We present this only as supporting evidence that the grantee is taken at face value, not as the lead scenario.

## Root cause

* Classification sets `managerAddress` from the subgraph's `domain.owner.id` (logic in `packages/migration`, re-exported via the shim `apps/manager/src/features/migration/service/classifyNames.ts`; **every defective consumption and execution step below is in-scope `apps/manager` code**).
* `buildMigrationPlan` embeds the grant via `buildRoleGrantCall` (`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-27`): `grantRoles(ETHRegistry_v2, labelToCanonicalId(name.label), ROLE_SET_RESOLVER, managerAddress)` (assembled at `buildAtomicMigrationBatches.ts:332,551,608-614`).
* `buildNextAtomicBatch` (`apps/manager/src/features/migration/service/migrationService.ts:886-964`) rebuilds batches from the frozen `plan.classified.managerAddress`: it re-resolves routes, resolver readiness and gas, but performs **zero** v1 registry-owner reads (proven by an instrumented execution in the PoC).
* The retry path is identical: `assertSourceTokensOwnedForRetry` (`migrationService.ts:705-752`) re-reads only the *registrant token* (`BaseRegistrar.ownerOf` / NameWrapper `balanceOf`), never the manager.
* `assertMigrationApprovalPlanCurrent` compares approval *keys*; the manager-restoration approval is address-independent (boolean `requiresManagerRestoration`), so a manager change cannot even trigger `MigrationPlanChangedError`.
* The post-state `manager-role` verification only proves the (stale) grant landed — it asserts the role was granted to the plan's address, not that the address is still the manager.

**The granted role is the real one.** The app's `ROLE_SET_RESOLVER = 1n << 24n` (`buildAtomicMigrationBatches.ts:48`, `buildRoleGrantCalls.ts:9`) matches the deployed-contract constants in `ensdomains/contracts-v2` (`contracts/script/deploy-constants.ts`: `REGISTRY.SET_RESOLVER: 1n << 24n`) — the deployment the in-scope manifest pins. (The e2e helper `e2e/helpers/migration-assertions.ts:46` uses `1n << 4n`, which is `REGISTRY.REGISTER_RESERVED` per the same source — an outdated test helper; the app is consistent with the deployed contracts.)

## Recommendation

Re-read the v1 registry `owner` for every classified unwrapped 2LD inside `buildNextAtomicBatch` (and the retry reconciliation) and rebuild or drop the `manager-role-grant` when it diverges from `plan.classified`. Defense in depth: invalidate `useV1Names` on migration-relevant changes, compare subgraph `_meta.block.number` against chain head at preview time, and surface any manager-restoration grant in the confirmation UI so it is never invisible.

## Proof of Concept

Against the **real, unmodified** migration code at the audited revision (`63772fd872af472ced58b009499355f3430c2a86`):

{% stepper %}
{% step %}

## Part 1

The migration plan's `manager-role-grant` inner execution decodes to `grantRoles(ETHRegistry_v2, labelToCanonicalId('alice'), ROLE_SET_RESOLVER = 1n << 24n, M_OLD)`, where `M_OLD` is the manager read from cached data at classification time.
{% endstep %}

{% step %}

## Part 2

The **real** execution path (`executeMigration` → `buildNextAtomicBatch` → `buildAtomicMigrationBatches` → `buildRoleGrantCall`, deliberately *not* mocked) rebuilds and submits the batch with the chain armed to answer `M_NEW` for any owner read — and the submitted `executeByOwner` calldata still grants **M\_OLD**, while the instrumented RPC log proves **zero** v1 registry-owner reads happen anywhere in the execution path. The grantee is purely plan-time cached data.
{% endstep %}
{% endstepper %}

Mocks are confined to external system boundaries (transaction submission capture, readiness/route/approval probes, RPC reads). Everything under test — plan builder, batch builders, role-grant encoder, execution orchestrator — is the real application code.

## Prerequisites

* Node v22+ and pnpm v10 (`corepack enable`)
* No RPC keys, wallets, or network access needed — the PoC runs fully offline.

## Steps

{% stepper %}
{% step %}

## Clone the competition repository and check out the audited revision

```bash
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596   # tree == audited revision 63772fd8…
pnpm install --frozen-lockfile
```

{% endstep %}

{% step %}

## Save the PoC file

Save the PoC file below as `apps/manager/src/features/migration/service/pocStaleManagerGrant.test.ts` (it runs with the app's own vitest config — no custom setup).
{% endstep %}

{% step %}

## Run it

```bash
cd apps/manager && npx vitest run pocStaleManagerGrant
```

{% endstep %}

{% step %}

## Expected output

```
✓ src/features/migration/service/pocStaleManagerGrant.test.ts (2 tests)
Test Files  1 passed (1)
     Tests  2 passed (2)
```

{% hint style="info" %}
An unrelated `[Reown Config] Failed to fetch remote project configuration` network warning may appear on stderr; it is a wallet-SDK telemetry call inside the app harness and does not affect the test.
{% endhint %}
{% endstep %}
{% endstepper %}

## How to read the result

* Part 1's assertions pin the grant's target (`V2_CONTRACTS.ETHRegistry`), resource (`labelToCanonicalId('alice')`), role (`1n << 24n`, matching `REGISTRY.SET_RESOLVER` in `contracts-v2/script/deploy-constants.ts`), and grantee (`M_OLD`, from the cached classification).
* Part 2 arms every on-chain owner read to return `M_NEW` (the current manager), runs the real execution path to completion (`result.completed === 1`, exactly one submitted transaction), decodes the submitted `executeByOwner` batch, and asserts the inner grant still names `M_OLD`. The final assertion — `ownerReads).toHaveLength(0)` — is the missing guard: the execution path re-resolves resolver readiness, routes, and gas, but never re-reads the v1 registry owner.
* Because `M_OLD` keeps live `ROLE_SET_RESOLVER` on the migrated v2 name, one `setResolver` transaction by `M_OLD` completes the hijack: the name then resolves attacker-controlled records.

## PoC file (`pocStaleManagerGrant.test.ts`)

```ts
/**
 * PoC — v1→v2 migration grants ROLE_SET_RESOLVER on the migrated v2 name to a
 * stale, cached v1 manager address that is never re-verified on-chain.
 *
 * Self-contained: runs with the app's own vitest config. No custom setup.
 *   cd apps/manager && npx vitest run pocStaleManagerGrant
 */
import type { Signer } from '@ens-apps/transaction-manager'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import type { RhinestoneAccount } from '@rhinestone/sdk'
import type { Config as WagmiConfig } from '@wagmi/core'
import {
  type Address,
  decodeFunctionData,
  type Hex,
  namehash,
  parseAbi,
  type PublicClient,
} from 'viem'
import { beforeEach, describe, expect, it, vi } from 'vitest'

const mocks = vi.hoisted(() => ({
  startTransaction: vi.fn(),
  waitForTransactionHash: vi.fn(),
  waitForTransaction: vi.fn(),
  verifyStandaloneHca: vi.fn(),
  checkResolverReadiness: vi.fn(),
  resolveDirectMigrationRoutes: vi.fn(),
  checkMigrationApprovals: vi.fn(),
  planMigrationApprovals: vi.fn(),
  reconcileAtomicMigrationBatch: vi.fn(),
  verifyAtomicMigrationBatch: vi.fn(),
}))

// Submission layer (external system) — captures every submitted request.
vi.mock('@ens-apps/transaction-manager', () => ({
  transactionManager: { startTransaction: mocks.startTransaction },
  waitForTransactionHash: mocks.waitForTransactionHash,
  waitForTransaction: mocks.waitForTransaction,
}))

// HCA account helpers: keep the REAL buildHcaOwnerExecutionCall /
// computeResolverAddress / getDestinationContracts; stub only chain probes.
vi.mock('@ens-apps/smart-account', async (importOriginal) => ({
  ...(await importOriginal<typeof import('@ens-apps/smart-account')>()),
  verifyStandaloneHca: mocks.verifyStandaloneHca,
}))

vi.mock('./migrationInvariants', () => ({
  checkDeterministicMigrationResolverReadiness: mocks.checkResolverReadiness,
}))

vi.mock('./directMigrationRoutes', async (importOriginal) => ({
  ...(await importOriginal<typeof import('./directMigrationRoutes')>()),
  resolveDirectMigrationRoutes: mocks.resolveDirectMigrationRoutes,
}))

vi.mock('./migrationApprovals', async (importOriginal) => ({
  ...(await importOriginal<typeof import('./migrationApprovals')>()),
  checkMigrationApprovals: mocks.checkMigrationApprovals,
  planMigrationApprovals: mocks.planMigrationApprovals,
}))

vi.mock('./verifyAtomicMigrationBatch', async (importOriginal) => ({
  ...(await importOriginal<typeof import('./verifyAtomicMigrationBatch')>()),
  reconcileAtomicMigrationBatch: mocks.reconcileAtomicMigrationBatch,
  verifyAtomicMigrationBatch: mocks.verifyAtomicMigrationBatch,
}))

import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { makeDomain, OWNER } from './_fixtures'
import { buildMigrationPlan } from './buildMigrationPlan'
import type { MigrationPreflight } from './computeMigrationPreflight'
import { executeMigration } from './migrationService'

const HCA: Address = '0x00000000000000000000000000000000000000ca'
const RECEIVER: Address = '0x0000000000000000000000000000000000000bbc'
const V1_CUSTOM_RESOLVER: Address = '0x00000000000000000000000000000000000000cc'
const M_OLD: Address = '0x0000000000000000000000000000000000000a01' // cached manager
const M_NEW: Address = '0x0000000000000000000000000000000000000b02' // on-chain manager now

const NAME = 'alice.eth'
const NODE = namehash(NAME)
// Matches contracts-v2 contracts/script/deploy-constants.ts REGISTRY.SET_RESOLVER.
const ROLE_SET_RESOLVER = 1n << 24n

const executeByOwnerAbi = parseAbi([
  'function executeByOwner((address target, uint256 value, bytes callData)[] executions) payable',
])

// ---- CHAIN TRUTH at execution time: the v1 manager is M_NEW ----------------
const chainTruthOwnerRead = (functionName: string): unknown => {
  if (functionName === 'owner' || functionName === 'ownerOf') return M_NEW
  return true
}

const readContractMock = vi.fn(
  (args: { functionName: string }): Promise<unknown> =>
    Promise.resolve(chainTruthOwnerRead(args.functionName)),
)

const publicClient = {
  chain: { id: 11155111 },
  getCode: vi.fn(() => Promise.resolve('0x6000' as Hex)),
  estimateGas: vi.fn(() => Promise.resolve(500_000n)),
  readContract: readContractMock,
  waitForTransactionReceipt: vi.fn(() =>
    Promise.resolve({ status: 'success', blockNumber: 123n }),
  ),
} as unknown as PublicClient

const unwrappedDomainWithManager = () =>
  makeDomain({
    id: NODE,
    name: NAME,
    labelName: 'alice',
    isWrapped: false,
    ownerId: M_OLD, // v1 registry owner ("manager") per the cached/stale read
    registrantId: OWNER, // victim wallet migrating
    resolverAddress: V1_CUSTOM_RESOLVER,
    registrationExpiry: '99999999999',
  })

const buildPlan = () =>
  buildMigrationPlan({
    domains: [unwrappedDomainWithManager()],
    hcaAddress: HCA,
    migrationOwner: OWNER,
    publicClient: { chain: { id: 11155111 } } as PublicClient,
    preflight: {
      preExistingOwnedPermRes: null,
      skipApprovalPhase: true,
      skipFetchProfilesPhase: true,
      baseRegistrarApproved: true,
      nameWrapperApproved: true,
      hcaReadiness: { status: 'deployment-required', hca: HCA },
      directMigrationRoutes: new Map([
        [
          NAME,
          {
            name: NAME,
            receiver: RECEIVER,
            parentDependency: null,
            expectedWrapperRegistry: null,
            receiverReadiness: 'migration-controller' as const,
          },
        ],
      ]),
    } satisfies MigrationPreflight,
  })

const decodeGrant = (data: Hex) => {
  const decoded = decodeFunctionData({ abi: ETH_REGISTRY_V2_ABI, data })
  expect(decoded.functionName).toBe('grantRoles')
  return decoded.args as unknown as readonly [bigint, bigint, Address]
}

beforeEach(() => {
  vi.clearAllMocks()
  localStorage.clear()

  let nextTransaction = 0
  mocks.startTransaction.mockImplementation(() => `tx-${nextTransaction++}`)
  const hashFor = (txId: string): Hex =>
    `0x${(Number.parseInt(txId.replace('tx-', ''), 10) + 1).toString(16).padStart(64, '0')}` as Hex
  mocks.waitForTransactionHash.mockImplementation((txId: string) =>
    Promise.resolve(hashFor(txId)),
  )
  mocks.waitForTransaction.mockImplementation((txId: string) =>
    Promise.resolve({
      hash: hashFor(txId),
      receipt: { status: 'success', blockNumber: 123n },
    }),
  )
  mocks.verifyStandaloneHca.mockResolvedValue(HCA)
  mocks.checkResolverReadiness.mockResolvedValue({
    status: 'verified',
    resolver: RECEIVER,
    implementation: RECEIVER,
    hcaHasRootRoles: true,
    walletHasWildcardRoles: true,
  })
  mocks.resolveDirectMigrationRoutes.mockResolvedValue(
    new Map([
      [
        NAME,
        {
          name: NAME,
          receiver: RECEIVER,
          parentDependency: null,
          expectedWrapperRegistry: null,
          receiverReadiness: 'migration-controller' as const,
        },
      ],
    ]),
  )
  mocks.checkMigrationApprovals.mockResolvedValue({})
  mocks.planMigrationApprovals.mockReturnValue([])
  mocks.reconcileAtomicMigrationBatch.mockResolvedValue({
    status: 'complete',
    verification: { batchIndex: 0, status: 'confirmed', results: [] },
  })
  mocks.verifyAtomicMigrationBatch.mockResolvedValue({
    batchIndex: 0,
    status: 'confirmed',
    results: [],
  })
})

describe('migration manager-role grant goes to the stale cached manager', () => {
  it('part 1 — plan encodes grantRoles(ROLE_SET_RESOLVER, M_OLD) on the v2 ETHRegistry', async () => {
    const plan = await buildPlan()

    // classification sources managerAddress purely from cached/subgraph owner
    expect(plan.classified[0]?.managerAddress?.toLowerCase()).toBe(
      M_OLD.toLowerCase(),
    )

    const batch = plan.atomicBatches[0]
    const grantExecution = batch?.innerExecutions.find(
      (e) => e.phase === 'manager-role-grant',
    )
    expect(grantExecution).toBeDefined()
    expect(grantExecution?.call.to).toBe(V2_CONTRACTS.ETHRegistry)

    const [resource, roles, account] = decodeGrant(
      grantExecution?.call.data as Hex,
    )
    expect(resource).toBe(labelToCanonicalId('alice'))
    expect(roles).toBe(ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(M_OLD.toLowerCase())

    // The post-state expectation only proves the grant to the stale address
    // happened; it cannot detect that M_OLD is no longer the manager.
    const expectation = batch?.verificationExpectations.find(
      (e) => e.type === 'manager-role',
    )
    expect(expectation).toMatchObject({
      account: M_OLD,
      roleBitmap: ROLE_SET_RESOLVER,
    })
  })

  it('part 2 — execution rebuilds and submits the stale grant without any on-chain owner re-read', async () => {
    // Sanity: chain truth at execution time is M_NEW.
    expect(chainTruthOwnerRead('owner')).toBe(M_NEW)

    const plan = await buildPlan()

    const result = await executeMigration({
      plan,
      wagmiConfig: {} as WagmiConfig,
      publicClient,
      signer: { type: 'eoa', walletClient: {} } as unknown as Signer,
      hcaClient: {
        getAddress: vi.fn(() => HCA),
        getInitData: vi.fn(),
      } as unknown as Pick<RhinestoneAccount, 'getAddress' | 'getInitData'>,
      refreshAccount: vi.fn(() => Promise.resolve()),
      onProgress: vi.fn(),
    })

    expect(result.completed).toBe(1)

    // Exactly one user transaction: the atomic migration batch.
    expect(mocks.startTransaction).toHaveBeenCalledTimes(1)
    const request = mocks.startTransaction.mock.calls[0]?.[0] as {
      request: { to: Address; data: Hex }
    }
    expect(request.request.to.toLowerCase()).toBe(HCA.toLowerCase())

    // Decode the REAL outer executeByOwner built at execution time by the REAL
    // buildNextAtomicBatch -> buildAtomicMigrationBatches -> buildRoleGrantCall.
    const outer = decodeFunctionData({
      abi: executeByOwnerAbi,
      data: request.request.data,
    })
    const executions = outer.args[0] as readonly {
      target: Address
      value: bigint
      callData: Hex
    }[]

    const grant = executions.find(
      (e) => e.target.toLowerCase() === V2_CONTRACTS.ETHRegistry.toLowerCase(),
    )
    expect(grant).toBeDefined()
    const [resource, roles, account] = decodeGrant(grant?.callData as Hex)
    expect(resource).toBe(labelToCanonicalId('alice'))
    expect(roles).toBe(ROLE_SET_RESOLVER)
    // THE EFFECT: the grantee is the stale cached manager, not the current
    // on-chain manager M_NEW.
    expect(account.toLowerCase()).toBe(M_OLD.toLowerCase())
    expect(account.toLowerCase()).not.toBe(M_NEW.toLowerCase())

    // The execution-time rebuild genuinely ran: readiness, routes and gas were
    // re-resolved at execution time.
    expect(mocks.checkResolverReadiness).toHaveBeenCalled()
    expect(mocks.resolveDirectMigrationRoutes).toHaveBeenCalled()
    expect(publicClient.estimateGas).toHaveBeenCalled()

    // THE GAP: across the whole execution path, no v1 registry-owner read was
    // ever performed — the grantee is purely plan-time cached data.
    const ownerReads = readContractMock.mock.calls.filter(([args]) => {
      const { functionName } = args as { functionName: string }
      return functionName === 'owner' || functionName === 'ownerOf'
    })
    expect(ownerReads).toHaveLength(0)
  })
})
```
