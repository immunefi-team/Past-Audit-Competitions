# 92597 w a high hca registration checkout can display exact rent while a recovered quote authorizes a larger wallet debit

**Submitted on Sep 10th 2026 at 20:35:04 UTC by @alpha064 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92597
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

In the ENS Manager registration flow, a user can be shown an exact checkout total equal to the name rent while the subsequent HCA funding permit and commit batch are constructed to debit the wallet for the rent plus the live Rhinestone execution budget. The demonstrated sequence is: the checkout budget query fails, the click-time re-fetch also fails and is swallowed, and the registration state machine then obtains a fresh successful quote. For a standalone-HCA registration with an 8.000021 USDC rent, the checkout displays `$8.00`, while the reproduced state machine sizes a permit at `20,196,054` USDC base units and the real commit actor encodes `transferFrom(wallet, HCA, 20,196,054)`. The PoC does not broadcast this batch.

This is a payment-integrity defect affecting Alice's own authenticated registration, not an attacker taking control of another user's account. No attacker-controlled trigger, wallet compromise, protocol break, or third-party wallet behaviour is required or demonstrated. The trigger is a controlled timing condition in which the UI quote and click-time quote fail, then the state-machine quote succeeds before the user signs the separate permit.

The assessed code is the unversioned ENS Audit Competition snapshot identified by the repository as audited tree `cda79ac`, which forks upstream commit `63772fd872af472ced58b009499355f3430c2a86`. No public release mapping or verified first affected/fixed release was available, so this report does not claim a release range. The recommended severity is **Critical**, matching the program's listed impact for showing the wrong amount on a transaction the user approves; the QA-03 text expressly distinguishes a display-only mismatch from a mismatch with the amount charged on-chain. This classification remains triage-sensitive because the timing window depends on transient quote recovery, the PoC verifies the encoded debit offline rather than broadcasting it, and the user signs a separate permit.

I reviewed the pinned source, ran the existing targeted tests, and ran the supplied offline integration PoC in a disposable repository copy. The PoC passed both its positive case and persistent-failure negative control. It exercised the real checkout component, registration machine, permit sizing, and commit-batch encoder with only external boundaries mocked.

## Vulnerability Details

### Background

The standalone-HCA route uses a user-owned smart account (the HCA) for ENS registration. The registration budget consists of the registration rent plus the two quoted Rhinestone execution legs. If the HCA already holds USDC, the wallet should fund only the shortfall; with a zero HCA balance, the wallet must fund the complete budget.

The checkout has two different responsibilities: it displays the amount the user is approving and it performs an affordability check. The source currently treats a missing budget as a reason to show the rent alone and let the machine decide later. The relevant query says:

```ts
// callers treat "no budget" as "show the price alone and let the machine
// surface any failure", never as a hard block.
retry: 1,
```

This is in `apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts`, in the audited snapshot described above. The route and exact deployment prevalence are not established by this report.

The failure path begins when the HCA budget query returns an error after its configured retry. `TokenPickerContent` computes the displayed total as `funding?.total ?? pricingData`; when no budget exists, `pricingData` is the rent. It also passes `!!funding` as the estimate flag. `PaymentTotalRow` therefore omits the “up to” wording when the fallback is active. The network-cost row is rendered only when funding exists or a quote is still loading. In the same function, the checkout affordability amount is also `funding?.walletDebit ?? pricingData`, so a wallet holding exactly the rent can pass this UI gate.

The decisive excerpts are in `apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx` and `PaymentTotalRow.tsx`:

```ts
const requiredAmount = funding?.walletDebit ?? pricingData
const displayTotal = funding?.total ?? pricingData
<PaymentTotalRow isEstimate={!!funding} total={displayTotal} />
```

On the register click, the component calls `fetchQuery(budgetQueryOptions).catch(() => null)`. With another quote failure, `walletDebitRaw` is `null`, the insufficient-funding check is skipped, and `registration.start` is dispatched. This does not itself debit funds; it permits the state machine to continue with a checkout that showed the rent-only fallback.

The state machine then makes an independent estimate in `packages/transaction-manager/src/machines/registration/registration.machine.ts`, state `computingHcaBudget`. Unlike the checkout fallback, an estimator error enters `error`. When the transient failure has cleared, the successful quote is assigned to `context.hcaBudget`. In `signingFundingPermit`, the permit value is calculated as:

```ts
value: bigintMax(
  context.hcaBudget! - (context.hcaUsdcBalance ?? 0n),
  0n,
)
```

`packages/smart-account/src/providers/rhinestone/budget.ts` sums `commitCost + registerCost + registrationPrice`. With a zero HCA balance, this value is the full budget. The supplied test uses a rent of `8,000,021` and a recovered budget of `20,196,054` base units, both matching values documented by the audited source fixtures.

The separate wallet signature is an EIP-2612 permit. In `packages/transaction-manager/src/machines/registration/registration.hca.actors.ts`, `submitFundingAndCommitActor` adds a USDC `permit` call followed by `transferFrom(input.permit.owner, input.hca, input.permit.value)` to the batch. Thus, if the constructed batch is submitted and succeeds, the same permit value produced by the machine is debited from Alice's wallet into Alice's HCA. The later registration then spends the rent and execution-leg costs according to the batch; any unused HCA balance remains user-owned. This report does not claim that the remainder is locked, withdrawable, or recoverable in every environment.

The post-registration display also derives `Total Paid` from the rent calculation in `apps/manager/src/features/register-v2/workflow/registering/components/RegistrationDetails.tsx`, rather than from the funding debit. That is a secondary confirmation of the same amount-binding problem, not a separate attack.

The exact source history available in the competition checkout contains the audited fork snapshot and no verified public release tags or fixing release for these paths. Consequently, the earliest affected release, affected range, and fixed release are unknown. The report is tied to the inspected audit snapshot, not to an invented product version range.

### Remediation

On the HCA route, a missing or stale budget must never be represented as an exact rent-only total. The safest fix is to keep the CTA blocked, or display “Total unavailable — network cost could not be quoted” with an explicit retry, until the budget used for the approval is available. The click-time refetch should also fail closed for this payment-integrity check instead of converting the error to `null` and continuing.

The amount shown before signing should be the same shortfall value that the machine will put in the permit, expressed in USDC units. The machine should carry one verified budget through the quote, display, permit, and batch stages, or invalidate the display whenever it re-quotes. The final registration details should report the actual wallet debit separately from the rent, rather than labeling rent alone as `Total Paid`.

Regression coverage should exercise: a failed initial quote, a failed click-time quote, a later successful machine quote, a zero HCA balance, a part-funded HCA, and a persistent machine quote failure. Tests should assert that no permit or `transferFrom` is produced for the persistent-failure case and that the displayed amount equals the permit shortfall for the successful case.

## Impact Details

The demonstrated primitive is an incorrect human-facing checkout amount followed by construction of a larger wallet-to-HCA debit in the normal registration flow. Alice must use the standalone-HCA route, have the relevant token selected, encounter a failed checkout quote and failed click-time quote, then have the state-machine quote succeed. She must also approve the separate permit. The PoC controls these events deterministically; it does not broadcast the batch, establish that a remote attacker can cause the quote failures, or establish that this sequence is reliable in production.

The important negative control is the state machine's own estimator error path: if the recovered quote does not succeed, the machine enters `error` before permit signing and batch submission. This rules out a claim that a persistent outage alone causes an automatic debit. The HCA balance is user-owned, so there is no demonstrated cross-user theft or attacker profit. The execution-leg portion can be consumed by the registration operation, but the exact economic outcome depends on the quoted batch and is not broadened here.

The program impact category concerning an incorrect amount on a transaction the user approves is the recommended Critical selection, and the program's QA-03 note makes an on-chain charge mismatch distinct from display-only QA-03. Acceptance at that severity is nevertheless not guaranteed: the trigger is not attacker-controlled, the timing is narrow, the PoC verifies the encoded debit without broadcasting it, and the user signs a separate permit whose raw base-unit value may be inspectable in some wallets. A downgrade or rejection based on exploitability and UX criteria remains plausible.

### Concise impact summary

In the audited ENS Manager snapshot, standalone-HCA checkout can show Alice an exact `$8.00` rent-only total after budget-query failure, while a recovered state-machine quote sizes a separate EIP-2612 permit and session-signed batch to debit `20.196054` USDC from her wallet into her HCA. The proof covers the wrong displayed amount and the encoded debit, not a broadcast transaction, attacker control, production reliability, wallet rendering, or cross-user theft.

No public release history was available to establish affected or fixed releases beyond the inspected unversioned audit snapshot. The report maps directly to the program's wrong-amount-on-an-approved-transaction impact language and the QA-03 on-chain-charge carve-out, while acknowledging the narrow timing window and separate user signature that may affect triage.

## References

* Audited asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* Checkout budget query: `apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts`
* Checkout total and click-time refetch: `apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx`
* Total-row rendering: `apps/manager/src/features/register-v2/workflow/pricing/components/PaymentTotalRow.tsx`
* Registration budget and permit sizing: `packages/transaction-manager/src/machines/registration/registration.machine.ts`
* Funding batch encoder: `packages/transaction-manager/src/machines/registration/registration.hca.actors.ts`
* Budget sum: `packages/smart-account/src/providers/rhinestone/budget.ts`

## Proof of Concept

The distributable PoC is `poc/f1-checkout-total-mismatch.poc.test.tsx`. It uses a disposable copy of the audited repository and no Sepolia, Rhinestone, wallet extension, WalletConnect, or third-party service. The test doubles control only external boundaries: pricing, availability, HCA balance, quote calls, permit signing, and submission. The actual Manager component, registration state machine, permit-value calculation, and commit-batch encoder remain in use.

From the PoC directory in a disposable repository copy, place the test at the repository-relative path shown below and run:

```sh
pnpm --filter manager exec vitest run src/features/register-v2/workflow/pricing/components/f1-checkout-total-mismatch.poc.test.tsx
```

The preserved run record reports:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

The positive test asserts that the settled checkout contains `$8.00`, contains no “up to” hedge, and has no network-cost row; that the click-time quote fails; that the real machine sizes the permit at `20_196_054`; and that the real commit actor encodes `transferFrom(wallet, HCA, 20_196_054)`. The negative control keeps the machine quote unavailable and asserts that the machine enters `error` without reading the HCA balance, requesting a permit signature, or constructing the submission batch. These are offline observations from the test run, not an on-chain transaction. No cleanup beyond deleting the disposable repository copy is required.

### Complete reproducible test source

```tsx
// Run from a disposable copy of the audited repository after placing this file at:
// apps/manager/src/features/register-v2/workflow/pricing/components/f1-checkout-total-mismatch.poc.test.tsx
// biome-ignore-all lint/suspicious/noExplicitAny: the PoC bridges a mocked UI event into the machine's typed event

import '@testing-library/jest-dom'
import { i18n } from '@lingui/core'
import { I18nProvider } from '@lingui/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { fireEvent, render, screen, waitFor } from '@testing-library/react'
import { errAsync, okAsync } from 'neverthrow'
import type { Address, Hex, PublicClient } from 'viem'
import { decodeFunctionData, parseAbi } from 'viem'
import { sepolia } from 'viem/chains'
import { describe, expect, it, vi } from 'vitest'
import { createActor, fromPromise, waitFor as waitForState } from 'xstate'

const mocks = vi.hoisted(() => ({
  checkoutEstimate: vi.fn(),
  checkoutHcaBalance: vi.fn(),
  navigate: vi.fn(),
  onUiSend: vi.fn(),
  startTransaction: vi.fn(() => 'poc-tx-1'),
}))

vi.mock('@xstate/react', () => ({
  useSelector: (_actor: unknown, selector: (state: unknown) => unknown) =>
    selector({
      context: { duration: 31_536_000, selectedToken: 'USDC' },
    }),
}))

vi.mock('@tanstack/react-router', async (importOriginal) => ({
  ...(await importOriginal<typeof import('@tanstack/react-router')>()),
  useNavigate: () => mocks.navigate,
}))

vi.mock('@/features/register-v2/state/registrationUi.context', () => ({
  useRegistrationV2Context: () => ({
    label: 'jeff',
    uiActor: { send: (event: unknown) => mocks.onUiSend(event) },
  }),
}))

const WALLET = '0x1111111111111111111111111111111111111111' as Address
const HCA = '0xaaaa000000000000000000000000000000000001' as Address
const BUDGET = 20_196_054n
const RENT = 8_000_021n

const SESSION_ENABLE = {
  enableData: { proof: 'poc' },
  permissionId: `0x${'33'.repeat(32)}` as Hex,
  sessionKey: '0x2222222222222222222222222222222222222222' as Address,
  validUntil: 2_000_000_000n,
}

const account = {
  accountAddress: HCA,
  ownerAddress: WALLET,
  signer: { type: 'rhinestone' as const },
  walletClient: {},
  getSessionEnablePayload: vi.fn(async () => SESSION_ENABLE),
  stablecoinBalances: [
    {
      address: '0x1c7d4b196cb0c7b01d743fbc6116a902379c7238',
      symbol: 'USDC',
      decimals: 6,
      balance: '50000000',
    },
  ],
  isLoadingBalances: false,
  isConnected: true,
}

vi.mock('@/lib/smart-account/SmartAccountContext', () => ({
  useSmartAccountContext: () => account,
}))

vi.mock('@/lib/wagmi', () => ({
  publicClient: { chain: sepolia },
}))

vi.mock('@/features/profile/service/profileReverseName', () => ({
  profileReverseNameQuery: () => ({
    queryKey: ['poc', 'primary-name'],
    queryFn: async () => null,
  }),
}))

vi.mock('@/features/shared/service/ownedNamesCount', () => ({
  ownedNamesCountQueryOptions: () => ({
    queryKey: ['poc', 'owned-count'],
    queryFn: async () => 0,
  }),
}))

vi.mock('@/features/register-v2/data/queries/availability.query', () => ({
  getRegistrationV2AvailabilityQueryOptions: () => ({
    queryKey: ['poc', 'availability'],
    queryFn: async () => ({ isAvailable: true }),
  }),
}))

vi.mock('@/features/register-v2/data/queries/pricing.query', () => ({
  getRegisterPriceQueryOptions: () => ({
    queryKey: ['poc', 'rent'],
    queryFn: async () => ({ basePrice: RENT, premium: 0n }),
  }),
}))

vi.mock(
  '@ens-apps/transaction-manager/machines/registration/registration.hca.actors',
  async (importOriginal) => ({
    ...(await importOriginal<
      typeof import('@ens-apps/transaction-manager/machines/registration/registration.hca.actors')
    >()),
    estimateHcaBudgetActor: (...args: unknown[]) =>
      mocks.checkoutEstimate(...args),
    readHcaUsdcBalanceActor: (...args: unknown[]) =>
      mocks.checkoutHcaBalance(...args),
  }),
)

vi.mock('@ens-apps/transaction-manager/providers/transactionManager', () => ({
  transactionManager: {
    startTransaction: (...args: unknown[]) => mocks.startTransaction(...args),
  },
}))

import type { PermitSignature } from '@ens-apps/transaction-manager/machines/registration/registration.actors'
import { submitFundingAndCommitActor } from '@ens-apps/transaction-manager/machines/registration/registration.hca.actors'
import { registrationMachine } from '@ens-apps/transaction-manager/machines/registration/registration.machine'
import type { Signer } from '@ens-apps/transaction-manager/types/signer.types'
import type { RhinestoneTransactionRequest } from '@ens-apps/transaction-manager/types/transaction.types'
import { TokenPickerContent } from './TokenPickerContent'

const erc20Abi = parseAbi([
  'function permit(address owner, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s)',
  'function transferFrom(address from, address to, uint256 amount) returns (bool)',
])

i18n.loadAndActivate({ locale: 'en', messages: {} })

describe('F1 PoC — exact checkout total differs from HCA wallet debit', () => {
  it('shows $8.00, then signs and transfers 20.196054 USDC', async () => {
    // Both the initial query and the click-time retry fail. The registration
    // machine is allowed to recover independently immediately afterwards.
    mocks.checkoutEstimate.mockImplementation(() =>
      errAsync(new Error('scripted orchestrator outage')),
    )
    mocks.checkoutHcaBalance.mockReturnValue(okAsync(0n))

    const machineEstimate = vi.fn(async () => ({
      total: BUDGET,
      commitCost: 4_000_000n,
      registerCost: 8_196_033n,
      registrationPrice: RENT,
      source: 'quote' as const,
    }))
    const readMachineHcaBalance = vi.fn(async () => 0n)
    const signFundingPermit = vi.fn(
      async ({ input }: { input: { value: bigint } }) =>
        ({
          owner: WALLET,
          spender: HCA,
          value: input.value,
          deadline: 1_800_000_000n,
          v: 27,
          r: `0x${'11'.repeat(32)}` as Hex,
          s: `0x${'22'.repeat(32)}` as Hex,
        }) satisfies PermitSignature,
    )
    const submitFundingAndCommit = vi.fn(() => new Promise(() => {}))

    mocks.onUiSend.mockImplementation((event: any) => {
      if (event.type !== 'registration.start') return
      const actor = createActor(
        registrationMachine.provide({
          actors: {
            estimateHcaBudget: fromPromise(machineEstimate) as never,
            readHcaUsdcBalance: fromPromise(readMachineHcaBalance) as never,
            signFundingPermit: fromPromise(signFundingPermit) as never,
            submitFundingAndCommit: fromPromise(
              submitFundingAndCommit,
            ) as never,
          },
        }),
        { input: { chainId: sepolia.id } },
      )
      actor.start()
      actor.send({
        type: 'START_REGISTRATION',
        name: `${event.label}.eth`,
        duration: event.duration,
        token: event.token,
        price: event.totalPrice,
        signer: event.account.signer as Signer,
        approvalSigner: {
          type: 'eoa',
          walletClient: event.account.walletClient,
        } as Signer,
        accountAddress: HCA,
        ownerAddress: WALLET,
        resolverOwnerAddress: WALLET,
        publicClient: { chain: sepolia } as unknown as PublicClient,
        hcaSessionEnable: event.hcaSessionEnable,
      })
      void waitForState(actor, (state) =>
        state.matches('submittingSetupBundle'),
      )
    })

    const queryClient = new QueryClient({
      defaultOptions: {
        queries: { retryDelay: 0 },
        mutations: { retry: false },
      },
    })
    render(
      <QueryClientProvider client={queryClient}>
        <I18nProvider i18n={i18n}>
          <TokenPickerContent />
        </I18nProvider>
      </QueryClientProvider>,
    )

    // Wait until the initial quote and its configured retry have both failed.
    await waitFor(() =>
      expect(mocks.checkoutEstimate.mock.calls.length).toBeGreaterThanOrEqual(
        2,
      ),
    )
    await waitFor(() =>
      expect(screen.queryByText('Network cost')).not.toBeInTheDocument(),
    )
    const failuresBeforeClick = mocks.checkoutEstimate.mock.calls.length
    expect(screen.getByText('$8.00')).toBeInTheDocument()
    expect(screen.queryByText('up to')).not.toBeInTheDocument()
    const button = screen.getByRole('button', { name: 'Register name' })
    expect(button).toBeEnabled()

    fireEvent.click(button)

    await waitFor(() =>
      expect(mocks.checkoutEstimate.mock.calls.length).toBeGreaterThan(
        failuresBeforeClick,
      ),
    )
    await waitFor(() => expect(signFundingPermit).toHaveBeenCalledOnce())
    expect(signFundingPermit.mock.calls[0][0].input.value).toBe(BUDGET)
    await waitFor(() => expect(submitFundingAndCommit).toHaveBeenCalledOnce())
    const permit = submitFundingAndCommit.mock.calls[0][0].input
      .permit as PermitSignature
    expect(permit.value).toBe(BUDGET)

    // Feed the exact permit produced by the real machine into the real commit
    // actor and decode the resulting session-signed batch.
    mocks.startTransaction.mockClear()
    const commitment = `0x${'cc'.repeat(32)}` as Hex
    const commitClient = {
      chain: sepolia,
      readContract: vi.fn().mockResolvedValue(commitment),
    } as unknown as PublicClient
    const result = await submitFundingAndCommitActor({
      name: 'jeff.eth',
      wallet: WALLET,
      hca: HCA,
      duration: 31_536_000n,
      permit,
      sessionEnable: SESSION_ENABLE as never,
      signer: { type: 'rhinestone' } as Signer,
      publicClient: commitClient,
    })
    expect(result.isOk()).toBe(true)

    const [intent] = mocks.startTransaction.mock.calls[0] as unknown as [
      { request: RhinestoneTransactionRequest },
    ]
    const transferCall = intent.request.rhinestoneParams.calls[1]
    const transfer = decodeFunctionData({
      abi: erc20Abi,
      data: transferCall.data,
    })
    expect(transfer.functionName).toBe('transferFrom')
    expect(transfer.args[0].toLowerCase()).toBe(WALLET.toLowerCase())
    expect(transfer.args[1].toLowerCase()).toBe(HCA.toLowerCase())
    expect(transfer.args[2]).toBe(BUDGET)
  })

  it('negative control: a persistent machine quote failure stops before permit or debit', async () => {
    const persistentFailure = vi.fn(async () => {
      throw new Error('scripted persistent orchestrator outage')
    })
    const readBalance = vi.fn(async () => 0n)
    const signPermit = vi.fn()
    const submitBundle = vi.fn()

    const actor = createActor(
      registrationMachine.provide({
        actors: {
          estimateHcaBudget: fromPromise(persistentFailure) as never,
          readHcaUsdcBalance: fromPromise(readBalance) as never,
          signFundingPermit: fromPromise(signPermit) as never,
          submitFundingAndCommit: fromPromise(submitBundle) as never,
        },
      }),
      { input: { chainId: sepolia.id } },
    )
    actor.start()
    actor.send({
      type: 'START_REGISTRATION',
      name: 'jeff.eth',
      duration: 31_536_000n,
      token: 'USDC',
      price: RENT,
      signer: { type: 'rhinestone' } as Signer,
      approvalSigner: {
        type: 'eoa',
        walletClient: account.walletClient,
      } as Signer,
      accountAddress: HCA,
      ownerAddress: WALLET,
      resolverOwnerAddress: WALLET,
      publicClient: { chain: sepolia } as unknown as PublicClient,
      hcaSessionEnable: SESSION_ENABLE as never,
    })

    await waitForState(actor, (state) => state.matches('error'))
    expect(persistentFailure).toHaveBeenCalledOnce()
    expect(readBalance).not.toHaveBeenCalled()
    expect(signPermit).not.toHaveBeenCalled()
    expect(submitBundle).not.toHaveBeenCalled()
    actor.stop()
  })
})
```
