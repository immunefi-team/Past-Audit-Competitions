# 89461 w a insight missing cleanup on an interrupted migration leaves a permanent registry wide setapprovalforall on the ens v2 registry exposing every name the wallet owns to theft

**Submitted on Aug 19th 2026 at 13:24:18 UTC by @OxPrince for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89461
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Direct theft of user NFTs

## Description

## Brief/Intro

The Manager app's v1→v2 migration asks the connected wallet to sign `ETHRegistry.setApprovalForAll(hcaAddress, true)` — a blanket operator grant that authorises the user's smart account (HCA) to move **every v2 ENS name that wallet owns**, not only the names being migrated.

The application treats this grant as strictly temporary: it is the one approval singled out by a `requiresMigrationApprovalCleanup` predicate, the code states *"Only direct HCA permissions are temporary"*, and the closing step of the flow is labelled *"Revoking temporary HCA access"*.

That revocation is skipped whenever the migration does not run to completion in the same browser session. Because the revocation list is derived from *"which approvals were missing at preflight"* rather than *"which approvals are currently granted"*, every subsequent attempt drops it from the plan and completes successfully while revoking nothing.

The user is left holding a permanent, registry-wide operator approval over their entire v2 portfolio, created by a flow that told them it was temporary, with no surface in the application that displays or removes it. Any party who subsequently exercises that smart account can transfer all of the user's names.

## Vulnerability Details

### 1. What the grant authorises, and when it is requested

The migration requests this approval whenever any selected name carries a v1 manager distinct from its registrant — the ordinary consequence of buying a v1 name on a marketplace, since a marketplace transfer moves the registrant but not the controller. That is precisely the population this migration serves.

[`computeMigrationPreflight.ts:205`](broken://pages/b5cfe39464f323f358fcd294e355ff418937a573#L205)

```ts
const requiresManagerRestoration = classified.some(
  (name) => name.managerAddress !== null,
)
```

When that holds, the approval is planned:

[`migrationApprovals.ts:320-323`](broken://pages/53bdb52024658e3a5c10b2d6c030e79c715e6b79#L320-L323)

```ts
if (needs.requiresManagerRestoration && !status.ethRegistryHcaApproved) {
  approvals.push(migrationApprovalForId({ id: 'eth-registry:hca', hcaAddress }))
}
```

and encoded:

[`migrationApprovals.ts:329-352`](broken://pages/53bdb52024658e3a5c10b2d6c030e79c715e6b79#L329-L352)

```ts
return {
  to: approval.contractAddress,          // V2_CONTRACTS.ETHRegistry
  data: encodeFunctionData({
    abi: OPERATOR_APPROVAL_ABI,
    functionName: 'setApprovalForAll',   // 0xa22cb465
    args: [approval.operatorAddress, true],   // operator = the user's HCA
  }),
  value: 0n,
}
```

`submitCall` sends it through `buildEOARequest`, i.e. **from the connected EOA**, so the resulting on-chain state is `ETHRegistry.isApprovedForAll(eoa, hca) === true`.

This is an operator approval scoped to the *registry*, not to a token set: it covers every v2 name the EOA holds there, including names acquired after the migration.

The application's own model treats this as the one grant that must not persist:

[`migrationApprovals.ts:74-80`](broken://pages/53bdb52024658e3a5c10b2d6c030e79c715e6b79#L74-L80)

```ts
/**
 * Only direct HCA permissions are temporary. MigrationHelper approvals are
 * intentionally reusable: the helper resolves every caller back to its
 * certified owner before it can move that owner's V1 names.
 */
export const requiresMigrationApprovalCleanup = (
  approval: MigrationApproval,
): approval is MigrationCleanupApproval =>
  approval.kind === 'operator' && approval.id === 'eth-registry:hca'
```

The user-facing copy makes the same promise:

* Wallet step: *"Allowing your HCA to restore managers"*
* Cleanup step: *"Removing manager-restoration access from your HCA"*
* Progress line: *"Revoking temporary HCA access"*

### 2. Defect one — the revocation sits behind an unguarded throw

[`migrationService.ts:1189-1212`](broken://pages/761ee912a18a91341732ac6030a2511e4761bfe9#L1189-L1212)

```ts
const approvalHashes = await ensureMigrationApprovals({
  ctx,
  plan: executionPlan,
  currentMissing,
})
txHashes.push(...approvalHashes)                     // <- grant lands on-chain here

txHashes.push(
  ...(await executeRemainingAtomicBatches({           // <- ANY throw exits the function
    ctx,
    plan: executionPlan,
    publicClient,
    onBatchComplete,
    retryPermissionHeadLag: approvalHashes.length > 0,
  })),
)
}
txHashes.push(
  ...(await revokeTemporaryOperatorApprovals({ ctx, plan })),   // <- never reached
)
ctx.tracker.complete('Migration complete', txHashes.at(-1))
```

There is no `try` / `catch` / `finally` anywhere in this block.

`executeRemainingAtomicBatches` is the long leg that drives every remaining wallet prompt and on-chain verification; it throws on a declined prompt, an RPC failure, a reverted batch, or a verification mismatch. Closing the tab produces the same outcome without any throw at all.

In every one of those cases, the grant is live on-chain and the revocation never runs.

### 3. Defect two — the revocation list is the preflight *missing* set

[`migrationService.ts:449-456`](broken://pages/761ee912a18a91341732ac6030a2511e4761bfe9#L449-L456)

```ts
const revokeTemporaryOperatorApprovals = async (params: {
  readonly ctx: MigrationCtx
  readonly plan: MigrationPlan
}): Promise<readonly Hex[]> => {
  const temporaryOperators = (
    params.plan.preflight.migrationApprovals ?? []
  ).filter(requiresMigrationApprovalCleanup)
```

`preflight.migrationApprovals` is documented in the type itself:

[`computeMigrationPreflight.ts:47-48`](broken://pages/b5cfe39464f323f358fcd294e355ff418937a573#L47-L48)

```ts
/** Missing grants only; confirmed operator entries remain available to the HCA. */
migrationApprovals?: readonly MigrationApproval[]
```

So a fresh session rebuilds the plan, `checkMigrationApprovals` reads `isApprovedForAll(eoa, hca) === true`, the `!status.ethRegistryHcaApproved` guard in `planMigrationApprovals` fails, and the approval is **absent from the plan**.

The revocation list filters an array that no longer contains it. The migration then reports success having revoked nothing.

`skipApprovalPhase: migrationApprovals.length === 0` ([`computeMigrationPreflight.ts:137`](broken://pages/b5cfe39464f323f358fcd294e355ff418937a573#L137)) additionally means neither an approval step nor a cleanup step is displayed on that run.

### 4. Why the existing recovery paths do not close it

| Mechanism                                                                                                                                                  | Why it does not help                                                                                                                                                                                                           |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `assertMigrationApprovalPlanCurrent` ([`:367`](broken://pages/761ee912a18a91341732ac6030a2511e4761bfe9#L367)) re-reads chain state before the first prompt | It only **compares** planned vs current approval keys and throws `MigrationPlanChangedError` on divergence. It never revokes. It is also skipped entirely on retries (`reconcileBeforeSubmit` ⇒ `currentMissing = undefined`). |
| Same-session retry (`resetForRetry`, [`migrationUi.machine.ts:203-215`](broken://pages/b420106da999369363fa734afea007a1d5d057ca#L203-L215))                | This one *does* work — `adjustPlanForRetry` spreads `...plan` and never touches `preflight`, and `revokeTemporaryOperatorApprovals` is passed the original `plan`. It covers only a retry inside the same page session.        |
| `cancel` / "Back" on the failure screen                                                                                                                    | Runs `resetAll` ([`:216`](broken://pages/b420106da999369363fa734afea007a1d5d057ca#L216)), discarding the plan — and with it the only record that a revocation is owed.                                                         |
| The **"Revoke temporary HCA access"** button ([`MigrationPage.tsx:375`](broken://pages/60c142341d5443303222b213d111e99100224d3d#L375))                     | Rendered only when `lastError.type === 'cleanup-failed'`, which is mapped from `MigrationCleanupError` — thrown **only from inside** `revokeTemporaryOperatorApprovals`. If that function never ran, the button never appears. |

`buildMigrationOperatorApprovalRevocationCall` has **exactly one call site** in the entire codebase ([`migrationService.ts:463`](broken://pages/761ee912a18a91341732ac6030a2511e4761bfe9#L463)), inside the block that no longer sees the approval.

Once the application is in this state, there is no path within the Manager that can clear it.

### 5. End-to-end failure narrative

{% stepper %}
{% step %}

## Grant approval

Alice owns `vault.eth` and `treasury.eth` on v2. She buys `alice.eth` on a marketplace — the seller remains its v1 manager — and opens the Manager to upgrade it.

`requiresManagerRestoration` is true, so the app prompts her wallet for `ETHRegistry.setApprovalForAll(hca, true)`. She approves.

The grant now covers `alice.eth`, `vault.eth`, `treasury.eth`, and anything she buys later.
{% endstep %}

{% step %}

## Interrupt the migration

Alice declines one of the later prompts, her wallet disconnects, or she closes the tab.

`executeRemainingAtomicBatches` exits; the revocation is skipped.
{% endstep %}

{% step %}

## Resume migration in a new session

Alice reopens the app the next day and the migration completes cleanly.

Preflight sees the approval already granted, omits it from the plan, and the revocation list is empty. She is shown **"Migration complete"**.
{% endstep %}

{% step %}

## Persistent approval remains

`ETHRegistry.isApprovedForAll(alice, hca)` remains `true`, indefinitely, with no UI indication and no in-app means of removal.
{% endstep %}
{% endstepper %}

## Impact Details

A permanent, registry-wide operator approval from the user's EOA to their HCA, covering **every v2 ENS name that EOA holds — present and future** — is created by a flow that states in its code, wallet-step label, and progress copy that the grant is temporary and will be revoked.

Nothing in the Manager surfaces it, and nothing in the Manager can remove it.

### Loss condition and scale

The residual becomes realised theft when the HCA is exercised by anyone other than the user.

Per the program's own known-issues entry **R2-03**, the HCA carries an ephemeral session key that is *"an unscoped full account owner"* for a 7-day lifetime, held in browser storage, and the entry instructs reporters to assume that key is exfiltratable.

Absent this approval, that key's blast radius is bounded by the HCA's own holdings — which after a settled registration is close to nothing, since the funding budget deliberately carries no buffer.

**With it, the blast radius is every v2 ENS name the EOA owns, without time limit.**

There is no cap on the number of names, and the exposure extends to names acquired long after the migration that created it.

## References

Add any relevant links to documentation or code

## Proof of Concept

## 3 POCs

```
PoC 1 — Control flow: an interrupted migration never reaches the revocation
        (real executeMigration; A/B on one variable)

PoC 2 — Sepolia fork: the approval is still live after a "successful" migration
        (real deployed ETHRegistry 0xBDC85dD5…F0E2, block 11,521,714)

PoC 3 — Unit: the next session drops the approval from the cleanup list
        (no anvil, no network — runs in ~10s)
```

### PoC 1 — Control flow: an interrupted migration never reaches the revocation (real `executeMigration`; A/B on one variable)

```rust
import type { Signer } from '@ens-apps/transaction-manager'
import type { RhinestoneAccount } from '@rhinestone/sdk'
import type { Config as WagmiConfig } from '@wagmi/core'
import {
  type Address,
  encodeErrorResult,
  type Hex,
  type PublicClient,
  parseAbi,
  type TransactionReceipt,
} from 'viem'
import { beforeEach, describe, expect, it, vi } from 'vitest'

const mocks = vi.hoisted(() => ({
  buildHcaDeploymentCall: vi.fn(),
  verifyStandaloneHca: vi.fn(),
  startTransaction: vi.fn(),
  waitForTransactionHash: vi.fn(),
  waitForTransaction: vi.fn(),
  buildAtomicMigrationBatches: vi.fn(),
  checkMigrationApprovals: vi.fn(),
  planMigrationApprovals: vi.fn(),
  buildMigrationApprovalCall: vi.fn(),
  checkResolverReadiness: vi.fn(),
  reconcileAtomicMigrationBatch: vi.fn(),
  verifyAtomicMigrationBatch: vi.fn(),
}))

vi.mock('@ens-apps/smart-account', async (importOriginal) => ({
  ...(await importOriginal<typeof import('@ens-apps/smart-account')>()),
  buildHcaDeploymentCall: mocks.buildHcaDeploymentCall,
  verifyStandaloneHca: mocks.verifyStandaloneHca,
}))

vi.mock('@ens-apps/transaction-manager', () => ({
  transactionManager: { startTransaction: mocks.startTransaction },
  waitForTransactionHash: mocks.waitForTransactionHash,
  waitForTransaction: mocks.waitForTransaction,
}))

vi.mock('./buildAtomicMigrationBatches', async (importOriginal) => ({
  ...(await importOriginal<typeof import('./buildAtomicMigrationBatches')>()),
  buildAtomicMigrationBatches: mocks.buildAtomicMigrationBatches,
}))

vi.mock('./migrationApprovals', async (importOriginal) => ({
  ...(await importOriginal<typeof import('./migrationApprovals')>()),
  checkMigrationApprovals: mocks.checkMigrationApprovals,
  planMigrationApprovals: mocks.planMigrationApprovals,
  buildMigrationApprovalCall: mocks.buildMigrationApprovalCall,
}))

vi.mock('./migrationInvariants', () => ({
  checkDeterministicMigrationResolverReadiness: mocks.checkResolverReadiness,
}))

vi.mock('./verifyAtomicMigrationBatch', async (importOriginal) => ({
  ...(await importOriginal<typeof import('./verifyAtomicMigrationBatch')>()),
  reconcileAtomicMigrationBatch: mocks.reconcileAtomicMigrationBatch,
  verifyAtomicMigrationBatch: mocks.verifyAtomicMigrationBatch,
}))

import type { BuildAtomicMigrationBatchesParams } from './buildAtomicMigrationBatches'
import type { MigrationPlan } from './buildMigrationPlan'
import type { ClassifiedName } from './classifyNames'
import type { MigrationApproval } from './migrationApprovals'
import {
  loadPendingAtomicMigrationIntents,
  loadSubmittedAtomicMigrationBatches,
  persistPendingAtomicMigrationIntent,
  persistSubmittedAtomicMigrationBatch,
} from './migrationBatchJournal'
import { executeMigration, type MigrationProgress } from './migrationService'
import type { V1Domain } from './v1SubgraphClient'
import {
  AtomicMigrationBatchReconciliationIndeterminateError,
  AtomicMigrationBatchVerificationError,
} from './verifyAtomicMigrationBatch'

const OWNER: Address = '0x0000000000000000000000000000000000000001'
const HCA: Address = '0x0000000000000000000000000000000000000002'
const FACTORY: Address = '0x0000000000000000000000000000000000000003'
const RESOLVER: Address = '0x0000000000000000000000000000000000000004'
const V1_RESOLVER: Address = '0x0000000000000000000000000000000000000005'
const APPROVAL_CONTRACT: Address = '0x0000000000000000000000000000000000000006'

const DEPLOY_DATA: Hex = '0xd3ad'
const OUTER_DATA: Hex = '0xcafe'
const GRANT_DATA: Hex = '0x01'
const DIRECT_PERMISSION_ERROR_ABI = parseAbi([
  'error ERC721InsufficientApproval(address operator, uint256 tokenId)',
])

const WAGMI = {} as WagmiConfig
const SIGNER = {
  type: 'eoa',
  walletClient: {},
} as unknown as Signer
const HCA_CLIENT = {
  getAddress: vi.fn(() => HCA),
  getInitData: vi.fn(),
} as unknown as Pick<RhinestoneAccount, 'getAddress' | 'getInitData'>

const getCodeMock = vi.fn()
const estimateGasMock = vi.fn()
const readContractMock = vi.fn()
const getTransactionReceiptMock = vi.fn()
const waitForReceiptMock = vi.fn()
const PUBLIC_CLIENT = {
  chain: { id: 11155111 },
  getCode: getCodeMock,
  estimateGas: estimateGasMock,
  readContract: readContractMock,
  getTransactionReceipt: getTransactionReceiptMock,
  waitForTransactionReceipt: waitForReceiptMock,
} as unknown as PublicClient

const APPROVAL: MigrationApproval = {
  kind: 'operator',
  id: 'base-registrar:hca',
  contractAddress: APPROVAL_CONTRACT,
  operatorAddress: HCA,
}
const MANAGER_APPROVAL: MigrationApproval = {
  kind: 'operator',
  id: 'eth-registry:hca',
  contractAddress: APPROVAL_CONTRACT,
  operatorAddress: HCA,
}

const hashFor = (value: number): Hex =>
  `0x${value.toString(16).padStart(64, '0')}` as Hex

const permissionMissingError = (): Error & { readonly data: Hex } =>
  Object.assign(new Error('ERC-721 approval is not visible yet'), {
    data: encodeErrorResult({
      abi: DIRECT_PERMISSION_ERROR_ABI,
      errorName: 'ERC721InsufficientApproval',
      args: [HCA, 2n],
    }),
  })

const domainFor = (label: string): V1Domain =>
  ({
    id: label,
    name: `${label}.eth`,
    labelName: label,
    labelhash:
      '0x0000000000000000000000000000000000000000000000000000000000000002',
    resolver: { address: V1_RESOLVER },
    owner: { id: OWNER },
    registrant: { id: OWNER },
    wrappedOwner: null,
    parent: { name: 'eth', wrappedDomain: null },
    registration: null,
    wrappedDomain: null,
  }) as V1Domain

const classifiedFor = (label: string): ClassifiedName => ({
  domain: domainFor(label),
  tokenType: 'unwrapped',
  label,
  parentName: 'eth',
  fuses: 0n,
  tokenHolder: OWNER,
  v1ResolverAddress: V1_RESOLVER,
  resolverStrategy: 'to-owned-permres',
  managerAddress: null,
})

const planFor = (labels: readonly string[] = ['alice']): MigrationPlan => {
  const classified = labels.map(classifiedFor)
  const nameExecutions = classified.map((classifiedName) => {
    const name = classifiedName.domain.name
    const directRoute = {
      name,
      receiver: RESOLVER,
      parentDependency: null,
      expectedWrapperRegistry: null,
      receiverReadiness: 'migration-controller' as const,
    }
    return {
      classified: classifiedName,
      directRoute,
      migrationData: {
        label: classifiedName.label,
        owner: OWNER,
        subregistry: RESOLVER,
        resolver: RESOLVER,
      },
      innerExecutions: [],
      verificationExpectations: [
        {
          id: `${name}:name-owner`,
          type: 'name-owner' as const,
          name,
          label: classifiedName.label,
          node: classifiedName.domain.id as Hex,
          resource: 1n,
          tokenType: classifiedName.tokenType,
          registryPath: {
            type: 'eth-registry-2ld' as const,
            registry: RESOLVER,
            label: classifiedName.label,
            resource: 1n,
          },
          expectedOwner: OWNER,
        },
      ],
    }
  })
  return {
    hcaAddress: HCA,
    hcaDeploymentRequired: false,
    migrationOwner: OWNER,
    domains: classified.map(({ domain }) => domain),
    classified,
    ineligible: [],
    groups: {
      unwrapped: classified,
      unlocked: [],
      locked2ld: [],
      childNames: new Map(),
    },
    preflight: {
      preExistingOwnedPermRes: null,
      skipApprovalPhase: false,
      skipFetchProfilesPhase: true,
      baseRegistrarApproved: false,
      nameWrapperApproved: false,
    },
    ownedPermRes: RESOLVER,
    profiles: new Map(),
    directRoutes: new Map(
      nameExecutions.map((execution) => [
        execution.classified.domain.name,
        execution.directRoute,
      ]),
    ),
    atomicBatches:
      nameExecutions.length === 0
        ? []
        : [
            {
              index: 0,
              names: nameExecutions.map(
                ({ classified: name }) => name.domain.name,
              ),
              nameExecutions,
              innerExecutions: [],
              outerCall: { to: HCA, data: OUTER_DATA, value: 0n },
              estimatedGas: 500_000n,
              verificationExpectations: nameExecutions.flatMap(
                (execution) => execution.verificationExpectations,
              ),
            },
          ],
    stepDescriptors: [],
  }
}

const runExecute = async (
  overrides: {
    plan?: MigrationPlan
    refreshAccount?: () => Promise<void>
    onBatchComplete?: (names: readonly string[], hash?: Hex) => void
    reconcileBeforeSubmit?: boolean
  } = {},
) => {
  const progressEvents: MigrationProgress[] = []
  const refreshAccount = overrides.refreshAccount ?? vi.fn()
  const result = await executeMigration({
    plan: overrides.plan ?? planFor(),
    wagmiConfig: WAGMI,
    publicClient: PUBLIC_CLIENT,
    signer: SIGNER,
    hcaClient: HCA_CLIENT,
    refreshAccount,
    onProgress: (progress) => progressEvents.push(progress),
    onBatchComplete: overrides.onBatchComplete,
    reconcileBeforeSubmit: overrides.reconcileBeforeSubmit,
  })
  return { progressEvents, refreshAccount, result }
}

beforeEach(() => {
  vi.clearAllMocks()
  localStorage.clear()

  let nextTransaction = 0
  mocks.startTransaction.mockImplementation(() => `tx-${nextTransaction++}`)
  const hashForTransaction = (txId: string) =>
    hashFor(Number.parseInt(txId.slice('tx-'.length), 10) + 1)
  mocks.waitForTransactionHash.mockImplementation((txId: string) =>
    Promise.resolve(hashForTransaction(txId)),
  )
  mocks.waitForTransaction.mockImplementation((txId: string) =>
    Promise.resolve({ hash: hashForTransaction(txId) }),
  )

  mocks.buildHcaDeploymentCall.mockReturnValue({
    to: FACTORY,
    data: DEPLOY_DATA,
    value: 0n,
  })
  mocks.verifyStandaloneHca.mockResolvedValue(HCA)
  mocks.checkMigrationApprovals.mockResolvedValue({})
  mocks.planMigrationApprovals.mockReturnValue([])
  mocks.buildMigrationApprovalCall.mockImplementation(
    (approval: MigrationApproval) => ({
      to: approval.contractAddress,
      data: GRANT_DATA,
      value: 0n,
    }),
  )
  mocks.checkResolverReadiness.mockResolvedValue({
    status: 'verified',
    resolver: RESOLVER,
    implementation: RESOLVER,
    hcaHasRootRoles: true,
    walletHasWildcardRoles: true,
  })
  mocks.buildAtomicMigrationBatches.mockImplementation(
    async ({
      hca,
      classified,
      estimateOuterGas,
    }: BuildAtomicMigrationBatchesParams) => {
      const names = classified.map(({ domain }) => domain.name)
      const outerCall = { to: hca, data: OUTER_DATA, value: 0n }
      const estimatedGas = await estimateOuterGas({
        call: outerCall,
        names,
        innerExecutions: [],
      })
      return {
        resolver: RESOLVER,
        batches: [
          {
            index: 0,
            names,
            nameExecutions: [],
            innerExecutions: [],
            outerCall,
            estimatedGas,
            verificationExpectations: [],
          },
        ],
      }
    },
  )
  mocks.reconcileAtomicMigrationBatch.mockResolvedValue({
    status: 'complete',
    verification: { batchIndex: 0, status: 'confirmed', results: [] },
  })
  mocks.verifyAtomicMigrationBatch.mockResolvedValue({
    batchIndex: 0,
    status: 'confirmed',
    results: [],
  })

  getCodeMock.mockResolvedValue('0x6000')
  estimateGasMock.mockResolvedValue(500_000n)
  readContractMock.mockImplementation(
    ({ functionName }: { functionName: string }) => {
      if (functionName === 'ownerOf') return Promise.resolve(OWNER)
      if (functionName === 'balanceOf') return Promise.resolve(1n)
      return Promise.resolve(true)
    },
  )
  getTransactionReceiptMock.mockResolvedValue({
    status: 'success',
    blockNumber: 123n,
  } as TransactionReceipt)
  waitForReceiptMock.mockResolvedValue({
    status: 'success',
    blockNumber: 123n,
  } as TransactionReceipt)
})

// ---------------------------------------------------------------------------
// W-01 CONTROL-FLOW POC
//
// Claim under test: `revokeTemporaryOperatorApprovals` sits AFTER
// `executeRemainingAtomicBatches` with no try/finally, so any throw from the
// batch leg skips the revocation entirely.
//
// The revocation call is built by the REAL
// `buildMigrationOperatorApprovalRevocationCall` (only the GRANT builder is
// mocked), so it is identifiable on the wire as setApprovalForAll(hca,false).
// ---------------------------------------------------------------------------

const REVOKE_SELECTOR = '0xa22cb465'
const revokeCalls = () =>
  mocks.startTransaction.mock.calls.filter((call) => {
    const data = (call?.[0] as { request?: { data?: string } })?.request?.data
    return (
      typeof data === 'string' &&
      data.startsWith(REVOKE_SELECTOR) &&
      // approved = false  ->  last word is all zeroes
      data.slice(-64) === '0'.repeat(64)
    )
  })

const planOwingRevocation = (): MigrationPlan => {
  const plan = planFor()
  return {
    ...plan,
    preflight: {
      ...plan.preflight,
      // Session A found this grant MISSING, so a revocation is owed.
      migrationApprovals: [MANAGER_APPROVAL],
    },
  }
}

describe('W-01: an interrupted migration never reaches the revocation', () => {
  beforeEach(() => {
    // The plan and the live chain agree, so assertMigrationApprovalPlanCurrent passes.
    mocks.planMigrationApprovals.mockReturnValue([MANAGER_APPROVAL])
  })

  it('CONTROL — when the batch leg succeeds, the revocation IS submitted', async () => {
    const { result } = await runExecute({ plan: planOwingRevocation() })

    expect(result.completed).toBe(1)
    expect(revokeCalls()).toHaveLength(1)

    const revoke = revokeCalls()[0]?.[0] as {
      request: { to: string; data: string; from: string }
    }
    // setApprovalForAll(HCA, false) sent from the owner EOA to the registry.
    expect(revoke.request.from).toBe(OWNER)
    expect(revoke.request.to).toBe(APPROVAL_CONTRACT)
    expect(revoke.request.data.toLowerCase()).toContain(HCA.slice(2).toLowerCase())
  })

  it('BUG — when the batch leg throws, the grant is submitted and the revocation is NOT', async () => {
    // Any failure in the long multi-prompt leg: a declined prompt, an RPC
    // failure, a reverted batch, a verification mismatch.
    mocks.verifyAtomicMigrationBatch.mockRejectedValue(
      new AtomicMigrationBatchVerificationError({
        message: 'batch verification failed',
        batchIndex: 0,
        names: ['alice.eth'],
      }),
    )

    await expect(runExecute({ plan: planOwingRevocation() })).rejects.toThrow()

    // The GRANT went out ...
    const grants = mocks.startTransaction.mock.calls.filter((call) => {
      const data = (call?.[0] as { request?: { data?: string } })?.request?.data
      return data === GRANT_DATA
    })
    expect(grants).toHaveLength(1)

    // ... and the REVOCATION never did.
    expect(revokeCalls()).toHaveLength(0)
  })
})
```

### PoC 2 — Sepolia fork: the approval is still live after a "successful" migration (real deployed ETHRegistry `0xBDC85dD5…F0E2`, block 11,521,714)

```rust
/**
 * W-01 FORK POC — real Sepolia contracts + real app code.
 *
 * Requires: anvil --fork-url <sepolia> --port 8545
 * and a prior `setApprovalForAll(HCA_GRANTED, true)` sent from EOA_GRANTED.
 */
import { createConfig, http } from '@wagmi/core'
import { sepolia } from '@wagmi/core/chains'
import { createPublicClient, http as viemHttp } from 'viem'
import { describe, expect, it } from 'vitest'
import { V2_CONTRACTS } from '../contracts/addresses'
import {
  checkMigrationApprovals,
  planMigrationApprovals,
  requiresMigrationApprovalCleanup,
} from './migrationApprovals'

const RPC = 'http://127.0.0.1:8545'

// Session A happened for this pair: the approval is already on-chain.
const EOA_GRANTED = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266' as const
const HCA_GRANTED = '0x1111111111111111111111111111111111111111' as const

// Control: never granted.
const EOA_FRESH = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as const
const HCA_FRESH = '0x2222222222222222222222222222222222222222' as const

const wagmiConfig = createConfig({
  chains: [sepolia],
  transports: { [sepolia.id]: http(RPC) },
})

const publicClient = createPublicClient({ chain: sepolia, transport: viemHttp(RPC) })

// A migration where a selected name has a v1 manager != registrant.
const needs = {
  hasUnwrapped: false,
  unwrappedTokenIds: [] as readonly bigint[],
  hasWrapped: false,
  requiresManagerRestoration: true,
}

const readApproval = (owner: `0x${string}`, operator: `0x${string}`) =>
  publicClient.readContract({
    address: V2_CONTRACTS.ETHRegistry,
    abi: [
      {
        name: 'isApprovedForAll',
        type: 'function',
        stateMutability: 'view',
        inputs: [
          { name: 'account', type: 'address' },
          { name: 'operator', type: 'address' },
        ],
        outputs: [{ type: 'bool' }],
      },
    ] as const,
    functionName: 'isApprovedForAll',
    args: [owner, operator],
  })

describe('W-01 fork PoC — the blanket approval survives a completed second migration', () => {
  it('CONTROL — a wallet that never granted it: the app plans the approval AND schedules its revocation', async () => {
    expect(await readApproval(EOA_FRESH, HCA_FRESH)).toBe(false)

    const status = await checkMigrationApprovals({
      eoa: EOA_FRESH,
      hcaAddress: HCA_FRESH,
      needs,
      wagmiConfig,
    })
    expect(status.ethRegistryHcaApproved).toBe(false)

    const planned = planMigrationApprovals({
      hcaAddress: HCA_FRESH,
      needs,
      status,
    })
    expect(planned.map((a) => a.id)).toEqual(['eth-registry:hca'])

    // This non-empty list is exactly what revokeTemporaryOperatorApprovals filters.
    expect(planned.filter(requiresMigrationApprovalCleanup)).toHaveLength(1)
  }, 60_000)

  it('SESSION B — the approval is live on-chain, and the app plans NO revocation for it', async () => {
    // The grant from the interrupted session A is real, on the real registry.
    expect(await readApproval(EOA_GRANTED, HCA_GRANTED)).toBe(true)

    // The app re-reads chain state exactly as computeMigrationPreflight does.
    const status = await checkMigrationApprovals({
      eoa: EOA_GRANTED,
      hcaAddress: HCA_GRANTED,
      needs,
      wagmiConfig,
    })
    expect(status.ethRegistryHcaApproved).toBe(true)

    // `preflight.migrationApprovals` is "missing grants only" -> empty.
    const planned = planMigrationApprovals({
      hcaAddress: HCA_GRANTED,
      needs,
      status,
    })
    expect(planned).toEqual([])

    // `revokeTemporaryOperatorApprovals` filters that list. Nothing to revoke.
    expect(planned.filter(requiresMigrationApprovalCleanup)).toEqual([])

    // So after a migration the app reports as COMPLETE, the blanket operator
    // approval over every v2 name this wallet owns is still live.
    expect(await readApproval(EOA_GRANTED, HCA_GRANTED)).toBe(true)
  }, 60_000)
})
```

### PoC 3 — Unit: the next session drops the approval from the cleanup list (no anvil, no network — runs in \~10s)

```rust
import { describe, expect, it } from 'vitest'
import type { Address } from 'viem'
import { V2_CONTRACTS } from '../contracts/addresses'
import {
  buildMigrationApprovalCall,
  type MigrationApprovalStatus,
  planMigrationApprovals,
  requiresMigrationApprovalCleanup,
} from './migrationApprovals'

const HCA = '0x1111111111111111111111111111111111111111' as Address

const needs = {
  hasUnwrapped: false,
  unwrappedTokenIds: [] as readonly bigint[],
  hasWrapped: false,
  requiresManagerRestoration: true, // a migrated name has a distinct v1 manager
}

const statusWith = (granted: boolean): MigrationApprovalStatus => ({
  baseRegistrarHcaApproved: true,
  unwrappedTokenApprovals: [],
  nameWrapperHcaApproved: true,
  ethRegistryHcaApproved: granted,
})

describe('PoC: the "temporary" ETHRegistry operator approval is never revoked after an abandoned attempt', () => {
  it('session A: the approval is planned, and it is a BLANKET setApprovalForAll on the V2 registry', () => {
    const planned = planMigrationApprovals({
      hcaAddress: HCA,
      needs,
      status: statusWith(false), // not yet granted on-chain
    })

    expect(planned).toHaveLength(1)
    const approval = planned[0]!
    expect(approval.id).toBe('eth-registry:hca')
    // The app itself marks this one as requiring cleanup.
    expect(requiresMigrationApprovalCleanup(approval)).toBe(true)

    const call = buildMigrationApprovalCall(approval)
    expect(call.to).toBe(V2_CONTRACTS.ETHRegistry)
    // setApprovalForAll(hca, true) — operator over EVERY v2 name the EOA owns,
    // not scoped to the names being migrated.
    expect(call.data.startsWith('0xa22cb465')).toBe(true)
    expect(call.data.toLowerCase()).toContain(HCA.slice(2).toLowerCase())
    expect(call.data.endsWith('1')).toBe(true) // approved = true
  })

  it('session B: once granted on-chain the approval drops out of the plan, so the cleanup list is EMPTY', () => {
    const planned = planMigrationApprovals({
      hcaAddress: HCA,
      needs,
      status: statusWith(true), // session A already granted it
    })

    // `preflight.migrationApprovals` is documented as "Missing grants only".
    expect(planned).toEqual([])

    // `revokeTemporaryOperatorApprovals` revokes exactly this filtered list.
    const cleanupTargets = planned.filter(requiresMigrationApprovalCleanup)
    expect(cleanupTargets).toEqual([])
  })
})
```
