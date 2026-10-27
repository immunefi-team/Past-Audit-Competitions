# 89727 w a high manager checkout does not bind the accepted usdc amount to the later permit and transfer

## #89727 \[W\&A-High] Manager checkout does not bind the accepted USDC amount to the later Permit and transfer

**Submitted on Aug 20th 2026 at 19:36:08 UTC by @rtw0x1 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89727
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

### Brief/Intro

In the standalone-HCA registration flow, ENS Manager can display an exact `8.00 USDC` checkout total and allow the user to continue after its funding quote is unavailable. The accepted amount is not carried into either `registration.start` or the downstream `START_REGISTRATION` event. The transaction manager then independently re-estimates the budget and can construct a `20.20 USDC` EIP-2612 Permit and ERC-20 `transferFrom` without returning the changed amount to an ENS confirmation screen.

This breaks the payment-consent invariant between the Manager checkout and the later transaction-construction path. Impact is Low because MetaMask still displays the changed numeric cap (`20.2`) and requires explicit confirmation before signing.

### Vulnerability Details

* Affected asset: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager`
* Supporting affected component: `packages/transaction-manager`
* Frozen version: commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` on `audit-comp-ready`
* Prerequisite: the ordinary standalone-HCA USDC registration path, with a checkout funding-quote failure or material quote drift followed by a larger successful runtime estimate

The root cause spans four target-owned steps:

1. `TokenPickerContent.tsx` deliberately falls back to `pricingData` when the funding quote is absent. That makes the displayed total and balance gate use the rent alone. The Register button remains available.
2. The click-time funding fetch is used only for a balance check. Its budget is not included when `registration.start` is dispatched.
3. `registrationUi.machine.ts` forwards `START_REGISTRATION` without `hcaBudget`.
4. `registration.machine.ts` sees an undefined `hcaBudget`, invokes `estimateHcaBudget`, and uses the new value for `signFundingPermitActor`. `submitFundingAndCommitActor` then constructs `permit(..., value)` and `transferFrom(..., value)` with that runtime value.

The two included tests are intentionally paired rather than presented as one monolithic UI-to-chain test. The first executes the real Manager checkout component and proves the exact `8.00` display and continuation state when the funding quote is absent. The second executes the real registration machine, Permit actor, funding-bundle builder, and ABI decoding. It proves the omitted accepted budget allows a deterministic runtime budget of `20.20` to become both `Permit.value` and `transferFrom.amount`.

The `20.20` runtime quote is a deterministic fixture used to exercise the target code path. It proves target-side consumption and field construction, while quote-service behavior and on-chain settlement remain outside the proof. The source itself documents that an 8.00 USDC Sepolia registration has reached approximately 20.20 USDC after the two execution legs.

### Impact Details

Websites and Applications — Low: the application does not preserve the amount the user accepted at checkout across its payment state machines. In the reproduced sequence, the ENS checkout shows `8.00`, while the later target code constructs authorization and transfer calldata for `20.20`. The `12.20` difference equals the modeled `4.00` commit-leg cost plus `8.20` register-leg cost.

The wallet is an important mitigation: MetaMask 13.41.0 visibly changes the spending cap from `8` in the control to `20.2` in the affected request and requires the user to press Confirm. The prompt identifies the token by contract address rather than displaying the `USDC` symbol. Both capture requests were cancelled, limiting the demonstrated impact to the application-side consent discontinuity before an explicit wallet confirmation.

The program's QA-03 known issue is limited to a display-only mismatch in Explorer and expressly distinguishes a mismatch that reaches the charged amount as a new finding. This report has a different Manager root and reaches the Permit and transfer construction boundary.

### Remediation

Treat the amount accepted on the checkout screen as an explicit authorization bound:

1. Carry the exact accepted `hcaBudget` (or maximum wallet debit plus quote identity/expiry) through `registration.start` and `START_REGISTRATION`.
2. Before signing the Permit, compare the fresh runtime budget against that bound.
3. If the amount increased, return to an ENS confirmation screen that clearly shows the new total and execution-cost breakdown. Do not rely on the wallet as the only reconfirmation layer.
4. If a trustworthy funding quote is unavailable, block continuation or label the amount as incomplete and require a successful quote before accepting checkout.

### References

* Frozen source commit: `https://github.com/immunefi-team/audit-comp-ens/commit/1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* Checkout fallback and dispatch: `https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx`
* UI-to-registration transition: `https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/state/registrationUi.machine.ts`
* Runtime budget machine: `https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts`
* Permit and transfer construction: `https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts`

### Proof of Concept

## Proof of Concept

### Prerequisites

* Node.js compatible with the repository
* Corepack and pnpm 10.27.0
* Git
* No wallet funds, live ENS deployment, public Sepolia RPC, or quote-service connection is needed for the executable proof

### Steps To Reproduce

1. Clone the exact frozen source and install dependencies.

   ```bash
   git clone https://github.com/immunefi-team/audit-comp-ens.git
   cd audit-comp-ens
   git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
   corepack enable
   corepack pnpm install --frozen-lockfile
   ```

   Expected: `git rev-parse HEAD` returns `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
2. Save the following as `apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.quote-failure.poc.test.tsx`:

   ```tsx
   import '@testing-library/jest-dom'
   import { i18n } from '@lingui/core'
   import { I18nProvider } from '@lingui/react'
   import { fireEvent, render, screen } from '@testing-library/react'
   import { describe, expect, it, vi } from 'vitest'
   import type { StablecoinBalance } from '@/lib/smart-account'
   import { TokenPickerContentBase } from './TokenPickerContent'

   i18n.loadAndActivate({ locale: 'en', messages: {} })

   const tenUsdc = {
     address: '0x1c7d4b196cb0c7b01d743fbc6116a902379c7238',
     symbol: 'USDC',
     decimals: 6,
     balance: '10000000',
   } as unknown as StablecoinBalance

   describe('HCA quote-failure checkout PoC', () => {
     it('shows rent as the exact total and enables registration without a budget', () => {
       const onNext = vi.fn()

       render(
         <I18nProvider i18n={i18n}>
           <TokenPickerContentBase
             isConnected
             isLoadingBalances={false}
             isQuotingFunding={false}
             label="myname"
             onNext={onNext}
             onSelectCoin={() => {}}
             pricingData={8}
             pricingLoading={false}
             selectedToken="USDC"
             stablecoinBalances={[tenUsdc]}
           />
         </I18nProvider>,
       )

       expect(screen.getByText('$8.00')).toBeInTheDocument()
       expect(screen.queryByText('up to')).not.toBeInTheDocument()

       const register = screen.getByRole('button', { name: 'Register name' })
       expect(register).toBeEnabled()
       fireEvent.click(register)
       expect(onNext).toHaveBeenCalledOnce()
     })
   })
   ```
3. Run the Manager control:

   ```bash
   corepack pnpm --filter manager exec vitest run src/features/register-v2/workflow/pricing/components/TokenPickerContent.quote-failure.poc.test.tsx --reporter=dot
   ```

   Expected: one test file and one test pass. The real checkout component renders `$8.00`, does not render `up to`, enables Register, and advances despite having no accepted funding budget.
4. Save the following as `packages/transaction-manager/src/machines/registration/hca-checkout-permit-accounting.poc.test.ts`:

   ```ts
   // biome-ignore-all lint/suspicious/noExplicitAny: decoded ABI args are intentionally inspected

   import {
     getDestinationContracts,
     type HcaBudgetBreakdown,
   } from '@ens-apps/smart-account'
   import type { Address, Hex, PublicClient } from 'viem'
   import { decodeFunctionData, parseAbi } from 'viem'
   import { sepolia } from 'viem/chains'
   import { describe, expect, it, vi } from 'vitest'
   import { createActor, fromPromise, waitFor } from 'xstate'
   import type { Signer } from '../../types/signer.types'
   import type { RhinestoneTransactionRequest } from '../../types/transaction.types'
   import {
     signFundingPermitActor,
     submitFundingAndCommitActor,
     type HcaSessionEnableParams,
   } from './registration.hca.actors'
   import { registrationMachine } from './registration.machine'

   const startTransaction = vi.fn(() => 'tx-poc')
   vi.mock('../../providers/transactionManager', () => ({
     transactionManager: {
       startTransaction: (...args: unknown[]) =>
         (startTransaction as unknown as (...values: unknown[]) => string)(...args),
     },
   }))

   const readContract = vi.fn()
   const signTypedData = vi.fn()
   const getEip712Domain = vi.fn()
   vi.mock('viem/actions', async (importOriginal) => ({
     ...(await importOriginal<typeof import('viem/actions')>()),
     readContract: (...args: unknown[]) => readContract(...args),
     signTypedData: (...args: unknown[]) => signTypedData(...args),
     getEip712Domain: (...args: unknown[]) => getEip712Domain(...args),
   }))

   const C = getDestinationContracts(sepolia.id)
   const WALLET = '0x1111111111111111111111111111111111111111' as Address
   const HCA = '0xaaaa000000000000000000000000000000000001' as Address
   const CHECKOUT_DISPLAYED = 8_000_000n
   const RUNTIME_BUDGET = 20_200_000n
   const COMMIT_COST = 4_000_000n
   const REGISTER_COST = 8_200_000n
   const REGISTRATION_PRICE = 8_000_000n
   const COMMITMENT = `0x${'cc'.repeat(32)}` as Hex

   const erc2612Abi = parseAbi([
     'function permit(address owner, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s)',
     'function transferFrom(address from, address to, uint256 amount) returns (bool)',
   ])

   const SESSION_ENABLE = {
     enableData: { poc: 'enable-data' },
     permissionId: `0x${'33'.repeat(32)}`,
     sessionKey: '0x2222222222222222222222222222222222222222',
     validUntil: 2_000_000_000n,
   } as unknown as HcaSessionEnableParams

   const walletClient = {
     account: { address: WALLET },
   } as unknown as Extract<Signer, { type: 'eoa' }>['walletClient']

   const approvalSigner = {
     type: 'eoa',
     walletClient,
   } as Extract<Signer, { type: 'eoa' }>

   const commitClient = {
     chain: sepolia,
     readContract: vi.fn().mockResolvedValue(COMMITMENT),
   } as unknown as PublicClient

   const rhinestoneSigner = { type: 'rhinestone' } as unknown as Signer

   describe('checkout -> permit -> transfer accounting PoC', () => {
     it('consumes the entire 12.20 USDC checkout delta as the two execution legs', async () => {
       vi.clearAllMocks()
       readContract.mockReset()
       getEip712Domain.mockReset()

       const runtimeBreakdown: HcaBudgetBreakdown = {
         total: RUNTIME_BUDGET,
         commitCost: COMMIT_COST,
         registerCost: REGISTER_COST,
         registrationPrice: REGISTRATION_PRICE,
         source: 'quote',
       }

       readContract
         .mockResolvedValueOnce(7n)
         .mockResolvedValueOnce(100_000_000n)
         .mockResolvedValue(COMMITMENT)
       getEip712Domain.mockResolvedValue({
         domain: {
           name: 'USDC',
           version: '2',
           chainId: sepolia.id,
           verifyingContract: C.usdc,
         },
       })
       signTypedData.mockResolvedValue(
         `0x${'11'.repeat(32)}${'22'.repeat(32)}1b` as Hex,
       )

       let submittedInput:
         | Parameters<typeof submitFundingAndCommitActor>[0]
         | undefined
       const actor = createActor(
         registrationMachine.provide({
           actors: {
             estimateHcaBudget: fromPromise(async () => runtimeBreakdown) as never,
             readHcaUsdcBalance: fromPromise(async () => 0n) as never,
             signFundingPermit: fromPromise(async ({ input }) => {
               const result = await signFundingPermitActor(input as never)
               return result._unsafeUnwrap()
             }) as never,
             submitFundingAndCommit: fromPromise(async ({ input }) => {
               submittedInput = input as Parameters<
                 typeof submitFundingAndCommitActor
               >[0]
               await new Promise<never>(() => undefined)
             }) as never,
           },
         }),
         { input: { chainId: sepolia.id } },
       )

       const startRegistration = {
         type: 'START_REGISTRATION' as const,
         name: 'myname.eth',
         duration: 31_536_000n,
         token: 'USDC' as const,
         price: REGISTRATION_PRICE,
         signer: rhinestoneSigner,
         approvalSigner,
         accountAddress: HCA,
         ownerAddress: WALLET,
         publicClient: commitClient,
         hcaSessionEnable: SESSION_ENABLE,
       }

       expect(Object.hasOwn(startRegistration, 'hcaBudget')).toBe(false)

       actor.start()
       actor.send(startRegistration)
       await waitFor(actor, (snapshot) => snapshot.matches('submittingSetupBundle'))
       expect(submittedInput).toBeDefined()

       const submitResult = await submitFundingAndCommitActor(submittedInput!)
       expect(submitResult.isOk()).toBe(true)

       const typedData = signTypedData.mock.calls[0][1]
       expect(typedData.primaryType).toBe('Permit')
       expect(typedData.domain).toMatchObject({
         name: 'USDC',
         version: '2',
         chainId: sepolia.id,
         verifyingContract: C.usdc,
       })
       expect(typedData.message).toMatchObject({
         owner: WALLET,
         spender: HCA,
         value: RUNTIME_BUDGET,
         nonce: 7n,
       })

       const [intent] = startTransaction.mock.calls[0] as unknown as [
         { request: RhinestoneTransactionRequest },
       ]
       const calls = intent.request.rhinestoneParams.calls
       const permitCall = decodeFunctionData({
         abi: erc2612Abi,
         data: calls[0].data,
       })
       const transferCall = decodeFunctionData({
         abi: erc2612Abi,
         data: calls[1].data,
       })

       expect(permitCall.functionName).toBe('permit')
       expect((permitCall.args as any)[2]).toBe(RUNTIME_BUDGET)
       expect(transferCall.functionName).toBe('transferFrom')
       expect((transferCall.args as any)[0].toLowerCase()).toBe(
         WALLET.toLowerCase(),
       )
       expect((transferCall.args as any)[1].toLowerCase()).toBe(HCA.toLowerCase())
       expect((transferCall.args as any)[2]).toBe(RUNTIME_BUDGET)

       const checkoutDelta = RUNTIME_BUDGET - CHECKOUT_DISPLAYED
       const executionLegsConsumed = COMMIT_COST + REGISTER_COST
       const hcaFinalBalance =
         RUNTIME_BUDGET -
         REGISTRATION_PRICE -
         COMMIT_COST -
         REGISTER_COST

       expect(runtimeBreakdown.total).toBe(
         REGISTRATION_PRICE + COMMIT_COST + REGISTER_COST,
       )
       expect(checkoutDelta).toBe(12_200_000n)
       expect(executionLegsConsumed).toBe(checkoutDelta)
       expect(hcaFinalBalance).toBe(0n)
       expect(actor.getSnapshot().context.hcaBudget).toBe(RUNTIME_BUDGET)
       expect(actor.getSnapshot().context.hcaBudgetBreakdown).toEqual(
         runtimeBreakdown,
       )

       console.log(
         JSON.stringify({
           checkoutDisplayedUsdc: '8.00',
           startRegistrationCarriesAcceptedBudget: false,
           runtimePermitValueUsdc: '20.20',
           transferFromUsdc: '20.20',
           registrationPriceUsdc: '8.00',
           commitLegCostUsdc: '4.00',
           registerLegCostUsdc: '8.20',
           checkoutDeltaUsdc: '12.20',
           executionLegsConsumedUsdc: '12.20',
           hcaFinalBalanceUsdc: '0.00',
         }),
       )

       actor.stop()
     }, 15_000)
   })
   ```
5. Run the transaction-construction proof:

   ```bash
   corepack pnpm --filter @ens-apps/transaction-manager exec vitest run src/machines/registration/hca-checkout-permit-accounting.poc.test.ts --reporter=dot
   ```

   Expected: one test file and one test pass, including:

   ```json
   {"checkoutDisplayedUsdc":"8.00","startRegistrationCarriesAcceptedBudget":false,"runtimePermitValueUsdc":"20.20","transferFromUsdc":"20.20","registrationPriceUsdc":"8.00","commitLegCostUsdc":"4.00","registerLegCostUsdc":"8.20","checkoutDeltaUsdc":"12.20","executionLegsConsumedUsdc":"12.20","hcaFinalBalanceUsdc":"0.00"}
   ```
6. Compare the raw MetaMask captures attached to this report:

   * `step-03-metamask-matched-8-spending-cap.png`: matched control, numeric spending cap `8`.
   * `step-04-metamask-drifted-20-2-spending-cap.png`: affected request, numeric spending cap `20.2`.

   Both captures use the exact audited manifest token address and actor-compatible EIP-712 domain on a local Sepolia-chain fixture. Both requests were cancelled. They prove wallet rendering and the explicit-confirmation mitigation; quote generation and settlement are represented only by the deterministic source reducers.

### Proof of Impact

| Run             | Security-relevant input/state                                                      | Observed result                                                                                                                              |
| --------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Matched control | Accepted checkout and Permit both use `8.00`                                       | MetaMask renders spending cap `8`; request cancelled                                                                                         |
| Affected path   | Checkout displays `8.00`; accepted budget omitted; runtime machine derives `20.20` | Real target actors construct `Permit.value=20.20` and `transferFrom.amount=20.20`; MetaMask renders spending cap `20.2` and requires Confirm |

The affected lifecycle is the normal standalone-HCA USDC registration path in the frozen competition source. The source reducers establish the target-owned amount discontinuity and exact calldata; the wallet captures establish the strongest user-side control. Together they support a Low transaction-consent integrity issue without overstating the result as a silent or completed loss.
