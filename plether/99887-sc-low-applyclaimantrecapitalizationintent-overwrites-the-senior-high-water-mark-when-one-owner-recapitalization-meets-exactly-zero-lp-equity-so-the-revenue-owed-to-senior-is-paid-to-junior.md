# \#99887 [SC-Low] `applyClaimantRecapitalizationIntent` overwrites the Senior high-water mark when one owner recapitalization meets exactly zero LP equity, so the revenue owed to Senior is paid to Junior

**Submitted on Nov 15th 2026 at 16:27:54 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99887
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Affected contracts (in-scope commit `a82d4fef`; live on Arbitrum Sepolia v2.0.0)

The program states that the v2.0.0 Arbitrum Sepolia release is built from a commit whose `packages/perps/src` is identical to `a82d4fef`. The defect is in `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`, which `HousePool` inlines and runs in `_reconcile` after `HousePoolReconcilePlanLib.planReconcile`. The proof of concept runs in the repository's Foundry suite at the in-scope commit, with the release parameters, Pyth mocked by the suite's `MockPyth` and USDG by the suite's mock settlement token. No in-scope contract is modified.

| Contract | Arbitrum Sepolia v2.0.0 address | Role in the issue |
|---|---|---|
| `HousePool` | `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e` | **Root cause.** `_reconcile` first absorbs the loss (Senior high-water mark kept), then `_applyPendingClaimantBucketsLive` runs the inlined `applyClaimantRecapitalizationIntent`, which **assigns** `seniorHighWaterMark = max(target, amount)` when Senior and Junior principal are both zero |
| `CfdEngine` | `0xca18a6783d7542f9d2f28f4f259772b74c4d6baa` | `recapitalizePool` and `clearDegradedMode`, the owner's documented recovery actions |
| `TrancheVault` (Senior) | `0x5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d` | Senior LP redemption, which loses the restoration revenue |
| `TrancheVault` (Junior) | `0x4a8b67184a1e3224c11a191cc69f22c50eef734e` | Junior redemption, which receives it |
| `OrderRouter` | `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444` | Public `commitOrder` / `executeOrder` / `updateMarkPrice` used for the market path and the later trading revenue |

Release configuration used (`packages/perps/DEPLOYMENT.md`): `vpiFactor` 0.01, `maxSkewRatio` 0.4, maintenance margin 0.1 %, initial margin 0.2 %, frozen-market margin 3 %, base carry 5 %, liquidation bounty 10 bps (minimum 1 USDG, 50 % keeper share), execution fee 0.04 % (asserted), Senior cap 40,000,000 USDG and 80 % of the pool (applied through the owner's timelocked setter and asserted), Senior coupon 800 bps and frozen LP fees 25/75 bps (pool defaults, asserted), Junior maintenance fee 0, 1 USDG seed in each tranche. The release Router configuration is applied through the owner's 72-hour timelocked path: `maxExecutionWindowSeconds` 60, `orderExecutionStalenessLimit` 60, `liquidationStalenessLimit` 15, `basketMaxConfidenceRatioBps` 10, `orderSettlementWindow` 15, `maxComponentPublishTimeDivergence` 5, `adverseConfidenceMultiplierBps` 2500, `minOpenNotionalAssets` 1,000 USDG, open-order bounty 1 bp (0.01–0.2 USDG), protection trigger bounty 0.2 USDG, `maxPendingOrders` 5, `minEngineGas` 600,000, `maxPruneOrdersPerCall` 64. Every Pyth mark moves at most 1 % and passes the oracle's ordering and freshness checks.

## Brief/Intro

The Senior high-water mark H is the Senior entitlement that later revenue must restore before Junior receives anything. Losses never lower it: `absorbLoss` leaves H "deliberately unchanged so future revenue continues to restore senior impairment", and the specification and whitepaper repeat that a loss does not reduce H.

`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` breaks this for a single owner recapitalization. In `_reconcile`, the loss is absorbed first. If that leaves Senior and Junior principal **both exactly zero**, the recapitalization takes a bootstrap branch that *assigns* `seniorHighWaterMark = max(recapitalization target, amount)` and discards the H that `absorbLoss` just preserved. If the loss leaves even one atom of principal, the gap branch runs instead and H survives. The same Senior shares remain outstanding in both cases, and no new Senior shares are minted.

The proof of concept drives the pool through a public market path to the price unit where terminal LP equity crosses zero. It then runs the owner's documented recovery (`recapitalizePool(200,000 USDG)`, `clearDegradedMode()`) and one public settlement. The two runs differ only by one 1e-8 price unit in the last crisis close:

- terminal LP equity **0.000000 USDG**: the stored H of 300,001.001552 USDG becomes **200,000.000000** USDG;
- terminal LP equity **0.011001 USDG**: H is kept and, with the Senior coupon checkpointed in the same reconcile, stands at **312,233.926714** USDG.

The following ordinary LP revenue of 71,970.345220 USDG then goes **entirely to Junior** in the first run and entirely to Senior in the second. The Senior LP redeems **71,966.456737 USDG less** (200,002.985257 instead of 271,969.441994), and the main Junior holder redeems **70,527.270000 USDG** instead of 0. 0.011001 USDG of LP equity decides 112,233.926714 USDG of Senior entitlement.

Impact: **High — Theft of unclaimed yield.** Revenue that the protocol owes to the impaired Senior tranche until it reaches its high-water mark is paid to Junior holders instead.

## Vulnerability Details

### Losses keep the Senior high-water mark

`HousePoolWaterfallAccountingLib.sol`, lines 161–184:

```solidity
/// @notice Absorbs loss against junior principal first and senior principal second.
/// @dev Principal subtraction saturates at zero if loss exceeds all claimed equity. The senior high-water mark is
///      deliberately unchanged so future revenue continues to restore senior impairment.
/// @param state Waterfall state before loss.
/// @param lossAssets Loss to absorb in 6-decimal settlement token.
/// @return nextState State after the junior-first loss waterfall.
function absorbLoss(
    WaterfallState memory state,
    uint256 lossAssets
) internal pure returns (WaterfallState memory nextState) {
    nextState = state;
    if (lossAssets <= nextState.juniorPrincipal) {
        nextState.juniorPrincipal -= lossAssets;
        return nextState;
    }

    uint256 seniorLoss = lossAssets - nextState.juniorPrincipal;
    nextState.juniorPrincipal = 0;
    if (nextState.seniorPrincipal > seniorLoss) {
        nextState.seniorPrincipal -= seniorLoss;
    } else {
        nextState.seniorPrincipal = 0;
    }
}
```

`ACCOUNTING_SPEC.md`, line 337: "Charge a shortfall to Junior first, then Senior. Losses do not reduce the Senior high-water mark." `WHITEPAPER.md`, lines 627–636: "Let \(H\) be the senior high-water mark: the unimpaired senior entitlement that future revenue must restore before junior receives residual value … 3. Loss does not reduce \(H\). 4. Later revenue first restores \(S\) to \(H\)." `SECURITY.md`, line 394, lists the invariant "High-water-mark protection | Senior impairment must be restored before junior extracts surplus".

### The loss is absorbed first, and the recapitalization is applied afterwards

`_reconcile` reserves the pending recapitalization out of distributable value (it is "settleable" up to distributable value, `HousePool.sol` lines 1829–1834). It plans the waterfall against the remaining, non-pending equity, stores that plan, and only then applies the recapitalization to the stored state:

`HousePool.sol`, lines 1564–1598 (abridged):

```solidity
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
    _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
...
HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
    _currentPendingAccountingState(),
    reconcileSnapshot,
    _pendingClaimantBucketAssets(settleableClaimantBuckets),
    poolConfig.seniorRateBps,
    couponElapsed,
    markFresh
);
...
_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;
...
_applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
```

In `planReconcile` the stored principal is compared with `distributable - pending` and the shortfall goes through `absorbLoss` (`HousePoolReconcilePlanLib.sol`, lines 84–105). With non-pending equity `e`, that leaves Junior at 0 and Senior at exactly `e`, with H unchanged.

### A recapitalization that meets zero principal assigns a new high-water mark

`_applyPendingClaimantBucketsLive` (`HousePool.sol`, lines 1759–1776) then calls:

`HousePoolPendingPreviewLib.sol`, lines 103–137:

```solidity
/// @notice Routes recapitalization to senior restoration or to unassigned assets.
/// @dev With senior owners and zero claimed principal, all `amount` bootstraps senior principal and sets the
///      high-water mark to `max(recapitalizationTargetAssets, amount)`. Otherwise, senior principal is restored only
///      up to its high-water mark. Any remainder, including all value when there are no senior owners, becomes
///      unassigned; recapitalization never accrues to junior principal. Mutates `state` in place.
/// @param state In-memory accounting state to update.
/// @param amount Settleable recapitalization assets to apply.
/// @param recapitalizationTargetAssets Full recapitalization intent used as the high-water mark on an empty bootstrap.
function applyClaimantRecapitalizationIntent(
    PendingAccountingState memory state,
    uint256 amount,
    uint256 recapitalizationTargetAssets
) internal pure {
    uint256 remaining = amount;
    if (state.seniorSupply > 0) {
        if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
            state.waterfall.seniorPrincipal += remaining;
            state.waterfall.seniorHighWaterMark =
                recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
            remaining = 0;
        } else {
            uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
                ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
                : 0;
            if (gap > 0) {
                uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
                state.waterfall.seniorPrincipal += seniorAssignedAssets;
                remaining -= seniorAssignedAssets;
            }
        }
    }
    if (remaining > 0) {
        state.unassignedAssets += remaining;
    }
}
```

- `e == 0`: lines 118–122 run. Senior principal becomes the recapitalization amount X, and H is **overwritten** with `max(target, X)`. With a single recapitalization, target and amount are both X. The H that `absorbLoss` preserved is lost.
- `e >= 1` atom: lines 123–131 run. Senior principal becomes `e + min(X, H - e)` and H is untouched.

The same outstanding Senior shares, the same owner action and the same recapitalization amount end with two different Senior entitlements. The only difference is whether one atom of LP equity remained when the reconcile ran. In the proof of concept the whole recapitalization is applied in that first reconcile in both runs (pending residual 0), and one more ordinary settlement before any revenue leaves H unchanged in both runs.

### The overwritten mark sends the next revenue to Junior

On the next reconcile with revenue, `distributeRevenue` restores Senior only up to H and credits the remainder to Junior (`HousePoolWaterfallAccountingLib.sol`, lines 140–159). After the overwrite, Senior already sits at its new H = X, so the whole 71,970.345220 USDG of revenue becomes Junior principal. With H intact, the same revenue restores Senior toward 312,233.926714 USDG and Junior receives nothing.

### Documented behavior and the repository tests

The NatSpec above (lines 104–107) documents the bootstrap branch, and two repository tests assert it:

- `test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol`, lines 24–55, `test_PostWipeoutRequiresExplicitRecapForHwmReset`: a 50,000 USDG Senior seed, all pool tokens burned, a 10,000 USDG recapitalization, then `assertEq(pool.seniorHighWaterMark(), recapAmount, "Explicit recapitalization should reset the HWM")`.
- the same file, lines 577–610, `test_SeniorRequiresExplicitRecapAfterFullWipeout`: "Recapitalization should seed a fresh HWM".

These tests describe the intended treatment of a tranche whose value is gone. They do not make the cliff in this report intended:

1. **Both tests create the wipe by removing every pool token directly** (`settlementToken.burn(address(pool), pool.totalAssets())`, or a `transfer` of the whole balance to `0xdead` by the pool itself). Equity is exactly zero by construction, so neither test ever looks at a pool with a residual atom. Under real trading losses, equity lands at an arbitrary value, and the branch depends on whether it rounds to exactly zero.
2. **The same file asserts the opposite principle**: lines 65–77, `test_SeniorHighWaterMarkMustSurviveFullWipeout`, assert "Senior recovery rights should survive wipeout". The public documents all state the survival rule: `ACCOUNTING_SPEC.md` line 337, `WHITEPAPER.md` lines 627–636 and `SECURITY.md` line 394. None of them mentions an exception for recapitalization.
3. **`ACCOUNTING_SPEC.md` lines 450–452** describe recapitalization as restoring the existing entitlement: "claimant recapitalization recorded through `recordClaimantInflow(amount, Recapitalization, cashMode)` may restore an existing protected senior entitlement above a reduced limit; it mints no new senior LP shares". In the zero-equity run, the existing entitlement is not restored. It is replaced by the recapitalization size, while the same Senior shares remain outstanding.
4. **The rule is discontinuous in a way no design would choose.** In the proof of concept, 0.011001 USDG of remaining LP equity decides between keeping and erasing 112,233.926714 USDG of Senior entitlement. Within the same crisis, that difference comes from one 1e-8 price unit in the last close. The supplementary library test shows that the threshold is exactly one atom (S = 10M, J = 2.5M, H = 10M, recapitalization 1M: D = R gives H = 1M, D = R + 1 wei gives H = 10M).

### Who triggers it and who benefits

The trigger is the owner's documented recovery runbook (`SECURITY.md` "Explicit terminal-deficit cascade": recapitalize, then clear degraded mode once `E >= L`), followed by any public `settleLpEpoch`. The program puts "a legitimate, documented action by a trusted role that triggers a bug harming users" in scope. The owner's recapitalization amount is an ordinary choice. It is meant to restore solvency and to restore Senior toward its entitlement. Nothing in the documentation tells the owner that the amount will also become Senior's new, lower entitlement.

The branch is decided by terminal LP equity at the first post-recapitalization reconcile. In the proof of concept that is set by the last crisis close, and no unprivileged actor steers it. The beneficiaries are the Junior holders at the time of the next revenue. They receive Senior's restoration revenue through ordinary Junior redemptions: the 980,000 USDG Junior holder redeems 70,527.270000 USDG that it would not otherwise receive. The losers are the Senior LPs, whose shares were never redeemed or diluted.

## Exploitation

All steps are in `test_SingleRecapErasesSeniorHighWaterMark_RevenueRedirectedToJunior`. LP capital, trades and settlements go through the public `TrancheVault`, `HousePool` and `OrderRouter` entry points. Pyth marks are pushed with `OrderRouter.updateMarkPrice`, one mark of at most 1 % per trading day.

1. **LPs.** Through the public request/settle/claim path, Junior receives 980,000 USDG from the main Junior holder and 4 × 5,000 USDG from four small Junior wallets. Senior receives 300,000 USDG from the Senior LP. Each tranche also holds its 1 USDG seed.
2. **Crisis (public market path).** A fully margined hedge is opened at 1.00, interleaved: three SHORT orders of 430,000 units and two LONG orders of 400,000 units (1 unit = 1 USDG notional at 1.00; the proof asserts that no position has a carry borrow base). The price rises 60 % in 48 daily marks (66 calendar days). A second LONG account opens 300,000 units at the top (1.60), and the winning SHORT closes, which latches degraded mode (asserted). The price then falls in daily 1 % marks (87 trading-day marks), until closing both LONG accounts would leave no LP equity.
3. **Crossing.** Inside the last 1 % move, the close price of both LONGs is bisected to adjacent 1e-8 price units: **0.67178025** (terminal LP equity 0.000000 USDG) and **0.67178026** (terminal LP equity 0.011001 USDG). Both runs start from one EVM snapshot. After the close, every step is identical.
4. **Owner (documented recovery).** `CfdEngine.recapitalizePool(200,000 USDG)` once, then `CfdEngine.clearDegradedMode()`. The stored H before the reconcile is 300,001.001552 USDG.
5. **First public settlement.** One small Junior wallet exits through `settleLpEpoch`, the first reconcile after the recapitalization. Zero-equity run: Senior principal 200,000.000000, **H 200,000.000000**. Positive-equity run: Senior principal 200,000.011001, **H 312,233.926714**. The increase over the stored 300,001.001552 is the Senior coupon accrued over the crisis and checkpointed in that reconcile. No recapitalization remains pending in either run.
6. A second ordinary settlement (another small Junior wallet) leaves H unchanged in both runs.
7. **Revenue.** A trader runs 20 identical losing LONG round trips of 70,000 USDG notional (five 1 % marks up, close, five back). LP revenue: 71,970.345220 USDG (zero-equity run) and 71,970.338965 USDG (positive-equity run).
8. A settlement (the third small wallet's exit) reconciles the revenue. Zero-equity run: Senior 200,000.000000, **Junior 71,970.345220**. Positive-equity run: **Senior 271,970.349966**, Junior 0.
9. The Senior LP and the main Junior holder request full redemptions, and a settlement funds them. Senior LP: **200,002.985257** USDG vs **271,969.441994** USDG. Junior holder: **70,527.270000** USDG vs 0.

`test_Control_OneAtomOfEquityKeepsHighWaterMark` runs the positive-equity run on its own. `test_LibraryOrder_OneWeiDecidesHighWaterMark` calls the real libraries in `_reconcile` order to show that the threshold is a single wei.

## Impact Details

**High — Theft of unclaimed yield.**

- **Senior LP:** −71,966.456737 USDG in redemption value (200,002.985257 vs 271,969.441994). Under the stored entitlement `min(H, S + J + U)`, Senior's shortfall is 71,970.345220 USDG. That is the whole revenue that should have restored it.
- **Junior holders:** +71,970.345220 USDG of Junior principal. The main Junior holder redeems 70,527.270000 USDG. The remaining small Junior wallets and the seed share the rest.
- **Entitlement erased:** 112,233.926714 USDG of Senior high-water mark (312,233.926714 → 200,000.000000). Every later dollar of revenue up to that amount goes to Junior instead of Senior. The proof of concept realizes 71,970 USDG of it.
- **Preconditions:** a pool loss that consumes all LP equity, which is the case the recapitalization runbook exists for, and the owner's documented recapitalization. All other steps are public. The cliff is at exactly zero: a recapitalization that meets any positive equity keeps H, and one that meets exactly zero erases it.

## Recommendation

Do not let a recapitalization lower the high-water mark of Senior shares that still exist. Remove the special case for existing Senior supply, so the zero-principal state follows the same gap logic as every other state:

```solidity
if (state.seniorSupply > 0) {
    uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
        ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
        : 0;
    uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
    state.waterfall.seniorPrincipal += seniorAssignedAssets;
    remaining -= seniorAssignedAssets;
}
```

If a fresh entitlement is wanted after a real wipeout, make it an explicit, separately documented owner decision (for example, a dedicated reset that emits an event), not a side effect of whether one atom of equity remained. At minimum, keep `seniorHighWaterMark = max(storedHighWaterMark, target, amount)` in the bootstrap branch.

Add a continuity test: for terminal LP equity `e` in {0, 1 wei, 1 USDG} and the same recapitalization, the Senior high-water mark after the first reconcile and the Senior/Junior split of later revenue must be the same, apart from `e` itself.

## Proof of Concept

Save the test below as `packages/perps/test/perps/poc/SeniorHighWaterMarkRecapReset.t.sol` and run from the repository root:

```bash
forge test --root packages/perps --match-contract RecapitalizationHighWaterMarkResetPoC -vv
```

The file contains three tests:

- `test_SingleRecapErasesSeniorHighWaterMark_RevenueRedirectedToJunior`: the zero-equity run and the positive-equity run from one snapshot, with the full comparison;
- `test_Control_OneAtomOfEquityKeepsHighWaterMark`: the positive-equity (control) run alone;
- `test_LibraryOrder_OneWeiDecidesHighWaterMark`: a supplementary illustration that calls the real libraries in `_reconcile` order. It is not an attack path.

It uses the repository's `BasePerpTest`, the suite's `MockPyth` and mock settlement token, and only public entry points apart from the owner's `recapitalizePool` and `clearDegradedMode`. There is no `vm.store`, `etch` or `mockCall` on in-scope contracts.

### Complete test:

<details>
<summary>Complete test </summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {HousePoolAccountingLib} from "@plether/perps/libraries/HousePoolAccountingLib.sol";
import {HousePoolPendingPreviewLib} from "@plether/perps/libraries/HousePoolPendingPreviewLib.sol";
import {HousePoolReconcilePlanLib} from "@plether/perps/libraries/HousePoolReconcilePlanLib.sol";
import {HousePoolWaterfallAccountingLib} from "@plether/perps/libraries/HousePoolWaterfallAccountingLib.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title One owner recapitalization of a just-wiped pool overwrites the Senior high-water mark
/// @notice Claimed impact: High - Theft of unclaimed yield (the revenue that must restore Senior to its high-water mark
///         is credited to Junior instead).
///
///         HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent (lines 118-121): when the reconcile leaves
///         S == J == 0, a recapitalization ASSIGNS seniorHighWaterMark = max(recapitalization target, amount), discarding
///         the stored high-water mark that absorbLoss deliberately preserves ("Losses do not reduce the Senior
///         high-water mark", ACCOUNTING_SPEC.md line 337). If the same reconcile leaves even one atom of equity, the gap
///         branch (lines 123-131) runs instead and the stored high-water mark survives.
///
///         Scenario (public calls only apart from the owner's documented recovery; release parameters; MockPyth and
///         mock USDG are the only mocks):
///           1. LP capital enters through the public request/settle/claim path (Senior 300k, Junior 1M).
///           2. A fully margined hedge, a +60% leg and a -32..33% leg at one <=1% Pyth mark per trading day wipe the LP
///              tranches (trader claims ~= pool cash). The final close price is bisected to the 1e-8 price unit at
///              which terminal LP equity crosses zero, so the two branches below differ by ONE price unit at the
///              close (about 0.011 USDG of equity).
///           3. The owner's documented runbook: CfdEngine.recapitalizePool(X) once, then clearDegradedMode().
///           4. A public settleLpEpoch (any LP's ordinary exit). This is the FIRST reconcile after the single recap.
///           5. Ordinary losing trades bring revenue B; one more public settlement; both LPs redeem.
///
///         ZERO-EQUITY branch (terminal equity exactly 0 before the recap): step 4 overwrites the high-water mark
///         ~312k -> X. Revenue B then goes to Junior because Senior already sits at its new high-water mark X.
///         POSITIVE-EQUITY branch (terminal equity +0.011001 USDG): step 4 keeps the high-water mark ~312k, and revenue B
///         restores Senior. No recapitalization stays pending in either branch, and one extra ordinary settlement before
///         the revenue is asserted not to move the high-water mark.
contract RecapitalizationHighWaterMarkResetPoC is BasePerpTest {

    // --- actors ---
    address internal constant SHORT_ACCOUNT = address(0xA770C2);
    address internal constant LONG_A = address(0xA770A1);
    address internal constant LONG_B = address(0xA770B1);
    address internal constant REV_TRADER = address(0x5EE0A1);
    address internal constant SENIOR_LP = address(0x5E1105);
    address internal constant JUNIOR_LP = address(0x1D1105);
    address internal constant LP_EXIT_A = address(0x1D2001);
    address internal constant LP_EXIT_B = address(0x1D2002);
    address internal constant LP_EXIT_C = address(0x1D2003);
    address internal constant LP_EXIT_D = address(0x1D2004);
    address internal constant KEEPER = address(0xE0EC);

    // --- scenario constants ---
    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant TOP_PRICE = 160_000_000; // +60% in 1% daily marks
    uint256 internal constant CROSSING_SEARCH_START = 72_000_000; // start evaluating the zero-equity crossing here
    uint256 internal constant TICK_INTERVAL = 2 minutes;
    uint256 internal constant JUNIOR_MAIN = 980_000e6;
    uint256 internal constant EXIT_WALLET_DEPOSIT = 5_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 300_000e6;
    uint256 internal constant RECAP = 200_000e6; // owner's single documented recapitalization
    uint256 internal constant BACKING_ROUNDS = 20;
    uint256 internal constant ROUND_NOTIONAL = 70_000e6; // < 40% skew cap of ~200k depth

    uint256 internal curPrice;
    uint256 internal orderNonce;

    struct Branch {
        uint256 closePrice;
        int256 equityBeforeRecap;
        uint256 hStoredBefore;
        uint256 sFirst;
        uint256 jFirst;
        uint256 hFirst;
        uint256 uFirst;
        uint256 pendingFirst;
        uint256 sExtra;
        uint256 hExtra;
        uint256 revenue;
        uint256 sFin;
        uint256 jFin;
        uint256 uFin;
        uint256 hFin;
        uint256 seniorUsdg;
        uint256 juniorUsdg;
    }

    // ------------------------------------------------------------------ configuration (release values)

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _applyReleaseRouterConfig();
        _setSeniorCapacity(40_000_000e6, 8_000);
        assertEq(pool.maxSeniorShareBps(), 8_000, "release Senior share cap");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior exposure cap");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(pool.seniorFrozenLpFeeBps(), 25, "release Senior frozen fee");
        assertEq(pool.juniorFrozenLpFeeBps(), 75, "release Junior frozen fee");
        assertEq(pool.seniorRateBps(), 800, "release Senior coupon");
        vm.warp(SETUP_TIMESTAMP + 7 days);
        curPrice = ENTRY_PRICE;
        _refreshMark(curPrice);

        uint256 jr = _requestDeposit(juniorVault, JUNIOR_LP, JUNIOR_MAIN);
        uint256 e1 = _requestDeposit(juniorVault, LP_EXIT_A, EXIT_WALLET_DEPOSIT);
        uint256 e2 = _requestDeposit(juniorVault, LP_EXIT_B, EXIT_WALLET_DEPOSIT);
        uint256 e3 = _requestDeposit(juniorVault, LP_EXIT_C, EXIT_WALLET_DEPOSIT);
        uint256 e4 = _requestDeposit(juniorVault, LP_EXIT_D, EXIT_WALLET_DEPOSIT);
        _warpTo(juniorVault.depositEpochStart(jr));
        _refreshMark(curPrice);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(juniorVault, JUNIOR_LP, jr);
        _claimDeposit(juniorVault, LP_EXIT_A, e1);
        _claimDeposit(juniorVault, LP_EXIT_B, e2);
        _claimDeposit(juniorVault, LP_EXIT_C, e3);
        _claimDeposit(juniorVault, LP_EXIT_D, e4);
        uint256 sr = _requestDeposit(seniorVault, SENIOR_LP, SENIOR_DEPOSIT);
        _warpTo(seniorVault.depositEpochStart(sr));
        _refreshMark(curPrice);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(seniorVault, SENIOR_LP, sr);
        _refreshMark(curPrice);
    }

    // ------------------------------------------------------------------ tests

    /// @notice Impact run vs control from ONE EVM snapshot. The branches differ only by one 1e-8 price unit at the last
    ///         crisis close; afterwards every step (single owner recap, settlements, revenue trades, redemptions) is
    ///         identical.
    function test_SingleRecapErasesSeniorHighWaterMark_RevenueRedirectedToJunior() external {
        (uint256 baseSnap, uint256 lo, uint256 hi) = _crisisToCrossing();
        emit log_named_decimal_uint("crossing: close price with equity <= 0 (ZERO-EQUITY branch)", lo, 8);
        emit log_named_decimal_uint("crossing: close price with equity  > 0 (POSITIVE-EQUITY branch)", hi, 8);
        assertEq(hi - lo, 1, "branches must differ by one price unit");

        Branch memory d = _runBranch(lo, "ZERO-EQUITY");
        vm.revertToState(baseSnap);
        Branch memory e = _runBranch(hi, "POSITIVE-EQUITY");

        // --- the discontinuity at the single recap's FIRST reconcile ---
        assertEq(d.equityBeforeRecap, 0, "ZERO-EQUITY branch must end the crisis at exactly 0 equity");
        assertGt(e.equityBeforeRecap, 0, "POSITIVE-EQUITY branch must end the crisis at > 0 equity");
        assertLt(uint256(e.equityBeforeRecap) + uint256(-d.equityBeforeRecap), 0.1e6, "equity gap not tiny");
        assertGe(d.hStoredBefore, SENIOR_DEPOSIT, "stored HWM before recap");
        assertEq(d.hStoredBefore, e.hStoredBefore, "same stored HWM");
        assertEq(d.hFirst, RECAP, "ZERO-EQUITY: HWM must be overwritten by the recap size");
        assertGe(e.hFirst, e.hStoredBefore, "POSITIVE-EQUITY: HWM must survive");
        assertEq(d.pendingFirst, 0, "ZERO-EQUITY: the whole recapitalization is applied in the first reconcile");
        assertEq(e.pendingFirst, 0, "POSITIVE-EQUITY: the whole recapitalization is applied in the first reconcile");
        // One more ordinary settlement before any revenue does not move the HWM in either branch.
        assertEq(d.hExtra, d.hFirst, "ZERO-EQUITY: extra settlement changed the HWM");
        assertEq(e.hExtra, e.hFirst, "POSITIVE-EQUITY: extra settlement changed the HWM");

        // --- the later revenue: Junior instead of Senior ---
        assertApproxEqRel(d.revenue, e.revenue, 0.01e18, "revenue must be comparable");
        assertLe(d.sFin, RECAP + 100e6, "ZERO-EQUITY: Senior capped at the recap-sized HWM (plus coupon dust)");
        assertGt(d.jFin, (d.revenue * 9) / 10, "ZERO-EQUITY: Junior did not receive the revenue");
        assertGt(e.sFin, RECAP + (e.revenue * 9) / 10, "POSITIVE-EQUITY: Senior was not restored by the revenue");
        assertLt(e.jFin, 1e6, "POSITIVE-EQUITY: Junior received revenue although Senior is below its HWM");
        assertGt(d.juniorUsdg, (d.revenue * 9) / 10 * 95 / 100, "ZERO-EQUITY: Junior holder did not redeem the revenue");
        assertGt(e.seniorUsdg, d.seniorUsdg + (d.revenue * 9) / 10, "Senior LP did not lose the revenue");

        uint256 total = d.sFin + d.jFin + d.uFin;
        // Stored HWM including the coupon paid in the first reconcile = the HWM the POSITIVE-EQUITY branch keeps.
        uint256 entitled = e.hFirst < total ? e.hFirst : total;
        emit log("==================== SUMMARY (same snapshot) ====================");
        emit log_named_decimal_int("ZERO-EQUITY equity before recap (USDG)", d.equityBeforeRecap, 6);
        emit log_named_decimal_int("POSITIVE-EQUITY equity before recap (USDG)", e.equityBeforeRecap, 6);
        emit log_named_decimal_uint("stored Senior HWM before the recap", d.hStoredBefore, 6);
        emit log_named_decimal_uint("owner recapitalization X (single call)", RECAP, 6);
        emit log_named_decimal_uint("ZERO-EQUITY HWM after first reconcile", d.hFirst, 6);
        emit log_named_decimal_uint("POSITIVE-EQUITY HWM after first reconcile", e.hFirst, 6);
        emit log_named_decimal_uint("ZERO-EQUITY revenue B", d.revenue, 6);
        emit log_named_decimal_uint("POSITIVE-EQUITY revenue B", e.revenue, 6);
        emit log_named_decimal_uint("ZERO-EQUITY final Senior principal", d.sFin, 6);
        emit log_named_decimal_uint("POSITIVE-EQUITY final Senior principal", e.sFin, 6);
        emit log_named_decimal_uint("ZERO-EQUITY final Junior principal", d.jFin, 6);
        emit log_named_decimal_uint("POSITIVE-EQUITY final Junior principal", e.jFin, 6);
        emit log_named_decimal_uint("ZERO-EQUITY Senior entitlement under the stored HWM min(H, S+J+U)", entitled, 6);
        emit log_named_decimal_uint("ZERO-EQUITY Senior shortfall vs stored-HWM entitlement", entitled - d.sFin, 6);
        emit log_named_decimal_uint("ZERO-EQUITY Senior LP USDG redeemed", d.seniorUsdg, 6);
        emit log_named_decimal_uint("POSITIVE-EQUITY Senior LP USDG redeemed", e.seniorUsdg, 6);
        emit log_named_decimal_uint("ZERO-EQUITY Junior holder USDG redeemed", d.juniorUsdg, 6);
        emit log_named_decimal_uint("POSITIVE-EQUITY Junior holder USDG redeemed", e.juniorUsdg, 6);
        emit log_named_decimal_uint("SENIOR LP LOSS (POSITIVE-EQUITY - ZERO-EQUITY)", e.seniorUsdg - d.seniorUsdg, 6);
        emit log_named_decimal_uint("JUNIOR HOLDER GAIN (ZERO-EQUITY - POSITIVE-EQUITY)", d.juniorUsdg - e.juniorUsdg, 6);
    }

    /// @notice Control alone: the same crisis, recapitalization and revenue, with the crisis closing one price unit higher
    ///         (terminal equity +0.011001 USDG). The stored high-water mark survives the recapitalization and the revenue
    ///         restores Senior; Junior receives nothing.
    function test_Control_OneAtomOfEquityKeepsHighWaterMark() external {
        (, , uint256 hi) = _crisisToCrossing();
        Branch memory e = _runBranch(hi, "POSITIVE-EQUITY");
        assertGt(e.equityBeforeRecap, 0, "control must end the crisis at > 0 equity");
        assertGe(e.hFirst, e.hStoredBefore, "control: HWM must survive the recapitalization");
        assertEq(e.hExtra, e.hFirst, "control: extra settlement changed the HWM");
        assertGt(e.sFin, RECAP + (e.revenue * 9) / 10, "control: Senior was not restored by the revenue");
        assertLt(e.jFin, 1e6, "control: Junior received revenue although Senior is below its HWM");
        emit log_named_decimal_uint("CONTROL HWM after first reconcile", e.hFirst, 6);
        emit log_named_decimal_uint("CONTROL Senior LP USDG redeemed", e.seniorUsdg, 6);
        emit log_named_decimal_uint("CONTROL Junior holder USDG redeemed", e.juniorUsdg, 6);
    }

    /// @notice Supplementary arithmetic illustration (NOT an attack path): the real libraries, in the order
    ///         HousePool._reconcile calls them, for S = 10M, J = 2.5M, H = 10M and a recapitalization R = 1M.
    ///         Distributable D == R gives H = 1M; D == R + 1 wei gives H = 10M.
    function test_LibraryOrder_OneWeiDecidesHighWaterMark() external {
        (uint256 s0, uint256 h0, uint256 u0) = _mirror(0);
        (uint256 s1, uint256 h1, uint256 u1) = _mirror(1);
        emit log_named_decimal_uint("D == R      : Senior principal", s0, 6);
        emit log_named_decimal_uint("D == R      : Senior HWM", h0, 6);
        emit log_named_decimal_uint("D == R + 1  : Senior principal", s1, 6);
        emit log_named_decimal_uint("D == R + 1  : Senior HWM", h1, 6);
        assertEq(h0, 1_000_000e6, "D == R must reset the HWM to R");
        assertEq(h1, 10_000_000e6, "D == R + 1 wei must keep the HWM");
        assertEq(s0, 1_000_000e6);
        assertEq(s1, 1_000_000e6 + 1);
        assertEq(u0 + u1, 0);
    }

    // ------------------------------------------------------------------ scenario

    function _runBranch(
        uint256 closePrice,
        string memory tag
    ) internal returns (Branch memory b) {
        emit log(string.concat("---------------- branch ", tag, " ----------------"));
        b.closePrice = closePrice;
        _closeBothLongsAt(closePrice);
        assertEq(_maxLiability(), 0, "positions remain");
        b.equityBeforeRecap = _equity();
        b.hStoredBefore = pool.seniorHighWaterMark();
        emit log_named_decimal_uint("crisis close price", closePrice, 8);
        emit log_named_decimal_int("terminal LP equity before recap (USDG)", b.equityBeforeRecap, 6);
        emit log_named_decimal_uint("stored Senior HWM before recap", b.hStoredBefore, 6);

        // Owner: SECURITY.md "Explicit terminal-deficit cascade" - recapitalize, then clear degraded mode.
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        engine.clearDegradedMode();

        // First public settlement after the single recap (an ordinary small Junior exit).
        _ordinaryExit(LP_EXIT_A);
        b.sFirst = pool.seniorPrincipal();
        b.jFirst = pool.juniorPrincipal();
        b.hFirst = pool.seniorHighWaterMark();
        b.uFirst = pool.unassignedAssets();
        b.pendingFirst = pool.pendingRecapitalizationAssets();
        emit log_named_decimal_uint("after FIRST reconcile: Senior principal", b.sFirst, 6);
        emit log_named_decimal_uint("after FIRST reconcile: Senior HWM", b.hFirst, 6);
        emit log_named_decimal_uint("after FIRST reconcile: Junior principal", b.jFirst, 6);
        emit log_named_decimal_uint("after FIRST reconcile: unassigned", b.uFirst, 6);
        emit log_named_decimal_uint("after FIRST reconcile: pending recap residual", b.pendingFirst, 6);

        // One more ordinary settlement before any revenue: the HWM set by the first reconcile does not move again.
        _ordinaryExit(LP_EXIT_B);
        b.sExtra = pool.seniorPrincipal();
        b.hExtra = pool.seniorHighWaterMark();
        emit log_named_decimal_uint("after extra settlement: Senior HWM", b.hExtra, 6);

        // Ordinary losing trades (LP revenue), fixed schedule identical in both branches.
        _fundTrader(REV_TRADER, 400_000e6);
        uint256 base = uint256(_equity());
        for (uint256 i; i < BACKING_ROUNDS; ++i) {
            _lossRound(ROUND_NOTIONAL);
        }
        b.revenue = uint256(_equity()) - base;
        emit log_named_decimal_uint("revenue B from ordinary losing trades", b.revenue, 6);
        IHousePool.LpEpochSettlementResult memory r = _ordinaryExit(LP_EXIT_C);
        b.sFin = pool.seniorPrincipal() + r.seniorFundedAssets;
        b.jFin = pool.juniorPrincipal() + r.juniorFundedAssets;
        b.uFin = pool.unassignedAssets();
        b.hFin = pool.seniorHighWaterMark();
        emit log_named_decimal_uint("final Senior principal", b.sFin, 6);
        emit log_named_decimal_uint("final Senior HWM", b.hFin, 6);
        emit log_named_decimal_uint("final Junior principal (incl. exiting wallet)", b.jFin, 6);
        emit log_named_decimal_uint("final unassigned", b.uFin, 6);
        (b.juniorUsdg, b.seniorUsdg) = _redeemBoth();
        emit log_named_decimal_uint("Junior holder (980k wallet) USDG redeemed", b.juniorUsdg, 6);
        emit log_named_decimal_uint("Senior LP (300k) USDG redeemed", b.seniorUsdg, 6);
    }

    /// @dev Public crisis path (fully margined hedge, +60% leg, winning shorts close -> degraded latch), then a daily
    ///      down-leg until closing both long legs would leave <= 0 LP equity; then bisection of the close price inside
    ///      the last 1% (one intraday mark move) to the single price unit where equity changes sign.
    function _crisisToCrossing() internal returns (uint256 baseSnap, uint256 lo, uint256 hi) {
        _fundTrader(SHORT_ACCOUNT, 1_330_000e6);
        _fundTrader(LONG_A, 830_000e6);
        _fundTrader(LONG_B, 500_000e6);
        _openAt(SHORT_ACCOUNT, CfdTypes.Side.SHORT, 430_000e18, 440_000e6);
        _openAt(LONG_A, CfdTypes.Side.LONG, 400_000e18, 410_000e6);
        _openAt(SHORT_ACCOUNT, CfdTypes.Side.SHORT, 430_000e18, 440_000e6);
        _openAt(LONG_A, CfdTypes.Side.LONG, 400_000e18, 410_000e6);
        _openAt(SHORT_ACCOUNT, CfdTypes.Side.SHORT, 430_000e18, 440_000e6);
        _assertNoCarryBase(SHORT_ACCOUNT);
        _assertNoCarryBase(LONG_A);

        (uint256 n, uint256 secs) = _dailyTickTo(TOP_PRICE);
        emit log_named_uint("up-leg: trading-day marks (<= 1% each)", n);
        emit log_named_uint("up-leg: calendar days", secs / 1 days);
        _openAt(LONG_B, CfdTypes.Side.LONG, 300_000e18, 490_000e6);
        _assertNoCarryBase(LONG_B);
        _closeLeg(SHORT_ACCOUNT, CfdTypes.Side.SHORT, curPrice);
        assertTrue(engine.degradedMode(), "no degraded latch");

        (n, secs) = _dailyTickTo(CROSSING_SEARCH_START);
        emit log_named_uint("down-leg to search start: trading-day marks", n);
        _assertNoCarryBase(LONG_A);
        _assertNoCarryBase(LONG_B);
        int256 eq = _equityIfLongsCloseAt(curPrice);
        assertGt(eq, 0, "search start already in deficit");
        uint256 prevSnap = vm.snapshotState();
        uint256 prevPrice = curPrice;
        while (true) {
            uint256 next = (curPrice * 99) / 100;
            curPrice = next;
            _nextTradingDay();
            _refreshMark(curPrice);
            ++n;
            eq = _equityIfLongsCloseAt(curPrice);
            if (eq <= 0) {
                break;
            }
            prevSnap = vm.snapshotState();
            prevPrice = curPrice;
            assertGt(curPrice, 50_000_000, "no crossing found");
        }
        emit log_named_uint("down-leg total trading-day marks", n);
        lo = curPrice;
        hi = prevPrice;
        vm.revertToState(prevSnap); // mark = prevPrice, both long legs open
        assertEq(curPrice, prevPrice, "revert restored the mark");
        while (hi - lo > 1) {
            uint256 mid = (lo + hi) / 2;
            if (_equityIfLongsCloseAt(mid) > 0) {
                hi = mid;
            } else {
                lo = mid;
            }
        }
        baseSnap = vm.snapshotState();
    }

    function _equityIfLongsCloseAt(
        uint256 price
    ) internal returns (int256 eq) {
        uint256 id = vm.snapshotState();
        _closeBothLongsAt(price);
        eq = _equity();
        vm.revertToState(id);
        vm.deleteStateSnapshot(id);
    }

    function _closeBothLongsAt(
        uint256 price
    ) internal {
        curPrice = price;
        _refreshMark(price);
        _closeLeg(LONG_A, CfdTypes.Side.LONG, price);
        _closeLeg(LONG_B, CfdTypes.Side.LONG, price);
    }

    /// @dev Terminal LP equity with no open positions: canonical pool assets minus trader claims.
    function _equity() internal view returns (int256) {
        return int256(pool.totalAssets()) - int256(engine.totalTraderClaimBalanceAssets());
    }

    /// @dev Pure mirror of HousePool._reconcile for one recapitalization bucket (supplementary).
    function _mirror(
        uint256 extraWei
    ) internal pure returns (uint256 s, uint256 h, uint256 u) {
        uint256 r = 1_000_000e6;
        HousePoolPendingPreviewLib.PendingAccountingState memory st;
        st.waterfall = HousePoolWaterfallAccountingLib.WaterfallState({
            seniorPrincipal: 10_000_000e6, juniorPrincipal: 2_500_000e6, seniorHighWaterMark: 10_000_000e6
        });
        st.seniorSupply = 1;
        st.juniorSupply = 1;
        HousePoolAccountingLib.ReconcileSnapshot memory snap =
            HousePoolAccountingLib.ReconcileSnapshot({distributable: r + extraWei, deficit: 0});
        HousePoolPendingPreviewLib.ClaimantPendingBuckets memory buckets =
            HousePoolPendingPreviewLib.ClaimantPendingBuckets({recapitalizationAssets: r, revenueAssets: 0});
        HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleable =
            HousePoolPendingPreviewLib.capClaimantBuckets(buckets, snap.distributable);
        HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
            st, snap, HousePoolPendingPreviewLib.claimantBucketAssets(settleable), 800, 0, true
        );
        HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(plan.state, settleable, buckets, true);
        return (plan.state.waterfall.seniorPrincipal, plan.state.waterfall.seniorHighWaterMark, plan.state.unassignedAssets);
    }

    /// @dev One ordinary losing LONG: open at the current mark, five +1% marks, close; then five -1% marks back.
    function _lossRound(
        uint256 notional
    ) internal {
        uint256 size = ((notional * 1e12 * 1e8 / curPrice) / 1e20) * 1e20;
        _openAt(REV_TRADER, CfdTypes.Side.LONG, size, notional / 10);
        for (uint256 i; i < 5; ++i) {
            _step(true);
        }
        _closeLeg(REV_TRADER, CfdTypes.Side.LONG, curPrice);
        for (uint256 i; i < 5; ++i) {
            _step(false);
        }
    }

    function _step(
        bool up
    ) internal {
        curPrice = up ? (curPrice * 101) / 100 : (curPrice * 99) / 100;
        vm.warp(vm.getBlockTimestamp() + TICK_INTERVAL);
        _refreshMark(curPrice);
    }

    function _dailyTickTo(
        uint256 target
    ) internal returns (uint256 n, uint256 secs) {
        uint256 t0 = vm.getBlockTimestamp();
        while (curPrice != target) {
            uint256 next = target > curPrice ? (curPrice * 101) / 100 : (curPrice * 99) / 100;
            if ((target > curPrice && next > target) || (target < curPrice && next < target)) {
                next = target;
            }
            curPrice = next;
            _nextTradingDay();
            _refreshMark(curPrice);
            ++n;
        }
        secs = vm.getBlockTimestamp() - t0;
    }

    function _nextTradingDay() internal {
        uint256 ts = vm.getBlockTimestamp();
        uint256 weekday = ((ts / 1 days) + 4) % 7; // 0 = Sunday ... 5 = Friday
        vm.warp(ts + (weekday == 5 ? 3 days : 1 days));
    }

    function _assertNoCarryBase(
        address account
    ) internal view {
        (uint256 borrowBase,,) = engine.positionCarryState(account);
        assertEq(borrowBase, 0, "position is not fully margined (carry would accrue)");
    }

    function _ordinaryExit(
        address wallet
    ) internal returns (IHousePool.LpEpochSettlementResult memory r) {
        uint256 shares = juniorVault.balanceOf(wallet);
        vm.prank(wallet);
        uint256 requestId = juniorVault.requestRedeem(shares, wallet, wallet);
        _warpTo(pool.lpEpochStart(requestId) + 10);
        _refreshMark(curPrice);
        vm.prank(KEEPER);
        r = pool.settleLpEpoch(0, 0); // no open positions: permissionless
    }

    function _redeemBoth() internal returns (uint256 juniorUsdg, uint256 seniorUsdg) {
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        uint256 j0 = settlementToken.balanceOf(JUNIOR_LP);
        uint256 s0 = settlementToken.balanceOf(SENIOR_LP);
        uint256 jShares = juniorVault.balanceOf(JUNIOR_LP);
        uint256 sShares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 jr = juniorVault.requestRedeem(jShares, JUNIOR_LP, JUNIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 sr = seniorVault.requestRedeem(sShares, SENIOR_LP, SENIOR_LP);
        _warpTo(pool.lpEpochStart(jr) + 10);
        _refreshMark(curPrice);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        uint256 sc = seniorVault.claimableRedeemRequest(sr, SENIOR_LP);
        assertEq(sc, sShares, "Senior redemption not fully funded");
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(sr, sc, SENIOR_LP, SENIOR_LP);
        uint256 jc = juniorVault.claimableRedeemRequest(jr, JUNIOR_LP);
        if (jc > 0) {
            vm.prank(JUNIOR_LP);
            juniorVault.claimRedeem(jr, jc, JUNIOR_LP, JUNIOR_LP);
        }
        juniorUsdg = settlementToken.balanceOf(JUNIOR_LP) - j0;
        seniorUsdg = settlementToken.balanceOf(SENIOR_LP) - s0;
    }

    // ------------------------------------------------------------------ public-path helpers

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertGt(claimable, 0, "deposit not activated");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _warpTo(
        uint256 t
    ) internal {
        if (vm.getBlockTimestamp() < t) {
            vm.warp(t);
        }
    }

    function _openAt(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("open", account, ++orderNonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        request.isClose = false;
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(KEEPER, orderId, curPrice);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open failed");
        _refreshMark(curPrice);
    }

    function _closeLeg(
        address account,
        CfdTypes.Side side,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("close", account, ++orderNonce));
        request.side = side;
        request.sizeDelta = _positionSize(account);
        request.marginDelta = 0;
        request.targetPrice = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        request.isClose = true;
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(account, orderId, price);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close failed");
        assertEq(_positionSize(account), 0, "leg not closed");
    }

    function _applyReleaseRouterConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.basketMaxConfidenceRatioBps = 10;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000;
        config.maxOpenOrderExecutionBountyAssets = 200_000;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        config.minEngineGas = 600_000;
        config.maxPruneOrdersPerCall = 64;
        _setRouterConfig(config);
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _executeAt(
        address keeper,
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 commitTime = router.lifecycleBook().orderTiming(orderId).commitTimestamp;
        uint256 publishTime = uint256(commitTime) + 1;
        if (vm.getBlockTimestamp() < publishTime) {
            vm.warp(publishTime);
        }
        _nextBlock();
        bytes[] memory tick = _ticks(price, publishTime, commitTime);
        vm.prank(keeper);
        result = router.executeOrder(orderId, tick);
    }

    function _ticks(
        uint256 price,
        uint256 publishTime,
        uint256 previousPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previousPublishTime)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        _nextBlock();
        router.updateMarkPrice(_ticks(price, vm.getBlockTimestamp(), engine.lastMarkTime()));
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

}
```

</details>

### Observed balances:

| Measurement (USDG unless noted) | Zero equity at the recapitalization (close 0.67178025) | One price unit higher (close 0.67178026) | Difference |
| --- | ---: | ---: | ---: |
| Terminal LP equity before the recapitalization | 0.000000 | 0.011001 | 0.011001 |
| Stored Senior high-water mark before the recapitalization | 300,001.001552 | 300,001.001552 | 0 |
| Owner recapitalization (single call) | 200,000.000000 | 200,000.000000 | 0 |
| Senior principal after the first reconcile | 200,000.000000 | 200,000.011001 | 0.011001 |
| **Senior high-water mark after the first reconcile** | **200,000.000000** | **312,233.926714** | **−112,233.926714** |
| Pending recapitalization after the first reconcile | 0.000000 | 0.000000 | 0 |
| Senior high-water mark after one more settlement | 200,000.000000 | 312,233.926714 | — |
| LP revenue from ordinary losing trades | 71,970.345220 | 71,970.338965 | 0.006255 |
| Final Senior principal | 200,000.000000 | 271,970.349966 | −71,970.349966 |
| Final Junior principal | 71,970.345220 | 0.000000 | +71,970.345220 |
| **Senior LP redemption** | **200,002.985257** | **271,969.441994** | **−71,966.456737** |
| **Junior holder redemption** | **70,527.270000** | **0.000000** | **+70,527.270000** |
