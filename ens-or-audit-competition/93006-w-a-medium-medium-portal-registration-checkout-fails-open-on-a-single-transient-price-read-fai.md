# 93006 w a medium medium portal registration checkout fails open on a single transient price read failure the ui freezes at 0 00 and drops every approval step while the machine charges the li&#x20;

**Submitted on Sep 13th 2026 at 20:15:09 UTC by @rector for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93006
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

A single transient failure of the registration-price read (the shared bundled dRPC endpoint's documented intermittent 500s) leaves the Explorer's name-registration checkout rendering tokens as selectable at $0.00 — no error state, no retry. The frozen $0 selection propagates into the transaction flow, where the registration machine — by design — re-reads and charges the live on-chain price: with a standing renewal allowance (the renewal flow approves 2× its quote at the same `ensEthRegistrar` spender), no approval prompt is constructed at all, so the completed charge carries no amount disclosure anywhere in the flow, and the only number the user ever sees is the post-charge "$0.00 / Paid" success banner. Demonstrated end-to-end with a 9/9 vitest PoC on the real checkout components and the real registration machine. This is exactly the class the program's own QA-03 known-issue entry carves out as a new finding ("a mismatch between the displayed total and the amount actually charged on-chain would be a new finding").

## Vulnerability Details

### Root cause

Four composing facts, each verified at the cited lines:

1. **The price query gives up on the first error.** The portal's global query client sets `retry: false` (and `staleTime` 1h) for every query — [`apps/portal/src/utils/queryClient.ts#L3-L9`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/utils/queryClient.ts#L3-L9) — and `resultQueryOptions` adds no retry override — [`packages/utils/src/tanstack-query/neverthrow/query.ts#L132-L148`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/utils/src/tanstack-query/neverthrow/query.ts#L132-L148). One transport blip leaves the registration-price query in `error` for the session.
2. **The picker gates only on `isLoading` and substitutes a $0 price for a missing one.** [`PaymentTokenPicker.tsx#L109-L115`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/components/PaymentTokenPicker.tsx#L109-L115) renders the token list the moment loading finishes — an errored query is not loading — and `buildTokenData` maps any absent/invalid price to `DEFAULT_PRICE`, whose `total` is `0n` — [`tokenData.ts#L13-L19`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/utils/tokenData.ts#L13-L19), [`:56`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/utils/tokenData.ts#L56). The failure is rendered as "selectable token at $0.00", not as an error state.
3. **Selection freezes the $0 at click time, and the CTA arms on selection alone.** Token selection is emitted only in the click handler — [`PaymentTokenPicker.tsx#L155-L158`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/components/PaymentTokenPicker.tsx#L155-L158) — and the section stores the click-time snapshot — [`PaymentTokenSection.tsx#L24-L25`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/components/PaymentTokenSection.tsx#L24-L25), [`:57`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/components/PaymentTokenSection.tsx#L57). The Register button is disabled by nothing but the absence of a selection — [`:59-L66`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/components/PaymentTokenSection.tsx#L59-L66). So one failure *before* the click freezes `tokenPrice = 0n` into the flow **even if the oracle recovers immediately after** (PoC test B proves the recovery is visible in the same screen while the CTA keeps the frozen $0).
4. **The machine layer — by design — never trusts the UI quote, and re-reads the live price at charge time.** The registration machine's own comment: *"Approve the live price, not the UI quote: the registrar pulls the CURRENT price at settlement"* — [`registration.machine.ts#L1284-L1289`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/transaction-manager/src/machines/registration/registration.machine.ts#L1284-L1289). It re-quotes the registration live (`actors.ts:592-631`), and when a standing allowance already covers the live price it **silently skips the approval step entirely** — guard `allowance >= livePrice` → straight to commitment cooldown, [`registration.machine.ts#L1273-L1281`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/transaction-manager/src/machines/registration/registration.machine.ts#L1273-L1281) — leaving the flow with no amount-bearing prompt at all.

The standing-allowance arm is structural, not hypothetical: **the portal's own renewal flow approves 2× its quote at the same `ensEthRegistrar` spender** the registration charge draws on — "Renewal approves 2× the price for headroom against price drift" — [`useRenewalTransactions.ts#L145-L157`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/renew/hooks/useRenewalTransactions.ts#L145-L157). Literal spender identity is proven in the PoC (test E1: all three sites resolve to the same address from the ensjs chain constants); after a completed renewal, the undrawn headroom is one full base-price of standing allowance at exactly the spender registrations pull from.

### Vulnerable code (branch permalinks)

(branch permalinks)

**1. Global no-retry** — [`apps/portal/src/utils/queryClient.ts#L3-L9`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/utils/queryClient.ts#L3-L9):

```ts
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 60, // 1 hour
      retry: false,
    },
  },
})
```

**2. Error ⇒ $0 substitution** — [`tokenData.ts#L13-L19`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/utils/tokenData.ts#L13-L19) + [`:56`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/utils/tokenData.ts#L56):

```ts
export const DEFAULT_PRICE: TokenPrice = { total: 0n, base: 0n, premium: 0n, ... }
// ...
price: isTokenPrice(price) ? price : DEFAULT_PRICE,   // a failed read is priced as free
```

**3. Loading-only gate + click-time selection freeze** — [`PaymentTokenPicker.tsx#L109-L115`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/components/PaymentTokenPicker.tsx#L109-L115), [`:155-L158`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/components/PaymentTokenPicker.tsx#L155-L158); CTA gate [`PaymentTokenSection.tsx#L59-L66`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/components/PaymentTokenSection.tsx#L59-L66).

**4. Approve step disappears at 0n** — [`useRegistrationTransactions.ts#L137-L140`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/register/hooks/useRegistrationTransactions.ts#L137-L140): `needsApproval = allowance < tokenPrice` — at `0n` no approval transaction is ever constructed (PoC C1/C2 contrast).

**5. The charge re-reads live and skips approval under standing allowance** — [`registration.machine.ts#L1273-L1296`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/transaction-manager/src/machines/registration/registration.machine.ts#L1273-L1296) (guard + the live-price comment + `tokenPrice := livePrice`).

**6. The sibling component shows the correct policy was known** — the multi-name renewal picker in the same app fails **closed**, with an explicit threat-model comment — [`MultiNamePaymentTokenPicker.tsx#L119-L124`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/renew/components/multi-name-extension/MultiNamePaymentTokenPicker.tsx#L119-L124):

```ts
// A settled-but-errored price read must NOT be treated as 0: that would
// understate a renewer's approval total (risking an on-chain revert) and make
// a token look affordable when it isn't. If any name's price failed to resolve,
// block selection and prompt a retry rather than price the batch wrong.
```

The single-name registration picker implements the exact opposite. The failure policy is not merely missing — it is inconsistent within the same application.

### Trigger path (step-by-step)

(step-by-step)

1. User opens the registration checkout for a name. The price query issues over the shared bundled dRPC key.
2. **One transient transport failure** (the class the repo's own fallback comments document) settles the price query in `error` — no retry, anywhere in the stack.
3. The picker renders (not loading anymore): tokens selectable at **$0.00**, balance checks against `total 0n`, no error state, no retry affordance.
4. User clicks a token — selection emits `(token, 0n)` and freezes. (Even if the oracle recovers a second later and the balance rows visibly update, the CTA below keeps the frozen $0 — PoC B.)
5. Register button enables on selection alone → `startFlow(token, 0n)` → at `0n` the portal's approval transaction is **never constructed** (`needsApproval` false).
6. The machine re-quotes the registration **live**, gets the true base+premium, and:
   * **Branch A (no standing allowance):** constructs `approve(ensEthRegistrar, live + 10%)` — the wallet popup shows a live amount (disclosure exists, weaker shape);
   * **Branch B (standing renewal allowance ≥ live price — the reported shape):** **no approval is constructed at all** — guard `allowance >= livePrice` sends the flow straight past `approvingToken`; `context.tokenPrice` stays `0n` and is never consulted; exactly one transaction (the register call, which carries no amount — the registrar pulls payment) is submitted.
7. The registration settles at the live price. The success banner renders **`$0.00 / Paid`** — the flow's only numeric surface, and it contradicts the charge rather than disclosing it.

**What fails closed (for calibration):** contract-level reverts do not trigger this — a reverted price call inside the machine's own read fails the flow (`allowFailure: false`, and the error-path `approve(0)` branch is fail-closed for funds, though it costs gas and clobbers the standing allowance); DAI is rejected pre-submission by `assertPaymentTokenSupported`. The realistic trigger is the **query layer's** first-error-settles policy — i.e., precisely the transient-blip class the indexer's fallback transport was added for.

## Impact Details

### Severity mapping

**Medium** — mapping onto the program's incorrect-transaction-detail impact row ("showing the wrong … amount on a transaction the user then approves") and squarely inside QA-03's own carve-out, which the known-issues disclosure words as:

> "Display-level only; **a mismatch between the displayed total and the amount actually charged on-chain would be a new finding.**"

This report demonstrates exactly the invited category, end to end: the displayed total is **$0.00**, the amount actually charged on-chain is the **live price**, and in the standing-allowance branch no approval prompt exists anywhere in the flow to disclose any amount — the only number the user ever sees is the post-charge **"$0.00 / Paid"** success banner.

* *Likelihood:* the arming event is **one transient transport failure on the shared bundled dRPC key** during the price read — a failure class the repository's own code documents as routine (`packages/indexer/src/chain.ts` comment: dRPC "intermittently returns `{"error":{"message":"Internal server error","code":3}}` (HTTP 500) on otherwise-valid `eth_call`s"). No attacker and no user error is needed; the global query client is configured `retry: false`, so the **first** failure settles the price query in its error state.
* *Impact:* a completed, unconsented real-token charge (USDC) whose only in-flow disclosure is a banner that asserts $0.00 was paid. Funds are charged at the genuine live price — no unintended recipient — but the amount was never disclosed at any consent point.
* Claimed Medium rather than Critical only because the charged amount is the correct live price (no overcharge, no wrong recipient), the deployment under audit is the Sepolia instance, and the arming event is an infrastructure blip rather than an attacker-controlled primitive. The fail-open shape itself — charge with zero amount surface — is demonstrated mechanically in the PoC (test D2), and the failure policy is chain-agnostic: a mainnet deployment differs only in chain constants.

### Impact

* **Unconsented real-token charge with zero amount disclosure.** In the standing-allowance branch the user's entire numeric experience of the payment is a **post-charge "$0.00 Paid"** banner (D2+F). There is no approval prompt to read, no wallet popup, no estimate — the register transaction itself carries no amount, because the registrar pulls payment against an allowance the user granted for a *different* flow (a renewal) at the same spender.
* **The arming population is every prior portal renewer.** The renewal flow's 2× headroom leaves one base-price of standing allowance at exactly the registration spender; the silent-charge bound is met at default 1yr durations whenever premium has decayed (E2's exact-equality result). No attacker, no phishing — a user who renewed a name last week and registers a new one during a dRPC blip is silently charged.
* **The app actively asserts the wrong number after the fact.** The receipt surface (success banner) contradicts the charge — the user's only in-app record says $0.00 was paid. This is worse than no disclosure: it is a false one, and it poisons every downstream dispute ("the app told me it was free" vs an on-chain USDC transfer).
* **Blast radius = the portal's registration checkout** — the primary name-purchase flow of the Explorer app, on the shared bundled RPC key, with `retry: false` globally. The failure class is documented as recurring in this repo's own transport layer.
* *(Availability footnote, fail-closed:)* the machine's error-path `approve(0)` branch resets the standing allowance on a failed run — funds-safe, but it silently destroys the renewal headroom and costs gas; noted for the fix.

### The missing invariant is systemic, not a one-off

is systemic, not a one-off

The Manager's registration funding path independently exhibits the same hole. There, a failed budget quote makes the UI render a **firm, un-hedged rent-only total** — the component's own comment: *"Without a budget the total is exact rent and the hedge would be a lie, so it is dropped"* ([`PaymentTotalRow.tsx#L8-L10`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/register-v2/workflow/pricing/components/PaymentTotalRow.tsx#L8-L10)) — while the machine signs a **full-budget permit** sized from its own fresh re-quote, and the code's own preflight error admits the funding amount *"is larger than the price shown at checkout"* ([`registration.hca.actors.ts#L627-L637`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L627-L637)). Both machines carry deliberate distrust-the-UI-quote comments; **neither UI defines what amount disclosure means when the pricing pipeline fails.** Two independent implementations of the same checkout class, one missing invariant — the register picker's fail-open is not an isolated slip but the uncovered edge of a checkout layer that has no amount-disclosure policy for pricing failure. (The two paths are distinct code and distinct roots; this report files the portal fail-open shape, whose demonstrated consequence is a completed no-disclosure charge.)

## References

### Recommendation

1. **Fail closed at the picker, mirroring the sibling that already does** — port `MultiNamePaymentTokenPicker`'s `pricesResolved` policy to the single-name registration picker: any errored price read blocks token selection and prompts a retry. A price of `0n` must never be selectable.
2. **Kill the `DEFAULT_PRICE` substitution for failed reads** — absent price is an error state (explicit retry UI), not a priced-at-free state. The `isLoading`-only gate should become `isLoading || isError`.
3. **Guard the flow entry** — `startFlow` should refuse a price that originated from a failed quote (a $0 registration price is never legitimate: base rent is always positive).
4. **Close the no-surface charge** — when the standing-allowance guard skips approval, surface the live price before submission (or require a fresh approval): no charge should complete without an amount the user was shown.

### Known-issues position (dup risk)

(known-issues position)

Checked against the program's full known-issues disclosure before filing (scope page re-scraped immediately prior):

* **QA-03** is the anchor and it carves this finding **in**: QA-03 itself covers a *display-level* price mismatch in the Explorer's registration/extension modals (per-year discount vs displayed total) and states verbatim that "a mismatch between the displayed total and the amount actually charged on-chain would be a new finding." Different app surface (Explorer modals' discount math vs the Portal register checkout's fail-open error handling), different root (arithmetic/display vs no-retry + error⇒$0 substitution + live-price charge), different consequence class (cosmetic vs completed charge). This report is the carve-out's invited category, not the fenced row.
* **WEB-1188** fences the *gas-funds warning* text — unrelated to price/charge amounts.
* The **R3-04** manager funding entry concerns the Manager's HCA funding path (referenced above as breadth context only); the filed bug is the Portal EOA checkout, a distinct root — the two are explicitly not duplicates of each other.
* The duration-stale two-quote display variant in the register modal was already fixed upstream (`key={duration}` remount, `RegisterName.tsx#L168-L172`) — distinct mechanism from the fail-open; no fix-bypass claim is made or needed (the vulnerable guard here is original code).

### Tools used

Manual source review (contrastive multi-role pipeline), vitest PoC on the real portal checkout components and the real registration machine (real charge actors, real call encoders — captured transactions decoded), literal spender equality proven from the ensjs chain constants, pure-arithmetic headroom verification with the code's own formulas, and a pre-filing anonymous re-scrape of the program's scope/known-issues page.

## Link to Proof of Concept

<https://gist.github.com/rz1989s/da39198111c7004cd72073400b36fb80>

## Proof of Concept

## Proof of Concept

**9/9 passing** (vitest 4.1.5; deterministic across 3 re-runs; full run log attached). Two additive test files over unmodified app code: the real portal checkout stack (`PaymentTokenSection` → `PaymentTokenPicker` → `PaymentTokenList`, real `useRegistrationTransactions`, real formatters) and the real registration machine with its real charge actors and both real call encoders. Numbers: base **5.00**, premium **3.00**, live total **8.00 USDC**; renewal approvals are 2× their quote.

| Test                          | World                                                                                                         | Proves                                                                                                                                                                                                                                     |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **A**                         | Price `queryFn` rejects once (the trigger); wallet holds 10.00                                                | USDC row renders **selectable** at total `0n`; Register CTA **enabled**; click emits `onConfirm(USDC, 0n)` — the exact `startFlow(token, 0n)` shape                                                                                        |
| **B**                         | Failure before click; oracle **recovers to 8.00 after** (balance rows visibly swap to "Insufficient balance") | Section keeps the click-time snapshot: CTA stays armed and still emits `(USDC, 0n)` — never `8_000_000n`. The two halves of one screen disagree                                                                                            |
| **C1 / C2**                   | `startFlow(USDC, 0n)` vs `startFlow(USDC, 8_000_000n)`, allowance 0n both                                     | At `0n` the transaction list contains **no approve step**; at 8.00 it does — the absence is the `0n` effect, not a broken hook                                                                                                             |
| **E1**                        | Pure imports, no mocks                                                                                        | Renewal approve spender === portal hook constant === machine charge spender — **literal address equality** from the ensjs chain definition                                                                                                 |
| **E2**                        | Pure arithmetic with the code's own formulas                                                                  | Renewal approve 2×base − renewal draw base = **base** headroom ≥ registration charge base + premium at default 1yr durations **at exact equality** — zero price movement is required for the silent-charge bound                           |
| **F**                         | `paid = formatPriceDisplay(0n, 6)`                                                                            | Banner renders **"$0.00 / Paid"**; the live 8.00 appears nowhere in the flow                                                                                                                                                               |
| **D1**                        | `START_REGISTRATION` carries `price: 0n`; multicall answers allowance 0n, live 8.00                           | Real machine walks to success; `context.tokenPrice` reassigned `0n → 8_000_000n`; captured approve decodes to `approve(ensEthRegistrar, 8_800_000n)` (= live + 10%); the register call carries **no amount** — the registrar pulls payment |
| **D2** *(the reported shape)* | Same, but allowance = a prior renewal's 2× headroom at the same spender                                       | Guard sends the machine straight to cooldown: **exactly one transaction submitted, zero approvals**, `tokenPrice` stays `0n` — a completed charge with **no amount surface anywhere in the flow**                                          |

**Isolation disclosure:** LEG 1 mocks four boundaries — the wagmi config (rebuilt on the same ensjs chain constants, address set asserted identical by E1), the wagmi React hooks (same seam the repo's own `ChangeResolverForm.test.tsx` fakes), the multicontract RPC read, and the price oracle `queryFn` (whose rejection is exactly the failure shape `buildTokenData` maps to `DEFAULT_PRICE`) — everything else is shipped code. LEG 2 mocks four boundaries — `viem/actions` multicall (asserted on), the wallet-submission `startTransaction` (captured, not signed), `assertPaymentTokenSupported`, and transport legs outside the charge slice — per the repo's own machine-test conventions; the charge slice (guard, both actors, both encoders) is real. No network calls; no Sepolia state touched; the target repository is unmodified (additive test files in a disposable install).

**Reproduce:** `pnpm vitest run src/features/register/components/poc-checkout-failsopen.test.tsx` (portal) and `pnpm vitest run src/machines/registration/poc-checkout-failsopen-machine.test.ts` (transaction-manager) at the pinned commit. **Full test files, run log, and live spender probe: <https://gist.github.com/rz1989s/da39198111c7004cd72073400b36fb80>** (secret gist — both legs re-run fresh immediately before filing: 9/9 passing again; the three spender constants re-resolved through the app's own unmocked modules to 0xa88553F454b77203B0D036A05c894d555EAAa2Cc, with deployed bytecode at that address confirmed on two independent Sepolia providers).

## Regression test

The PoC assertions are the contract, drop-in: after a fix, (i) an errored price read must **block selection and prompt retry** (A inverts — no selectable $0 row; the multi-name picker's `pricesResolved` gate is the in-repo reference implementation), (ii) `startFlow` must never receive a price originating from a failed quote (B/C1 invert), and (iii) no charge may complete without an amount disclosure — under standing allowance the flow must surface the live price before submission or re-auth (D2 inverts), and the success banner must never render a price the flow did not actually charge (F inverts).
