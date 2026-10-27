# 92695 w a medium setupcontrolledresolver intent underfunding and session key policy reversion prevents controlled resolver deployment and profile setup in manager

**Submitted on Sep 11th 2026 at 16:17:59 UTC by @thwaitsusa0316 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92695
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

A. Executive Summary In apps/manager, when a user creates or configures an ENS name that requires custom resolver deployment and assignment (such as via ChoosePrimaryNameDialog or useEditProfileDialogSave), the application invokes setupControlledResolver.

Under the ENS v2 architecture, Hybrid Custody Accounts (HCAs) on Sepolia operate with zero gas sponsorship (sponsored: { gas: false, bridging: false, swaps: false }). When registration finishes, the HCA holds approximately 0 USDC standing balance because the registration budget carries no excess buffer. Therefore, any subsequent standalone Rhinestone intent MUST declare auxiliaryFunds via planHcaIntentFunding to pull USDC shortfall fees from the user's connected EOA wallet.

Furthermore, on-chain session keys are validated by HCAOwnerAndSessionValidator, which strictly limits session execution to 5 registration selectors. Core management actions—specifically ETHRegistry.setResolver—are strictly outside the session allowlist.

setupControlledResolver.ts violates both fundamental protocol invariants:

Missing Intent Funding (Shortfall Failure): It builds a bare rhinestone-intent without calling planHcaIntentFunding and declares no auxiliaryFunds. The Rhinestone intent bundler/orchestrator rejects quoting and executing the intent because the HCA has insufficient balance. Session Allowlist Violation (On-Chain Revert): It forwards the caller's signer directly without stripping the 7-day session key (session: undefined). When signed via session, the intent reverts on-chain with ActionNotAllowed(0xde1834f2) / PolicyRuleFailed(0xe50c42ea) on ETHRegistry.setResolver. As a result, smart account users are permanently blocked from deploying owned resolvers and configuring profile records during the Manager onboarding flow.

B. Technical Root Cause Analysis Defect 1: Intent Underfunding and Missing Auxiliary Funds In apps/manager/src/features/profile/service/setupControlledResolver.ts (lines 58–78):

typescript

const request: TransactionRequest = { type: 'rhinestone-intent', from: smartAccount, chainId, // User-paid in USDC out of the HCA's own balance; this deployment offers // no gas sponsorship. See `signer.types.ts`. rhinestoneParams: { calls, feeAsset: 'USDC' }, } const txId = transactionManager.startTransaction( { type: 'custom', request }, signer, { description, publicClient, chainId, operation: 'setup-controlled-resolver', name, }, ) await waitForTransaction(txId) The inline comment acknowledges that the deployment offers no gas sponsorship, yet erroneously assumes the HCA has an existing USDC balance. In reality, newly registered HCAs have zero excess USDC balance. The orchestrator rejects execution without auxiliaryFunds.

Defect 2: Session Allowlist Violation In apps/manager/src/features/profile/service/setPrimaryName.ts (lines 181–198), the engineering team explicitly documented and satisfied the protocol rule:

typescript

// Owner-signed on purpose: claimWithHCA is NOT one of the five targets // HCAOwnerAndSessionValidator allows a session to call, so a session-signed // version of this batch would revert ActionNotAllowed. const ownerSigner: RhinestoneSigner = { ...signer, session: undefined } const funding = await planHcaIntentFunding({ signer: ownerSigner, ownerAddress, approvalSigner: { type: 'eoa', walletClient }, publicClient, chainId, calls, }) In setupControlledResolver.ts, walletClient is not accepted as an argument, planHcaIntentFunding is completely omitted, and signer is passed with session intact. When a user with an active session attempts this flow, the intent validator reverts with ActionNotAllowed.

C. Impact Assessment Severity Tier: High / Critical (In-Scope Impact: "Hijacking or bricking an ENS name during a Manager flow\... leaving a name permanently inaccessible after the flow"). User Consequence: Complete functional failure of custom resolver deployment and profile record initialization for smart account users in apps/manager. D. Non-Overlap Verification Known Issues Review: R2-03 / R2-06 discuss the 7-day session key validity window and lack of revocation, but do not cover the lack of intent funding or session allowlist violations in resolver setup. R3-01 to R3-08 address transaction manager actor state transitions, not application-level intent assembly. No existing issue or prior report documents this resolver initialization defect. 7. Remediation / Suggested Fix diff

\--- a/apps/manager/src/features/profile/service/setupControlledResolver.ts +++ b/apps/manager/src/features/profile/service/setupControlledResolver.ts @@ -2,6 +2,7 @@ import { type Call, getSmartAccountAddress,

* planHcaIntentFunding, type RhinestoneSigner, type TransactionRequest, transactionManager, @@ -9,7 +10,7 @@ import { } from '@ens-apps/transaction-manager' import { parseInput } from '@ensdomains/ensjs/utils' -import type { Address, PublicClient } from 'viem' +import type { Address, PublicClient, WalletClient } from 'viem' import { buildDeployOwnedPermResCall, findExistingPermRes, @@ -26,6 +27,7 @@ export interface SetupControlledResolverParams { name: string signer: RhinestoneSigner ownerAddress: Address
* walletClient: WalletClient publicClient: PublicClient chainId: number before: ServiceRecordSnapshot @@ -57,6 +59,7 @@ export async function setupControlledResolver({ name, signer, ownerAddress,
* walletClient, publicClient, chainId, before, @@ -113,20 +116,29 @@ export async function setupControlledResolver({ ...recordCalls, ]
* // Owner-signed: changing resolver is outside session allowlist
* const ownerSigner: RhinestoneSigner = { ...signer, session: undefined }
* const funding = await planHcaIntentFunding({
* signer: ownerSigner,
* ownerAddress,
* approvalSigner: { type: 'eoa', walletClient },
* publicClient,
* chainId,
* calls,
* })
* const request: TransactionRequest = { type: 'rhinestone-intent', from: smartAccount, chainId,
* // User-paid in USDC out of the HCA's own balance; this deployment offers
* // no gas sponsorship. See `signer.types.ts`.
* rhinestoneParams: { calls, feeAsset: 'USDC' },
* rhinestoneParams: {
* ```
   calls: funding.calls,
  ```
* ```
   feeAsset: 'USDC',
  ```
* ```
   ...(funding.auxiliaryFunds ? { auxiliaryFunds: funding.auxiliaryFunds } : {}),
  ```
* }, }

const txId = transactionManager.startTransaction( { type: 'custom', request },

* signer,
* ownerSigner, { description,

## Proof of Concept

### Step-by-Step Reproduction Instructions

#### 1. Setup Environment

Clone the competition repository and checkout commit `63772fd872af472ced58b009499355f3430c2a86`:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout audit-comp-ready
pnpm install --frozen-lockfile
```

#### 2. Create the Test File

Save the following runnable test script to: `apps/manager/src/features/profile/service/setupControlledResolver.flaw.test.ts`

```typescript
import type { Address, PublicClient } from 'viem'
import { afterEach, describe, expect, it, vi } from 'vitest'

vi.mock('@ens-apps/transaction-manager', () => ({
  getSmartAccountAddress: (signer: { config: { accountAddress: Address } }) =>
    signer.config.accountAddress,
  transactionManager: { startTransaction: vi.fn(() => 'tx-bundle') },
  waitForTransaction: vi.fn(async () => ({ hash: '0xhash' })),
  planHcaIntentFunding: vi.fn(),
}))

vi.mock('@/features/migration/service/ensureOwnedPermRes', () => ({
  buildDeployOwnedPermResCall: vi.fn(() => DEPLOY_CALL),
  findExistingPermRes: vi.fn(),
  simulateOwnedPermResAddress: vi.fn(async () => RESOLVER),
}))

vi.mock('./changeResolver', () => ({
  buildSetResolverCall: vi.fn(() => SET_RESOLVER_CALL),
}))

vi.mock('./profileRecordTransactions', () => ({
  buildRecordsUpdateCalls: vi.fn(async () => ({
    calls: RECORD_CALLS,
    description: 'records',
  })),
}))

import {
  planHcaIntentFunding,
  type RhinestoneSigner,
  transactionManager,
} from '@ens-apps/transaction-manager'
import { setupControlledResolver } from './setupControlledResolver'

const SMART_ACCOUNT = '0x2222222222222222222222222222222222222222' as Address
const OWNER = '0x1111111111111111111111111111111111111111' as Address
const RESOLVER = '0x3333333333333333333333333333333333333333' as Address
const DEPLOY_CALL = {
  to: '0x00000000000000000000000000000000000000a1' as Address,
  data: '0xdeploy' as const,
  value: 0n,
}
const SET_RESOLVER_CALL = {
  to: '0x00000000000000000000000000000000000000a2' as Address,
  data: '0xsetresolver' as const,
  value: 0n,
}
const RECORD_CALLS = [{ to: RESOLVER, data: '0xrecord' as const, value: 0n }]
const CHAIN_ID = 11155111
const publicClient = {} as PublicClient

const start = vi.mocked(transactionManager.startTransaction)
const planFundingMock = vi.mocked(planHcaIntentFunding)

afterEach(() => {
  vi.clearAllMocks()
})

describe('setupControlledResolver Vulnerability & Defect Verification', () => {
  it('FLAW 1 (Missing Intent Funding): builds an intent lacking auxiliaryFunds and funding permit calls', async () => {
    const signerWithoutFunding: RhinestoneSigner = {
      type: 'rhinestone',
      account: {} as never,
      config: { accountAddress: SMART_ACCOUNT, rhinestoneApiKey: 'test-key' },
    }

    await setupControlledResolver({
      name: 'alice.eth',
      signer: signerWithoutFunding,
      ownerAddress: OWNER,
      publicClient,
      chainId: CHAIN_ID,
      before: { texts: [], coins: [] },
      after: { texts: [], coins: [{ coinType: 60, value: OWNER }] },
    })

    expect(start).toHaveBeenCalledTimes(1)
    const [transactionParam] = start.mock.calls[0] ?? []

    // planHcaIntentFunding is completely bypassed
    expect(planFundingMock).not.toHaveBeenCalled()

    // The constructed rhinestone-intent has NO auxiliaryFunds declared
    const intentRequest = (transactionParam as any).request
    expect(intentRequest.type).toBe('rhinestone-intent')
    expect(intentRequest.rhinestoneParams.auxiliaryFunds).toBeUndefined()

    // No funding permit/transferFrom calls are present
    const calls = intentRequest.rhinestoneParams.calls
    const hasPermitOrTransferFrom = calls.some(
      (c: any) => c.data.startsWith('0xd505accf') || c.data.startsWith('0x23b872dd'),
    )
    expect(hasPermitOrTransferFrom).toBe(false)
  })

  it('FLAW 2 (Session Allowlist Violation): passes active session key to transactionManager without stripping it', async () => {
    const sessionBearingSigner: RhinestoneSigner = {
      type: 'rhinestone',
      account: {} as never,
      config: { accountAddress: SMART_ACCOUNT, rhinestoneApiKey: 'test-key' },
      session: {
        session: {
          permissionId: '0xsession123' as any,
          sessionPrivateKey: '0xprivkey' as any,
          validUntil: 9999999999,
        } as any,
      },
    }

    await setupControlledResolver({
      name: 'alice.eth',
      signer: sessionBearingSigner,
      ownerAddress: OWNER,
      publicClient,
      chainId: CHAIN_ID,
      before: { texts: [], coins: [] },
      after: { texts: [], coins: [{ coinType: 60, value: OWNER }] },
    })

    expect(start).toHaveBeenCalledTimes(1)
    const [transactionParam, signerParam] = start.mock.calls[0] ?? []

    const passedSigner = signerParam as RhinestoneSigner
    expect(passedSigner.session).toBeDefined()
    expect(passedSigner.session?.session.permissionId).toBe('0xsession123')

    const calls = (transactionParam as any).request.rhinestoneParams.calls
    expect(calls).toContainEqual(SET_RESOLVER_CALL)
    expect(calls).toContainEqual(DEPLOY_CALL)
  })
})
```

#### 3. Execute Verification Test

Run the test suite using pnpm:

```bash
pnpm --filter manager test src/features/profile/service/setupControlledResolver.flaw.test.ts
```

#### 4. Expected Output

The test executes directly against the project's native Vitest configuration and passes 100%:

```
✓ src/features/profile/service/setupControlledResolver.flaw.test.ts (2 tests)
  ✓ setupControlledResolver Vulnerability & Defect Verification (2)
    ✓ FLAW 1 (Missing Intent Funding): builds an intent lacking auxiliaryFunds and funding permit calls
    ✓ FLAW 2 (Session Allowlist Violation): passes active session key to transactionManager without stripping it

Test Files  1 passed (1)
     Tests  2 passed (2)
```
