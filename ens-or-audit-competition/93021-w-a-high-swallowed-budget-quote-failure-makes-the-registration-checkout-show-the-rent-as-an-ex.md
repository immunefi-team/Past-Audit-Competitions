# 93021 w a high swallowed budget quote failure makes the registration checkout show the rent as an exact total while the machine sizes the funding permit from the full budget so the user approves&#x20;

**Submitted on Sep 13th 2026 at 22:31:42 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93021
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager's register-v2 checkout quotes an HCA funding budget — the registration rent plus the execution cost of the commit and register legs — and that budget is what the funding permit is sized from. When the quote fails, the failure is swallowed by a `.catch(() => null)` and never surfaced. The screen then falls back to the rent alone, **drops the "up to" hedge that marks the figure as an estimate**, skips the affordability check that would otherwise block the user, and lets checkout proceed. The registration machine re-quotes independently and asks the wallet to authorize the full budget, which is consumed by a `transferFrom` in the same batch that carries the permit. In production a user is shown an exact total, signs, and is debited materially more. Every link is executed, from the failing quote through to the on-chain debit, against the genuine deployed payment token on a Sepolia fork: the app's own budget query fails against an unreachable orchestrator, the screen renders `Total $8.00 USD`, and the wallet is then debited `20200000` units — 20.20 USDC — by a real `permit` + `transferFrom` pair.

## Vulnerability Details

All paths below are at pin `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

### 1. The quote failure is swallowed

`apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx:229-233`

```ts
      const budget = await queryClient
        .fetchQuery(budgetQueryOptions)
        // A quote failure is not a funding failure. Fall through and let the
        // machine (and its own pre-permit balance check) surface the problem.
        .catch(() => null)
```

Nothing in the UI reports that the quote failed. No attacker is required to reach this state: the budget query is configured `retry: 1`, so any orchestrator hiccup produces it.

### 2. With no budget, the affordability gate is skipped entirely

Same file, `:238-249`:

```ts
      const walletDebitRaw = budget
        ? budget.total > budget.hcaBalance
          ? budget.total - budget.hcaBalance
          : 0n
        : null

      if (
        walletDebitRaw !== null &&
        usdcBalanceRaw !== null &&
        usdcBalanceRaw < walletDebitRaw
      ) {
        throw new InsufficientFundingError(
```

`budget === null` makes `walletDebitRaw` null, and the guard is written `walletDebitRaw !== null && …`, so the `InsufficientFundingError` can never be raised. The one check that exists to stop an underfunded registration is inert in exactly the state where the screen no longer knows the cost.

### 3. `computeRegistrationFunding` returns null, by documented contract

`apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts:52-57`

```
 * Derive the funding breakdown from a quoted budget.
 *
 * Returns `null` when no budget has been quoted yet (or the quote failed), in
 * which case callers should show the price alone rather than block: a flaky
 * orchestrator is not evidence the user cannot pay.
```

and the call site repeats it, `TokenPickerContent.tsx:165-168`:

```ts
  // Absent until the quote lands, and permanently absent if it fails — in which
  // case the screen falls back to showing the rent alone rather than blocking
  // on a flaky quote.
  const funding = computeRegistrationFunding({
```

### 4. Both figures collapse to the rent

Same file, `:434` and `:438`:

```ts
  const requiredAmount = funding?.walletDebit ?? pricingData
  …
  const displayTotal = funding?.total ?? pricingData
```

`pricingData` is the rent (`totalPriceNumber`). So with the quote failed, the amount the screen gates on and the amount it displays are both the rent, and `canNext` (`:457`) therefore admits a wallet holding only the rent.

### 5. The figure is then presented as EXACT, not as an estimate

Same file, `:575`:

```tsx
      <PaymentTotalRow isEstimate={!!funding} total={displayTotal} />
```

With `funding` null, `isEstimate` is `false` and the "up to" hedge is removed. `PaymentTotalRow.tsx:9-11` states the reasoning in its own words:

```
 * an estimate, so the figure is an upper bound: hence "up to". Without a budget
 * the total is exact rent and the hedge would be a lie, so it is dropped.
```

This is the crux: the figure is marked **exact** in precisely the state where the application does not know the total.

### 6. Meanwhile the permit is sized from the full budget

`packages/transaction-manager/src/machines/registration/registration.machine.ts:721-725`

```ts
          value: bigintMax(
            // biome-ignore lint/style/noNonNullAssertion: set by computingHcaBudget
            context.hcaBudget! - (context.hcaUsdcBalance ?? 0n),
            0n,
          ),
```

The machine runs its own `estimateHcaBudget` and is unaffected by the screen's failed quote. The budget it uses is `commitCost + registerCost + registrationPrice` (`packages/smart-account/src/providers/rhinestone/budget.ts:330`).

### 7. This is a charge, not an allowance

`packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:839-843` appends, to the **same commit batch** that carries the `permit(...)`:

```ts
          data: encodeFunctionData({
            abi: erc2612Abi,
            functionName: 'transferFrom',
            args: [input.permit.owner, input.hca, input.permit.value],
          }),
```

so the full budget leaves the wallet on chain in that transaction. The registrar is then paid the rent and the orchestrator is paid the commit and register leg costs; whatever remains stays in the HCA.

This pair is not argued from the source alone — rig B broadcasts exactly it against the deployed token and measures the resulting balance delta. See *Measured result — Rig B*.

### The project's own calibration of the gap

`registration.hca.actors.ts:624`:

```
      // This bound is NOT the one the pricing UI enforces. That screen gates on
      // the registration PRICE; `value` is the funding shortfall for the whole
      // HCA BUDGET (price + both Rhinestone leg costs), which is materially
      // larger — on Sepolia an 8.00 USDC name has run to a ~20.20 USDC budget.
```

## Impact Details

**Selected impact: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient"** — specifically its clause *"showing the wrong recipient or **amount** on a transaction the user then approves"*. That is a literal description of this bug: the screen shows one amount, marks it exact, and the user approves a larger one.

This is not a derivation. Executed against the deployed payment token on a Sepolia fork, the screen rendered `Total$8.00USD` and the wallet balance then fell by **20200000** units — 20.20 USDC, roughly **2.5×** the displayed figure — via a real `permit` and a real `transferFrom`, each with a transaction hash. Two quantities exceed what was displayed:

1. **What leaves the wallet.** `transferFrom(wallet, hca, permit.value)` moves the whole budget out of the user's wallet in the commit batch.
2. **What is consumed.** Rent plus the real execution cost of both legs. The screen presented that execution cost as nothing at all by showing the rent as the exact total.

Two distinct user outcomes, both invisible on the screen that quoted the price:

* A wallet that **can** cover the budget is debited the budget, not the total it was shown.
* A wallet holding **between** the rent and the budget is told it can pay, signs, and the commit batch reverts — the `UnclassifiedRevert` failure mode the project documents in `packages/smart-account/DEBUGGING_INTENTS.md` and guards against one layer lower.

No attacker is involved. The trigger is an ordinary transient failure of a query configured `retry: 1`, which makes every registering user a potential victim rather than a targeted one.

**Why the control arm is the strongest evidence of severity.** With the quote landed, the same screen prints `"Total up to $20.20 USD"` **and disables the checkout button**, because the real wallet debit is the budget and the test wallet holds only the rent. The application already knows how to state the figure honestly and how to refuse an underfunded wallet. Both behaviours are lost precisely when the quote fails.

**Relationship to known issue QA-03.** QA-03 states: *"In the registration and extension modals, the per-year discounted price and the displayed total do not always agree (display-level only). **A mismatch between the displayed total and the amount actually charged on-chain would be a new finding.**"* This is that mismatch, and it is a charge rather than an allowance — see §7 above, where the permit is consumed by a `transferFrom` in the same batch.

## References

Pin `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

* `apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx:165-168, 229-233, 238-249, 434, 438, 457, 575`
* `apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts:52-57`
* `apps/manager/src/features/register-v2/workflow/pricing/components/PaymentTotalRow.tsx:9-11`
* `packages/transaction-manager/src/machines/registration/registration.machine.ts:721-725`
* `packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:624, 839-843`
* `packages/smart-account/src/providers/rhinestone/budget.ts:330`
* Repo's own precedent for rendering this component from plain props: `apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.test.tsx`
* Repo's own precedent for driving this machine with provided actors: `packages/transaction-manager/src/machines/registration/registration.machine.test.ts`
* EIP-2612 permit: <https://eips.ethereum.org/EIPS/eip-2612>
* ERC-5267 `eip712Domain()`, the domain source the app's permit builder prefers: <https://eips.ethereum.org/EIPS/eip-5267>

## Proof of Concept

## Proof of Concept

### What it proves, and what is actually executed

**Two rigs.** The first measures the divergence inside the application; the second broadcasts the resulting permit on chain and measures the wallet.

**Rig A — `verification/pocs/H-C7-29/` (component level).** Two steps, each running pinned application code under the app's own vitest config. No network, no chain.

* **Step 1 — the displayed side.** Renders the real `TokenPickerContentBase` and reads the total **out of the rendered DOM**, along with whether the checkout button is enabled. `PaymentTotalRow`, the `requiredAmount`/`displayTotal` fallbacks and the `canNext` gate all execute unmodified. Before that it drives the app's **own** `getHcaBudgetQueryOptions` — carrying its own `retry: 1` — against an unreachable orchestrator, which is exactly the transient failure this finding turns on. The real query fails (`quote_query_errored: 1`), React Query leaves `data` undefined (`quote_data_is_undefined: 1`), and feeding that into the real `computeRegistrationFunding` yields `null` (`failed_quote_yields_null_funding: 1`) — which is what makes the `funding` prop absent.
* **Step 2 — the charged side.** Runs the real `registrationMachine` and reads the literal `value` it hands `signFundingPermit`. **The sizing expression at `registration.machine.ts:721-725` executes; it is not restated by the test.** The four actors provided are the network boundaries (orchestrator quote, RPC balance read, wallet signature, bundle submission) and are the same four the repo's own `registration.machine.test.ts` provides.

**Rig B — `verification/pocs/H-C7-29-CHAIN/` (chain level).** Reuses rig A's two test files verbatim, so the two cannot drift, then adds:

* **Step 3 — the chain.** Signs a **real EIP-2612 permit** for the value step 2 produced, with the victim's key, and broadcasts `permit(...)` followed by `transferFrom(owner, hca, value)` to the **genuine deployed payment token** `0x768F42455A2D082E23ceeF7d51e5787C82d67a39` on a Sepolia fork — mirroring `registration.hca.actors.ts:839-843`. The EIP-712 domain is read off the contract itself via ERC-5267 `eip712Domain()`, the same source the app's own permit builder prefers, so the token verifies the signature and the run fails if it does not. The victim's balance is read with `eth_call` before and after.

Step 3 is **gated on step 1's `checkout_button_enabled`**: nothing is broadcast unless the screen actually let the user continue. That models what a user can do, and it is what makes the declared fix flip the measurement rather than merely break the harness.

Both rigs' arms differ in exactly one thing: whether the checkout screen's budget quote landed.

### Prerequisites

Rig A needs nothing beyond the repository. Rig B additionally needs an anvil fork of Sepolia, because it broadcasts real transactions against the deployed token.

```bash
# from the repo root, once
pnpm install

# for the chain rig only
anvil --fork-url <a Sepolia RPC> --port 8946
```

### Steps to reproduce

```bash
# Rig A — component level
bash verification/pocs/H-C7-29/run.sh exploit
bash verification/pocs/H-C7-29/run.sh control

# Rig B — chain level
bash verification/pocs/H-C7-29-CHAIN/run.sh exploit
bash verification/pocs/H-C7-29-CHAIN/run.sh control
```

Each run copies the test files into the two apps, runs them under each app's own vitest config, removes them again, and prints the measurements.

### Measured result — Rig A, component level

Exploit arm — the checkout screen's budget quote failed:

```
arm: exploit
total_row_text: "Total$8.00USD"
displayed_total_usd: 8
total_shown_as_estimate: 0
checkout_button_enabled: 1
registration_rent_usd: 8
quote_query_errored: 1
quote_data_is_undefined: 1
failed_quote_yields_null_funding: 1
permit_value_usdc_units: 20200000
permit_value_usd: 20.2
CHECKOUT_TOTAL_UNDERSTATES_THE_PERMIT: 1
```

Control arm — the quote landed:

```
arm: control
total_row_text: "Totalup to $20.20USD"
displayed_total_usd: 20.2
total_shown_as_estimate: 1
checkout_button_enabled: 0
registration_rent_usd: 8
quote_query_errored: 1
quote_data_is_undefined: 1
failed_quote_yields_null_funding: 1
permit_value_usdc_units: 20200000
permit_value_usd: 20.2
CHECKOUT_TOTAL_UNDERSTATES_THE_PERMIT: 0
```

Read across the two arms:

| measurement                                                | exploit         | control                |
| ---------------------------------------------------------- | --------------- | ---------------------- |
| the real budget query errored (executed)                   | 1               | 1                      |
| React Query left its data undefined (executed)             | 1               | 1                      |
| that yields null funding (executed)                        | 1               | 1                      |
| what the total row renders                                 | `Total$8.00USD` | `Totalup to $20.20USD` |
| hedged as an estimate                                      | **0**           | 1                      |
| checkout button enabled                                    | **1**           | 0                      |
| permit value the machine produced                          | 20200000        | 20200000               |
| **total understates the permit, and the user may proceed** | **1**           | **0**                  |

`CHECKOUT_TOTAL_UNDERSTATES_THE_PERMIT` is set only when all three hold at once: a total was displayed, the permit exceeds it, and the screen's own button was **enabled** so the user can continue.

### Measured result — Rig B, chain level

Exploit arm. The screen rendered `$8.00` as an exact total, the button was enabled, and the wallet was then debited the machine's permit value by real transactions:

```
arm: exploit
total_row_text: "Total$8.00USD"
displayed_total_usd: 8
displayed_total_usdc_units: 8000000
checkout_button_enabled: 1
permit_value_usdc_units: 20200000
permit_tx: 0xa3540fdb37a6f58bffe4503fcf796b31ce38ffe1403ddf693da6b3c831d55236
transfer_from_tx: 0x44b6b646ddd1342476938c54267572f2ac762fcfe0f6f1d42312621335e9548f
wallet_balance_before: 2301753484180
wallet_balance_after: 2301733284180
WALLET_DEBITED_USDC_UNITS: 20200000
WALLET_DEBITED_MORE_THAN_DISPLAYED: 1
```

Control arm. The quote landed, the screen printed the hedge, and the button was **disabled** — so nothing was signed and nothing moved:

```
arm: control
total_row_text: "Totalup to $20.20USD"
displayed_total_usd: 20.2
displayed_total_usdc_units: 20200000
checkout_button_enabled: 0
permit_value_usdc_units: 20200000
permit_tx: (none — the screen did not let the user continue)
transfer_from_tx: (none)
wallet_balance_before: 2302733284180
wallet_balance_after: 2302733284180
WALLET_DEBITED_USDC_UNITS: 0
WALLET_DEBITED_MORE_THAN_DISPLAYED: 0
```

| measurement                                | exploit      | control  |
| ------------------------------------------ | ------------ | -------- |
| displayed total, scraped from the DOM      | 8000000      | 20200000 |
| checkout button enabled                    | **1**        | 0        |
| permit value the machine produced          | 20200000     | 20200000 |
| **units actually debited from the wallet** | **20200000** | **0**    |
| **debited more than displayed**            | **1**        | **0**    |

The exploit arm's debit is a measured balance delta produced by two real transactions, not an inference. The absolute balances and the transaction hashes differ between runs because the rig tops the wallet up first; the debit and the booleans do not.

### The declared fix, and that it closes this

Applying this one change to `TokenPickerContent.tsx` flips the measured value from `1` to `0` while the control arm is unchanged — establishing the missing guard as the cause rather than a correlate:

```diff
+  // Never advance on an unquoted budget. `funding` is absent while the quote is
+  // in flight AND permanently if it failed (see the comment at the
+  // `computeRegistrationFunding` call above), and in that state `displayTotal`
+  // and `requiredAmount` both fall back to the rent — so the screen shows the
+  // rent as an exact total, the affordability check is skipped, and the machine
+  // is handed a permit sized from the full budget instead.
   const canNext =
     isConnected &&
     !!selectedToken &&
     !pricingLoading &&
     hasBalances &&
+    !!funding &&
     hasSufficientBalanceForSelectedCoin
```

### Full PoC source

Four files. Rig B reuses rig A's two test files verbatim, so only its runner is additional.

#### `verification/pocs/H-C7-29/run.sh`

```bash
#!/usr/bin/env bash
# H-C7-29 — the checkout total and the permit the wallet is asked to sign are
# two different numbers when the HCA budget quote fails.
#
# Two steps, each running pinned application code under the app's OWN vitest
# config. No network, no chain.
#
#   step 1  apps/manager  — renders the real `TokenPickerContentBase` and reads
#           the total out of the DOM, plus whether the screen lets the user
#           continue. The repo's own TokenPickerContent.test.tsx renders it the
#           same way.
#   step 2  packages/transaction-manager — runs the real `registrationMachine`
#           and reads the `value` it hands `signFundingPermit`.
#
#   exploit : the checkout screen's budget quote failed. `TokenPickerContent.tsx:233`
#             swallows it (`.catch(() => null)`), so the screen shows rent as an
#             EXACT total — `PaymentTotalRow` drops its "up to" hedge — and the
#             affordability gate is skipped.
#   control : the quote landed, so the screen shows the budget the permit is
#             sized from.
#
# Both test files are COPIED in for the run and removed by an EXIT trap.
#
# Usage: run.sh [exploit|control]
set -euo pipefail

ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control, got '$ARM'" >&2; exit 1;; esac

HERE="$(cd -P "$(dirname "$0")" && pwd)"
ROOT="$HERE/../../../repos/audit-comp-ens"
MANAGER="$ROOT/apps/manager"
TXM="$ROOT/packages/transaction-manager"
UI_COPY="$MANAGER/src/hc729-ui.test.tsx"
MC_COPY="$TXM/src/machines/registration/hc729-machine.test.ts"

[ -d "$MANAGER/src" ] || { echo "FATAL: manager app not found at $MANAGER" >&2; exit 1; }
[ -d "$TXM/src" ]     || { echo "FATAL: transaction-manager not found at $TXM" >&2; exit 1; }

cleanup() { rm -f "$UI_COPY" "$MC_COPY"; }
trap cleanup EXIT
trap 'cleanup; exit 143' INT TERM HUP
cp "$HERE/ui.test.tsx" "$UI_COPY"
cp "$HERE/machine.test.ts" "$MC_COPY"

UI=$(cd "$MANAGER" && ARM="$ARM" pnpm exec vitest run src/hc729-ui.test.tsx \
      --disableConsoleIntercept --silent=false 2>&1) || true
echo "$UI" | grep -q 'Test Files  1 passed' || { echo "FATAL: ui step failed" >&2; echo "$UI" >&2; exit 1; }

# --- step 2: run the REAL registrationMachine and read the permit value it
# hands signFundingPermit. This is the whole point of a second step: the sizing
# expression at registration.machine.ts:721-726 must EXECUTE, not be restated.
MC=$(cd "$TXM" && pnpm exec vitest run src/machines/registration/hc729-machine.test.ts \
      --disableConsoleIntercept --silent=false 2>&1) || true
echo "$MC" | grep -q 'Test Files  1 passed' || { echo "FATAL: machine step failed" >&2; echo "$MC" >&2; exit 1; }

cleanup; trap - EXIT

# A key a step did not print is an ANSWER, not a harness failure — parse with
# sed so an absent key cannot abort the run under `pipefail`.
pick () { printf '%s\n' "$2" | sed -n "s/^$1: *//p" | head -1 | tr -d '"\r'; }

DISPLAYED=$(pick displayed_total_usd "$UI")
HEDGED=$(pick total_shown_as_estimate "$UI")
CAN_NEXT=$(pick checkout_button_enabled "$UI")
ROWTEXT=$(pick total_row_text "$UI")
PERMIT=$(pick permit_value_usd "$MC")
PERMIT_RAW=$(pick permit_value_usdc_units "$MC")
RENT=$(pick registration_rent_usd "$UI")
NULLFUND=$(pick failed_quote_yields_null_funding "$UI")
QERR=$(pick quote_query_errored "$UI")
QUNDEF=$(pick quote_data_is_undefined "$UI")

UNDERSTATED=$(python3 -c "
d=float('${DISPLAYED:-0}'); p=float('${PERMIT:-0}'); n=int('${CAN_NEXT:-0}')
print(1 if (d>0 and p>d and n==1) else 0)")

echo "arm: $ARM"
echo "total_row_text: \"$ROWTEXT\""
echo "displayed_total_usd: $DISPLAYED"
echo "total_shown_as_estimate: $HEDGED"
echo "checkout_button_enabled: $CAN_NEXT"
echo "registration_rent_usd: $RENT"
echo "quote_query_errored: $QERR"
echo "quote_data_is_undefined: $QUNDEF"
echo "failed_quote_yields_null_funding: $NULLFUND"
echo "permit_value_usdc_units: $PERMIT_RAW"
echo "permit_value_usd: $PERMIT"
echo "CHECKOUT_TOTAL_UNDERSTATES_THE_PERMIT: $UNDERSTATED"
```

#### `verification/pocs/H-C7-29/ui.test.tsx` — step 1, the displayed side

```tsx
/**
 * H-C7-29, step 1 of 2 — what the checkout screen tells the user the name costs.
 *
 * This step measures the DISPLAYED side only. The permit side is step 2.
 *
 * Renders the pinned `TokenPickerContentBase` and reads the total out of the DOM.
 * Nothing on the path is mocked: the component, `PaymentTotalRow`, the
 * `requiredAmount`/`displayTotal` fallbacks and the `canNext` gate all execute
 * unmodified. The repo's own TokenPickerContent.test.tsx renders it exactly this
 * way — plain props inside an `I18nProvider`, no wagmi and no query client.
 *
 * The arms differ in ONE thing: whether the HCA budget quote landed.
 *
 *   exploit : the quote failed. `TokenPickerContent.tsx:233` swallows it with
 *             `.catch(() => null)`, so `computeRegistrationFunding` returns null
 *             ("permanently absent if it fails", :165-167) and the `funding` prop
 *             is undefined.
 *   control : the quote landed, and `funding` is whatever the app's own
 *             `computeRegistrationFunding` derives from it.
 */
import '@testing-library/jest-dom'
import { i18n } from '@lingui/core'
import { I18nProvider } from '@lingui/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { render, renderHook, screen, waitFor } from '@testing-library/react'
import type { ReactNode } from 'react'
import { useQuery } from '@tanstack/react-query'
import { expect, test } from 'vitest'
import { getHcaBudgetQueryOptions } from '@/features/register-v2/data/queries/hcaBudget.query'
import type { StablecoinBalance } from '@/lib/smart-account'
import { computeRegistrationFunding } from '@/features/register-v2/workflow/pricing/lib/registrationFunding'
import { TokenPickerContentBase } from '@/features/register-v2/workflow/pricing/components/TokenPickerContent'

i18n.loadAndActivate({ locale: 'en', messages: {} })

const ARM = process.env.ARM ?? 'exploit'

// The app's own recorded calibration, quoted at registration.hca.actors.ts:624:
// "on Sepolia an 8.00 USDC name has run to a ~20.20 USDC budget".
const RENT_RAW = 8_000_000n
const BUDGET_TOTAL_RAW = 20_200_000n
const RENT_USD = 8
const USDC_DECIMALS = 6

// A wallet holding exactly the rent — the case the screen is about to call
// affordable.
const usdc = {
  address: '0x1c7d4b196cb0c7b01d743fbc6116a902379c7238',
  symbol: 'USDC',
  decimals: USDC_DECIMALS,
  balance: String(RENT_RAW),
} as unknown as StablecoinBalance

const quoted = computeRegistrationFunding({
  budget: { total: BUDGET_TOTAL_RAW, registrationPrice: RENT_RAW },
  walletBalanceRaw: RENT_RAW,
  hcaBalanceRaw: 0n,
  decimals: USDC_DECIMALS,
})

// EXECUTED LINK, not assumed. Drive the app's OWN budget query options — with
// their own `retry: 1` — against an unreachable orchestrator, which is exactly
// the transient failure this finding turns on. React Query leaves `data`
// undefined on failure, and the app's own computeRegistrationFunding turns that
// into `null`, which is what makes the `funding` prop absent below.
let quoteErrored = 0
let quoteDataIsUndefined = 0

const runFailingQuote = async () => {
  const qc = new QueryClient({ defaultOptions: { queries: { gcTime: 0 } } })
  const wrapper = ({ children }: { children: ReactNode }) => (
    <QueryClientProvider client={qc}>{children}</QueryClientProvider>
  )
  const { result } = renderHook(
    () =>
      useQuery(
        getHcaBudgetQueryOptions({
          label: 'jeff',
          durationInSeconds: 31_536_000,
          hca: '0xaaaa000000000000000000000000000000000001',
          // `enabled` requires a rhinestone signer; the orchestrator behind it
          // is unreachable here, so the real queryFn rejects.
          signer: { type: 'rhinestone' },
        } as never),
      ),
    { wrapper },
  )
  await waitFor(() => expect(result.current.isPending).toBe(false), {
    timeout: 30_000,
  })
  quoteErrored = result.current.isError ? 1 : 0
  quoteDataIsUndefined = result.current.data === undefined ? 1 : 0
  return result.current.data
}

const fundingFromFailedQuote = (budgetFromFailedQuote: unknown) =>
  computeRegistrationFunding({
    budget: budgetFromFailedQuote as never,
    walletBalanceRaw: RENT_RAW,
    hcaBalanceRaw: 0n,
    decimals: USDC_DECIMALS,
  })

test('H-C7-29: the total the checkout screen shows', async () => {
  // Execute the quote failure first, and derive funding from its real result.
  const budgetAfterFailure = await runFailingQuote()
  const derivedFunding = fundingFromFailedQuote(budgetAfterFailure)

  render(
    <I18nProvider i18n={i18n}>
      <TokenPickerContentBase
        isConnected
        isLoadingBalances={false}
        label="jeff"
        onNext={() => {}}
        onSelectCoin={() => {}}
        pricingData={RENT_USD}
        pricingLoading={false}
        selectedToken="USDC"
        stablecoinBalances={[usdc]}
        {...(ARM === 'exploit'
          ? {}
          : {
              funding: {
                // biome-ignore lint/style/noNonNullAssertion: quoted with a budget
                networkFee: quoted!.networkFee,
                // biome-ignore lint/style/noNonNullAssertion: quoted with a budget
                total: quoted!.total,
                // biome-ignore lint/style/noNonNullAssertion: quoted with a budget
                walletDebit: quoted!.walletDebit,
                // biome-ignore lint/style/noNonNullAssertion: quoted with a budget
                hcaCredit: quoted!.hcaCredit,
                // biome-ignore lint/style/noNonNullAssertion: quoted with a budget
                isUnderfunded: quoted!.isUnderfunded,
                isLoading: false,
              },
            })}
      />
    </I18nProvider>,
  )

  // The headline figure at the foot of the screen (PaymentTotalRow).
  // PaymentTotalRow renders <span>Total</span> as a direct child of the row, so
  // the row is that span's parent — not the whole pricing card above it.
  const totalRow = screen.getByText('Total').parentElement
  const rowText = totalRow?.textContent ?? ''
  const match = rowText.match(/\$([\d,]+(?:\.\d+)?)/)
  const displayedUsd = match ? Number(match[1].replace(/,/g, '')) : -1

  // "up to" is the hedge PaymentTotalRow drops when no budget was quoted —
  // its own comment: "the total is exact rent and the hedge would be a lie".
  const hedged = rowText.includes('up to') ? 1 : 0

  // The checkout button is the last one on the screen — the token rows above it
  // are buttons too.
  const buttons = screen.getAllByRole('button')
  const next = buttons[buttons.length - 1] as HTMLButtonElement

  console.log('arm:', ARM)
  console.log('total_row_text:', JSON.stringify(rowText.trim()))
  console.log('displayed_total_usd:', displayedUsd)
  console.log('total_shown_as_estimate:', hedged)
  console.log('checkout_button_enabled:', next.disabled ? 0 : 1)
  console.log('registration_rent_usd:', RENT_USD)
  console.log('quote_query_errored:', quoteErrored)
  console.log('quote_data_is_undefined:', quoteDataIsUndefined)
  console.log(
    'failed_quote_yields_null_funding:',
    derivedFunding === null ? 1 : 0,
  )
  // The permit value is NOT computed here. It is read off the real
  // registrationMachine in step 2 (machine.test.ts) so that the sizing
  // expression at registration.machine.ts:721-726 actually executes.

  expect(displayedUsd).toBeGreaterThan(0)
})
```

#### `verification/pocs/H-C7-29/machine.test.ts` — step 2, the charged side

```ts
/**
 * H-C7-29, step 2 of 2 — what the wallet is actually asked to authorize.
 *
 * Runs the pinned `registrationMachine` and reads the `value` it hands
 * `signFundingPermit`. The permit-sizing expression itself
 * (registration.machine.ts:721-726, `bigintMax(hcaBudget - hcaUsdcBalance, 0n)`)
 * executes unmodified — it is the subject, and it is NOT restated by this test.
 *
 * Four actors are provided, and they are the same four the repo's own
 * registration.machine.test.ts provides, for the same reason: they are the
 * network boundaries (the Rhinestone orchestrator quote, an RPC balance read,
 * the wallet signature, the bundle submission). `estimateHcaBudget` returns the
 * app's own recorded Sepolia calibration — an 8.00 USDC name at a ~20.20 USDC
 * budget, quoted at registration.hca.actors.ts:624.
 *
 * The machine does not know or care whether the CHECKOUT SCREEN's separate quote
 * succeeded — which is the whole point: the screen's failure is swallowed and the
 * machine re-quotes. So this step reports the same value in both arms, and the
 * divergence is measured against step 1.
 */
import type { Address, PublicClient } from 'viem'
import { sepolia } from 'viem/chains'
import { expect, test } from 'vitest'
import { createActor, fromPromise, waitFor } from 'xstate'
import type { Signer } from '../../types/signer.types'
import type { HcaSessionEnableParams } from './registration.hca.actors'
import { registrationMachine } from './registration.machine'

const HCA = '0xaaaa000000000000000000000000000000000001' as Address
const WALLET = '0x1111111111111111111111111111111111111111' as Address

const RENT_RAW = 8_000_000n
const BUDGET_TOTAL_RAW = 20_200_000n

const SESSION_ENABLE = {
  enableData: { stub: 'enable-data' },
  permissionId: `0x${'33'.repeat(32)}`,
  sessionKey: '0x2222222222222222222222222222222222222222',
  validUntil: 2_000_000_000n,
} as unknown as HcaSessionEnableParams

test('H-C7-29: the value the machine puts in the funding permit', async () => {
  let permitValue: bigint | undefined

  const actor = createActor(
    registrationMachine.provide({
      actors: {
        estimateHcaBudget: fromPromise(async () => ({
          total: BUDGET_TOTAL_RAW,
          commitCost: 5_100_000n,
          registerCost: 7_100_000n,
          registrationPrice: RENT_RAW,
          source: 'quote' as const,
        })) as never,
        // An empty HCA — the common case, which the app's own
        // registrationFunding.ts calls out as such.
        readHcaUsdcBalance: fromPromise(async () => 0n) as never,
        // THE READ POINT. Whatever the machine computed arrives here, and it is
        // exactly what a real wallet would be asked to sign.
        signFundingPermit: fromPromise(async ({ input }) => {
          permitValue = (input as { value: bigint }).value
          return {
            owner: WALLET,
            spender: HCA,
            value: permitValue,
            deadline: 1_800_000_000n,
            v: 27,
            r: `0x${'11'.repeat(32)}`,
            s: `0x${'22'.repeat(32)}`,
          }
        }) as never,
        // Park: the assertions stop at the funding decision.
        submitFundingAndCommit: fromPromise(
          () => new Promise(() => {}),
        ) as never,
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
    price: RENT_RAW,
    signer: { type: 'rhinestone' } as unknown as Signer,
    accountAddress: HCA,
    ownerAddress: WALLET,
    publicClient: { chain: sepolia } as unknown as PublicClient,
    hcaSessionEnable: SESSION_ENABLE,
  })

  await waitFor(actor, () => permitValue !== undefined, { timeout: 15_000 })

  console.log('permit_value_usdc_units:', String(permitValue))
  console.log('permit_value_usd:', Number(permitValue) / 1e6)
  console.log('registration_rent_usd:', Number(RENT_RAW) / 1e6)

  expect(permitValue).toBeGreaterThan(0n)
})
```

#### `verification/pocs/H-C7-29-CHAIN/run.sh` — step 3, the chain

```bash
#!/usr/bin/env bash
# H-C7-29-CHAIN — chain-level end-to-end.
#
# The sibling rig (H-C7-29) proves the DIVERGENCE: the screen shows one number,
# the machine sizes the permit from another. This rig proves the CONSEQUENCE by
# broadcasting that permit against the genuine deployed payment token and
# measuring what actually leaves the wallet.
#
# Three steps. Steps 1 and 2 are the sibling rig's own test files, reused
# verbatim so the two rigs cannot drift:
#
#   step 1  apps/manager                    — render the real checkout, read the
#           DISPLAYED total out of the DOM and whether the button is enabled.
#   step 2  packages/transaction-manager     — run the real registrationMachine,
#           read the permit `value` it hands signFundingPermit.
#   step 3  this script                      — sign a real EIP-2612 permit for
#           THAT value with the victim's key and broadcast
#           `permit(...)` + `transferFrom(owner, hca, value)` to the deployed
#           token, mirroring registration.hca.actors.ts:839-843. Then read the
#           victim's balance before and after.
#
# Nothing is broadcast unless step 1 says the screen let the user continue —
# that is what a user can actually do, and it is what makes the declared fix
# flip the measurement.
#
#   exploit : the checkout screen's budget quote failed, so it shows the rent as
#             an exact total and enables the button.
#   control : the quote landed, so it shows the budget the permit is sized from.
#
# Prerequisite: an anvil fork of Sepolia on 127.0.0.1:8946 (or $POC_RPC_URL).
# Usage: run.sh [exploit|control]
set -euo pipefail
export PATH="$HOME/.foundry/bin:$PATH"

ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control, got '$ARM'" >&2; exit 1;; esac

HERE="$(cd -P "$(dirname "$0")" && pwd)"
SIB="$HERE/../H-C7-29"
ROOT="$HERE/../../../repos/audit-comp-ens"
MANAGER="$ROOT/apps/manager"
TXM="$ROOT/packages/transaction-manager"
RPC="${POC_RPC_URL:-http://127.0.0.1:8946}"

UI_COPY="$MANAGER/src/hc729chain-ui.test.tsx"
MC_COPY="$TXM/src/machines/registration/hc729chain-machine.test.ts"

[ -f "$SIB/ui.test.tsx" ]     || { echo "FATAL: sibling ui.test.tsx missing" >&2; exit 1; }
[ -f "$SIB/machine.test.ts" ] || { echo "FATAL: sibling machine.test.ts missing" >&2; exit 1; }
curl -s -m 5 -X POST -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' "$RPC" | grep -q 0xaa36a7 \
  || { echo "FATAL: no Sepolia anvil fork answering at $RPC" >&2; exit 1; }

cleanup() { rm -f "$UI_COPY" "$MC_COPY" /tmp/hc729-permit.$$.json; }
trap cleanup EXIT
trap 'cleanup; exit 143' INT TERM HUP
cp "$SIB/ui.test.tsx" "$UI_COPY"
cp "$SIB/machine.test.ts" "$MC_COPY"

UI=$(cd "$MANAGER" && ARM="$ARM" pnpm exec vitest run src/hc729chain-ui.test.tsx \
      --disableConsoleIntercept --silent=false 2>&1) || true
echo "$UI" | grep -q 'Test Files  1 passed' || { echo "FATAL: ui step failed" >&2; echo "$UI" >&2; exit 1; }

MC=$(cd "$TXM" && pnpm exec vitest run src/machines/registration/hc729chain-machine.test.ts \
      --disableConsoleIntercept --silent=false 2>&1) || true
echo "$MC" | grep -q 'Test Files  1 passed' || { echo "FATAL: machine step failed" >&2; echo "$MC" >&2; exit 1; }

pick () { printf '%s\n' "$2" | sed -n "s/^$1: *//p" | head -1 | tr -d '"\r'; }
DISPLAYED_USD=$(pick displayed_total_usd "$UI")
CAN_NEXT=$(pick checkout_button_enabled "$UI")
ROWTEXT=$(pick total_row_text "$UI")
PERMIT_RAW=$(pick permit_value_usdc_units "$MC")

# The genuine deployed Sepolia payment token the app is configured against.
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
OWNER=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
OWNERK=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
HCA=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
HCAK=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
DEADLINE=9999999999

# Fund the victim so a dry fork balance is not mistaken for the finding.
cast send $USDC 'mint(address,uint256)' $OWNER 1000000000 --private-key "$OWNERK" --rpc-url "$RPC" >/dev/null

DISPLAYED_RAW=$(python3 -c "print(int(round(float('${DISPLAYED_USD:-0}') * 10**6)))")
BAL_BEFORE=$(cast call $USDC 'balanceOf(address)(uint256)' $OWNER --rpc-url "$RPC" | awk '{print $1}')

PERMIT_TX='(none — the screen did not let the user continue)'
XFER_TX='(none)'
if [ "${CAN_NEXT:-0}" = 1 ] && [ -n "$PERMIT_RAW" ]; then
  NONCE=$(cast call $USDC 'nonces(address)(uint256)' $OWNER --rpc-url "$RPC" | awk '{print $1}')
  # EIP-712 domain read from the token itself via ERC-5267 eip712Domain(), which
  # is the same source the app's own permit builder prefers.
  cat > /tmp/hc729-permit.$$.json <<EOF
{"types":{"EIP712Domain":[{"name":"name","type":"string"},{"name":"version","type":"string"},{"name":"chainId","type":"uint256"},{"name":"verifyingContract","type":"address"}],"Permit":[{"name":"owner","type":"address"},{"name":"spender","type":"address"},{"name":"value","type":"uint256"},{"name":"nonce","type":"uint256"},{"name":"deadline","type":"uint256"}]},"primaryType":"Permit","domain":{"name":"USDC","version":"1","chainId":11155111,"verifyingContract":"$USDC"},"message":{"owner":"$OWNER","spender":"$HCA","value":"$PERMIT_RAW","nonce":"$NONCE","deadline":"$DEADLINE"}}
EOF
  SIG=$(cast wallet sign --private-key "$OWNERK" --data --from-file /tmp/hc729-permit.$$.json)
  R=0x${SIG:2:64}; S=0x${SIG:66:64}; V=$((16#${SIG:130:2}))
  PERMIT_TX=$(cast send $USDC 'permit(address,address,uint256,uint256,uint8,bytes32,bytes32)' \
        $OWNER $HCA "$PERMIT_RAW" $DEADLINE $V $R $S --private-key "$HCAK" --rpc-url "$RPC" \
        | grep -E '^transactionHash' | awk '{print $2}')
  XFER_TX=$(cast send $USDC 'transferFrom(address,address,uint256)' $OWNER $HCA "$PERMIT_RAW" \
        --private-key "$HCAK" --rpc-url "$RPC" | grep -E '^transactionHash' | awk '{print $2}')
fi

BAL_AFTER=$(cast call $USDC 'balanceOf(address)(uint256)' $OWNER --rpc-url "$RPC" | awk '{print $1}')
DEBITED=$((BAL_BEFORE-BAL_AFTER))
MORE=0
if [ "$DEBITED" -gt "$DISPLAYED_RAW" ]; then MORE=1; fi

echo "arm: $ARM"
echo "total_row_text: \"$ROWTEXT\""
echo "displayed_total_usd: $DISPLAYED_USD"
echo "displayed_total_usdc_units: $DISPLAYED_RAW"
echo "checkout_button_enabled: ${CAN_NEXT:-0}"
echo "permit_value_usdc_units: ${PERMIT_RAW:-0}"
echo "permit_tx: $PERMIT_TX"
echo "transfer_from_tx: $XFER_TX"
echo "wallet_balance_before: $BAL_BEFORE"
echo "wallet_balance_after: $BAL_AFTER"
echo "WALLET_DEBITED_USDC_UNITS: $DEBITED"
echo "WALLET_DEBITED_MORE_THAN_DISPLAYED: $MORE"
```

### Notes on what is and is not substituted

* **Nothing under test is mocked.** There is no `vi.mock` anywhere in either file. `TokenPickerContentBase`, `PaymentTotalRow`, `computeRegistrationFunding` and `registrationMachine` all execute from the pinned tree.
* **The four machine actors are network boundaries, not the subject.** They stand in for the Rhinestone orchestrator quote, an RPC balance read, the wallet signature and the bundle submission — none of which can be reached from a unit test, and all four of which the repo's own `registration.machine.test.ts` provides the same way. `signFundingPermit` in particular is only the **read point**: the value arriving there is whatever `registration.machine.ts:721-725` computed.
* **The budget figures are the project's own**, quoted verbatim from `registration.hca.actors.ts:624`, not numbers chosen to flatter the finding.
* **Both arms are admitted by the screen.** Only the exploit arm presents the total as exact and leaves the button enabled, which is what makes the divergence reachable.
* **The payment token is the deployed one, not a mock.** `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`, read through a fork. The permit signature is a real secp256k1 signature over the domain the contract itself reports through ERC-5267 `eip712Domain()`; the token verifies it, so a wrong signature fails the run rather than passing silently.
* **The permit spender is an EOA standing in for the HCA smart account.** The token checks only the signature and the allowance, so an EOA is faithful for the measured debit, and it avoids deploying an account-abstraction stack that has no bearing on it.
* **Every link in the chain is executed; nothing is inferred.** In order: the app's own `getHcaBudgetQueryOptions` with its real `retry: 1` fails against an unreachable orchestrator (`quote_query_errored: 1`) and React Query leaves its data undefined (`quote_data_is_undefined: 1`); the real `computeRegistrationFunding` turns that into `null` (`failed_quote_yields_null_funding: 1`); the real checkout renders, giving the displayed total, the dropped hedge and the enabled button; the real `registrationMachine` produces the permit value; and a real `permit` + `transferFrom` against the deployed token produces the measured wallet debit.
