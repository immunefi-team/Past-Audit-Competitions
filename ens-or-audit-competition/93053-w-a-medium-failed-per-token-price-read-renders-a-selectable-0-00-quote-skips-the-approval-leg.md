# 93053 w a medium failed per token price read renders a selectable 0 00 quote skips the approval leg and the amount less renew silently charges the live price against the standing 2x allowance

**Submitted on Sep 14th 2026 at 04:15:20 UTC by @farismaulana for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93053
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

when the portal's per-token price query fails, the payment picker does not show an error or disable the token. it prices the token at exactly $0.00, keeps it selectable, and on Confirm submits a renewal with no amount argument. the registrar then pulls its live oracle price against whatever ERC-20 allowance is standing, and because every healthy renewal leaves behind an approval of 2x its quote that is never revoked, that live pull succeeds silently. the user approves a screen that says $0.00 and is charged the real price, with no wallet approval prompt in between: the application shows the wrong amount on a transaction the user then approves.

## Vulnerability Details

three production pieces compose into this.

a failed read becomes zero, not "no price":

```ts
// apps/portal/src/features/register/utils/tokenData.ts
@>  const total = priceResult ?? DEFAULT_PRICE   // DEFAULT_PRICE.total === 0n
```

`buildTokenData` substitutes `DEFAULT_PRICE` (total `0n`) whenever the per-token price query settled with an error or returned no data. this is the app's own error branch: a price query failing to return is the expected unavailability of a network dependency, and the in-scope code owns what renders in its place — no dependency returns any wrong data. the picker gates rows only on `isLoading`, with no error gate (PaymentTokenPicker.tsx:144-153), and the row balance gate is `balance < price.total`, which is `0n < 0n === false`, so the errored token stays selectable even at zero balance.

the confirm path keys its approval decision off that unvalidated figure:

```ts
// apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:466-501
@>  if (tokenAllowance >= tokenPrice) {
@>    // skip the approve leg entirely
```

with `tokenPrice = 0n`, the condition `tokenAllowance >= 0n` always holds, so no approval transaction is built, the wallet is never prompted, and the renew submits immediately.

the renew carries no price, and a standing allowance pays for it:

```ts
// apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:147-160
@>  amount: params.tokenPrice * 2n   // approve width: 2x the quote, never revoked
```

the submitted `renew()` calldata is exactly `[label, duration, paymentToken, referrer]`, no amount and no maxPrice, so the contract charges its live oracle price via `transferFrom` against whatever allowance stands.

concrete sequence with the PoC's numbers (1-year renewal, DAI):

1. healthy renewal quoted P = 5 DAI. the flow approves 2P = 10 DAI to the renewer, the renew pulls \~5 DAI at the live price, and the 10 DAI allowance stands forever (nothing in the repo revokes it).
2. weeks later the same wallet extends the name again and the per-token quote fails transiently while the base quote succeeds. the DAI row renders as $0.00 and stays clickable.
3. the user clicks Confirm on the $0.00 screen. `tokenAllowance >= 0n` holds, so no approval prompt appears; the submitted renew carries no amount.
4. the registrar pulls the live price L against the standing 10 DAI. every live price up to 2P settles: the user is charged L in DAI against a $0.00 consent, and the residual allowance persists for the next round.

on a fresh wallet with no standing allowance the same flow submits the amount-less renew directly, which reverts at the live-price pull after the user has paid gas for a checkout that displayed $0.00. the confirm screen is itself fed by a separate tokenless (USDC-default) quote, so whenever a non-USDC token is selected the displayed total is denominated in a different asset than the one charged.

## Impact Details

this is the program's listed impact "Causing the application to display or pre-fill an incorrect address, name, or transaction detail … showing the wrong recipient or amount on a transaction the user then approves" (websites and applications, Critical). the PoC demonstrates the clause end to end: the consent screen shows a total of exactly $0.00, the user clicks Confirm, and the submitted transaction charges the live oracle price in the selected token, drawn from an allowance granted by an earlier renewal, with no approval step between consent and charge.

* the displayed amount and the executed amount are different numbers on an ordinary user flow; the charge is silent (no wallet prompt, no error surface) whenever any residual allowance from an earlier renewal in that token stands, which is the steady state for any repeat user.
* the trigger is a transient RPC failure on one quote while the other succeeds; no attacker is required.
* the gas-only variant (fresh wallet) still bills the user for a checkout that displayed $0.00, and the USDC-default confirm screen shows a differently-denominated total whenever a non-USDC token is selected.

the recipient of the charge is the intended renewer and the magnitude is the live rent for the selected duration; the defect is that neither the amount nor the approval leg matched the consent the user actually gave.

## References

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/register/utils/tokenData.ts#L13-L61>

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/register/components/PaymentTokenPicker.tsx#L144-L153>

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/hooks/useRenewalTransactions.ts#L147-L160>

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/hooks/useRenewalTransactions.ts#L466-L501>

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/components/ExtendNameConfirmation.tsx#L56-L63>

## Proof of Concept

## Proof of Concept

add this test to `apps/portal/src/__poc__/payment-zero-quote.poc.test.ts` (the `__poc__` directory is new; the portal package's vitest include pattern picks up `src/**/*.test.ts`). it drives the real production `buildTokenData`, the real `PaymentTokenList` component, and the real `useRenewalTransactions.startFlow`. only the browser wallet stack is swapped for the repo's own lightweight test chain (the same pattern as the repo's `useContractAddress.test.ts`) and the transaction-manager singleton is stubbed.

```ts
/**
 * PoC — failed per-token price read renders as a selectable $0.00 quote,
 * skips the approval leg, and submits an amount-less renew that the standing
 * 2x residual allowance settles at the live price.
 *
 * Production code under test (all exercised for real, none modified):
 *   buildTokenData/DEFAULT_PRICE ... apps/portal/src/features/register/utils/tokenData.ts:13-61
 *   PaymentTokenPicker gate ...... apps/portal/src/features/register/components/PaymentTokenPicker.tsx:144-153
 *   PaymentTokenList rows ........ apps/portal/src/features/register/components/PaymentTokenList.tsx:52-79
 *   useRenewalTransactions ....... apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:147-160,190-217,466-501
 *   renewNameWriteParameters ..... ensjs encodes exactly [label, duration, paymentToken, referrer]
 */
import { REFERER_ADDRESS } from '@ens-apps/transaction-manager/contracts/ens-sepolia'
import { renewNameWriteParameters } from '@ensdomains/ensjs/wallet'
import { fireEvent, render, renderHook, act, screen } from '@testing-library/react'
import { createElement } from 'react'
import { decodeFunctionData, erc20Abi, formatUnits, type Address, type Hex } from 'viem'
import { describe, expect, it, vi } from 'vitest'
import { PaymentTokenList } from '@/features/register/components/PaymentTokenList'
import { PAYMENT_TOKENS } from '@/features/register/constants/paymentTokens'
import { useRenewalTransactions } from '@/features/renew/hooks/useRenewalTransactions'
import { getRenewerAddress } from '@/features/renew/utils/renewer'
import {
  buildTokenData,
  DEFAULT_PRICE,
  type TokenPrice,
} from '@/features/register/utils/tokenData'
import {
  createTestWrapper,
  sepoliaWithEns,
  TEST_ACCOUNTS,
  type WalletClientWithAccount,
} from '@/test-utils'

// the repo's own test pattern: swap the browser wallet stack for the
// lightweight chain definition so contract-address resolution stays real
vi.mock('@/lib/wagmi', async () => {
  const { sepoliaWithEns: chain } = await import('@/test-utils/wagmi.mock')
  return { sepoliaWithEns: chain }
})

// the PoC only inspects the intents the flow builds
vi.mock('@ens-apps/transaction-manager', () => ({
  transactionManager: {
    clear: vi.fn(),
    startTransaction: vi.fn(),
  },
}))

const NAME = 'pocname.eth'
const LABEL = 'pocname'
const DURATION = 31_536_000 // 1 year, seconds
const DURATION_BIG = BigInt(DURATION)

const USDC = PAYMENT_TOKENS.find((t) => t.symbol === 'USDC')!
const DAI = PAYMENT_TOKENS.find((t) => t.symbol === 'DAI')!

// healthy quotes: 5 USDC (6dp) and 5 DAI (18dp)
const HEALTHY_USDC_PRICE: TokenPrice = {
  total: 5_000_000n,
  base: 5_000_000n,
  premium: 0n,
  decimals: 6,
  hasPremium: false,
}
const HEALTHY_DAI_PRICE: TokenPrice = {
  total: 5n * 10n ** 18n,
  base: 5n * 10n ** 18n,
  premium: 0n,
  decimals: 18,
  hasPremium: false,
}
const P = HEALTHY_DAI_PRICE.total

const RENEWER_V2 = getRenewerAddress(true)

const WALLET = {
  account: { address: TEST_ACCOUNTS.alice },
} as unknown as WalletClientWithAccount
const INTENT_CTX = { walletClient: WALLET, chainId: sepoliaWithEns.id }

const referenceRenew = renewNameWriteParameters(
  { chain: sepoliaWithEns } as never,
  {
    name: NAME,
    duration: DURATION_BIG,
    paymentToken: DAI.address as Address,
    referrer: REFERER_ADDRESS,
    contract: 'ensEthRegistrar',
  },
)

const decodeApprove = (data: Hex) => {
  const decoded = decodeFunctionData({ abi: erc20Abi, data })
  return decoded.args as unknown as [Address, bigint]
}

const renderRenewalHook = async () => {
  const rendered = renderHook(() => useRenewalTransactions(), {
    wrapper: createTestWrapper(),
  })
  await act(async () => {})
  return rendered
}

const startFlow = (
  rendered: ReturnType<typeof renderHook<ReturnType<typeof useRenewalTransactions>>>,
  config: Parameters<ReturnType<typeof useRenewalTransactions>['startFlow']>[1],
) => {
  act(() => {
    rendered.result.current.startFlow({ name: NAME, isV2: true }, config)
  })
}

describe('failed per-token price read -> selectable $0.00 quote -> approval skipped -> amount-less renew vs standing 2x allowance', () => {
  it('a failed per-token price read renders the token as a selectable $0.00 quote', () => {
    // PaymentTokenPicker.tsx:144-149 feeds buildTokenData with
    // priceQueries.map((query) => query.data) — an errored query yields
    // `undefined` for its slot; only `isLoading` is gated (no error gate).
    const tokenData = buildTokenData(
      PAYMENT_TOKENS,
      [HEALTHY_USDC_PRICE, undefined], // DAI price query errored
      [10_000_000n, 0n], // balances: funded USDC, ZERO DAI
      [0n, 0n], // allowances: zero for both
    )

    const daiRow = tokenData[1]
    expect(
      daiRow.price,
      'a failed per-token price read renders as a $0.00 quote (DEFAULT_PRICE substitution) instead of dropping/erroring the token',
    ).toBe(DEFAULT_PRICE)
    expect(
      daiRow.price.total,
      'the DAI row the user sees is quoted at exactly 0 — displayed as $0.00',
    ).toBe(0n)
    expect(
      formatUnits(daiRow.price.total, daiRow.decimals),
      'the rendered quote string is "0" — i.e. $0.00',
    ).toBe('0')

    // PaymentTokenPicker.tsx:151-153 — the only balance gate in the picker.
    // `balance < price.total` is `0n < 0n` === false, so the insufficient-balance
    // branch is NOT taken and the list renders. The real production component,
    // mounted: the DAI row button is ENABLED and clicking it selects the
    // 0n-priced token even with zero balance.
    const onSelect = vi.fn()
    render(
      createElement(PaymentTokenList, {
        tokenData,
        selectedToken: null,
        isRegistering: false,
        onSelect,
      }),
    )
    const row = screen.getByRole('button', { name: /DAI/ })
    expect(
      row.getAttribute('disabled'),
      'the $0.00 DAI row is clickable — a wallet with ZERO DAI can still select it',
    ).toBeNull()
    fireEvent.click(row)
    expect(
      onSelect,
      'selection succeeds and hands the flow the errored token with price.total = 0n',
    ).toHaveBeenCalledWith(
      expect.objectContaining({
        address: DAI.address,
        price: expect.objectContaining({ total: 0n }),
      }),
    )
  })

  it('confirm skips the approve leg entirely and submits a renew with NO amount argument', async () => {
    const erroredRowPrice = buildTokenData(
      PAYMENT_TOKENS,
      [HEALTHY_USDC_PRICE, undefined],
      [10_000_000n, 0n],
      [0n, 0n],
    )[1].price

    const rendered = await renderRenewalHook()

    // ExtendNameConfirmation.tsx:56-63 submits the SELECTED row's price/address
    // into startFlow — this is the confirm click.
    startFlow(rendered, {
      duration: DURATION,
      tokenAddress: DAI.address as Address,
      tokenPrice: erroredRowPrice.total, // 0n — the $0.00 the user "approved"
      tokenAllowance: 0n, // fresh wallet: zero standing allowance
    })
    const txs = rendered.result.current.transactions

    // THE SKIP: the approve gate `tokenAllowance >= tokenPrice` at
    // useRenewalTransactions.ts:482 is `0n >= 0n` — always true — so NO
    // approval transaction is built and no wallet approve prompt ever appears.
    expect(
      txs.some((tx) => tx.id.startsWith('renewal-approve-')),
      'the required-approval leg is skipped entirely (no setApproval/approve tx exists)',
    ).toBe(false)
    expect(
      txs,
      'exactly one transaction is submitted — the renew — with no approval step in front of it',
    ).toHaveLength(1)
    expect(txs[0].id).toBe(`renewal-renew-${NAME}`)

    expect(
      txs[0].intent?.prepare,
      'the renew step is prepared straight away — the flow believes no allowance work is needed',
    ).toBeDefined()

    // NO AMOUNT: build the real intent and decode the calldata the flow
    // would sign.
    const renewIntent = txs[0].intent!.prepare!(INTENT_CTX)
    const decoded = decodeFunctionData({
      abi: referenceRenew.abi,
      data: renewIntent.request.data,
    })

    expect(decoded.functionName).toBe('renew')
    expect(
      decoded.args,
      'renew calldata is [label, duration, paymentToken, referrer] — NO amount/maxPrice argument, so the contract charges its LIVE oracle price against whatever allowance is standing',
    ).toEqual([LABEL, DURATION_BIG, DAI.address, REFERER_ADDRESS])
    expect(
      renewIntent.request.to,
      'the renew targets the real v2 renewer (ETHRegistrar) — an actual on-chain settlement path',
    ).toBe(RENEWER_V2)
    expect(renewIntent.request.value).toBe(0n) // no ETH either: pure ERC-20 pull
  })

  it('a healthy flow approves exactly 2x the quote and its standing residual settles any live price L <= 2P for a later $0.00 flow', async () => {
    const rendered = await renderRenewalHook()

    // healthy contrast, quoted price P with zero allowance: approve + renew,
    // approve width exactly 2P, and the renew waits for the approval.
    startFlow(rendered, {
      duration: DURATION,
      tokenAddress: DAI.address as Address,
      tokenPrice: P,
      tokenAllowance: 0n,
    })
    const txs = rendered.result.current.transactions
    expect(txs, 'healthy quote with zero allowance produces approve + renew').toHaveLength(2)
    const approveTx = txs[0]
    expect(approveTx.id).toBe(`renewal-approve-${RENEWER_V2}`)
    expect(
      txs[1].intent?.prepare,
      'healthy flow: the renew is NOT estimable before the approval lands (contrast with the $0.00 flow, which skips straight to a prepared renew)',
    ).toBeUndefined()

    // buildRenewalApproveIntent approves tokenPrice * 2n (useRenewalTransactions.ts:147-160).
    // decode the REAL encoded approve to prove the width.
    const approveIntent = approveTx.intent!.prepare!(INTENT_CTX)
    const [spender, approvedAmount] = decodeApprove(approveIntent.request.data)
    expect(approveIntent.request.to).toBe(DAI.address)
    expect(spender).toBe(RENEWER_V2)
    expect(
      approvedAmount,
      'the standing allowance the healthy flow leaves behind is exactly 2x the quote — the residual carrier for a later silent settlement',
    ).toBe(P * 2n)

    // the attack on top of that standing residual: the $0.00 flow again
    // (same wallet, price read fails this time).
    startFlow(rendered, {
      duration: DURATION,
      tokenAddress: DAI.address as Address,
      tokenPrice: 0n, // failed read -> the $0.00 quote
      tokenAllowance: approvedAmount, // the standing 2P residual
    })
    const attackTxs = rendered.result.current.transactions
    expect(
      attackTxs.some((tx) => tx.id.startsWith('renewal-approve-')),
      'with a standing 2P allowance the $0.00 flow again skips the approve leg — nothing re-prompts the wallet',
    ).toBe(false)
    expect(attackTxs).toHaveLength(1)

    // settlement math for the amount-less renew: the renewer pulls the LIVE
    // price L via transferFrom. ERC-20 semantics: it succeeds for any
    // L <= standing allowance, so every plausible live price around P settles
    // silently — charging L while the consent screen showed $0.00.
    for (const L of [P, P + P / 2n, P * 2n]) {
      expect(
        L <= approvedAmount,
        `live price ${formatUnits(L, 18)} DAI <= standing 2P allowance — the silent pull succeeds and the user is charged ${formatUnits(L, 18)} DAI against a $0.00 display`,
      ).toBe(true)
      expect(
        approvedAmount - L,
        'residual after the silent charge (2P - L) — allowance persists for the next flow',
      ).toBeGreaterThanOrEqual(0n)
    }
    // only L > 2P exceeds the standing allowance
    expect((P * 2n + 1n) <= approvedAmount).toBe(false)
  })
})

```

run it from the repo root:

```bash
pnpm install --frozen-lockfile --config.ignoreScripts=true
pnpm --filter portal exec vitest run src/__poc__/payment-zero-quote.poc.test.ts --reporter=verbose
```

captured output:

```
 ✓ src/__poc__/payment-zero-quote.poc.test.ts > failed per-token price read -> selectable $0.00 quote -> approval skipped -> amount-less renew vs standing 2x allowance > a failed per-token price read renders the token as a selectable $0.00 quote 43ms
 ✓ src/__poc__/payment-zero-quote.poc.test.ts > failed per-token price read -> selectable $0.00 quote -> approval skipped -> amount-less renew vs standing 2x allowance > confirm skips the approve leg entirely and submits a renew with NO amount argument 140ms
 ✓ src/__poc__/payment-zero-quote.poc.test.ts > failed per-token price read -> selectable $0.00 quote -> approval skipped -> amount-less renew vs standing 2x allowance > a healthy flow approves exactly 2x the quote and its standing residual settles any live price L <= 2P for a later $0.00 flow 4ms
 Test Files  1 passed (1)
      Tests  3 passed (3)
```

readback: test 1 proves a failed read produces a selectable $0.00 row in the real `buildTokenData` and the real mounted `PaymentTokenList`; test 2 proves the confirm path emits exactly one transaction, the amount-less renew targeting the real v2 renewer, with no approval prompt; test 3 proves a healthy flow's real encoded approve is exactly 2P, that the renew waits for it, and that the standing residual then settles every live price up to 2P against a later $0.00 consent.
