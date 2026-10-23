# 92836 w a medium payment token picker silently treats a failed price read as 0 hiding insufficient balance and skipping the real approval amount

**Submitted on Sep 12th 2026 at 18:15:57 UTC by @W3B\_SR\_0xki for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92836
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

`apps/portal`'s single-name registration and renewal flows share one `PaymentTokenPicker` component to let a user pick which stablecoin (USDC or DAI) they'll pay in. When the on-chain price read for a given token settles with an error (a transient RPC failure, an oracle revert, any real-world partial-multicall failure), the component's own data-shaping helper, `buildTokenData`, silently falls back to a hardcoded `price.total: 0n` instead of surfacing the failure. This defaults propagate into two places the user actually relies on: the "Insufficient balance" warning (which compares balance against this fabricated $0 price, so it can never fire for that token) and the price value forwarded to the confirm/approval step.

The app's own codebase shows the team already identified and fixed this exact risk once — in the **multi-name** renewal picker, whose code carries an explicit comment warning that a settled-but-errored price "must NOT be treated as 0 ... risking an on-chain revert." That fix was never applied to the **single-name** picker, which both register and renew still use.

Confirmed live, using the real, unmodified production code: a wallet holding **zero** of both supported tokens sees no insufficient-balance warning and DAI rendered as a selectable, apparently free option, because its real price was never successfully read.

## Assets Affected

* `apps/portal/src/features/register/utils/tokenData.ts:34-44` (root cause — `isTokenPrice`/`buildTokenData`)
* `apps/portal/src/features/register/components/PaymentTokenPicker.tsx:92-153` (consumer — shared by register and renew)
* `apps/portal/src/features/register/components/PaymentTokenSection.tsx:42-46` (forwards the fabricated price onward as `tokenPrice`)
* Sibling that already has the fix: `apps/portal/src/features/renew/components/multi-name-extension/MultiNamePaymentTokenPicker.tsx`

## Vulnerability Details

### The fallback

```ts
// apps/portal/src/features/register/utils/tokenData.ts:13-19,34-44
export const DEFAULT_PRICE: TokenPrice = {
  total: 0n,
  base: 0n,
  premium: 0n,
  decimals: USDC_DECIMALS,
  hasPremium: false,
}

function isTokenPrice(value: TokenPrice | undefined): value is TokenPrice {
  if (typeof value !== 'object' || value === null) return false
  ...
}

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
      ...
    }
  })
}
```

A TanStack Query that has settled into an error state reports `data: undefined` — indistinguishable, to `isTokenPrice`, from "still loading" or "never asked." Both cases fall back to `DEFAULT_PRICE` (`total: 0n`).

### The loading gate only checks `isLoading`, never `isError`

```ts
// apps/portal/src/features/register/components/PaymentTokenPicker.tsx:109-115
if (
  balancesQuery.isLoading ||
  allowancesQuery.isLoading ||
  priceQueries.some((query) => query.isLoading)
) {
  return <Skeleton />
}
```

Once a price query **settles** — successfully or with an error — `isLoading` becomes `false` and the component proceeds past the skeleton, straight into `buildTokenData` with `query.data === undefined` for the failed token.

### The consequence: the fabricated $0 drives both the warning and the confirm step

```ts
// PaymentTokenPicker.tsx:144-153
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

For the errored token, `price.total` is `0n`, so `balance < 0n` is `false` for any real (non-negative) ERC-20 balance — **including a balance of exactly zero**. The token is treated as affordable regardless of what the wallet actually holds.

```ts
// PaymentTokenSection.tsx:42-46
const handleBuyName = () => {
  if (selectedTokenData) {
    onConfirm(selectedTokenData.address, selectedTokenData.price.total)
  }
}
```

If the user selects that token, `0n` — not the real, unfetched price — is what gets forwarded onward as `tokenPrice`, the value the rest of the confirm/approval flow treats as authoritative.

### The team already fixed this exact class of bug — just not here

```ts
// apps/portal/src/features/renew/components/multi-name-extension/MultiNamePaymentTokenPicker.tsx
// (paraphrased from its own comment)
// "A settled-but-errored price read must NOT be treated as 0 ... risking
//  an on-chain revert" — blocks selection and shows a retry warning
//  instead of defaulting silently.
```

The multi-name picker explicitly distinguishes "still loading," "settled successfully," and "settled with an error," and refuses to proceed on the last case. The single-name picker — used by both register and renew, per the shared-component design — has no such distinction.

## Impact

Any user attempting to register or renew a name during a transient price- oracle/RPC failure for one supported token sees that token presented as freely selectable with no balance warning, even holding zero of it, and with a $0 price silently substituted for the real rent cost. This is a genuine render/sign mismatch: what the confirmation UI implies ("this token needs no balance, no warning") does not match the real, unfetched on-chain price the eventual `register`/`renew` call would actually require. If the user happens to hold a pre-existing token allowance from an earlier interaction that already covers the real price, the flow's approval-needed check (computed against the fabricated $0) would also be satisfied trivially, letting the transaction proceed straight to submission without ever confirming or displaying the real amount about to be spent.

## Recommendation

* In `buildTokenData` (or its caller), distinguish "price query still loading," "price query succeeded," and "price query settled with an error" — the way `MultiNamePaymentTokenPicker.tsx` already does — and never substitute `DEFAULT_PRICE` for the error case.
* Extend `PaymentTokenPicker.tsx`'s loading gate to also check `priceQueries.some(query => query.isError)`, blocking selection (or excluding that token) and surfacing a retry affordance, matching the multi-name picker's existing behavior.
* Since this exact fix already exists in one sibling component, consider extracting the multi-name picker's price-resolution logic into a shared helper both pickers use, so a future fix doesn't need to be applied twice again.

## Proof of Concept

## Proof of Concept

Executed against the real, unmodified production chain — `useQueries` + `getRegistrationPriceQueryOptions` (both from `PaymentTokenPicker.tsx` itself) and `buildTokenData` — mocking only the one true external boundary: ensjs's `getRegisterPrice`, the actual on-chain price read, made to genuinely throw for DAI (simulating a real RPC/oracle failure) while USDC succeeds normally.

```ts
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { renderHook, waitFor } from '@testing-library/react'
import type { ReactNode } from 'react'
import { useQueries } from '@tanstack/react-query'
import { describe, expect, it, vi } from 'vitest'
import { SUPPORTED_TOKENS } from '@/lib/constants/tokens'
import { PAYMENT_TOKENS } from '@/features/register/constants/paymentTokens'
import { getRegistrationPriceQueryOptions } from '@/features/register/hooks/useRegistrationPrice'
import { buildTokenData } from '@/features/register/utils/tokenData'

const REAL_DAI_ADDRESS = SUPPORTED_TOKENS.DAI.toLowerCase()

vi.mock('@ensdomains/ensjs/public/v2', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@ensdomains/ensjs/public/v2')>()
  return {
    ...actual,
    getRegisterPrice: vi.fn(async (_client, params) => {
      if (params.paymentToken.toLowerCase() === REAL_DAI_ADDRESS) {
        throw new Error('oracle read reverted for DAI (simulated RPC failure)')
      }
      return { base: 8_000_021n, premium: 0n } // real-shaped USDC price
    }),
  }
})

vi.mock('@/lib/wagmi/helpers', async (importOriginal) => {
  const { ok } = await import('neverthrow')
  const actual = await importOriginal<typeof import('@/lib/wagmi/helpers')>()
  return { ...actual, safeGetClient: () => ok({}) }
})

const createWrapper = () => {
  const queryClient = new QueryClient({ defaultOptions: { queries: { retry: false } } })
  return function Wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  }
}

describe('PaymentTokenPicker PoC — errored price silently becomes $0', () => {
  it('buildTokenData defaults DAI to price=0 after its real price query errors, hiding the insufficient-balance warning at 0 balance', async () => {
    // Exactly PaymentTokenPicker.tsx:92-106's own priceQueries construction.
    const { result } = renderHook(
      () =>
        useQueries({
          queries: PAYMENT_TOKENS.map((token) =>
            getRegistrationPriceQueryOptions({
              name: 'pocname.eth',
              duration: 31536000,
              token: token.address,
            }),
          ),
        }),
      { wrapper: createWrapper() },
    )

    await waitFor(() => {
      expect(result.current.every((q) => !q.isLoading)).toBe(true)
    })

    const usdcQuery = result.current[0]
    const daiQuery = result.current[1]

    expect(daiQuery.isError).toBe(true)
    expect(daiQuery.data).toBeUndefined()
    expect(usdcQuery.isError).toBe(false)
    expect(usdcQuery.data).toBeDefined()

    // PaymentTokenPicker.tsx:109-115's own gate: only checks isLoading.
    const wouldShowSkeleton = result.current.some((q) => q.isLoading)
    expect(wouldShowSkeleton).toBe(false)

    // PaymentTokenPicker.tsx:144-149's own call, verbatim.
    const balances = [0n, 0n]
    const allowances = [0n, 0n]
    const tokenData = buildTokenData(
      PAYMENT_TOKENS,
      result.current.map((q) => q.data),
      balances,
      allowances,
    )

    const dai = tokenData.find((t) => t.symbol === 'DAI')!
    expect(dai.price.total).toBe(0n)

    // PaymentTokenPicker.tsx:151-153's own exact check, verbatim.
    const noSupportedTokenHasSufficientBalance = tokenData.every(
      (token) => token.balance < token.price.total,
    )
    expect(noSupportedTokenHasSufficientBalance).toBe(false)
  })
})
```
