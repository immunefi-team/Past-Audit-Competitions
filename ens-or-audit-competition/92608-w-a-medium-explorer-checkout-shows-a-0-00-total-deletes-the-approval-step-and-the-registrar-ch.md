# 92608 w a medium explorer checkout shows a 0 00 total deletes the approval step and the registrar charges the full rent price the user was never shown

**Submitted on Sep 10th 2026 at 21:45:54 UTC by @v4rvl for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92608
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

`buildTokenData` substitutes `DEFAULT_PRICE` - a record whose `total` is `0n` - for any payment token whose price read did not resolve. The Explorer's neverthrow query wrapper converts a failed price read into a thrown query, so `query.data` is `undefined` and the substitution fires. `0n` then propagates as a real quoted price through three independent gates that all invert: the "Insufficient balance" guard becomes unreachable because `balance < 0n` is never true, the token row renders "available" and stays clickable on a wallet holding zero USDC and zero DAI, and the amount carried into the flow is `0n`, so `needsApproval = allowance < 0n` is false and the ERC-20 approval step is removed from the transaction list. The registration summary above the picker fetches the same price under a different query key, so it can show the real total while the picker shows nothing. The user is shown `$0.00` for a purchase that is not free, and the transaction submitted is a `register` / `renew` whose on-chain charge is the real price rather than the displayed one. Driven to its endpoint on a Sepolia fork: displayed total `$0.00`, wallet asked to sign `approve(registrar, 8800023)` for a step the modal no longer listed, registrar pulled `8000021` by `transferFrom`, buyer's USDC balance `100000000` to `91999979`, name registered to the buyer on the ENSv2 registry. The multi-name renewal picker in the same codebase blocks this exact condition and its code comment names the consequence.

## Root cause

**The substitution.** `apps/portal/src/features/register/utils/tokenData.ts:13-19,46-61`:

```ts
export const DEFAULT_PRICE: TokenPrice = {
  total: 0n,
  base: 0n,
  premium: 0n,
  decimals: USDC_DECIMALS,
  hasPremium: false,
}
```

```ts
export function buildTokenData(
  tokens: readonly TokenInput[],
  prices: readonly (TokenPrice | undefined)[],
  balances: readonly bigint[],
  allowances: readonly bigint[],
): TokenWithPriceAndBalance[] {
  return tokens.map((token, i) => {
    const price = prices[i]
    return {
      ...token,
      price: isTokenPrice(price) ? price : DEFAULT_PRICE,
```

**How `undefined` gets there.** `packages/utils/src/tanstack-query/neverthrow/query.ts:132-141` turns a neverthrow `Err` into a throw, so a failed price read leaves the query in `error` state with `data === undefined`:

```ts
  const queryFn =
    typeof rawQueryFn === 'function'
      ? (context: QueryFunctionContext<QueryKey>) =>
          rawQueryFn(context).match(
            (value) => value,
            (error) => {
              throw error
            },
          )
      : rawQueryFn
```

`getRegistrationPrice` and `getRenewalPrice` both return `Err(GetRegistrationPriceError)` when the on-chain read fails (`apps/portal/src/features/register/hooks/useRegistrationPrice.ts:81-85`, `useRenewalPrice.ts:36-45`; the latter's doc comment states pricing "reverts if the name isn't renewable via the renewer").

**The picker checks only `isLoading`, never `isError`.** `apps/portal/src/features/register/components/PaymentTokenPicker.tsx:109-115,144-153`:

```ts
  if (
    balancesQuery.isLoading ||
    allowancesQuery.isLoading ||
    priceQueries.some((query) => query.isLoading)
  ) {
    return <Skeleton />
  }
```

```ts
  const tokenData = buildTokenData(
    PAYMENT_TOKENS,
    priceQueries.map((query) => query.data),
    balances,
    allowances,
  )

  const noSupportedTokenHasSufficientBalance = tokenData.every(
    (token) => token.balance < token.price.total,
  )
```

**The three gates that invert.** `apps/portal/src/features/register/components/PaymentTokenList.tsx:34,41,60-64`:

```tsx
      const hasSufficientBalance = token.balance >= token.price.total
```

```tsx
          disabled={!hasSufficientBalance || isRegistering}
```

```tsx
            {hasSufficientBalance ? (
              <p className="text-muted-foreground text-xs">available</p>
            ) : (
              <p className="text-destructive text-xs">Insufficient balance</p>
            )}
```

`apps/portal/src/features/register/components/PaymentTokenSection.tsx:42-46` hands the zero straight to the flow:

```ts
  const handleBuyName = () => {
    if (selectedTokenData) {
      onConfirm(selectedTokenData.address, selectedTokenData.price.total)
    }
  }
```

`apps/portal/src/features/register/hooks/useRegistrationTransactions.ts:137-140,230-254,311-313`:

```ts
  const needsApproval =
    !savedParams ||
    allowanceQuery.data === undefined ||
    allowanceQuery.data < savedParams.tokenPrice
```

```ts
    if (needsApproval) {
      steps.push({
        id: REGISTRATION_TX_IDS.approve,
```

```ts
  const paid = savedParams
    ? formatPriceDisplay(savedParams.tokenPrice, savedParams.tokenDecimals)
    : undefined
```

The renewal flow has the same twin. `apps/portal/src/features/renew/components/ExtendNameModal.tsx:117-124` forwards `token.price.total`, and `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:482` drops the approval on `>=`:

```ts
      if (single.tokenAllowance >= single.tokenPrice) return [renewTx]
```

**The sibling that gets it right.** `apps/portal/src/features/renew/components/multi-name-extension/MultiNamePaymentTokenPicker.tsx:119-137` refuses the same condition, and its comment states the consequence this report demonstrates:

```ts
  // A settled-but-errored price read must NOT be treated as 0: that would
  // understate a renewer's approval total (risking an on-chain revert) and make
  // a token look affordable when it isn't. If any name's price failed to resolve,
  // block selection and prompt a retry rather than price the batch wrong.
  const pricesResolved =
    usdcPriceQueries.every((q) => q.data && isPriceResult(q.data)) &&
    daiPriceQueries.every((q) => q.data && isPriceResult(q.data))

  if (!pricesResolved) {
    return (
      <MessageCard
        variant="warning"
```

**Why the summary and the picker can disagree.** `RegisterNameCheckoutSummary` fetches the price with no `token` argument (`apps/portal/src/features/register/components/RegisterNameSummary.tsx:58-64`), while `PaymentTokenPicker` fetches it with `token: token.address` (`PaymentTokenPicker.tsx:92-107`). `createQueryKey` includes the parameters, so these are two independent cache entries for the same USDC price. One can be in `success` while the other is in `error`.

## Steps to reproduce

1. Connect a wallet to the Explorer and open `/<name>/register` for an available `.eth` name.
2. Cause the picker's two price reads (`get-registration-price` with `token: USDC` and `token: DAI`) to settle in `error`. Any RPC failure on those two calls does it - a rate-limited or briefly unavailable provider is sufficient. The checkout summary's own price query uses a different key and is unaffected.
3. Observe the payment section: both tokens render with `available` beneath the balance, and neither shows "Insufficient balance" - including on a wallet with a zero balance of both.
4. Select a token. The Register button enables.
5. Click Register. The transaction modal opens listing "Approve payment" while the allowance read is still in flight, and drops that row once the read settles, leaving "Deploy resolver", "Submit commitment" and "Register name".
6. The success screen's `paid` value is `$0.00`.
7. Press through the modal. The registration completes. The registrar charges its real rent price by `transferFrom`, and the wallet is prompted along the way to sign an `approve` for 110 percent of that price, a step the modal's list no longer contains. Measured on the fork: displayed `$0.00`, `approve` signed for `8800023`, `transferFrom` pulled `8000021`, USDC balance `100000000` to `91999979`, allowance `0` to `800002`, name owned by the buyer on the ENSv2 registry.

## Proof of Concept

## Proof of concept

```tsx
/**
 * A failed price read renders the checkout as $0.00, enables the buy button on
 * an empty wallet, and deletes the approval step.
 *
 * Real production code driven here, unmodified:
 *   - features/register/utils/tokenData.ts            buildTokenData / DEFAULT_PRICE
 *   - features/register/components/PaymentTokenList   the rendered picker
 *   - features/register/utils/registrationPrice.ts    formatPriceDisplay
 *   - packages/utils tanstack-query/neverthrow        resultQueryOptions
 *   - features/register/hooks/useRegistrationPrice    getRegistrationPrice (Err path)
 *
 * The only stand-in is the network: a queryFn that yields the same neverthrow
 * `Err` the production `getRegistrationPrice` yields when the on-chain read
 * fails. The Err -> `data === undefined` conversion under test is performed by
 * the real `resultQueryOptions`, not by the test.
 */
import { QueryClient } from '@tanstack/react-query'
import { render, screen } from '@testing-library/react'
import { err } from 'neverthrow'
import { describe, expect, it } from 'vitest'
import { resultQueryOptions } from '@ens-apps/utils/tanstack-query/neverthrow'
import { PaymentTokenList } from '@/features/register/components/PaymentTokenList'
import { PAYMENT_TOKENS } from '@/features/register/constants/paymentTokens'
import { GetRegistrationPriceError } from '@/features/register/hooks/useRegistrationPrice'
import { formatPriceDisplay } from '@/features/register/utils/registrationPrice'
import {
  buildTokenData,
  DEFAULT_PRICE,
  type TokenPrice,
} from '@/features/register/utils/tokenData'

/** The real price of the name, as the checkout summary renders it. */
const REAL_USDC_PRICE: TokenPrice = {
  base: 5_000_000n,
  premium: 0n,
  total: 5_000_000n,
  decimals: 6,
  hasPremium: false,
}
const REAL_DAI_PRICE: TokenPrice = {
  base: 5_000_000_000_000_000_000n,
  premium: 0n,
  total: 5_000_000_000_000_000_000n,
  decimals: 18,
  hasPremium: false,
}

/** Wallet holds no USDC and no DAI. */
const EMPTY_BALANCES = [0n, 0n] as const
/** No standing ERC-20 allowance to the registrar. */
const NO_ALLOWANCES = [0n, 0n] as const

describe('zero-price checkout', () => {
  it('resultQueryOptions turns a production price Err into data === undefined', async () => {
    const client = new QueryClient({
      defaultOptions: { queries: { retry: false } },
    })

    const options = resultQueryOptions({
      queryKey: ['portal2', 'price'] as const,
      queryFn: () =>
        err(
          new GetRegistrationPriceError({
            cause: new Error('execution reverted') as never,
          }),
        ),
    })

    await expect(client.fetchQuery(options)).rejects.toBeInstanceOf(
      GetRegistrationPriceError,
    )

    const state = client.getQueryState(['portal2', 'price'])
    expect(state?.status).toBe('error')
    expect(state?.data).toBeUndefined()

    console.log(
      '[STEP 1] query status =',
      state?.status,
      '| query.data =',
      String(state?.data),
    )
  })

  it('DEFECT: undefined price -> $0.00 total, enabled button on a 0-balance wallet, no approval', () => {
    // PaymentTokenPicker.tsx:144-149 passes `priceQueries.map(q => q.data)`
    // straight through. Both reads failed, so both are undefined.
    const tokenData = buildTokenData(
      PAYMENT_TOKENS,
      [undefined, undefined],
      EMPTY_BALANCES,
      NO_ALLOWANCES,
    )

    expect(tokenData[0].price).toEqual(DEFAULT_PRICE)
    expect(tokenData[0].price.total).toBe(0n)
    expect(tokenData[1].price.total).toBe(0n)

    // PaymentTokenPicker.tsx:151-153 - the "Insufficient balance" guard.
    const noSupportedTokenHasSufficientBalance = tokenData.every(
      (token) => token.balance < token.price.total,
    )
    expect(noSupportedTokenHasSufficientBalance).toBe(false)

    render(
      <PaymentTokenList
        tokenData={tokenData}
        selectedToken={null}
        isRegistering={false}
        onSelect={() => undefined}
      />,
    )

    // PaymentTokenList.tsx:34,41,60-64 - a wallet holding 0 USDC is told the
    // token is available and the row is clickable.
    const buttons = screen.getAllByRole('button')
    expect(buttons).toHaveLength(2)
    expect(buttons[0]).not.toBeDisabled()
    expect(buttons[1]).not.toBeDisabled()
    expect(screen.getAllByText('available')).toHaveLength(2)
    expect(screen.queryAllByText('Insufficient balance')).toHaveLength(0)

    // PaymentTokenSection.tsx:44 -> RegisterName.tsx:79 -> startFlow(addr, price)
    const tokenPrice = tokenData[0].price.total
    expect(tokenPrice).toBe(0n)

    // useRegistrationTransactions.ts:311-313 - the amount shown as paid.
    const paid = formatPriceDisplay(tokenPrice, tokenData[0].price.decimals)
    expect(paid).toBe('$0.00')

    // useRegistrationTransactions.ts:137-140 with a settled allowance read of 0.
    const allowance = 0n
    const needsApproval = allowance < tokenPrice
    expect(needsApproval).toBe(false)

    // useRenewalTransactions.ts:482 - the renewal flow's twin gate.
    const renewalSkipsApproval = allowance >= tokenPrice
    expect(renewalSkipsApproval).toBe(true)

    // useRenewalTransactions.ts:157 - the amount the renewal would approve.
    const renewalApproveAmount = tokenPrice * 2n
    expect(renewalApproveAmount).toBe(0n)

    console.log(
      '[DEFECT] displayed total =',
      paid,
      '| buttons disabled =',
      buttons.map((b) => (b as HTMLButtonElement).disabled).join(','),
      '| insufficient-balance warnings =',
      screen.queryAllByText('Insufficient balance').length,
      '| needsApproval =',
      needsApproval,
      '| renewal approve amount =',
      renewalApproveAmount.toString(),
    )
  })

  it('CONTROL: same wallet, price reads succeed -> $5.00, disabled buttons, approval required', () => {
    // ONE load-bearing condition changed: the two price reads resolve.
    const tokenData = buildTokenData(
      PAYMENT_TOKENS,
      [REAL_USDC_PRICE, REAL_DAI_PRICE],
      EMPTY_BALANCES,
      NO_ALLOWANCES,
    )

    expect(tokenData[0].price.total).toBe(5_000_000n)

    const noSupportedTokenHasSufficientBalance = tokenData.every(
      (token) => token.balance < token.price.total,
    )
    expect(noSupportedTokenHasSufficientBalance).toBe(true)

    render(
      <PaymentTokenList
        tokenData={tokenData}
        selectedToken={null}
        isRegistering={false}
        onSelect={() => undefined}
      />,
    )

    const buttons = screen.getAllByRole('button')
    expect(buttons).toHaveLength(2)
    expect(buttons[0]).toBeDisabled()
    expect(buttons[1]).toBeDisabled()
    expect(screen.getAllByText('Insufficient balance')).toHaveLength(2)

    const tokenPrice = tokenData[0].price.total
    const paid = formatPriceDisplay(tokenPrice, tokenData[0].price.decimals)
    expect(paid).toBe('$5.00')

    const allowance = 0n
    expect(allowance < tokenPrice).toBe(true)
    expect(tokenPrice * 2n).toBe(10_000_000n)

    console.log(
      '[CONTROL] displayed total =',
      paid,
      '| buttons disabled =',
      buttons.map((b) => (b as HTMLButtonElement).disabled).join(','),
      '| insufficient-balance warnings =',
      screen.getAllByText('Insufficient balance').length,
      '| needsApproval =',
      allowance < tokenPrice,
      '| renewal approve amount =',
      (tokenPrice * 2n).toString(),
    )
  })

  it('the sibling multi-name picker blocks the same condition', async () => {
    // MultiNamePaymentTokenPicker.tsx:123-137 refuses to price a batch when any
    // read failed. Reproduced verbatim against the same undefined inputs.
    const { isPriceResult } = await import(
      '@/features/register/utils/registrationPrice'
    )
    const failedQueries = [{ data: undefined }, { data: undefined }]
    const okQueries = [{ data: REAL_USDC_PRICE }, { data: REAL_USDC_PRICE }]

    const pricesResolvedWhenFailed = failedQueries.every(
      (q) => q.data && isPriceResult(q.data),
    )
    const pricesResolvedWhenOk = okQueries.every(
      (q) => q.data && isPriceResult(q.data),
    )

    expect(pricesResolvedWhenFailed).toBe(false)
    expect(pricesResolvedWhenOk).toBe(true)

    console.log(
      '[SIBLING GUARD] multi-name pricesResolved(failed reads) =',
      pricesResolvedWhenFailed,
      '-> selection blocked; single-name picker has no equivalent gate',
    )
  })
})
```

Command:

```
pnpm --filter portal exec vitest run src/portal2_zero_price_checkout.test.tsx --reporter=verbose --silent=false
```

Captured output:

```
stdout | src/portal2_zero_price_checkout.test.tsx > zero-price checkout > resultQueryOptions turns a production price Err into data === undefined
[STEP 1] query status = error | query.data = undefined

stdout | src/portal2_zero_price_checkout.test.tsx > zero-price checkout > DEFECT: undefined price -> $0.00 total, enabled button on a 0-balance wallet, no approval
[DEFECT] displayed total = $0.00 | buttons disabled = false,false | insufficient-balance warnings = 0 | needsApproval = false | renewal approve amount = 0

stdout | src/portal2_zero_price_checkout.test.tsx > zero-price checkout > CONTROL: same wallet, price reads succeed -> $5.00, disabled buttons, approval required
[CONTROL] displayed total = $5.00 | buttons disabled = true,true | insufficient-balance warnings = 2 | needsApproval = true | renewal approve amount = 10000000

stdout | src/portal2_zero_price_checkout.test.tsx > zero-price checkout > the sibling multi-name picker blocks the same condition
[SIBLING GUARD] multi-name pricesResolved(failed reads) = false -> selection blocked; single-name picker has no equivalent gate

 ✓ src/portal2_zero_price_checkout.test.tsx > zero-price checkout > resultQueryOptions turns a production price Err into data === undefined 7ms
 ✓ src/portal2_zero_price_checkout.test.tsx > zero-price checkout > DEFECT: undefined price -> $0.00 total, enabled button on a 0-balance wallet, no approval 136ms
 ✓ src/portal2_zero_price_checkout.test.tsx > zero-price checkout > CONTROL: same wallet, price reads succeed -> $5.00, disabled buttons, approval required 62ms
 ✓ src/portal2_zero_price_checkout.test.tsx > zero-price checkout > the sibling multi-name picker blocks the same condition 1ms

 Test Files  1 passed (1)
      Tests  4 passed (4)
   Start at  22:00:30
   Duration  10.93s (transform 1.37s, setup 360ms, import 9.09s, tests 210ms, environment 856ms)
```

### The price read is a revert-prone contract call, not only an RPC risk

`getRegistrationPrice` and `getRenewalPrice` end in a contract read, and those reads carry named revert paths. Measured against a local Anvil Sepolia fork (`http://127.0.0.1:8546`, chain id 11155111), on the v2 `ETHRegistrar` `0xa88553F454b77203B0D036A05c894d555EAAa2Cc` and the v1 `ETHRenewerV1` `0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa`:

```
$ cast call $REGISTRAR 'getRegisterPrice(string,uint64,address)(uint256,uint256)' 'zzfinalizerprobe' 31536000 $USDC
8000021 [8e6] 0

$ cast call $REGISTRAR 'getRegisterPrice(string,uint64,address)(uint256,uint256)' 'nike' 31536000 $USDC
Error: execution reverted: custom error 0x477707e8:  nike

$ cast call $REGISTRAR 'getRenewPrice(string,uint64,address)(uint256)' 'zzfinalizerprobe' 31536000 $USDC
Error: execution reverted: custom error 0x1caefaa0:  zzfinalizerprobe

$ cast call $RENEWER 'getRenewPrice(string,uint64,address)(uint256)' 'nike' 31536000 $USDC
160000009 [1.6e8]

$ cast 4byte 0x477707e8
NameNotAvailable(string)
$ cast 4byte 0x1caefaa0
NameNotRenewable(string)
```

So the same call that returns `8000021` for one label reverts `NameNotAvailable` or `NameNotRenewable` for another, purely on the chain state of the label at read time. Both revert paths land in `Err(GetRegistrationPriceError)` and therefore in `data === undefined`, which is the input the defect arm above is driven with. The picker issues one price query per token (`PaymentTokenPicker.tsx:92-107`), so the USDC and DAI reads settle independently and either can be the one that errors while the other succeeds.

### Terminal measurement: what was displayed, what was signed, what was charged

The section above stops at the render and the step list. This one carries the same checkout to its endpoint on the local Anvil Sepolia fork and measures four quantities side by side: the total the app displayed, the amount the wallet was actually asked to approve, the amount the registrar pulled, and the buyer's token balance before and after.

Both arms run the shipped containers: `PaymentTokenSection`, which mounts the real `PaymentTokenPicker` and lets it issue its own price, balance and allowance queries; the real `useRegistrationTransactions`; the real `TransactionModal`; and the real `registrationMachine` with every actor in `packages/transaction-manager`. The buyer is a real wagmi connection to a clean EOA, not a prop. The clicks are real clicks on the rendered token row, the `Register` button, the modal's `Start` button and the modal's `Open wallet` button. The single substituted dependency is the JSON-RPC endpoint that the app's own client accessor `safeGetClient` returns to the PRICE query (`useRegistrationPrice.ts:51` through `lib/wagmi/helpers.ts:16`): in the defect arm it points at a closed port, in the control arm at the fork. Balances, allowances, the machine's `publicClient` and every write go to the fork in both arms.

The one thing the earlier section could not see is what the registration state machine does on its own. `registration.machine.ts:1247-1300` enters `checkingAllowance`, reads the LIVE register price with `readPaymentAuthorizationActor` (`registration.actors.ts:592-630`), and when the allowance is short it submits an approval for that live price. `registration.actors.ts:82-84` is the amount:

```ts
function authorizedPaymentAmount(price: bigint): bigint {
  return price + price / 10n
}
```

So the wallet is prompted to approve `8800023` base units while the app's own step list, its success screen and its `paid` value all say `$0.00`, and the registrar then pulls `8000021`.

Save as `apps/portal/src/__poc_term3__/portal_zero_price_terminal.poc.test.tsx`:

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { connect, disconnect } from '@wagmi/core'
import { mock } from '@wagmi/connectors'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { act, fireEvent, render, screen, waitFor } from '@testing-library/react'
import {
  type Address,
  createPublicClient,
  createWalletClient,
  erc20Abi,
  http,
  parseAbiItem,
} from 'viem'
import { sepolia } from 'viem/chains'
import { useEffect } from 'react'
import { createConfig, usePublicClient, WagmiProvider } from 'wagmi'
import { afterEach, beforeAll, describe, expect, it, vi } from 'vitest'

const FORK = 'http://127.0.0.1:8546'
/** Closed port: the price RPC is the one dependency that fails. */
const DEAD_RPC = 'http://127.0.0.1:1'

const USDC = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39' as Address
const ETH_REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc' as Address
/** Clean EOA (no code), funded with 100 USDC and ETH on the fork. */
const VICTIM = '0xbEeF00000000000000000000000000000000e301' as Address

const DURATION_SECONDS = 31_536_000
/**
 * Ten-character labels, so both arms carry the identical registrar price, and
 * freshly randomised per run, because a label registered by a previous run is
 * no longer available and `getRegisterPrice` then reverts `NameNotAvailable`.
 */
const suffix = () => {
  const alphabet = 'abcdefghijklmnopqrstuvwxyz0123456789'
  let out = ''
  for (let i = 0; i < 4; i++) {
    out += alphabet[Math.floor(Math.random() * alphabet.length)]
  }
  return out
}
const DEFECT_LABEL = `term3z${suffix()}`
const CONTROL_LABEL = `term3c${suffix()}`
const TARGET_USDC = 100_000_000n

const forkChain = {
  ...extendChainWithEns(sepolia),
  rpcUrls: {
    default: { http: [FORK] },
    public: { http: [FORK] },
  },
}

const observer = createPublicClient({
  chain: forkChain,
  transport: http(FORK, { timeout: 120_000, retryCount: 8, retryDelay: 2_000 }),
})

// `safeGetClient` is the app's own accessor for its module-level wagmi client
// (lib/wagmi/helpers.ts:16). Only the price path uses it; balances, allowances,
// the machine's publicClient and every write go through the WagmiProvider
// config below, which points at the fork.
vi.mock('@/lib/wagmi/helpers', async () => {
  const { ok } = await import('neverthrow')
  return {
    // biome-ignore lint/suspicious/noExplicitAny: test global
    safeGetClient: () => ok((globalThis as any).__TERM3_PRICE_CLIENT__),
    safeGetConnectorClient: () => {
      throw new Error('not used in this PoC')
    },
  }
})

// Fire-and-forget Etherscan call made after the resolver proxy deploys
// (useRegistrationTransactions.ts:103). External service, no bearing on any
// asserted value.
vi.mock('@/utils/blockExplorer/verifyProxyContract', () => ({
  verifyProxyContract: async () => undefined,
}))

const { TransactionManagerProvider, transactionManager } = await import(
  '@ens-apps/transaction-manager'
)
const { PaymentTokenSection } = await import(
  '@/features/register/components/PaymentTokenSection'
)
const { useRegistrationTransactions } = await import(
  '@/features/register/hooks/useRegistrationTransactions'
)
const { TransactionModal } = await import(
  '@/features/transaction-manager/components/TransactionModal'
)
const { useTransactionModal, closeTransactionModal } = await import(
  '@/features/transaction-manager/hooks/useTransactionModal'
)
const { getEnsOwnerQueryOptions } = await import(
  '@/features/profile/hooks/useEnsOwner'
)

const priceClientFor = (url: string) =>
  createPublicClient({
    chain: forkChain,
    transport: http(url, {
      timeout: url === FORK ? 120_000 : 2_000,
      retryCount: url === FORK ? 8 : 0,
      retryDelay: 2_000,
    }),
  })

const wagmiConfigForFork = () =>
  createConfig({
    chains: [forkChain],
    connectors: [mock({ accounts: [VICTIM] })],
    storage: null,
    transports: {
      [forkChain.id]: http(FORK, {
        timeout: 120_000,
        retryCount: 8,
        retryDelay: 2_000,
      }),
    },
  })

type Captured = {
  paid: string | undefined
  stepTitles: string[]
  stepIds: string[]
  isSuccess: boolean
  confirmedPrice: bigint | null
}

const captured: Captured = {
  paid: undefined,
  stepTitles: [],
  stepIds: [],
  isSuccess: false,
  confirmedPrice: null,
}

/** Mirrors __root.tsx:22-42 `TransactionManagerSetup`, minus the router-only hooks. */
const TxManagerSetup = ({ children }: { children: React.ReactNode }) => {
  const publicClient = usePublicClient()
  useEffect(() => {
    if (publicClient) {
      transactionManager.setPublicClient(forkChain.id, publicClient)
    }
  }, [publicClient])
  if (!publicClient) return <>{children}</>
  return (
    <TransactionManagerProvider publicClient={publicClient}>
      {children}
    </TransactionManagerProvider>
  )
}

/**
 * Mirrors RegisterName.tsx:39, :79-81, :168-180 - the same hook, the same
 * onConfirm body (startFlow then openModal), the same modal.
 */
const CheckoutHarness = ({ name }: { name: string }) => {
  const { transactions, isRegistering, isSuccess, paid, startFlow } =
    useRegistrationTransactions({ name, duration: DURATION_SECONDS })
  const { openModal } = useTransactionModal()

  captured.paid = paid
  captured.stepTitles = transactions.map((t) => t.title)
  captured.stepIds = transactions.map((t) => t.id)
  captured.isSuccess = isSuccess

  return (
    <>
      <PaymentTokenSection
        name={name}
        duration={DURATION_SECONDS}
        isConnected
        isRegistering={isRegistering}
        onConfirm={(tokenAddress, tokenPrice) => {
          captured.confirmedPrice = tokenPrice
          startFlow(tokenAddress, tokenPrice)
          openModal()
        }}
      />
      <TransactionModal transactions={transactions} />
    </>
  )
}

const readUsdc = async (fn: 'balanceOf' | 'allowance') =>
  observer.readContract({
    address: USDC,
    abi: erc20Abi,
    functionName: fn,
    args: fn === 'balanceOf' ? [VICTIM] : [VICTIM, ETH_REGISTRAR],
    // biome-ignore lint/suspicious/noExplicitAny: two arities, one call site
  } as any) as Promise<bigint>

const APPROVAL_EVENT = parseAbiItem(
  'event Approval(address indexed owner, address indexed spender, uint256 value)',
)
const TRANSFER_EVENT = parseAbiItem(
  'event Transfer(address indexed from, address indexed to, uint256 value)',
)

const victimWallet = () =>
  createWalletClient({
    account: VICTIM,
    chain: forkChain,
    transport: http(FORK, { timeout: 120_000 }),
  })

/**
 * Fixture reset so both arms start from the identical wallet state: exactly
 * 100.000000 USDC and a zero allowance to the registrar. Both are ordinary
 * ERC-20 calls on the fork's mock token; neither touches app code.
 */
async function resetWalletFixture() {
  await observer.request({
    method: 'anvil_setBalance',
    params: [VICTIM, '0x56BC75E2D63100000'],
    // biome-ignore lint/suspicious/noExplicitAny: anvil namespace
  } as any)
  const wallet = victimWallet()
  if ((await readUsdc('allowance')) !== 0n) {
    const hash = await wallet.writeContract({
      address: USDC,
      abi: erc20Abi,
      functionName: 'approve',
      args: [ETH_REGISTRAR, 0n],
    })
    await observer.waitForTransactionReceipt({ hash })
  }
  const held = await readUsdc('balanceOf')
  if (held < TARGET_USDC) {
    const hash = await wallet.writeContract({
      address: USDC,
      abi: [parseAbiItem('function mint(address to, uint256 amount)')],
      functionName: 'mint',
      args: [VICTIM, TARGET_USDC - held],
    })
    await observer.waitForTransactionReceipt({ hash })
  }
}

/** One arm: drive the checkout to the chain and measure both sides. */
async function runArm(opts: { label: string; priceRpc: string }) {
  const name = `${opts.label}.eth`
  await resetWalletFixture()
  // biome-ignore lint/suspicious/noExplicitAny: test global
  ;(globalThis as any).__TERM3_PRICE_CLIENT__ = priceClientFor(opts.priceRpc)

  const livePrice = (await observer.readContract({
    address: ETH_REGISTRAR,
    abi: [
      parseAbiItem(
        'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
      ),
    ],
    functionName: 'getRegisterPrice',
    args: [opts.label, BigInt(DURATION_SECONDS), USDC],
  })) as readonly [bigint, bigint]
  const liveTotal = livePrice[0] + livePrice[1]

  const fromBlock = (await observer.getBlockNumber()) + 1n
  const balanceBefore = await readUsdc('balanceOf')
  const allowanceBefore = await readUsdc('allowance')

  const config = wagmiConfigForFork()
  await connect(config, { connector: config.connectors[0] })

  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false, gcTime: 0, staleTime: 0 } },
  })

  const view = render(
    <WagmiProvider config={config} reconnectOnMount={false}>
      <QueryClientProvider client={queryClient}>
        <TxManagerSetup>
          <CheckoutHarness name={name} />
        </TxManagerSetup>
      </QueryClientProvider>
    </WagmiProvider>,
  )

  // 1. the picker settles and renders the token rows
  const usdcButton = await screen.findByRole(
    'button',
    { name: /USDC/ },
    { timeout: 60_000 },
  )
  const rowText = usdcButton.textContent ?? ''
  const insufficientNodes = screen.queryAllByText('Insufficient balance').length
  const availableNodes = screen.queryAllByText('available').length

  // 2. select the token, then press Register
  await act(async () => {
    fireEvent.click(usdcButton)
  })
  const registerButton = screen.getByRole('button', { name: 'Register' })
  await act(async () => {
    fireEvent.click(registerButton)
  })

  // 3. the modal is open: read the step list the user is shown. The list is
  //    read twice - once on open, and once after every query the page issued
  //    has settled (`isFetching() === 0`), which is when the allowance read
  //    that `needsApproval` depends on has returned. Waiting on "no query in
  //    flight" does not presume either outcome.
  await waitFor(() => expect(screen.getByText('Deploy resolver')).toBeTruthy(), {
    timeout: 30_000,
  })
  const stepTitlesAtOpen = captured.stepTitles.slice()
  await waitFor(() => expect(queryClient.isFetching()).toBe(0), {
    timeout: 60_000,
    interval: 200,
  })
  const approveStepShown = screen.queryByText('Approve payment') !== null
  const stepTitlesShown = captured.stepTitles.slice()
  const paidShown = captured.paid

  // 4. drive the flow: Start -> Open wallet. One click on "Open wallet" runs
  //    the machine end to end (handleStart sends START_REGISTRATION).
  const startButton = screen.getByRole('button', { name: 'Start' })
  await act(async () => {
    fireEvent.click(startButton)
  })
  const openWallet = await screen.findByRole(
    'button',
    { name: 'Open wallet' },
    { timeout: 30_000 },
  )
  await act(async () => {
    fireEvent.click(openWallet)
  })

  // 5. wait for the machine to reach success (commit cooldown is 60s on-chain)
  await waitFor(() => expect(captured.isSuccess).toBe(true), {
    timeout: 420_000,
    interval: 1_000,
  })

  const balanceAfter = await readUsdc('balanceOf')
  const allowanceAfter = await readUsdc('allowance')

  const approvals = await observer.getLogs({
    address: USDC,
    event: APPROVAL_EVENT,
    args: { owner: VICTIM, spender: ETH_REGISTRAR },
    fromBlock,
    toBlock: 'latest',
  })
  const transfers = await observer.getLogs({
    address: USDC,
    event: TRANSFER_EVENT,
    args: { from: VICTIM },
    fromBlock,
    toBlock: 'latest',
  })

  view.unmount()
  await disconnect(config)
  act(() => closeTransactionModal())

  return {
    name,
    liveTotal,
    rowText,
    insufficientNodes,
    availableNodes,
    confirmedPrice: captured.confirmedPrice,
    paidShown,
    stepTitlesAtOpen,
    stepTitlesShown,
    approveStepShown,
    balanceBefore,
    balanceAfter,
    allowanceBefore,
    allowanceAfter,
    approvedValues: approvals.map((l) => l.args.value as bigint),
    transferredValues: transfers.map((l) => l.args.value as bigint),
  }
}

/** Reads the registry through the app's own owner resolution. */
async function ownerOf(name: string) {
  // biome-ignore lint/suspicious/noExplicitAny: test global
  ;(globalThis as any).__TERM3_PRICE_CLIENT__ = priceClientFor(FORK)
  const options = getEnsOwnerQueryOptions({ name })
  // biome-ignore lint/suspicious/noExplicitAny: query context is unused here
  const o = options as any
  const result = await o.queryFn({ queryKey: o.queryKey })
  return result as { owner: string; protocolVersion: string } | null
}

const report = (tag: string, r: Awaited<ReturnType<typeof runArm>>) =>
  console.log(
    [
      `[${tag} ${r.name}]`,
      `live registrar price = ${r.liveTotal}`,
      `| displayed total = ${r.paidShown}`,
      `| price handed to startFlow = ${r.confirmedPrice}`,
      `| USDC row = ${JSON.stringify(r.rowText)}`,
      `| "Insufficient balance" nodes = ${r.insufficientNodes}`,
      `| "available" nodes = ${r.availableNodes}`,
      `| modal steps on open = ${JSON.stringify(r.stepTitlesAtOpen)}`,
      `| modal steps once every read settled = ${JSON.stringify(r.stepTitlesShown)}`,
      `| "Approve payment" row present = ${r.approveStepShown}`,
      `| USDC balance ${r.balanceBefore} -> ${r.balanceAfter} (charged ${r.balanceBefore - r.balanceAfter})`,
      `| approve() amounts signed on chain = ${JSON.stringify(r.approvedValues.map(String))}`,
      `| transferFrom amounts pulled = ${JSON.stringify(r.transferredValues.map(String))}`,
      `| allowance ${r.allowanceBefore} -> ${r.allowanceAfter}`,
    ].join(' '),
  )

describe('zero-price checkout, terminal measurement on the Sepolia fork', () => {
  beforeAll(async () => {
    expect(await observer.getChainId()).toBe(11155111)
    await resetWalletFixture()
    expect(await readUsdc('balanceOf')).toBe(TARGET_USDC)
    expect(await readUsdc('allowance')).toBe(0n)
  }, 300_000)

  afterEach(() => {
    act(() => closeTransactionModal())
  })

  it('DEFECT: price read fails -> $0.00 shown, no approval step, registrar charges the live price', async () => {
    const r = await runArm({ label: DEFECT_LABEL, priceRpc: DEAD_RPC })
    report('DEFECT', r)
    const owner = await ownerOf(r.name)
    console.log(`[DEFECT ${r.name}] registry owner after = ${JSON.stringify(owner)}`)

    // what the user was shown
    expect(r.paidShown).toBe('$0.00')
    expect(r.confirmedPrice).toBe(0n)
    expect(r.insufficientNodes).toBe(0)
    expect(r.availableNodes).toBe(2)
    expect(r.approveStepShown).toBe(false)
    expect(r.stepTitlesShown).toEqual([
      'Deploy resolver',
      'Submit commitment',
      'Register name',
    ])

    // what the wallet was asked to approve, and what was charged
    expect(r.liveTotal).toBe(8_000_021n)
    expect(r.balanceBefore).toBe(100_000_000n)
    expect(r.balanceAfter).toBe(91_999_979n)
    expect(r.balanceBefore - r.balanceAfter).toBe(8_000_021n)
    expect(r.approvedValues).toEqual([8_800_023n])
    expect(r.transferredValues).toEqual([8_000_021n])
    expect(r.allowanceBefore).toBe(0n)
    expect(r.allowanceAfter).toBe(800_002n)
    expect(owner?.owner?.toLowerCase()).toBe(VICTIM.toLowerCase())
  }, 900_000)

  it('CONTROL: same wallet, same duration, price read succeeds -> $8.00 shown, approval step present, same charge', async () => {
    const r = await runArm({ label: CONTROL_LABEL, priceRpc: FORK })
    report('CONTROL', r)
    const owner = await ownerOf(r.name)
    console.log(`[CONTROL ${r.name}] registry owner after = ${JSON.stringify(owner)}`)

    expect(r.paidShown).toBe('$8.00')
    expect(r.confirmedPrice).toBe(8_000_021n)
    expect(r.approveStepShown).toBe(true)
    expect(r.stepTitlesShown).toEqual([
      'Deploy resolver',
      'Submit commitment',
      'Approve payment',
      'Register name',
    ])

    expect(r.liveTotal).toBe(8_000_021n)
    expect(r.balanceBefore).toBe(100_000_000n)
    expect(r.balanceAfter).toBe(91_999_979n)
    expect(r.balanceBefore - r.balanceAfter).toBe(8_000_021n)
    expect(r.transferredValues).toEqual([8_000_021n])
    expect(r.approvedValues).toEqual([8_800_023n])
    expect(r.allowanceBefore).toBe(0n)
    expect(r.allowanceAfter).toBe(800_002n)
    expect(owner?.owner?.toLowerCase()).toBe(VICTIM.toLowerCase())
  }, 900_000)
})
```

Run:

```bash
pnpm --filter portal exec vitest run src/__poc_term3__/portal_zero_price_terminal.poc.test.tsx --reporter=verbose --silent=false
```

Captured output. The transaction-manager's own progress logging is omitted:

```
[DEFECT term3z49ga.eth] live registrar price = 8000021 | displayed total = $0.00 | price handed to startFlow = 0 | USDC row = "USDCUSDC100available" | "Insufficient balance" nodes = 0 | "available" nodes = 2 | modal steps on open = ["Deploy resolver","Submit commitment","Approve payment","Register name"] | modal steps once every read settled = ["Deploy resolver","Submit commitment","Register name"] | "Approve payment" row present = false | USDC balance 100000000 -> 91999979 (charged 8000021) | approve() amounts signed on chain = ["8800023"] | transferFrom amounts pulled = ["8000021"] | allowance 0 -> 800002
[DEFECT term3z49ga.eth] registry owner after = {"owner":"0xbEeF00000000000000000000000000000000e301","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"}

[CONTROL term3c37ju.eth] live registrar price = 8000021 | displayed total = $8.00 | price handed to startFlow = 8000021 | USDC row = "USDCUSDC100available" | "Insufficient balance" nodes = 1 | "available" nodes = 1 | modal steps on open = ["Deploy resolver","Submit commitment","Approve payment","Register name"] | modal steps once every read settled = ["Deploy resolver","Submit commitment","Approve payment","Register name"] | "Approve payment" row present = true | USDC balance 100000000 -> 91999979 (charged 8000021) | approve() amounts signed on chain = ["8800023"] | transferFrom amounts pulled = ["8000021"] | allowance 0 -> 800002
[CONTROL term3c37ju.eth] registry owner after = {"owner":"0xbEeF00000000000000000000000000000000e301","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"}

 ✓ zero-price checkout, terminal measurement on the Sepolia fork > DEFECT: price read fails -> $0.00 shown, no approval step, registrar charges the live price 80699ms
 ✓ zero-price checkout, terminal measurement on the Sepolia fork > CONTROL: same wallet, same duration, price read succeeds -> $8.00 shown, approval step present, same charge 89669ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
   Start at  22:26:22
   Duration  258.29s (transform 10.28s, setup 545ms, import 44.21s, tests 212.43s, environment 672ms)
```

Read across the two arms. The buyer opened each arm holding exactly `100000000` USDC base units and a zero allowance to the registrar, and asked for the same one-year registration of a ten-character `.eth` label, which the registrar prices at `8000021` in both. The only difference is whether the picker's price read resolved.

| quantity                            | price read fails                                        | price read resolves                                                        |
| ----------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------------------- |
| registrar's live price              | `8000021`                                               | `8000021`                                                                  |
| total the checkout displayed        | `$0.00`                                                 | `$8.00`                                                                    |
| price handed to `startFlow`         | `0`                                                     | `8000021`                                                                  |
| USDC row `textContent`              | `USDCUSDC100available`                                  | `USDCUSDC100available`                                                     |
| `Insufficient balance` nodes        | 0                                                       | 1                                                                          |
| `available` nodes                   | 2                                                       | 1                                                                          |
| modal steps once every read settled | `Deploy resolver`, `Submit commitment`, `Register name` | `Deploy resolver`, `Submit commitment`, `Approve payment`, `Register name` |
| `Approve payment` row present       | `false`                                                 | `true`                                                                     |
| `approve()` amount signed on chain  | `8800023`                                               | `8800023`                                                                  |
| `transferFrom` amount pulled        | `8000021`                                               | `8000021`                                                                  |
| USDC balance before and after       | `100000000` to `91999979`                               | `100000000` to `91999979`                                                  |
| allowance before and after          | `0` to `800002`                                         | `0` to `800002`                                                            |
| registry owner of the new name      | the buyer, ENSv2                                        | the buyer, ENSv2                                                           |

Two things this settles that the component-level arm could not. First, the transaction does not revert: the registration completes, the name is owned by the buyer on the ENSv2 registry, and the charge is the registrar's real rent price. The user is charged `8000021` against a screen that said `$0.00`. Second, the missing approval step is not merely cosmetic and it is not a precondition either: with a zero standing allowance the machine still signs an `approve` for `8800023`, a wallet prompt for a step the app's own step list does not contain, and the `800002` residue it leaves behind is a standing allowance the user was never shown.

## Control

Same wallet (0 USDC, 0 DAI, 0 allowance), same tokens, same rendered component, same downstream expressions. One load-bearing condition changed: whether the two price reads resolve.

| condition           | `price.total` | `noSupportedTokenHasSufficientBalance` | buttons disabled | "Insufficient balance" nodes | displayed total | `needsApproval` | renewal approve amount |
| ------------------- | ------------- | -------------------------------------- | ---------------- | ---------------------------- | --------------- | --------------- | ---------------------- |
| price reads fail    | `0n`          | `false`                                | `false,false`    | 0                            | `$0.00`         | `false`         | `0`                    |
| price reads resolve | `5000000n`    | `true`                                 | `true,true`      | 2                            | `$5.00`         | `true`          | `10000000`             |

The end-to-end control is the second arm of the terminal measurement above: the same buyer, the same opening balance of `100000000` USDC base units, the same zero allowance, the same one-year duration, the same ten-character label length and therefore the same `8000021` registrar price. One condition changed, whether the picker's price read resolved. The control displayed `$8.00` and listed "Approve payment"; the defect arm displayed `$0.00` and did not. Both charged `8000021`.

## Actor ledger

Every row measured in the defect arm of the terminal test, on the Sepolia fork. Amounts are USDC base units, six decimals.

| Actor                        | Opening state                                                                      | Attack outcome                                                                                                     | Control outcome                                                                                                         | Entitled change                           | Evidence                                                                                             |
| ---------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Buyer `0xbEeF...e301`        | `100000000` USDC, `0` allowance to the registrar, owns no `.eth` name in this flow | `91999979` USDC, `800002` standing allowance, owns `term3z49ga.eth`; the only price the app ever named was `$0.00` | `91999979` USDC, `800002` standing allowance, owns `term3c37ju.eth`; the app named `$8.00` before the wallet was opened | pay the quoted price, or decline it       | `USDC balance 100000000 -> 91999979`, `allowance 0 -> 800002`, `displayed total = $0.00`             |
| ETHRegistrar `0xa885...a2Cc` | rent price `8000021` for the label and duration                                    | pulled `8000021` by `transferFrom`                                                                                 | pulled `8000021` by `transferFrom`                                                                                      | receive its own quoted rent               | `transferFrom amounts pulled = ["8000021"]`, `getRegisterPrice -> 8000021`                           |
| The wallet prompt            | no pending approval                                                                | asked to sign `approve(registrar, 8800023)`, a step absent from the modal's list                                   | asked to sign `approve(registrar, 8800023)`, a step present in the modal's list                                         | be asked only for steps the app disclosed | `approve() amounts signed on chain = ["8800023"]`, `"Approve payment" row present = false` vs `true` |

The headline quantity is one number: the buyer paid `8000021` base units, `8.000021` USDC, against a displayed total of `$0.00`. The gap is the whole price.

## Impact

Predicate by predicate against the selected row.

**The application displays an incorrect transaction detail.** The checkout renders `$0.00` as the amount for a registration or renewal whose on-chain rent price is not zero. Measured: displayed total `$0.00` in the defect arm against `$5.00` in the control arm, from identical component inputs except the price read.

**So that a user submits a transaction.** The Register button is enabled in the defect arm on a wallet holding zero of both accepted tokens, where in the control arm it is disabled with "Insufficient balance" rendered twice. Measured: `buttons disabled = false,false` against `true,true`, and `insufficient-balance warnings = 0` against `2`.

**Showing the wrong amount on a transaction the user then approves.** This is the sub-bullet the evidence reaches, and it is measured to its endpoint. The registration completed on the fork: the wallet signed an `approve` for `8800023` base units and the registrar pulled `8000021` by `transferFrom`, while the app displayed `$0.00`, handed `0` to `startFlow`, and listed three steps that did not include "Approve payment". The buyer's USDC balance went from `100000000` to `91999979`. The name is owned by the buyer on the ENSv2 registry, so the flow is not merely broken; it settles at a price the user was never shown.

**No standing allowance is needed.** The wallet in the measurement opened with an allowance of `0` to the registrar. The registration state machine reads the live price itself at `registration.machine.ts:1247-1300` and submits the approval regardless of what the picker quoted, so the charge does not depend on a residual allowance left by an earlier flow. The residual-allowance path is a second, independent way to reach the same charge: the renewal flow approves twice the price at `useRenewalTransactions.ts:157` and contains no revoke step, and after one completed renewal of a five-character name on the same fork the residual allowance on the v1 renewer measured `160000009`.

**Who loses what.** The buyer is charged `8000021` USDC base units against a screen that said `$0.00`, a difference of `8000021` base units, or the entire price. The buyer also ends the flow with a standing `800002` allowance to the registrar that no screen in the flow named. A buyer holding zero of both tokens is separately driven into the same flow, having been told on a zero balance that both tokens are `available` with no "Insufficient balance" warning rendered.

**The program named this case.** `QA-03` closes with "A mismatch between the displayed total and the amount actually charged on-chain would be a new finding." That is this mismatch: displayed `0`, charged the registrar's rent price.

**Strongest counterargument, and the evidence against it.** The strongest rejection is that a failed price read is an infrastructure problem rather than an application defect, so the app is not at fault. Two pieces of evidence defeat it. First, the codebase has already ruled on this question in the opposite direction: `MultiNamePaymentTokenPicker.tsx:119-122` states that a settled-but-errored price read must not be treated as `0`, names both consequences this report measures, and blocks selection. The single-name register and renew pickers do the thing that comment forbids, so the disagreement is internal to the app, not between the app and its provider. Second, the failure is not only an RPC risk: the price read is a contract call with named revert paths, measured above as `NameNotAvailable(string)` and `NameNotRenewable(string)` on the deployed registrar and renewer, and both land in the same `data === undefined` the defect arm is driven with.

**Honest bound on the trigger.** The precondition is that a price query settles in `error` while the rest of the page works. I did not find an attacker-controlled way to force a specific user's price query to fail. The reachable causes established here are provider failure and the two measured on-chain revert paths. What is not conditional is the handling: three independent gates invert on the same substituted `0n`, and the picker checks only `isLoading`, never `isError` (`PaymentTokenPicker.tsx:109-115`).

**Bound.** One checkout per occurrence, for as long as the price query stays in `error`. The zero does not persist in chain state. Two things do: the amount the `register` or `renew` transaction charged, measured `8000021`, and the allowance residue the machine's own approval leaves behind, measured `800002` on a wallet that opened the flow at `0`.

## Suggested fix

1. Delete `DEFAULT_PRICE` as a fallback for a failed read. `buildTokenData` should return `price: undefined` (or a discriminated `{ status: 'unavailable' }`) so that "no price" is not representable as "free".
2. Give the single-name picker the guard its multi-name sibling already has. `PaymentTokenPicker` should compute `pricesResolved` with `isPriceResult` over `priceQueries.map(q => q.data)`, and render the same "Couldn't load prices" card instead of a token list when it is false.
3. Make the balance and approval gates fail closed. `hasSufficientBalance` must be false and `needsApproval` must be true whenever the price is unknown; `needsApproval` should be `!(allowance > 0n && allowance >= tokenPrice)` so a zero price never satisfies it, and `useRenewalTransactions.ts:482` needs the same treatment.
4. Refuse to start a flow with `tokenPrice === 0n`. `startFlow` in both `useRegistrationTransactions.ts:301` and `useRenewalTransactions.ts:505` should throw rather than record a zero price in `savedParams`.
5. Share one query key between the checkout summary and the picker for the same `(name, duration, token)` triple, so the two surfaces cannot hold different beliefs about the price.
