# \#99745 [SC-Critical] Refundable open VPI is priced into LP exits as final revenue, letting a Junior LP who trades steal other LPs' funds

**Submitted on Nov 15th 2026 at 04:25:59 UTC by @itsactuallyomar for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99745
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An open's positive VPI (the price-impact charge) is booked as LP revenue the moment the position opens, and the LP share price has no term for it, although the same position's close can rebate up to the whole charge out of pool cash. A Junior LP that also trades opens a skew-widening position just before its own redemption is funded, redeems at a price that contains the charge, then closes and is rebated the whole charge out of the cash of the LPs who stayed. On a market deployed with the release parameters, the PoC's attacker ends with 2,779.027696 USDG more than all the USDG it brought, in one cycle and with no price move, and the other LPs lose principal: an LP that entered during the window exits 6,423.466462 USDG below its 6,500,000 deposit (6,228.351096 below its exit without the attack), and after three cycles a stayer exits 2,732.868082 USDG below its 1,000,000 deposit.

## Vulnerability Details

**1. An open books its VPI as pool revenue at once.** `executeOpen` records the trade cost, minus the execution fee credited to the treasury, as `Revenue` with `CashArrived` (`packages/perps/src/CfdEngineSettlementSidecar.sol:454-464`):

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

`recordClaimantInflow` adds the amount to `accountedAssets` (`HousePool.sol:640-641`), and `totalAssets()` is the lesser of the raw balance and `accountedAssets` (`HousePool.sol:426-429`), so pool assets rise by the charge immediately. The same amount becomes the position's lifetime VPI (`CfdEngineSettlementSidecar.sol:517`):

```solidity
vpiAccrued: currentPosition.vpiAccrued + delta.posVpiAccruedDelta,
```

**2. The LP price treats the charge as final.** Every Junior entry and exit is priced from one reconcile snapshot: pool assets minus trader claims, plus or minus the marked price delta (`HousePoolAccountingLib.sol:60-78`, fed by `CfdEngineProtocolLens.sol:59-73`, where `physicalAssets = pool.totalAssets()`):

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
...
if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
```

There is no term for VPI that an open position can still take back. The epoch settlement reconciles (`HousePool.sol:835`), prices Junior redemptions at the reconciled `juniorPrincipal` (`HousePool.sol:840-841`), and then prices Junior deposits at the principal left after the exits (`HousePool.sol:914-919`). Revenue goes to Junior once Senior is at its high-water mark (`HousePoolWaterfallAccountingLib.sol:140-159`).

**3. The same position takes the charge back at its close, out of pool cash.** The close gives the closed part its share of lifetime VPI, evaluates a fresh curve delta at the current skew and depth, and clamps only so that lifetime VPI cannot go below zero (`CloseAccountingLib.sol:103-118`):

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
...
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

A close that narrows the skew is therefore rebated up to the whole charge. The planner nets the VPI delta with the execution fee, the frozen spread and carry, and treats a negative result as a rebate (`CfdEnginePlanLib.sol:1036-1061`), pays it from free pool cash (`CfdEnginePlanLib.sol:970-978`), and the sidecar pays it out of the pool (`CfdEngineSettlementSidecar.sol:621-624`; `payOut` lowers `accountedAssets` and transfers, `HousePool.sol:604-613`).

Between an open and its close, a charge that the position can still reclaim sits in Junior principal as if it were final. An LP that exits in that window is paid a share of it, an LP that enters in that window pays for a share of it, and the LPs in the pool at the close fund the refund.

**The attack.** It needs no role and no flash loan: order execution and LP epoch settlement are permissionless (`OrderRouter.sol:233-247`, `OrderRouter.sol:474-484`).

1. The attacker deposits into Junior. After the 1 h deposit cooldown (`TrancheVault.sol:38`) it queues a full redemption, which matures at the next hourly epoch boundary (`HousePool.sol:178`).
2. Shortly before that boundary it opens a skew-widening position and executes the order itself. The VPI is booked as revenue.
3. The epoch is settled, and the redemption is paid at a Junior principal that contains the charge.
4. It closes the position. The close's VPI is minus the whole charge, and the pool pays the rebate.

The attacker is paid its own charge twice: its share of it through the redemption price, and all of it through the rebate. The LPs who stay pay the difference.

## Impact Details

Selected impact: **Critical, "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield"**. The funds taken are other LPs' deposits held by `HousePool`.

All figures are from the PoC, in USDG, on unmodified contracts deployed and configured as the repository's deploy script does (release parameters: `vpiFactor` 1 %, skew cap 40 % of depth, execution fee 4 bps), on a market seeded through public entry points with 12,000,303.713891 of LP assets (Junior principal 4,000,000.999982).

**Principal loss.**
- An LP that deposits 6,500,000 into Junior, activated in the attacker's exit settlement, exits with 6,493,576.533538: 6,423.466462 below its deposit, and 6,228.351096 below its exit in the same timeline without the attack (`test_depthKept`).
- After three attack cycles, the stayer `lpJunior2` (1,000,000 deposited) exits with 997,267.131918: 2,732.868082 below its deposit, and 3,954.737551 below its exit without the attack (`test_repeatedCycles`).
- After one cycle the same stayer exits with 999,970.990600, against 1,001,287.649740 without the attack: 1,316.659140 less, and below its deposit (`test_attack`).

**Attacker profit (`test_attack`).** The attacker holds 60 % of Junior (6,000,000 deposited). Its 6,000,000 LONG, opened 58 s before the settlement that funds its redemption, moves the book skew from 855,100.246500 to 6,855,087.176118 (3,808 bps of depth; the cap is 4,000 bps) and pays 12,849.948255 of VPI. The redemption pays 6,007,634.974119, against 5,999,912.373449 without the trade (7,722.600670 more). The close's VPI is -12,849.948255, and the pool pays a net rebate of 10,449.942343 in full, after withholding 2,400.005912 of close fee and carry. The attacker brought 6,610,000.000000 and holds 6,612,779.027696 at the end: **+2,779.027696 absolute**, and +2,866.654247 against the no-trade control, after both execution fees. All other LPs together lose 5,266.637877; the test asserts that this equals the attacker's gain plus the treasury's execution fee on the open (2,399.983630). Senior principal is unchanged (0.000000): the loss falls on Junior.

**Scale and repetition.**
- At the release Senior cap (Senior 39,990,220.179461 and Junior 11,505,210.096430 before the attacker), the attacker gains +9,355.786495 per cycle (+8,917.607346 absolute) and the other LPs lose 19,460.247941 (`test_releaseScale`).
- Three consecutive cycles (32,389 s) end at +2,779.027696, +2,785.365753 and +2,791.719508 (`test_repeatedCycles`).
- A day-long hold also pays: with the redemption funded 89,997 s after the open, the gain is +2,505.098748 and the stayer exits 1,226.270288 lower, below its deposit (`test_hold25h`).
- Depositing just before the open works too: +3,647.114276 over the deposit-only timeline, with the stayer 1,711.815978 lower (`test_freshDepositPath`).
- The transfer itself needs no attacker: an LP that exits while an unrelated trader's skew-widening LONG is open is paid 2,185.420985 more, and a stayer 206.386339 less, than without the trade (`test_passiveExit`).

**Conditions.**
- The attacker needs a large share of Junior. The gain grows with the share, and break-even lies between the sweep's rows at 35 % (gain -172.923923) and 40 % (3,999 bps, gain +319.285320); the stayer pays in every row (`test_juniorShareSweep`). The programme's rules state that "A high capital requirement is generally not a valid reason to downgrade a report" and that feasibility limitations "do not change the impact or severity of a finding".
- It is not an artefact of a flat price: over 32 seeded random walks of the basket (sigma 0.17 bp per sqrt(second), price exposure unhedged), every seed gains over its control and ends in absolute profit; the two 16-seed halves have mean gains of +2,649.732224 and +2,849.615808, and the worst seed gains +1,783.778435 (`test_movingBasket_seeds01to16`, `test_movingBasket_seeds17to32`).
- It is not an artefact of a tight oracle confidence: on a flat basket, at per-feed Pyth confidence of 0.25, 2, 5, 8 and 10 bp of price, the attack gains +2,866.654247, +2,492.896384, +1,857.834455, +1,229.215221 and +805.924345 over its same-confidence control, ends in absolute profit at every level, and the stayer loses 1,316.659140 down to 801.329902 against its control; the last level sits at the oracle's 10 bp admission cap (`PletherOracle.sol:869-873`; aggregate 997 hundredths of a bp), the widest confidence the oracle admits (`test_widerPythConfidence`).

**Nearest known items.**
- The AI-scan item AI-31, "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" (`audits/scan-plether-fi-plether-core-d4541b49.md:1742`, a known issue under programme known issue #2), is about temporarily cutting pool depth to enlarge a skew-reducing close's rebate, whereas here the depth change adds only 9.047481 to the rebate (31 bps of the gain; +2,857.606766 without it, `test_depthCutShareOfRebate`), the attack still gains +2,516.079004 and the entrant still loses principal when depth at the close exceeds depth at the open (18,505,518.688027 against 18,000,303.713891) and the clamp does not bind (close VPI -12,499.250615, `test_depthKept`), and the fix below, which adds no depth guard and only defers when an open's VPI becomes revenue, makes every attack test fail.
- M10 in the repository's history ("JIT LP attack via unrealized PnL", commit `e93999ee`: "Mitigated by C3 deposit cooldown (1-hour capital lockup). Full fix (streaming revenue) deferred to V2.") names a different cause: here the attacker waits out the cooldown, the basket is flat so there is no unrealized PnL, the cause is a refundable charge booked as final revenue, and a hold that outlasts a 24 h streaming window still pays (`test_hold25h`; a supplementary check with a streaming sketch applied is at the end of this description).
- The documented collection-time accounting ("Positive VPI increases physical pool assets", `WHITEPAPER.md:411-412`) is not what this report disputes: the bug is that a collected charge the position can still reclaim is paid out as final, which breaks the stated property that new LPs "do not inherit old trader obligations at a higher deposit NAV" (`SECURITY.md:859-860`; the entrant in `test_depthKept`), while the documents accept a collection-time cohort transfer in words for carry (`ACCOUNTING_SPEC.md:737-740`) and keep the opposite-sign VPI reserve out of the price "so LPs cannot withdraw it as marked equity before the matching position obligation settles" (`WHITEPAPER.md:887-888`).

## Recommendation

Keep a position's positive lifetime VPI out of LP equity until that position releases it. At an open, record as revenue only the part of the trade cost that is not a new increase in positive lifetime VPI. When the position lowers its lifetime VPI (a reduce, a close, a liquidation), record the released part as revenue before any rebate is paid.

```diff
diff --git a/packages/perps/src/CfdEngineSettlementSidecar.sol b/packages/perps/src/CfdEngineSettlementSidecar.sol
index 64a4710..6e123f1 100644
--- a/packages/perps/src/CfdEngineSettlementSidecar.sol
+++ b/packages/perps/src/CfdEngineSettlementSidecar.sol
@@ -378,6 +378,38 @@ contract CfdEngineSettlementSidecar is ICfdEngineSettlementSidecar {
         }
     }
 
+    function _positiveVpiPart(
+        int256 vpiAssets
+    ) private pure returns (uint256) {
+        return vpiAssets > 0 ? uint256(vpiAssets) : 0;
+    }
+
+    /// @dev A position's positive lifetime VPI is refundable by that position, so its cash stays outside
+    ///      LP NAV (unrecorded pool cash) until the position releases it (reduce, close, liquidation, rebate on open).
+    function _deferredVpiIncrease(
+        int256 beforeAssets,
+        int256 afterAssets
+    ) private pure returns (uint256) {
+        uint256 b = _positiveVpiPart(beforeAssets);
+        uint256 a = _positiveVpiPart(afterAssets);
+        return a > b ? a - b : 0;
+    }
+
+    function _recognizeDeferredVpi(
+        ICfdEngineSettlementHost host,
+        int256 beforeAssets,
+        int256 afterAssets
+    ) private {
+        uint256 b = _positiveVpiPart(beforeAssets);
+        uint256 a = _positiveVpiPart(afterAssets);
+        if (b > a) {
+            IHousePool(host.pool())
+                .recordClaimantInflow(
+                    b - a, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
+                );
+        }
+    }
+
     function _liveMark(
         ICfdEngineAccountActionView engine,
         ICfdEngineSettlementHost host
@@ -439,6 +471,7 @@ contract CfdEngineSettlementSidecar is ICfdEngineSettlementSidecar {
             );
         }
 
+        _recognizeDeferredVpi(host, currentPosition.vpiAccrued, currentPosition.vpiAccrued + delta.posVpiAccruedDelta);
         if (delta.poolRebatePayoutAssets > 0) {
             IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
         }
@@ -452,7 +485,8 @@ contract CfdEngineSettlementSidecar is ICfdEngineSettlementSidecar {
             delta.executionFeeAssets
         );
         if (delta.tradeCostAssets > 0) {
-            uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
+            uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets
+                - _deferredVpiIncrease(currentPosition.vpiAccrued, currentPosition.vpiAccrued + delta.posVpiAccruedDelta);
             if (poolCashInflowAssets > 0) {
                 IHousePool(host.pool())
                     .recordClaimantInflow(
@@ -618,6 +652,11 @@ contract CfdEngineSettlementSidecar is ICfdEngineSettlementSidecar {
         if (delta.pricePayoutAssets > 0) {
             host.settlementRecordTraderClaim(delta.account, delta.pricePayoutAssets, !delta.deletePosition);
         }
+        _recognizeDeferredVpi(
+            host,
+            currentPosition.vpiAccrued,
+            delta.deletePosition ? int256(0) : currentPosition.vpiAccrued - delta.posVpiAccruedReduction
+        );
         if (delta.actionRebatePaidAssets > 0) {
             IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
             clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
@@ -783,6 +822,10 @@ contract CfdEngineSettlementSidecar is ICfdEngineSettlementSidecar {
         ICfdEngineSettlementHost host = ICfdEngineSettlementHost(msg.sender);
         host.settlementApplyCarryAndMark(delta.price, publishTime);
         IMarginClearinghouse clearinghouse = IMarginClearinghouse(host.clearinghouse());
+        {
+            (,,,,,, int256 vpiBeforeLiquidation) = ICfdEngineAccountActionView(ENGINE).positions(delta.account);
+            _recognizeDeferredVpi(host, vpiBeforeLiquidation, 0);
+        }
         uint256 marginBefore = clearinghouse.pnlPledgeAssets(delta.account);
         if (marginBefore != delta.posMargin) {
             revert CfdEngineSettlementSidecar__SettlementMismatch();
```

With this diff applied, every attack test in the PoC fails at its first gain assertion and `test_honestFlows` passes (1 passed, 12 failed).

The diff demonstrates the rule; it is not a production patch. It keeps the deferred cash outside `accountedAssets`, so every `totalAssets()` read, including the pool depth used for VPI and the skew cap, leaves it out. A production version can instead keep an Engine aggregate of open positions' positive lifetime VPI and subtract it as a liability in `buildReconcileSnapshot` (not tested).

## Proof of Concept: tests, expected output and checks

### What each test proves (all on unmodified code)

- `test_attack`: one cycle at 60 % of Junior. The attacker ends with more USDG than it brought and gains more than 2,000 over the no-trade control; the close is rebated exactly the open's charge and the pool pays it in full; the stayer exits below its control and below its deposit; the other LPs lose more than the attacker gains, exactly its gain plus the treasury's fee on the open.
- `test_releaseScale`: the same at the release Senior cap (Senior above 39,000,000): absolute profit, gain above 5,000, the stayer below its control and its deposit.
- `test_repeatedCycles`: three cycles in a row, each ending in absolute profit; the stayer then exits below its control and its deposit.
- `test_depthKept`: an unrelated 6,500,000 Junior deposit activates in the attacker's exit settlement, so the close runs at more depth than the open and its curve rebate is below the charge (the clamp does not bind); the attack still gains more than 2,000, and the new LP exits below its control and its deposit.
- `test_depthCutShareOfRebate`: the rebate the depth change adds is less than 1 % of the gain, and without it the gain is still above 2,000.
- `test_passiveExit`: no attacker; an LP that exits while an unrelated trader's skew-widening position is open is paid more than 1,000 above the no-trade run, and a stayer less.
- `test_freshDepositPath`: deposit, open right after activation, redeem, close: absolute profit, a gain above 2,000 over the deposit-only timeline, and the pool and the stayer end lower.
- `test_hold25h`: the redemption is funded at least 24 h after the open: absolute profit, a gain above 2,000, and the stayer more than 1,000 lower and below its deposit.
- `test_juniorShareSweep`: the attacker's share of Junior at 30 / 35 / 40 / 45 / 60 / 75 %: 30 % loses, 45 / 60 / 75 % gain, the gain grows with the share, and the stayer pays in every row.
- `test_movingBasket_seeds01to16`, `test_movingBasket_seeds17to32`: 16 seeded random walks of the basket each, price exposure unhedged: mean gain above 2,000, every seed gains over its control and ends in absolute profit.
- `test_widerPythConfidence`: per-feed Pyth confidence of 0.25, 2, 5, 8 and 10 bp of price, attack and control at the same confidence: at every level the attack gains, ends in absolute profit, and the stayer exits below its control; the last level is at the oracle's cap (aggregate at least 9.95 bp).
- `test_honestFlows`: control for the fix run: the seeded traders close, and LPs exit and enter normally.

### Output (`-vv`, Forge 1.5.1; compiler warnings omitted, `...` marks omitted log lines)

```text
Ran 1 test for test/perps/poc/VpiRefundLpExit.t.sol:VpiRefundLpExitWiderConfidenceTest
[PASS] test_widerPythConfidence() (gas: 179815193)
Logs:
  WIDER PYTH CONFIDENCE: the attack and its no-trade control at the same per-feed confidence
    per-feed confidence 0.25 bp of price (raw price / 40000)
      aggregate basket confidence the oracle accepted (hundredths of a bp of price): 18
      attacker gain over the control (USDG): 2866.654247
      attacker net, attack (USDG): 2779.027696
    ...
      stayer lpJunior2 change vs the control (USDG): -1316.659140
    per-feed confidence 2 bp of price (raw price / 5000)
      aggregate basket confidence the oracle accepted (hundredths of a bp of price): 196
      attacker gain over the control (USDG): 2492.896384
      attacker net, attack (USDG): 2405.269833
    ...
      stayer lpJunior2 change vs the control (USDG): -1223.193056
    per-feed confidence 5 bp of price (raw price / 2000)
      aggregate basket confidence the oracle accepted (hundredths of a bp of price): 498
      attacker gain over the control (USDG): 1857.834455
      attacker net, attack (USDG): 1770.207904
    ...
      stayer lpJunior2 change vs the control (USDG): -1064.382345
    per-feed confidence 8 bp of price (raw price / 1250)
      aggregate basket confidence the oracle accepted (hundredths of a bp of price): 796
      attacker gain over the control (USDG): 1229.215221
      attacker net, attack (USDG): 1141.588670
    ...
      stayer lpJunior2 change vs the control (USDG): -907.182767
    per-feed confidence 10 bp of price (raw price / 1000)
      aggregate basket confidence the oracle accepted (hundredths of a bp of price): 997
      attacker gain over the control (USDG): 805.924345
      attacker net, attack (USDG): 718.297794
    ...
      stayer lpJunior2 change vs the control (USDG): -801.329902

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 215.41ms (206.16ms CPU time)

Ran 12 tests for test/perps/poc/VpiRefundLpExit.t.sol:VpiRefundLpExitTest
[PASS] test_attack() (gas: 133742194)
Logs:
  boot: pool assets / Junior principal / Senior principal (USDG):
     12000303.713891 4000000.999982 8000001.000018
  ATTACK: 60% of Junior, 6,000,000 LONG
    claim-net pool depth right before the open (USDG): 18000303.713891
    open: VPI paid by the attacker's LONG (lifetime VPI after the open, USDG): 12849.948255
    book skew right before the open, |long - short| at the mark (USDG): 855100.246500
    book skew right after the open (USDG): 6855087.176118
    book skew after the open, share of the depth before it (bps): 3808
    open: USDG the open cost the attacker, VPI + execution fee (USDG): 15249.931885
    open -> settlement that funds the redemption (seconds): 58
    redemption paid, attack (USDG): 6007634.974119
    redemption paid, no-trade control (USDG): 5999912.373449
    redemption paid above the control (USDG): 7722.600670
    claim-net pool depth right before the close (USDG): 12005518.688027
    close: VPI of the close, preview right before it (USDG, negative = rebate): -12849.948255
    close: net action rebate assessed, event (USDG): 10449.942343
    close: net action rebate paid out of the pool, event (USDG): 10449.942343
    close: net action rebate waived, event (USDG): 0.000000
    close: execution fee and carry withheld from the VPI rebate, kept by the pool (USDG): 2400.005912
    attacker: USDG brought, deposit + margin + buffer (USDG): 6610000.000000
    attacker: USDG held at the end (USDG): 6612779.027696
    attacker net, attack (USDG): 2779.027696
    attacker net, no-trade control (USDG): -87.626551
    attacker gain over the control (USDG): 2866.654247
    attacker LP leg, redeemed - deposited (USDG): 7634.974119
    attacker trading leg, clearinghouse out - in (USDG): -4855.946423
    stayer lpJunior2 (deposited 1,000,000) real exit, attack (USDG): 999970.990600
    stayer lpJunior2 real exit, no-trade control (USDG): 1001287.649740
    stayer lpJunior2 change vs the control (USDG): -1316.659140
    other Junior principal after that settlement, change vs the control (USDG): -3949.978737
    Senior principal after that settlement, change vs the control (USDG): 0.000000
    all stayers (every LP but the attacker), change vs the control (USDG): -5266.637877
    protocol treasury, change vs the control (USDG): 2399.983630
    gas of the whole attack cycle as run here (every call incl. the keeper's, plus test overhead): 8440989

[PASS] test_depthCutShareOfRebate() (gas: 133534643)
Logs:
  DEPTH CUT: share of the close rebate and of the gain
    open: VPI charged (USDG): 12849.948255
    claim-net depth right before the open (USDG): 18000303.713891
    pool assets right before the funding settlement (USDG): 18013153.662146
    ...
    close VPI at unchanged depth, preview after the open and before the funding (USDG): -12840.900774
    close VPI at unchanged depth, simulateClose at the close's instant (USDG): -12840.900603
    close VPI the real close receives, clamped (USDG): -12849.948255
    rebate added by the depth cut, against the pre-funding preview (USDG): 9.047481
    rebate added by the depth cut, against simulateClose at the close's instant (USDG): 9.047652
    rebate without the depth cut, as a share of the charge (basis points): 9992
    redemption paid above the control (USDG): 7722.600670
    attacker gain over the control (USDG): 2866.654247
    attacker gain without the depth cut's addition (USDG): 2857.606766
    depth cut's addition as a share of the gain (basis points): 31

[PASS] test_depthKept() (gas: 136986464)
Logs:
  DEPTH KEPT: unrelated 6,500,000 Junior deposit activates in the attacker's exit settlement
    claim-net pool depth right before the open (USDG): 18000303.713891
    open: VPI paid by the attacker's LONG (lifetime VPI after the open, USDG): 12849.948255
    ...
    claim-net pool depth right before the close (USDG): 18505518.688027
    close: VPI of the close, preview right before it (USDG, negative = rebate): -12499.250615
    ...
    attacker gain over the control (USDG): 2516.079004
    ...
    stayer lpJunior2 (deposited 1,000,000) real exit, attack (USDG): 1001627.038607
    stayer lpJunior2 real exit, no-trade control (USDG): 1001298.951333
    stayer lpJunior2 change vs the control (USDG): 328.087274
    ...
    entrant lpJunior3 (deposited 6,500,000) real exit, attack (USDG): 6493576.533538
    entrant lpJunior3 real exit, no-trade control (USDG): 6499804.884634
    entrant lpJunior3 change vs the control (USDG): -6228.351096

[PASS] test_freshDepositPath() (gas: 132759431)
Logs:
  FRESH DEPOSIT: 20,000,000 Junior, 8,000,000 LONG right after activation
    attacker net, deposit + trade (USDG): 3525.373235
    attacker net, deposit only (USDG): -121.741041
    attacker gain over the deposit-only timeline (USDG): 3647.114276
    ...
    pool assets change (USDG): -6847.265623
    ...
    stayer lpJunior2 change (USDG): -1711.815978

[PASS] test_hold25h() (gas: 134210006)
Logs:
  HOLD 25 H: redemption funded about 25 h after the open
    ...
    open -> settlement that funds the redemption (seconds): 89997
    ...
    close: net action rebate paid out of the pool, event (USDG): 10449.942343
    close: net action rebate waived, event (USDG): 0.000000
    close: execution fee and carry withheld from the VPI rebate, kept by the pool (USDG): 2400.005912
    attacker: USDG brought, deposit + margin + buffer (USDG): 6610000.000000
    attacker: USDG held at the end (USDG): 6611365.953589
    attacker net, attack (USDG): 1365.953589
    attacker net, no-trade control (USDG): -1139.145159
    attacker gain over the control (USDG): 2505.098748
    attacker LP leg, redeemed - deposited (USDG): 6583.455511
    attacker trading leg, clearinghouse out - in (USDG): -5217.501922
    stayer lpJunior2 (deposited 1,000,000) real exit, attack (USDG): 999885.890918
    stayer lpJunior2 real exit, no-trade control (USDG): 1001112.161206
    stayer lpJunior2 change vs the control (USDG): -1226.270288
    other Junior principal after that settlement, change vs the control (USDG): -3678.812090
    ...
[PASS] test_honestFlows() (gas: 132284864)
    ...

[PASS] test_juniorShareSweep() (gas: 187373277)
Logs:
  JUNIOR SHARE SWEEP
    Junior share 3000 bps, deposit 1715000.000000, LONG 4351700.000000
      attacker gain over the control (USDG): -632.713332
      stayer lpJunior2 real exit, lower than the control by (USDG): 276.985123
    Junior share 3500 bps, deposit 2153847.000000, LONG 4518461.860000
      attacker gain over the control (USDG): -172.923923
      stayer lpJunior2 real exit, lower than the control by (USDG): 408.611850
    Junior share 3999 bps, deposit 2666667.000000, LONG 4713333.460000
      attacker gain over the control (USDG): 319.285320
      stayer lpJunior2 real exit, lower than the control by (USDG): 551.150260
    Junior share 4501 bps, deposit 3275000.000000, LONG 4944500.000000
      attacker gain over the control (USDG): 853.555769
      stayer lpJunior2 real exit, lower than the control by (USDG): 707.835613
    Junior share 5999 bps, deposit 6000000.000000, LONG 5980000.000000
      attacker gain over the control (USDG): 2837.148468
      stayer lpJunior2 real exit, lower than the control by (USDG): 1307.280767
    Junior share 7499 bps, deposit 12000000.000000, LONG 8260000.000000
      attacker gain over the control (USDG): 6206.922473
      stayer lpJunior2 real exit, lower than the control by (USDG): 2377.722847

[PASS] test_movingBasket_seeds01to16() (gas: 364348445)
Logs:
  MOVING BASKET: seeded random walks, sigma 0.17 bp per sqrt(second); seeds 1 to 16
    ...
    seeds: 16
    mean gain over the control (USDG): 2649.732224
    standard deviation of the gain (USDG): 393.007100
    minimum gain (USDG): 1783.778435
    maximum gain (USDG): 3251.709268
    seeds with a gain over the control: 16
    seeds where the attacker ends with more USDG than it brought: 16

[PASS] test_movingBasket_seeds17to32() (gas: 363862017)
Logs:
  MOVING BASKET: seeded random walks, sigma 0.17 bp per sqrt(second); seeds 17 to 32
    ...
    seeds: 16
    mean gain over the control (USDG): 2849.615808
    standard deviation of the gain (USDG): 228.818553
    minimum gain (USDG): 2490.826383
    maximum gain (USDG): 3289.603375
    seeds with a gain over the control: 16
    seeds where the attacker ends with more USDG than it brought: 16

[PASS] test_passiveExit() (gas: 130787309)
Logs:
  PASSIVE: an LP exits while an unrelated trader's skew-widening LONG is open
    day trader's 3,800,000 LONG: VPI paid at the open (USDG): 8724.046135
    day trader's close: VPI, preview right before it (USDG, negative = rebate): -8724.046135
    day trader net over the round trip (USDG): -3086.440662
    exiter lpJunior2 paid, no trade (USDG): 1001320.538955
    exiter lpJunior2 paid, with the trade (USDG): 1003505.959940
    exiter paid above the no-trade run (USDG): 2185.420985
    stayer (1/3 of lpJunior1's shares) paid, no trade (USDG): 1001296.185395
    stayer (1/3 of lpJunior1's shares) paid, with the trade (USDG): 1001089.799056
    stayer change vs the no-trade run (USDG): -206.386339

[PASS] test_releaseScale() (gas: 136139435)
Logs:
  Senior principal before the attacker (USDG): 39990220.179461
  Junior principal before the attacker (USDG): 11505210.096430
    ...
  attacker LONG notional (USDG): 25261315.411278
  RELEASE SCALE: Senior at the cap, attacker 60% of Junior
    ...
    open: VPI paid by the attacker's LONG (lifetime VPI after the open, USDG): 49558.067247
    ...
    attacker: USDG brought, deposit + margin + buffer (USDG): 19786131.541127
    attacker: USDG held at the end (USDG): 19795049.148473
    attacker net, attack (USDG): 8917.607346
    attacker net, no-trade control (USDG): -438.179149
    attacker gain over the control (USDG): 9355.786495
    ...
    stayer lpJunior2 (deposited 1,000,000) real exit, attack (USDG): 999531.524220
    stayer lpJunior2 real exit, no-trade control (USDG): 1001225.176231
    stayer lpJunior2 change vs the control (USDG): -1693.652011
    ...
    all stayers (every LP but the attacker), change vs the control (USDG): -19460.247941
    ...

[PASS] test_repeatedCycles() (gas: 149824388)
Logs:
    cycle 1: attacker net (USDG): 2779.027696
    cycle 2: attacker net (USDG): 2785.365753
    cycle 3: attacker net (USDG): 2791.719508
  REPEATED CYCLES: 3 cycles of the attack
    time for the three cycles (seconds): 32389
    attacker net over 3 cycles, attack (USDG): 8356.112957
    attacker net over 3 cycles, no-trade control (USDG): -262.890312
    attacker gain over the control, 3 cycles (USDG): 8619.003269
    stayer lpJunior2 (deposited 1,000,000) real exit, attack (USDG): 997267.131918
    stayer lpJunior2 real exit, no-trade control (USDG): 1001221.869469
    stayer lpJunior2 change vs the control (USDG): -3954.737551
    all stayers (every LP but the attacker), change vs the control (USDG): -15818.954159

Suite result: ok. 12 passed; 0 failed; 0 skipped; finished in 575.85ms (2.08s CPU time)

Ran 2 test suites in 587.72ms (791.26ms CPU time): 13 tests passed, 0 failed, 0 skipped (13 total tests)
```

### The fix

Save the Recommendation's diff as `fix.diff` in the repository root, then:

```bash
git apply fix.diff
forge test --root packages/perps --match-path test/perps/poc/VpiRefundLpExit.t.sol -vv
git apply -R fix.diff
```

Every attack test fails at its first gain assertion, and `test_honestFlows` passes:

```text
[FAIL: the attack gains over its same-confidence control: -4839236422 <= 0] test_widerPythConfidence() (gas: 199451517)
[FAIL: the attacker ends with more USDG than all the USDG it brought: 6605073098457 <= 6610000000000] test_attack() (gas: 141489915)
[FAIL: without the depth cut's addition the attack still gains > 2,000 USDG: -4839236422 <= 2000000000] test_depthCutShareOfRebate() (gas: 141305779)
[FAIL: the attack still gains > 2,000 USDG over the control: -5186503810 <= 2000000000] test_depthKept() (gas: 145593773)
[FAIL: the attacker ends with more USDG than it brought: -6586839521 <= 0] test_freshDepositPath() (gas: 140564763)
[FAIL: the attacker ends with more USDG than all the USDG it brought: 6603659252084 <= 6610000000000] test_hold25h() (gas: 142044932)
[PASS] test_honestFlows() (gas: 132433786)
[FAIL: 60% of Junior gains: -4823089411 <= 0] test_juniorShareSweep() (gas: 209908904)
[FAIL: mean gain over the control > 2,000 USDG: -5055902547 <= 2000000000] test_movingBasket_seeds01to16() (gas: 416705662)
[FAIL: mean gain over the control > 2,000 USDG: -4856104208 <= 2000000000] test_movingBasket_seeds17to32() (gas: 416233547)
[FAIL: the exiter is paid > 1,000 USDG more than without the trade: 1000224447065 <= 1001220037068] test_passiveExit() (gas: 138016359)
[FAIL: the attacker ends with more USDG than all the USDG it brought: 19765153364376 <= 19785964264799] test_releaseScale() (gas: 144447825)
[FAIL: every cycle ends with more USDG than the attacker brought to it: -4926901543 <= 0] test_repeatedCycles() (gas: 162583512)
Ran 2 test suites in 552.76ms (741.54ms CPU time): 1 tests passed, 12 failed, 0 skipped (13 total tests)
```

### Optional supplementary check: the deferred streaming fix

The repository records M10 ("JIT LP attack via unrealized PnL") as mitigated by the 1 h deposit cooldown, with the full fix (streaming revenue) deferred. `streaming.diff` below is a sketch of that fix, not part of the PoC: open and close LP revenue vests linearly over 24 h before it reaches tranche principal. Save it as `streaming.diff` in the repository root, run `git apply streaming.diff`, run the same `forge test` command, and revert with `git apply -R streaming.diff`. Result: 2 passed, 11 failed. The 58 s cycle stops (`test_attack` fails; gain over the control -4,834.355830), and the 25 h hold still pays (`test_hold25h` passes; gain +2,508.213644, and the stayer exits 1,227.049012 below its control):

```text
[FAIL: the attacker ends with more USDG than all the USDG it brought: 6605297973795 <= 6610000000000] test_attack() (gas: 141669805)
    ...
    attacker net, attack (USDG): -4702.026205
    ...
    attacker gain over the control (USDG): -4834.355830
    ...
[PASS] test_hold25h() (gas: 134500215)
    ...
    attacker net, attack (USDG): 3788.699060
    ...
    attacker gain over the control (USDG): 2508.213644
    ...
    stayer lpJunior2 (deposited 1,000,000) real exit, attack (USDG): 999280.204702
    stayer lpJunior2 real exit, no-trade control (USDG): 1000507.253714
    stayer lpJunior2 change vs the control (USDG): -1227.049012
    ...
[PASS] test_honestFlows() (gas: 132587423)
    ...
Ran 2 test suites in 602.12ms (808.77ms CPU time): 2 tests passed, 11 failed, 0 skipped (13 total tests)
```

```diff
diff --git a/packages/perps/src/CfdEngineSettlementSidecar.sol b/packages/perps/src/CfdEngineSettlementSidecar.sol
index 64a4710d..2d2e6d22 100644
--- a/packages/perps/src/CfdEngineSettlementSidecar.sol
+++ b/packages/perps/src/CfdEngineSettlementSidecar.sol
@@ -122,6 +122,10 @@ interface ICfdEngineAccountActionView {
 ///      settlement-host hooks. The sidecar does not own independent protocol state and does not validate planner deltas;
 ///      the bound engine must supply a valid delta derived from matching live state. Unless stated otherwise, settlement token
 ///      amounts use 6 decimals, prices use 8 decimals, sizes use 18 decimals, and timestamps are Unix seconds.
+interface IRevenueVestingPool {
+    function lockRevenue(uint256 amount) external;
+}
+
 contract CfdEngineSettlementSidecar is ICfdEngineSettlementSidecar {
 
     /// @notice Sole engine address authorized to call settlement entrypoints.
@@ -461,6 +465,11 @@ contract CfdEngineSettlementSidecar is ICfdEngineSettlementSidecar {
                         IHousePool.ClaimantInflowCashMode.CashArrived
                     );
             }
+            // Streaming revenue (sketch of M10's deferred fix): the open's LP revenue (VPI net of the treasury fee)
+            // vests.
+            if (poolCashInflowAssets > 0) {
+                IRevenueVestingPool(host.pool()).lockRevenue(poolCashInflowAssets);
+            }
         }
 
         if (delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets) {
@@ -614,6 +623,13 @@ contract CfdEngineSettlementSidecar is ICfdEngineSettlementSidecar {
 
         uint256 feeWithheldAssets = delta.executionFeeAssets - protocolFeeCreditedAssets;
         _recordRetainedCloseRevenue(host, delta.actionChargeWithheldAssets - feeWithheldAssets);
+        // Streaming revenue (sketch of M10's deferred fix): the close's LP action revenue (VPI, carry, spread; not
+        // the treasury fee) vests.
+        uint256 closeRevenueAssets = delta.actionChargeCollectedAssets - protocolFeeCreditedAssets
+            + delta.actionChargeWithheldAssets - feeWithheldAssets;
+        if (closeRevenueAssets > 0) {
+            IRevenueVestingPool(host.pool()).lockRevenue(closeRevenueAssets);
+        }
 
         if (delta.pricePayoutAssets > 0) {
             host.settlementRecordTraderClaim(delta.account, delta.pricePayoutAssets, !delta.deletePosition);
diff --git a/packages/perps/src/HousePool.sol b/packages/perps/src/HousePool.sol
index fceb093d..ce3451d1 100644
--- a/packages/perps/src/HousePool.sol
+++ b/packages/perps/src/HousePool.sol
@@ -196,6 +196,38 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
     /// @notice Earliest Unix timestamp at which the pending configuration may be finalized, or zero if none.
     uint256 public poolConfigActivationTime;
 
+    // Streaming revenue (sketch of M10's deferred fix): open and close LP action revenue vests linearly over 24 h
+    // before reconcile assigns it to tranche principal.
+    uint256 public lockedRevenueAssets;
+    uint256 public lockedRevenueAt;
+    uint256 public constant REVENUE_VESTING = 24 hours;
+
+    function lockedRevenueNow() public view returns (uint256) {
+        uint256 elapsed = block.timestamp - lockedRevenueAt;
+        if (elapsed >= REVENUE_VESTING) {
+            return 0;
+        }
+        return (lockedRevenueAssets * (REVENUE_VESTING - elapsed)) / REVENUE_VESTING;
+    }
+
+    function lockRevenue(uint256 amount) external {
+        if (msg.sender != address(ENGINE) && msg.sender != ENGINE.settlementSidecar()) {
+            revert HousePool__Unauthorized();
+        }
+        lockedRevenueAssets = lockedRevenueNow() + amount;
+        lockedRevenueAt = block.timestamp;
+    }
+
+    function _holdLockedRevenue(HousePoolAccountingLib.ReconcileSnapshot memory snap) internal view {
+        uint256 claimed = seniorPrincipal + juniorPrincipal;
+        if (snap.distributable <= claimed) {
+            return;
+        }
+        uint256 hold = lockedRevenueNow();
+        uint256 surplus = snap.distributable - claimed;
+        snap.distributable -= hold < surplus ? hold : surplus;
+    }
+
     modifier onlyPauserOrOwner() {
         if (msg.sender != owner() && msg.sender != pauser) {
             revert HousePool__UnauthorizedPauser();
@@ -1561,6 +1593,7 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
         bool markFresh = _markIsFreshForReconcile(accountingSnapshot, statusSnapshot);
         HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot =
             HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot);
+        _holdLockedRevenue(reconcileSnapshot);
         HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
         HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
             _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
@@ -1614,6 +1647,7 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
         ctx.statusSnapshot = statusSnapshot;
         HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot =
             HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot);
+        _holdLockedRevenue(reconcileSnapshot);
         HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
         HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
             _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
```

## References

- Open books VPI as revenue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464
- Lifetime VPI written at the open: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L517-L517
- Close rebate paid out of the pool: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624
- `totalAssets()`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L426-L429
- `payOut`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L604-L613
- `recordClaimantInflow` (`CashArrived`): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L641
- Epoch settlement, reconcile and Junior redemption price: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841
- Junior deposit price: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L919
- Reconcile snapshot: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78
- Snapshot inputs: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L59-L73
- Revenue waterfall: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L140-L159
- Close VPI and lifetime clamp: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118
- Close action netting: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1036-L1061
- Rebate paid from free pool cash: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L978
- Permissionless `executeOrder`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L233-L247
- Permissionless `settleLpEpoch`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484
- Epoch length: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L178-L178
- Deposit cooldown: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L38-L38
- Oracle confidence cap: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L869-L873
- AI-31: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742-L1800
- Positive VPI increases pool assets: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L411-L412
- VPI rebate reserve kept out of the price: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L887-L888
- New LPs do not inherit old trader obligations: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L859-L860
- Carry cohort transfer accepted: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L737-L740


## Proof of Concept

## Proof of Concept

Steps (`test_attack`; USDG; release parameters; unmodified code; public entry points only, no privileged role acts):
1. The attacker deposits 6,000,000 into Junior (60 % of Junior), waits out the 1 h deposit cooldown and queues a full redemption for the next hourly epoch.
2. 60 s before the epoch boundary it opens a 6,000,000 LONG and executes it itself. The open's VPI, 12,849.948255, is booked to the pool at once.
3. The epoch is settled through the permissionless Router entry point. The redemption pays 6,007,634.974119, which is 7,722.600670 more than without the trade.
4. It closes: the close's VPI is -12,849.948255 and the pool pays a 10,449.942343 net rebate. The attacker ends with 6,612,779.027696 against 6,610,000.000000 brought (+2,779.027696), and the stayer `lpJunior2` exits at 999,970.990600, below its 1,000,000 deposit (1,001,287.649740 without the attack).

Run: in a clone of https://github.com/immunefi-team/audit-comp-plether at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (with `git submodule update --init --recursive`), save the file below as `packages/perps/test/perps/poc/VpiRefundLpExit.t.sol`, then:

```bash
forge test --root packages/perps --match-path test/perps/poc/VpiRefundLpExit.t.sol -vv
```

All 13 tests pass in about a minute (Forge 1.5.1, the version CI pins, or Forge 1.8.3). What each test proves, the expected output, and the fix and streaming checks are at the end of the Description.

```solidity
// SPDX-License-Identifier: AGPL-3.0
// PoC: refundable open VPI is priced into LP exits as final revenue. Audited commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
// Save as packages/perps/test/perps/poc/VpiRefundLpExit.t.sol; run (Forge 1.5.1, the version CI pins, or 1.8.3):
//   forge test --root packages/perps --match-path test/perps/poc/VpiRefundLpExit.t.sol -vv
// Every production contract of packages/perps/src is deployed from its artifact (deployCode) in the order, wiring,
// roles and release parameters of script/DeployPerpsArbitrumSepolia.s.sol, bootstrapped as
// script/BootstrapPerpsArbitrumSepolia.s.sol does; the release-only constructor wrappers and MockUSDG are taken from
// that script. Mocks: MockPyth and MockUSDG only. No vm.store, vm.mockCall, vm.etch, deployCodeTo or vm.chainId;
// nothing in scope is modified. Cheat codes: prank (actors' own calls), deal (actors' ETH for Pyth fees), warp/roll
// (epochs, cooldown, order timing), snapshotState/revertToState (attack and control from the same state), recordLogs
// (the close's rebate event), getNonce/computeCreateAddress (the Router address, as the script computes it).
// Assumptions: six production feeds, confidence 0.0025% of price (wider in test_widerPythConfidence); basket flat at
// 1.006 except in the moving-basket tests; no other flow in the window (test_depthKept adds one deposit on purpose).
pragma solidity 0.8.35;
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAccountLens} from "@plether/perps/CfdEngineAccountLens.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdClosePreview} from "@plether/perps/CfdClosePreview.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PerpsPublicLens} from "@plether/perps/PerpsPublicLens.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {SettlementMonitorLens} from "@plether/perps/SettlementMonitorLens.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineSettlementSidecar} from "@plether/perps/interfaces/ICfdEngineSettlementSidecar.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test, console2} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";
contract ArbitrumSepoliaReleaseHousePool is HousePool {
	constructor(
		address settlementToken,
		address engine,
		address redemptionMathSidecar
	) HousePool(settlementToken, engine, redemptionMathSidecar) {
		poolConfig.maxSeniorExposureAssets = 40_000_000e6;
		poolConfig.maxSeniorShareBps = 8000;
		_validatePoolConfig(poolConfig);
	}
}
contract ArbitrumSepoliaReleaseOracle is PletherOracle {
	constructor(
		address engine,
		address housePool,
		address pyth,
		bytes32[] memory feedIds,
		uint256[] memory quantities,
		uint256[] memory basePrices,
		bool[] memory inversions
	) PletherOracle(engine, housePool, pyth, feedIds, quantities, basePrices, inversions) {
		adverseConfidenceMultiplierBps = 2500;
	}
}
contract ArbitrumSepoliaReleaseRouter is OrderRouter {
	constructor(
		address engine,
		address engineLens,
		address housePool,
		address pletherOracle_,
		address keeperSidecar,
		address policyEvaluator,
		address executionSidecar,
		address lifecycleBook
	)
		OrderRouter(
			engine,
			engineLens,
			housePool,
			pletherOracle_,
			keeperSidecar,
			policyEvaluator,
			executionSidecar,
			lifecycleBook
		)
	{
		minOpenNotionalAssets = 1000e6;
	}
}
contract MockUSDG is ERC20 {
	constructor() ERC20("Mock USDG", "USDG") {}
	function decimals() public pure override returns (uint8) {
		return 6;
	}
	function mint(
		address to,
		uint256 amount
	) external {
		_mint(to, amount);
	}
}
abstract contract VpiRefundLpExitFixture is Test {
	uint32 internal constant CAP_PRICE = 2e8;
	uint256 internal constant FROZEN_CLOSE_SPREAD_BPS = 50;
	uint256 internal constant JUNIOR_MAINTENANCE_FEE_APR_BPS = 0;
	uint256 internal constant RELEASE_SEED_ASSETS = 1e6;
	uint256 internal constant T0 = 1_794_218_400;
	uint256 internal constant PYTH_FEE_WEI = 1;
	uint256 internal constant ONE = 1e8;
	string internal constant ART_USDG = "VpiRefundLpExit.t.sol:MockUSDG";
	string internal constant ART_PYTH = "MockPyth.sol:MockPyth";
	string internal constant ART_CLEARINGHOUSE = "MarginClearinghouse.sol:MarginClearinghouse";
	string internal constant ART_ENGINE = "CfdEngine.sol:CfdEngine";
	string internal constant ART_NAV_BOOK = "TerminalNavBookV2.sol:TerminalNavBookV2";
	string internal constant ART_PLANNER = "CfdEnginePlanner.sol:CfdEnginePlanner";
	string internal constant ART_SETTLEMENT_SIDECAR = "CfdEngineSettlementSidecar.sol:CfdEngineSettlementSidecar";
	string internal constant ART_ENGINE_ADMIN = "CfdEngineAdmin.sol:CfdEngineAdmin";
	string internal constant ART_REDEMPTION_SIDECAR =
		"HousePoolRedemptionMathSidecar.sol:HousePoolRedemptionMathSidecar";
	string internal constant ART_POOL = "VpiRefundLpExit.t.sol:ArbitrumSepoliaReleaseHousePool";
	string internal constant ART_VAULT = "TrancheVault.sol:TrancheVault";
	string internal constant ART_ACCOUNT_LENS = "CfdEngineAccountLens.sol:CfdEngineAccountLens";
	string internal constant ART_ENGINE_LENS = "CfdEngineLens.sol:CfdEngineLens";
	string internal constant ART_ORACLE = "VpiRefundLpExit.t.sol:ArbitrumSepoliaReleaseOracle";
	string internal constant ART_POLICY = "CfdOrderPolicyEvaluator.sol:CfdOrderPolicyEvaluator";
	string internal constant ART_CLOSE_PREVIEW = "CfdClosePreview.sol:CfdClosePreview";
	string internal constant ART_EXEC_SIDECAR = "OrderRouterExecutionSidecar.sol:OrderRouterExecutionSidecar";
	string internal constant ART_LIFECYCLE_BOOK = "OrderLifecycleBook.sol:OrderLifecycleBook";
	string internal constant ART_LIQ_SIDECAR =
		"OrderRouterLiquidationBatchSidecar.sol:OrderRouterLiquidationBatchSidecar";
	string internal constant ART_ROUTER = "VpiRefundLpExit.t.sol:ArbitrumSepoliaReleaseRouter";
	string internal constant ART_PUBLIC_LENS = "PerpsPublicLens.sol:PerpsPublicLens";
	string internal constant ART_MONITOR_LENS = "SettlementMonitorLens.sol:SettlementMonitorLens";
	string internal constant ART_EMERGENCY = "EmergencyPauseCoordinator.sol:EmergencyPauseCoordinator";
	MockUSDG internal usdg;
	MockPyth internal pyth;
	MarginClearinghouse internal clearinghouse;
	CfdEngine internal engine;
	TerminalNavBookV2 internal terminalNavBook;
	CfdEnginePlanner internal planner;
	CfdEngineSettlementSidecar internal settlementSidecar;
	CfdEngineAdmin internal engineAdmin;
	HousePoolRedemptionMathSidecar internal redemptionMathSidecar;
	HousePool internal pool;
	TrancheVault internal seniorVault;
	TrancheVault internal juniorVault;
	CfdEngineAccountLens internal accountLens;
	CfdEngineLens internal engineLens;
	PletherOracle internal oracle;
	CfdOrderPolicyEvaluator internal policyEvaluator;
	CfdClosePreview internal closePreview;
	OrderRouterExecutionSidecar internal executionSidecar;
	OrderLifecycleBook internal lifecycleBook;
	OrderRouterLiquidationBatchSidecar internal liquidationBatchSidecar;
	OrderRouter internal router;
	OrderRouterAdmin internal routerAdmin;
	address internal protectionBook;
	PerpsPublicLens internal publicLens;
	SettlementMonitorLens internal monitorLens;
	EmergencyPauseCoordinator internal emergency;
	address internal deployer = makeAddr("deployer");
	address internal guardian = makeAddr("guardian");
	address internal seedReceiver = makeAddr("seedReceiver");
	address internal keeper = makeAddr("keeper");
	address internal attacker = makeAddr("attacker");
	address internal lpJunior1 = makeAddr("lpJunior1");
	address internal lpJunior2 = makeAddr("lpJunior2");
	address internal lpSenior1 = makeAddr("lpSenior1");
	address internal lpSenior2 = makeAddr("lpSenior2");
	address internal alice = makeAddr("alice");
	address internal bob = makeAddr("bob");
	address internal carol = makeAddr("carol");
	address internal dave = makeAddr("dave");
	address internal erin = makeAddr("erin");
	address internal frank = makeAddr("frank");
	uint256 internal constant LP_JUNIOR1 = 3_000_000e6;
	uint256 internal constant LP_JUNIOR2 = 1_000_000e6;
	uint256 internal constant LP_SENIOR1 = 5_000_000e6;
	uint256 internal constant LP_SENIOR2 = 3_000_000e6;
	uint256 internal constant SEEDED_BASKET = 1.006e8;
	uint256 internal lastBasket;
	uint256 internal lastPublishTime;
	uint256 internal orderNonce;
	mapping(uint64 => uint256) internal commitTimeOf;
	mapping(uint64 => uint256) internal commitBlockOf;
	int256 internal constant WALK_SIGMA_E8 = 1700;
	uint256 internal constant WALK_STEPS = 64;
	bool internal walkOn;
	uint256 internal walkSeed;
	uint256 internal walkT0;
	uint256 internal walkP0;
	function _boot() internal {
		vm.warp(T0);
		vm.roll(100_000_000);
		_deployProtocol();
		_bootstrapRelease();
		_seedState();
	}
	function _deploy(
		string memory artifact,
		bytes memory args
	) internal returns (address addr) {
		addr = deployCode(artifact, args);
		vm.label(addr, artifact);
	}
	function _deployProtocol() internal {
		vm.startPrank(deployer);
		usdg = MockUSDG(_deploy(ART_USDG, ""));
		pyth = MockPyth(_deploy(ART_PYTH, ""));
		pyth.setFee(PYTH_FEE_WEI);
		clearinghouse = MarginClearinghouse(_deploy(ART_CLEARINGHOUSE, abi.encode(address(usdg))));
		engine = CfdEngine(
			_deploy(
				ART_ENGINE,
				abi.encode(address(usdg), address(clearinghouse), CAP_PRICE, _riskParams(), FROZEN_CLOSE_SPREAD_BPS)
			)
		);
		terminalNavBook = TerminalNavBookV2(_deploy(ART_NAV_BOOK, abi.encode(address(engine), CAP_PRICE)));
		engine.setTerminalNavBook(address(terminalNavBook));
		planner = CfdEnginePlanner(_deploy(ART_PLANNER, ""));
		settlementSidecar = CfdEngineSettlementSidecar(_deploy(ART_SETTLEMENT_SIDECAR, abi.encode(address(engine))));
		engineAdmin = CfdEngineAdmin(_deploy(ART_ENGINE_ADMIN, abi.encode(address(engine), deployer)));
		engine.setDependencies(address(planner), address(settlementSidecar), address(engineAdmin));
		redemptionMathSidecar = HousePoolRedemptionMathSidecar(_deploy(ART_REDEMPTION_SIDECAR, ""));
		pool = HousePool(
			_deploy(ART_POOL, abi.encode(address(usdg), address(engine), address(redemptionMathSidecar)))
		);
		seniorVault = TrancheVault(
			_deploy(
				ART_VAULT,
				abi.encode(address(usdg), address(pool), true, "Plether Senior LP", "psLP", uint256(0), address(0))
			)
		);
		pool.setSeniorVault(address(seniorVault));
		address juniorFeeRecipient = engine.protocolTreasury();
		juniorVault = TrancheVault(
			_deploy(
				ART_VAULT,
				abi.encode(
					address(usdg),
					address(pool),
					false,
					"Plether Junior LP",
					"pjLP",
					JUNIOR_MAINTENANCE_FEE_APR_BPS,
					juniorFeeRecipient
				)
			)
		);
		pool.setJuniorVault(address(juniorVault));
		engine.setPool(address(pool));
		accountLens = CfdEngineAccountLens(_deploy(ART_ACCOUNT_LENS, abi.encode(address(engine))));
		engineLens = CfdEngineLens(_deploy(ART_ENGINE_LENS, abi.encode(address(engine))));
		oracle = PletherOracle(
			_deploy(
				ART_ORACLE,
				abi.encode(
					address(engine),
					address(pool),
					address(pyth),
					_feedIds(),
					_quantities(),
					_basePrices(),
					_inversions()
				)
			)
		);
		policyEvaluator = CfdOrderPolicyEvaluator(_deploy(ART_POLICY, ""));
		closePreview = CfdClosePreview(_deploy(ART_CLOSE_PREVIEW, ""));
		executionSidecar = OrderRouterExecutionSidecar(_deploy(ART_EXEC_SIDECAR, ""));
		address expectedRouter = vm.computeCreateAddress(deployer, uint256(vm.getNonce(deployer)) + 2);
		lifecycleBook = OrderLifecycleBook(
			_deploy(
				ART_LIFECYCLE_BOOK,
				abi.encode(expectedRouter, address(engine), address(clearinghouse), address(pool))
			)
		);
		liquidationBatchSidecar =
			OrderRouterLiquidationBatchSidecar(_deploy(ART_LIQ_SIDECAR, abi.encode(expectedRouter)));
		router = OrderRouter(
			payable(_deploy(
					ART_ROUTER,
					abi.encode(
						address(engine),
						address(engineLens),
						address(pool),
						address(oracle),
						address(liquidationBatchSidecar),
						address(policyEvaluator),
						address(executionSidecar),
						address(lifecycleBook)
					)
				))
		);
		require(address(router) == expectedRouter, "OrderRouter CREATE address mismatch");
		protectionBook = address(router.positionProtectionBook());
		routerAdmin = OrderRouterAdmin(router.admin());
		engine.setOrderRouter(address(router));
		clearinghouse.setEngine(address(engine));
		publicLens = PerpsPublicLens(
			_deploy(
				ART_PUBLIC_LENS,
				abi.encode(address(accountLens), address(engine), address(router), address(pool))
			)
		);
		monitorLens = SettlementMonitorLens(_deploy(ART_MONITOR_LENS, abi.encode(address(router))));
		emergency = EmergencyPauseCoordinator(
			_deploy(ART_EMERGENCY, abi.encode(address(routerAdmin), address(pool), deployer))
		);
		pool.setPauser(address(emergency));
		routerAdmin.setPauser(address(emergency));
		vm.stopPrank();
		vm.label(address(routerAdmin), "OrderRouterAdmin");
		vm.label(protectionBook, "PositionProtectionBook");
		vm.label(address(pool.ENGINE_PROTOCOL_LENS()), "CfdEngineProtocolLens");
		vm.label(address(monitorLens.SIDECAR()), "SettlementMonitorLensSidecar");
		vm.deal(keeper, 100 ether);
	}
	function _bootstrapRelease() internal {
		vm.startPrank(deployer);
		usdg.mint(deployer, 2 * RELEASE_SEED_ASSETS);
		usdg.approve(address(pool), 2 * RELEASE_SEED_ASSETS);
		pool.initializeSeedPosition(false, RELEASE_SEED_ASSETS, seedReceiver);
		pool.initializeSeedPosition(true, RELEASE_SEED_ASSETS, seedReceiver);
		emergency.setGuardian(guardian);
		vm.stopPrank();
		_setPrice(ONE);
		vm.prank(deployer);
		pool.activateTrading();
	}
	function _seedState() internal {
		uint256 j1 = _lpRequestDeposit(juniorVault, lpJunior1, LP_JUNIOR1);
		uint256 j2 = _lpRequestDeposit(juniorVault, lpJunior2, LP_JUNIOR2);
		_settleEpoch();
		_lpClaimDeposit(juniorVault, lpJunior1, j1);
		_lpClaimDeposit(juniorVault, lpJunior2, j2);
		uint256 s1 = _lpRequestDeposit(seniorVault, lpSenior1, LP_SENIOR1);
		uint256 s2 = _lpRequestDeposit(seniorVault, lpSenior2, LP_SENIOR2);
		_settleEpoch();
		_lpClaimDeposit(seniorVault, lpSenior1, s1);
		_lpClaimDeposit(seniorVault, lpSenior2, s2);
		_openAtNotional(alice, CfdTypes.Side.LONG, 2_000_000e6, 200_000e6);
		_openAtNotional(bob, CfdTypes.Side.LONG, 1_000_000e6, 50_000e6);
		_openAtNotional(carol, CfdTypes.Side.LONG, 250_000e6, 5000e6);
		_openAtNotional(dave, CfdTypes.Side.SHORT, 1_500_000e6, 300_000e6);
		_openAtNotional(erin, CfdTypes.Side.SHORT, 800_000e6, 32_000e6);
		_openAtNotional(frank, CfdTypes.Side.SHORT, 100_000e6, 10_000e6);
		_setPrice(SEEDED_BASKET);
		_accrueCarry(1 hours);
		_setPrice(SEEDED_BASKET);
	}
	function _riskParams() internal pure returns (CfdTypes.RiskParams memory) {
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
	function _feedIds() internal pure returns (bytes32[] memory ids) {
		ids = new bytes32[](6);
		ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
		ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
		ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
		ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
		ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
		ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
	}
	function _quantities() internal pure returns (uint256[] memory q) {
		q = new uint256[](6);
		q[0] = 576e15;
		q[1] = 136e15;
		q[2] = 119e15;
		q[3] = 91e15;
		q[4] = 42e15;
		q[5] = 36e15;
	}
	function _basePrices() internal pure returns (uint256[] memory b) {
		b = new uint256[](6);
		b[0] = 117_500_000;
		b[1] = 638_000;
		b[2] = 134_480_000;
		b[3] = 72_880_000;
		b[4] = 10_860_000;
		b[5] = 126_100_000;
	}
	function _inversions() internal pure returns (bool[] memory inv) {
		inv = new bool[](6);
		inv[1] = true;
		inv[3] = true;
		inv[4] = true;
		inv[5] = true;
	}
	function _pythData(
		uint256 basket8,
		uint256 publishTime,
		uint256 prevPublishTime
	) internal view virtual returns (bytes[] memory data) {
		uint256[] memory bases = _basePrices();
		bytes32[] memory ids = _feedIds();
		MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](6);
		for (uint256 i; i < 6; ++i) {
			uint256 usdPerUnit = (bases[i] * basket8 + ONE / 2) / ONE;
			bool inverted = (i == 1 || i == 3 || i == 4 || i == 5);
			int32 expo = i == 1 ? int32(-3) : int32(-5);
			uint256 raw;
			if (inverted) {
				uint256 scale = 10 ** uint256(uint32(-expo));
				raw = (scale * ONE + usdPerUnit / 2) / usdPerUnit;
			} else {
				raw = (usdPerUnit + 500) / 1000;
			}
			uint64 conf = uint64(raw / 40_000);
			if (conf == 0) {
				conf = 1;
			}
			u[i] = MockPyth.FeedUpdate(
				ids[i], MockPyth.MockPrice(int64(uint64(raw)), conf, expo, publishTime, prevPublishTime)
			);
		}
		data = new bytes[](1);
		data[0] = abi.encode(u);
	}
	function _priceAt(
		uint256 t,
		uint256 flatBasket
	) internal view returns (uint256) {
		if (!walkOn || walkT0 == 0 || t <= walkT0) {
			return flatBasket;
		}
		uint256 n = t - walkT0;
		if (n > WALK_STEPS) {
			n = WALK_STEPS;
		}
		int256 w;
		for (uint256 i = 1; i <= n; ++i) {
			w += _gaussE6(walkSeed, i);
		}
		return uint256(int256(walkP0) + (int256(walkP0) * WALK_SIGMA_E8 * w) / 1e14);
	}
	function _gaussE6(
		uint256 seed,
		uint256 step
	) internal pure returns (int256) {
		uint256 h = uint256(keccak256(abi.encode("basket walk", seed, step)));
		uint256 m = (1 << 21) - 1;
		uint256 sum;
		for (uint256 j; j < 12; ++j) {
			sum += (h >> (21 * j)) & m;
		}
		return ((int256(sum) - int256(6 * m)) * 1e6) / int256(m);
	}
	function _setPrice(
		uint256 basket8
	) internal {
		_nextBlock(1);
		uint256 t = vm.getBlockTimestamp();
		basket8 = _priceAt(t, basket8);
		lastBasket = basket8;
		lastPublishTime = t;
		bytes[] memory data = _pythData(basket8, t, t - 1);
		uint256 fee = oracle.getUpdateFee(data);
		vm.prank(keeper);
		router.updateMarkPrice{value: fee}(data);
	}
	function _warp(
		uint256 dt
	) internal {
		vm.warp(vm.getBlockTimestamp() + dt);
		vm.roll(vm.getBlockNumber() + (dt == 0 ? 1 : dt * 4));
	}
	function _nextBlock(
		uint256 dt
	) internal {
		vm.roll(vm.getBlockNumber() + 1);
		vm.warp(vm.getBlockTimestamp() + dt);
	}
	function _warpToNextEpoch() internal {
		uint256 next = pool.lpEpochStart(pool.currentLpEpoch() + 1);
		_warp(next + 1 - vm.getBlockTimestamp());
	}
	function _accrueCarry(
		uint256 dt
	) internal {
		_warp(dt);
		engine.checkpointCarryIndexes();
	}
	function _fundTrader(
		address trader,
		uint256 amount
	) internal {
		usdg.mint(trader, amount);
		vm.startPrank(trader);
		usdg.approve(address(clearinghouse), amount);
		clearinghouse.deposit(trader, amount);
		vm.stopPrank();
	}
	function _req(
		CfdTypes.Side side,
		uint256 sizeDelta,
		uint256 marginDelta,
		bool isClose
	) internal returns (OrderV3Types.OrderRequest memory r) {
		r.clientOrderId = keccak256(abi.encode("order", ++orderNonce));
		r.side = side;
		r.sizeDelta = sizeDelta;
		r.marginDelta = marginDelta;
		r.isClose = isClose;
		r.targetPrice = (side == CfdTypes.Side.LONG) != isClose ? 1 : type(uint256).max;
		uint256 window = router.maxExecutionWindowSeconds();
		r.bounds = OrderV3Types.ExecutionBounds({
			submitBy: uint64(vm.getBlockTimestamp() + window),
			executionWindowSeconds: uint32(window),
			allowedExecutionModes: 7,
			expectedConfigHash: lifecycleBook.currentExecutionConfigHash(),
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
	function _commitReq(
		address trader,
		OrderV3Types.OrderRequest memory r
	) internal returns (uint64 id) {
		vm.prank(trader);
		id = router.commitOrder(r);
		commitTimeOf[id] = vm.getBlockTimestamp();
		commitBlockOf[id] = vm.getBlockNumber();
	}
	function _commit(
		address trader,
		CfdTypes.Side side,
		uint256 sizeDelta,
		uint256 marginDelta,
		bool isClose
	) internal returns (uint64 id) {
		id = _commitReq(trader, _req(side, sizeDelta, marginDelta, isClose));
	}
	function _executeAs(
		address executor,
		uint64 id,
		uint256 basket8
	) internal returns (OrderV3Types.ExecutionResult memory res) {
		uint256 ct = commitTimeOf[id];
		if (vm.getBlockNumber() <= commitBlockOf[id]) {
			vm.roll(commitBlockOf[id] + 1);
		}
		if (vm.getBlockTimestamp() < ct + 2) {
			vm.warp(ct + 2);
		}
		uint256 publishTime = ct + 1 > lastPublishTime ? ct + 1 : lastPublishTime;
		if (publishTime > vm.getBlockTimestamp()) {
			vm.warp(publishTime);
		}
		basket8 = _priceAt(publishTime, basket8);
		lastBasket = basket8;
		lastPublishTime = publishTime;
		bytes[] memory data = _pythData(basket8, publishTime, ct);
		uint256 fee = oracle.getOrderExecutionFee(data);
		vm.prank(executor);
		res = router.executeOrder{value: fee}(id, data);
	}
	function _execute(
		uint64 id,
		uint256 basket8
	) internal {
		OrderV3Types.ExecutionResult memory res = _executeAs(keeper, id, basket8);
		if (res.status != OrderV3Types.LifecycleStatus.Executed) {
			console2.log("order", uint256(id), "status", uint256(res.status));
			revert("order not executed");
		}
	}
	function _open(
		address trader,
		CfdTypes.Side side,
		uint256 sizeDelta,
		uint256 marginDelta,
		uint256 basket8
	) internal {
		uint64 id = _commit(trader, side, sizeDelta, marginDelta, false);
		_execute(id, basket8);
	}
	function _open(
		address trader,
		CfdTypes.Side side,
		uint256 sizeDelta,
		uint256 marginDelta
	) internal {
		_open(trader, side, sizeDelta, marginDelta, lastBasket);
	}
	function _openAtNotional(
		address trader,
		CfdTypes.Side side,
		uint256 notional6,
		uint256 margin6
	) internal {
		_fundTrader(trader, margin6 + notional6 / 100);
		_open(trader, side, _sizeFor(notional6, lastBasket), margin6);
	}
	function _close(
		address trader,
		uint256 basket8
	) internal {
		(uint256 size,,,, CfdTypes.Side side,,) = engine.positions(trader);
		require(size > 0, "no position");
		uint64 id = _commit(trader, side, size, 0, true);
		_execute(id, basket8);
	}
	function _lpRequestDeposit(
		TrancheVault vault,
		address lp,
		uint256 assets
	) internal returns (uint256 requestId) {
		usdg.mint(lp, assets);
		vm.startPrank(lp);
		usdg.approve(address(vault), assets);
		requestId = vault.requestDeposit(assets, lp, lp);
		vm.stopPrank();
	}
	function _lpClaimDeposit(
		TrancheVault vault,
		address lp,
		uint256 requestId
	) internal returns (uint256 shares) {
		uint256 claimable = vault.claimableDepositRequest(requestId, lp);
		require(claimable > 0, "deposit not claimable");
		vm.prank(lp);
		shares = vault.claimDeposit(requestId, claimable, lp, lp);
	}
	function _lpDeposit(
		TrancheVault vault,
		address lp,
		uint256 assets
	) internal returns (uint256 shares) {
		uint256 id = _lpRequestDeposit(vault, lp, assets);
		_settleEpoch();
		shares = _lpClaimDeposit(vault, lp, id);
	}
	function _lpRequestRedeem(
		TrancheVault vault,
		address lp,
		uint256 shares
	) internal returns (uint256 requestId) {
		vm.prank(lp);
		requestId = vault.requestRedeem(shares, lp, lp);
	}
	function _lpClaimRedeem(
		TrancheVault vault,
		address lp,
		uint256 requestId
	) internal returns (uint256 assets) {
		uint256 claimable = vault.claimableRedeemRequest(requestId, lp);
		require(claimable > 0, "redeem not claimable");
		vm.prank(lp);
		assets = vault.claimRedeem(requestId, claimable, lp, lp);
	}
	function _settleEpoch() internal {
		_warpToNextEpoch();
		_settleEpochNow();
	}
	function _settleEpochNow() internal {
		uint256 t = vm.getBlockTimestamp();
		uint256 basket8 = _priceAt(t, lastBasket);
		lastBasket = basket8;
		lastPublishTime = t;
		bytes[] memory data = _pythData(basket8, t, t - 1);
		uint256 fee = oracle.getUpdateFee(data);
		vm.prank(keeper);
		router.settleLpEpoch{value: fee}(data);
	}
	function _sizeFor(
		uint256 notional6,
		uint256 basket8
	) internal pure returns (uint256 size) {
		size = (notional6 * 1e20) / basket8;
		size -= size % CfdTypes.SIZE_QUANTUM;
	}
	function _size(
		address trader
	) internal view returns (uint256 size) {
		(size,,,,,,) = engine.positions(trader);
	}
	function _usdgOf(
		address who
	) internal view returns (uint256) {
		return usdg.balanceOf(who) + clearinghouse.balanceAssets(who);
	}
	function _equity(
		address who
	) internal view returns (uint256) {
		return _usdgOf(who) + engine.traderClaimBalanceAssets(who);
	}
	function _depth() internal view returns (uint256) {
		uint256 p = pool.totalAssets();
		uint256 c = engine.totalTraderClaimBalanceAssets();
		return p > c ? p - c : 0;
	}
	function _skew() internal view returns (uint256) {
		(, uint256 longOi,,) = engine.sides(uint8(CfdTypes.Side.LONG));
		(, uint256 shortOi,,) = engine.sides(uint8(CfdTypes.Side.SHORT));
		uint256 p = engine.lastMarkPrice();
		uint256 l = longOi * p / 1e20;
		uint256 sh = shortOi * p / 1e20;
		return l > sh ? l - sh : sh - l;
	}
	function _usdS(
		int256 x
	) internal pure returns (string memory) {
		uint256 a = x < 0 ? uint256(-x) : uint256(x);
		bytes memory f = bytes(vm.toString(a % 1e6));
		bytes memory pad = new bytes(6 - f.length);
		for (uint256 i; i < pad.length; ++i) {
			pad[i] = "0";
		}
		return string.concat(x < 0 ? "-" : "", vm.toString(a / 1e6), ".", string(pad), string(f));
	}
	function _usd(
		uint256 x
	) internal pure returns (string memory) {
		return _usdS(int256(x));
	}
}
abstract contract VpiRefundLpExitScenario is VpiRefundLpExitFixture {
	uint256 internal constant DEP = 6_000_000e6;
	uint256 internal constant NOTIONAL = 6_000_000e6;
	uint256 internal constant BUFFER = 10_000e6;
	uint256 internal constant HOLD_25H = 24 hours + 1;
	uint256 internal constant ENTRANT_DEP = 6_500_000e6;
	uint256 internal constant WALK_SEEDS = 32;
	uint256 internal entrantReqId;
	struct Cycle {
		uint256 startUsdg;
		uint256 deposit;
		uint256 brought;
		uint256 finalUsdg;
		int256 net;
		uint256 redeemed;
		int256 tradingLeg;
		uint256 openCost;
		uint256 skewBeforeOpen;
		uint256 skewAfterOpen;
		int256 vpiCharged;
		uint256 openTime;
		uint256 fundTime;
		uint256 depthOpen;
		uint256 depthClose;
		uint256 poolBeforeFunding;
		int256 vpiCloseUnchangedDepth;
		int256 vpiCloseSimulated;
		int256 vpiAtClose;
		uint256 closeRebateAssessed;
		uint256 closeRebatePaid;
		uint256 closeRebateWaived;
		uint256 gasUsed;
	}
	struct Stayers {
		uint256 exitPaid;
		uint256 juniorLeft;
		uint256 seniorLeft;
		uint256 treasury;
		uint256 entrantPaid;
	}
	function _cycle(
		address who,
		uint256 dep,
		uint256 notional,
		bool trade,
		uint256 hold,
		uint256 entrantDep
	) internal returns (Cycle memory c) {
		vm.deal(who, 1 ether);
		c.startUsdg = _equity(who);
		c.deposit = dep;
		c.brought = dep;
		uint256 rid = _lpRequestDeposit(juniorVault, who, dep);
		_settleEpoch();
		uint256 shares = _lpClaimDeposit(juniorVault, who, rid);
		_warpToNextEpoch();
		_setPrice(lastBasket);
		uint256 redId;
		if (hold == 0) {
			redId = _lpRequestRedeem(juniorVault, who, shares);
			if (entrantDep > 0) {
				entrantReqId = _lpRequestDeposit(juniorVault, makeAddr("lpJunior3"), entrantDep);
			}
			uint256 next = pool.lpEpochStart(pool.currentLpEpoch() + 1);
			_warp(next - 60 - vm.getBlockTimestamp());
			if (walkOn) {
				walkT0 = vm.getBlockTimestamp();
				walkP0 = lastBasket;
			}
			_setPrice(lastBasket);
		}
		uint256 clearinghouseIn;
		if (trade) {
			uint256 margin = notional / 10;
			clearinghouseIn = margin + BUFFER;
			c.brought += clearinghouseIn;
			_fundTrader(who, clearinghouseIn);
			uint256 equityBefore = _equity(who);
			OrderV3Types.OrderRequest memory r = _req(CfdTypes.Side.LONG, _sizeFor(notional, lastBasket), margin, false);
			uint64 id = _commitReq(who, r);
			c.depthOpen = _depth();
			c.skewBeforeOpen = _skew();
			require(
				_executeAs(who, id, lastBasket).status == OrderV3Types.LifecycleStatus.Executed, "open not executed"
			);
			c.skewAfterOpen = _skew();
			c.openCost = equityBefore - _equity(who);
			(,,,,,, c.vpiCharged) = engine.positions(who);
		}
		c.openTime = vm.getBlockTimestamp();
		if (hold > 0) {
			uint256 e = pool.currentLpEpoch() + 1;
			while (pool.lpEpochStart(e) < c.openTime + hold) {
				++e;
			}
			uint256 boundary = pool.lpEpochStart(e);
			_warp(boundary - 360 - vm.getBlockTimestamp());
			_setPrice(lastBasket);
			redId = _lpRequestRedeem(juniorVault, who, shares);
			if (entrantDep > 0) {
				entrantReqId = _lpRequestDeposit(juniorVault, makeAddr("lpJunior3"), entrantDep);
			}
			_warp(boundary - 60 - vm.getBlockTimestamp());
			_setPrice(lastBasket);
		}
		c.poolBeforeFunding = pool.totalAssets();
		if (trade) {
			c.vpiCloseUnchangedDepth =
				engineLens.previewClose(who, _size(who), engine.lastMarkPrice()).vpiDeltaAssets;
		}
		_settleEpoch();
		c.fundTime = vm.getBlockTimestamp();
		c.redeemed = _lpClaimRedeem(juniorVault, who, redId);
		if (trade) {
			_setPrice(lastBasket);
			c.depthClose = _depth();
			ICfdEngineTypes.ClosePreview memory pv = engineLens.previewClose(who, _size(who), engine.lastMarkPrice());
			c.vpiAtClose = pv.vpiDeltaAssets;
			c.vpiCloseSimulated = engineLens.simulateClose(
					who, _size(who), engine.lastMarkPrice(), c.poolBeforeFunding
				).vpiDeltaAssets;
			OrderV3Types.OrderRequest memory r2 = _req(CfdTypes.Side.LONG, _size(who), 0, true);
			uint64 cid = _commitReq(who, r2);
			vm.recordLogs();
			require(
				_executeAs(who, cid, lastBasket).status == OrderV3Types.LifecycleStatus.Executed, "close not executed"
			);
			(c.closeRebateAssessed, c.closeRebatePaid, c.closeRebateWaived) = _closeRebate(who);
			uint256 bal = clearinghouse.balanceAssets(who);
			c.tradingLeg = int256(bal + engine.traderClaimBalanceAssets(who)) - int256(clearinghouseIn);
			if (bal > 0) {
				vm.prank(who);
				clearinghouse.withdraw(who, bal);
			}
		}
		c.finalUsdg = _equity(who);
		c.net = int256(c.finalUsdg) - int256(c.startUsdg) - int256(c.brought);
	}
	function _closeRebate(
		address who
	) internal view returns (uint256 assessed, uint256 paid, uint256 waived) {
		Vm.Log[] memory logs = vm.getRecordedLogs();
		bytes32 sig = ICfdEngineSettlementSidecar.ActionRebateSettled.selector;
		for (uint256 i; i < logs.length; ++i) {
			if (
				logs[i].emitter == address(settlementSidecar) && logs[i].topics.length == 2 && logs[i].topics[0] == sig
					&& logs[i].topics[1] == bytes32(uint256(uint160(who)))
			) {
				(assessed, paid, waived) = abi.decode(logs[i].data, (uint256, uint256, uint256));
			}
		}
	}
	function _stayerExit() internal returns (Stayers memory st) {
		_setPrice(lastBasket);
		uint256 rid = _lpRequestRedeem(juniorVault, lpJunior2, juniorVault.balanceOf(lpJunior2));
		_settleEpoch();
		st.exitPaid = _lpClaimRedeem(juniorVault, lpJunior2, rid);
		st.juniorLeft = pool.juniorPrincipal();
		st.seniorLeft = pool.seniorPrincipal();
		st.treasury = _usdgOf(engine.protocolTreasury());
	}
	function _entrantExit() internal returns (uint256 paid) {
		address lpJunior3 = makeAddr("lpJunior3");
		uint256 shares3 = _lpClaimDeposit(juniorVault, lpJunior3, entrantReqId);
		_warpToNextEpoch();
		_warpToNextEpoch();
		_setPrice(lastBasket);
		uint256 r3 = _lpRequestRedeem(juniorVault, lpJunior3, shares3);
		_settleEpoch();
		paid = _lpClaimRedeem(juniorVault, lpJunior3, r3);
	}
	function _branch(
		uint256 dep,
		uint256 notional,
		bool trade,
		uint256 hold,
		uint256 entrantDep
	) internal returns (Cycle memory c, Stayers memory st) {
		uint256 g = gasleft();
		c = _cycle(attacker, dep, notional, trade, hold, entrantDep);
		c.gasUsed = g - gasleft();
		st = _stayerExit();
		if (entrantDep > 0) {
			st.entrantPaid = _entrantExit();
		}
	}
	function _attackVsControl(
		uint256 dep,
		uint256 notional,
		uint256 hold,
		uint256 entrantDep
	) internal returns (Cycle memory atk, Cycle memory ctl, Stayers memory sAtk, Stayers memory sCtl) {
		uint256 s = vm.snapshotState();
		(ctl, sCtl) = _branch(dep, notional, false, hold, entrantDep);
		vm.revertToState(s);
		(atk, sAtk) = _branch(dep, notional, true, hold, entrantDep);
	}
	function _stayersChange(
		Stayers memory sAtk,
		Stayers memory sCtl
	) internal pure returns (int256) {
		return (int256(sAtk.exitPaid) - int256(sCtl.exitPaid)) + (int256(sAtk.juniorLeft) - int256(sCtl.juniorLeft))
			+ (int256(sAtk.seniorLeft) - int256(sCtl.seniorLeft));
	}
	function _log(
		string memory title,
		Cycle memory atk,
		Cycle memory ctl,
		Stayers memory sAtk,
		Stayers memory sCtl
	) internal pure {
		console2.log(title);
		console2.log("  claim-net pool depth right before the open (USDG):", _usd(atk.depthOpen));
		console2.log("  open: VPI paid by the attacker's LONG (lifetime VPI after the open, USDG):", _usdS(atk.vpiCharged));
		console2.log("  book skew right before the open, |long - short| at the mark (USDG):", _usd(atk.skewBeforeOpen));
		console2.log("  book skew right after the open (USDG):", _usd(atk.skewAfterOpen));
		console2.log("  book skew after the open, share of the depth before it (bps):", atk.skewAfterOpen * 10_000 / atk.depthOpen);
		console2.log("  open: USDG the open cost the attacker, VPI + execution fee (USDG):", _usd(atk.openCost));
		console2.log("  open -> settlement that funds the redemption (seconds):", atk.fundTime - atk.openTime);
		console2.log("  redemption paid, attack (USDG):", _usd(atk.redeemed));
		console2.log("  redemption paid, no-trade control (USDG):", _usd(ctl.redeemed));
		console2.log(
			"  redemption paid above the control (USDG):", _usdS(int256(atk.redeemed) - int256(ctl.redeemed))
		);
		console2.log("  claim-net pool depth right before the close (USDG):", _usd(atk.depthClose));
		console2.log("  close: VPI of the close, preview right before it (USDG, negative = rebate):", _usdS(atk.vpiAtClose));
		console2.log("  close: net action rebate assessed, event (USDG):", _usd(atk.closeRebateAssessed));
		console2.log("  close: net action rebate paid out of the pool, event (USDG):", _usd(atk.closeRebatePaid));
		console2.log("  close: net action rebate waived, event (USDG):", _usd(atk.closeRebateWaived));
		console2.log(
			"  close: execution fee and carry withheld from the VPI rebate, kept by the pool (USDG):",
			_usdS(-atk.vpiAtClose - int256(atk.closeRebateAssessed))
		);
		console2.log("  attacker: USDG brought, deposit + margin + buffer (USDG):", _usd(atk.brought));
		console2.log("  attacker: USDG held at the end (USDG):", _usd(atk.finalUsdg));
		console2.log("  attacker net, attack (USDG):", _usdS(atk.net));
		console2.log("  attacker net, no-trade control (USDG):", _usdS(ctl.net));
		console2.log("  attacker gain over the control (USDG):", _usdS(atk.net - ctl.net));
		console2.log(
			"  attacker LP leg, redeemed - deposited (USDG):", _usdS(int256(atk.redeemed) - int256(atk.deposit))
		);
		console2.log("  attacker trading leg, clearinghouse out - in (USDG):", _usdS(atk.tradingLeg));
		console2.log("  stayer lpJunior2 (deposited 1,000,000) real exit, attack (USDG):", _usd(sAtk.exitPaid));
		console2.log("  stayer lpJunior2 real exit, no-trade control (USDG):", _usd(sCtl.exitPaid));
		console2.log(
			"  stayer lpJunior2 change vs the control (USDG):", _usdS(int256(sAtk.exitPaid) - int256(sCtl.exitPaid))
		);
		console2.log(
			"  other Junior principal after that settlement, change vs the control (USDG):",
			_usdS(int256(sAtk.juniorLeft) - int256(sCtl.juniorLeft))
		);
		console2.log(
			"  Senior principal after that settlement, change vs the control (USDG):",
			_usdS(int256(sAtk.seniorLeft) - int256(sCtl.seniorLeft))
		);
		console2.log(
			"  all stayers (every LP but the attacker), change vs the control (USDG):",
			_usdS(_stayersChange(sAtk, sCtl))
		);
		console2.log(
			"  protocol treasury, change vs the control (USDG):", _usdS(int256(sAtk.treasury) - int256(sCtl.treasury))
		);
	}
}
contract VpiRefundLpExitTest is VpiRefundLpExitScenario {
	function test_attack() public {
		_boot();
		console2.log("boot: pool assets / Junior principal / Senior principal (USDG):");
		console2.log("  ", _usd(pool.totalAssets()), _usd(pool.juniorPrincipal()), _usd(pool.seniorPrincipal()));
		(Cycle memory atk, Cycle memory ctl, Stayers memory sAtk, Stayers memory sCtl) =
			_attackVsControl(DEP, NOTIONAL, 0, 0);
		_log("ATTACK: 60% of Junior, 6,000,000 LONG", atk, ctl, sAtk, sCtl);
		console2.log(
			"  gas of the whole attack cycle as run here (every call incl. the keeper's, plus test overhead):",
			atk.gasUsed
		);
		int256 gain = atk.net - ctl.net;
		int256 lpLeg = int256(atk.redeemed) - int256(atk.deposit);
		assertGt(atk.vpiCharged, 0, "the open paid VPI");
		assertGt(atk.finalUsdg, atk.brought, "the attacker ends with more USDG than all the USDG it brought");
		assertGt(atk.net, 0, "attacker net > 0 after both execution fees");
		assertGt(gain, 2000e6, "gain over the no-trade control > 2,000 USDG, after both execution fees");
		assertGt(atk.redeemed, ctl.redeemed, "the redemption is funded at the VPI-raised price");
		assertLt(sAtk.exitPaid, sCtl.exitPaid, "the stayer's real exit pays less than in the control");
		assertLt(sAtk.exitPaid, LP_JUNIOR2, "the stayer's real exit is below its own 1,000,000 deposit");
		assertGt(-_stayersChange(sAtk, sCtl), gain, "the stayers lose more than the attacker gains");
		assertEq(
			-_stayersChange(sAtk, sCtl),
			gain + (int256(sAtk.treasury) - int256(sCtl.treasury)),
			"the stayers' loss is exactly the attacker's gain plus the treasury's open fee"
		);
		assertGt(
			int256(atk.redeemed) - int256(ctl.redeemed),
			atk.vpiCharged / 2,
			"the redemption carries more than half of the attacker's own charge"
		);
		assertEq(-atk.vpiAtClose, atk.vpiCharged, "the close is rebated exactly the open's charge");
		assertEq(atk.closeRebatePaid, atk.closeRebateAssessed, "the pool pays the close's net rebate in full");
		assertEq(atk.closeRebateWaived, 0, "nothing of the close's rebate is waived");
		assertEq(atk.net, lpLeg + atk.tradingLeg, "the attacker's result is its LP leg plus its trading leg");
		assertEq(_size(attacker), 0, "no position left");
		assertEq(juniorVault.balanceOf(attacker), 0, "no Junior shares left");
		assertEq(clearinghouse.balanceAssets(attacker), 0, "no clearinghouse balance left");
		assertEq(engine.traderClaimBalanceAssets(attacker), 0, "no trader claim left");
	}
	function test_releaseScale() public {
		_boot();
		address lpJunior3 = makeAddr("lpJunior3");
		address lpSenior3 = makeAddr("lpSenior3");
		_setPrice(lastBasket);
		_lpDeposit(juniorVault, lpJunior3, 7_500_000e6);
		_setPrice(lastBasket);
		_lpDeposit(seniorVault, lpSenior3, 31_990_000e6);
		console2.log("Senior principal before the attacker (USDG):", _usd(pool.seniorPrincipal()));
		console2.log("Junior principal before the attacker (USDG):", _usd(pool.juniorPrincipal()));
		uint256 dep = 17_250_000e6;
		uint256 depth = pool.totalAssets() + dep;
		uint256 notional = depth * 38 / 100 - 860_000e6;
		console2.log("pool depth after the attacker's deposit (USDG):", _usd(depth));
		console2.log("attacker LONG notional (USDG):", _usd(notional));
		(Cycle memory atk, Cycle memory ctl, Stayers memory sAtk, Stayers memory sCtl) =
			_attackVsControl(dep, notional, 0, 0);
		_log("RELEASE SCALE: Senior at the cap, attacker 60% of Junior", atk, ctl, sAtk, sCtl);
		int256 gain = atk.net - ctl.net;
		assertGt(pool.seniorPrincipal(), 39_000_000e6, "Senior near the 40M release cap");
		assertGt(atk.finalUsdg, atk.brought, "the attacker ends with more USDG than all the USDG it brought");
		assertGt(gain, 5000e6, "gain over the no-trade control > 5,000 USDG at release scale");
		assertLt(sAtk.exitPaid, sCtl.exitPaid, "the stayer's real exit pays less than in the control");
		assertLt(sAtk.exitPaid, LP_JUNIOR2, "the stayer's real exit is below its own 1,000,000 deposit");
		assertGt(-_stayersChange(sAtk, sCtl), gain, "the stayers lose more than the attacker gains");
	}
	function test_repeatedCycles() public {
		_boot();
		uint256 n = 3;
		uint256 s = vm.snapshotState();
		int256 ctlSum;
		for (uint256 i; i < n; ++i) {
			ctlSum += _cycle(attacker, DEP, NOTIONAL, false, 0, 0).net;
		}
		Stayers memory sCtl = _stayerExit();
		vm.revertToState(s);
		uint256 t0 = vm.getBlockTimestamp();
		int256 atkSum;
		int256[] memory nets = new int256[](n);
		for (uint256 i; i < n; ++i) {
			Cycle memory c = _cycle(attacker, DEP, NOTIONAL, true, 0, 0);
			nets[i] = c.net;
			atkSum += c.net;
			console2.log(string.concat("  cycle ", vm.toString(i + 1), ": attacker net (USDG):"), _usdS(c.net));
		}
		uint256 elapsed = vm.getBlockTimestamp() - t0;
		Stayers memory sAtk = _stayerExit();
		console2.log("REPEATED CYCLES: 3 cycles of the attack");
		console2.log("  time for the three cycles (seconds):", elapsed);
		console2.log("  attacker net over 3 cycles, attack (USDG):", _usdS(atkSum));
		console2.log("  attacker net over 3 cycles, no-trade control (USDG):", _usdS(ctlSum));
		console2.log("  attacker gain over the control, 3 cycles (USDG):", _usdS(atkSum - ctlSum));
		console2.log("  stayer lpJunior2 (deposited 1,000,000) real exit, attack (USDG):", _usd(sAtk.exitPaid));
		console2.log("  stayer lpJunior2 real exit, no-trade control (USDG):", _usd(sCtl.exitPaid));
		console2.log(
			"  stayer lpJunior2 change vs the control (USDG):", _usdS(int256(sAtk.exitPaid) - int256(sCtl.exitPaid))
		);
		console2.log(
			"  all stayers (every LP but the attacker), change vs the control (USDG):",
			_usdS(_stayersChange(sAtk, sCtl))
		);
		for (uint256 i; i < n; ++i) {
			assertGt(nets[i], 0, "every cycle ends with more USDG than the attacker brought to it");
		}
		assertGt(atkSum - ctlSum, 0, "the three cycles beat the no-trade control");
		assertLt(sAtk.exitPaid, LP_JUNIOR2, "principal loss: the stayer exits below its own 1,000,000 deposit");
		assertLt(sAtk.exitPaid, sCtl.exitPaid, "the stayer's real exit is below the control's");
	}
	function test_depthKept() public {
		_boot();
		(Cycle memory atk, Cycle memory ctl, Stayers memory sAtk, Stayers memory sCtl) =
			_attackVsControl(DEP, NOTIONAL, 0, ENTRANT_DEP);
		_log("DEPTH KEPT: unrelated 6,500,000 Junior deposit activates in the attacker's exit settlement", atk, ctl, sAtk, sCtl);
		int256 gain = atk.net - ctl.net;
		console2.log("  entrant lpJunior3 (deposited 6,500,000) real exit, attack (USDG):", _usd(sAtk.entrantPaid));
		console2.log("  entrant lpJunior3 real exit, no-trade control (USDG):", _usd(sCtl.entrantPaid));
		console2.log(
			"  entrant lpJunior3 change vs the control (USDG):",
			_usdS(int256(sAtk.entrantPaid) - int256(sCtl.entrantPaid))
		);
		assertGt(atk.depthClose, atk.depthOpen, "the close runs at more depth than the open");
		assertLt(-atk.vpiAtClose, atk.vpiCharged, "the close's curve rebate is below the charge: the clamp does not bind");
		assertGt(gain, 2000e6, "the attack still gains > 2,000 USDG over the control");
		assertLt(sAtk.entrantPaid, sCtl.entrantPaid, "the entrant's real exit is below the control's");
		assertLt(sAtk.entrantPaid, ENTRANT_DEP, "principal loss: the entrant exits below its own 6,500,000 deposit");
	}
	function test_depthCutShareOfRebate() public {
		_boot();
		(Cycle memory atk, Cycle memory ctl,,) = _attackVsControl(DEP, NOTIONAL, 0, 0);
		int256 gain = atk.net - ctl.net;
		int256 depthCut = atk.vpiCloseUnchangedDepth - atk.vpiAtClose;
		console2.log("DEPTH CUT: share of the close rebate and of the gain");
		console2.log("  open: VPI charged (USDG):", _usdS(atk.vpiCharged));
		console2.log("  claim-net depth right before the open (USDG):", _usd(atk.depthOpen));
		console2.log("  pool assets right before the funding settlement (USDG):", _usd(atk.poolBeforeFunding));
		console2.log("  claim-net depth right before the close (USDG):", _usd(atk.depthClose));
		console2.log(
			"  close VPI at unchanged depth, preview after the open and before the funding (USDG):",
			_usdS(atk.vpiCloseUnchangedDepth)
		);
		console2.log(
			"  close VPI at unchanged depth, simulateClose at the close's instant (USDG):", _usdS(atk.vpiCloseSimulated)
		);
		console2.log("  close VPI the real close receives, clamped (USDG):", _usdS(atk.vpiAtClose));
		console2.log("  rebate added by the depth cut, against the pre-funding preview (USDG):", _usdS(depthCut));
		console2.log(
			"  rebate added by the depth cut, against simulateClose at the close's instant (USDG):",
			_usdS(atk.vpiCloseSimulated - atk.vpiAtClose)
		);
		console2.log(
			"  rebate without the depth cut, as a share of the charge (basis points):",
			-atk.vpiCloseUnchangedDepth * 10_000 / atk.vpiCharged
		);
		console2.log("  redemption paid above the control (USDG):", _usdS(int256(atk.redeemed) - int256(ctl.redeemed)));
		console2.log("  attacker gain over the control (USDG):", _usdS(gain));
		console2.log("  attacker gain without the depth cut's addition (USDG):", _usdS(gain - depthCut));
		console2.log("  depth cut's addition as a share of the gain (basis points):", depthCut * 10_000 / gain);
		assertGt(gain - depthCut, 2000e6, "without the depth cut's addition the attack still gains > 2,000 USDG");
		assertLt(depthCut * 100, gain, "the depth cut adds less than 1% of the gain");
		assertApproxEqAbs(
			atk.vpiCloseSimulated, atk.vpiCloseUnchangedDepth, 0.01e6, "both unchanged-depth reads agree within 0.01 USDG"
		);
		assertEq(-atk.vpiAtClose, atk.vpiCharged, "the real close is rebated exactly the charge (the clamp binds)");
		assertLt(-atk.vpiCloseUnchangedDepth, atk.vpiCharged, "at unchanged depth the curve rebates less than the charge");
	}
	function _passive(
		bool trade
	) internal returns (uint256 exiterPaid, uint256 stayerPaid, int256 vpi, int256 closeVpi, int256 traderNet) {
		address t = makeAddr("dayTrader");
		vm.deal(t, 1 ether);
		uint256 rid = _lpRequestRedeem(juniorVault, lpJunior2, juniorVault.balanceOf(lpJunior2));
		uint256 next = pool.lpEpochStart(pool.currentLpEpoch() + 1);
		_warp(next - 1800 - vm.getBlockTimestamp());
		_setPrice(lastBasket);
		if (trade) {
			_fundTrader(t, 380_000e6 + 38_000e6);
			_open(t, CfdTypes.Side.LONG, _sizeFor(3_800_000e6, lastBasket), 380_000e6);
			(,,,,,, vpi) = engine.positions(t);
		}
		_settleEpoch();
		exiterPaid = _lpClaimRedeem(juniorVault, lpJunior2, rid);
		_warp(30 minutes);
		_setPrice(lastBasket);
		if (trade) {
			closeVpi = engineLens.previewClose(t, _size(t), engine.lastMarkPrice()).vpiDeltaAssets;
			_close(t, lastBasket);
			traderNet = int256(_equity(t)) - int256(380_000e6 + 38_000e6);
		}
		_setPrice(lastBasket);
		uint256 rid1 = _lpRequestRedeem(juniorVault, lpJunior1, juniorVault.balanceOf(lpJunior1) / 3);
		_settleEpoch();
		stayerPaid = _lpClaimRedeem(juniorVault, lpJunior1, rid1);
	}
	function test_passiveExit() public {
		_boot();
		uint256 s = vm.snapshotState();
		(uint256 exitCtl, uint256 stayCtl,,,) = _passive(false);
		vm.revertToState(s);
		(uint256 exitAtk, uint256 stayAtk, int256 vpi, int256 closeVpi, int256 traderNet) = _passive(true);
		console2.log("PASSIVE: an LP exits while an unrelated trader's skew-widening LONG is open");
		console2.log("  day trader's 3,800,000 LONG: VPI paid at the open (USDG):", _usdS(vpi));
		console2.log("  day trader's close: VPI, preview right before it (USDG, negative = rebate):", _usdS(closeVpi));
		console2.log("  day trader net over the round trip (USDG):", _usdS(traderNet));
		console2.log("  exiter lpJunior2 paid, no trade (USDG):", _usd(exitCtl));
		console2.log("  exiter lpJunior2 paid, with the trade (USDG):", _usd(exitAtk));
		console2.log("  exiter paid above the no-trade run (USDG):", _usdS(int256(exitAtk) - int256(exitCtl)));
		console2.log("  stayer (1/3 of lpJunior1's shares) paid, no trade (USDG):", _usd(stayCtl));
		console2.log("  stayer (1/3 of lpJunior1's shares) paid, with the trade (USDG):", _usd(stayAtk));
		console2.log("  stayer change vs the no-trade run (USDG):", _usdS(int256(stayAtk) - int256(stayCtl)));
		assertGt(exitAtk, exitCtl + 1000e6, "the exiter is paid > 1,000 USDG more than without the trade");
		assertLt(stayAtk, stayCtl, "the stayer's real exit is below the no-trade run");
	}
	function _freshDeposit(
		bool trade
	) internal returns (int256 net, uint256 poolEnd, uint256 stayerPaid) {
		vm.deal(attacker, 1 ether);
		uint256 start = _equity(attacker);
		uint256 minted = 20_000_000e6;
		uint256 rid = _lpRequestDeposit(juniorVault, attacker, 20_000_000e6);
		_settleEpoch();
		_lpClaimDeposit(juniorVault, attacker, rid);
		_setPrice(SEEDED_BASKET);
		if (trade) {
			minted += 1_600_000e6 + 80_000e6;
			_fundTrader(attacker, 1_600_000e6 + 80_000e6);
			_open(attacker, CfdTypes.Side.LONG, _sizeFor(8_000_000e6, SEEDED_BASKET), 1_600_000e6, SEEDED_BASKET);
		}
		_warp(1 hours);
		_setPrice(SEEDED_BASKET);
		uint256 rr = _lpRequestRedeem(juniorVault, attacker, juniorVault.balanceOf(attacker));
		_settleEpoch();
		_lpClaimRedeem(juniorVault, attacker, rr);
		if (trade) {
			_setPrice(SEEDED_BASKET);
			_close(attacker, SEEDED_BASKET);
		}
		net = int256(_equity(attacker)) - int256(start) - int256(minted);
		poolEnd = pool.totalAssets();
		stayerPaid = _stayerExit().exitPaid;
	}
	function test_freshDepositPath() public {
		_boot();
		uint256 s = vm.snapshotState();
		(int256 bNet, uint256 bPool, uint256 bStay) = _freshDeposit(false);
		vm.revertToState(s);
		(int256 dNet, uint256 dPool, uint256 dStay) = _freshDeposit(true);
		console2.log("FRESH DEPOSIT: 20,000,000 Junior, 8,000,000 LONG right after activation");
		console2.log("  attacker net, deposit + trade (USDG):", _usdS(dNet));
		console2.log("  attacker net, deposit only (USDG):", _usdS(bNet));
		console2.log("  attacker gain over the deposit-only timeline (USDG):", _usdS(dNet - bNet));
		console2.log("  pool assets after the attacker has left, deposit only (USDG):", _usd(bPool));
		console2.log("  pool assets after the attacker has left, deposit + trade (USDG):", _usd(dPool));
		console2.log("  pool assets change (USDG):", _usdS(int256(dPool) - int256(bPool)));
		console2.log("  stayer lpJunior2 real exit, deposit only (USDG):", _usd(bStay));
		console2.log("  stayer lpJunior2 real exit, deposit + trade (USDG):", _usd(dStay));
		console2.log("  stayer lpJunior2 change (USDG):", _usdS(int256(dStay) - int256(bStay)));
		assertGt(dNet, 0, "the attacker ends with more USDG than it brought");
		assertGt(dNet - bNet, 2000e6, "gain over the deposit-only timeline > 2,000 USDG");
		assertLt(dPool, bPool, "the pool the stayers own ends lower");
		assertLt(dStay, bStay, "the stayer's real exit is below the deposit-only timeline");
	}
	function test_hold25h() public {
		_boot();
		(Cycle memory atk, Cycle memory ctl, Stayers memory sAtk, Stayers memory sCtl) =
			_attackVsControl(DEP, NOTIONAL, HOLD_25H, 0);
		_log("HOLD 25 H: redemption funded about 25 h after the open", atk, ctl, sAtk, sCtl);
		int256 gain = atk.net - ctl.net;
		assertGe(atk.fundTime - atk.openTime, 24 hours, "the redemption is funded at least 24 h after the open");
		assertGt(atk.finalUsdg, atk.brought, "the attacker ends with more USDG than all the USDG it brought");
		assertGt(gain, 2000e6, "gain over the control > 2,000 USDG after a day of carry");
		assertGt(int256(sCtl.exitPaid) - int256(sAtk.exitPaid), 1000e6, "the stayer's real exit pays > 1,000 less");
		assertLt(sAtk.exitPaid, LP_JUNIOR2, "principal loss: the stayer exits below its own 1,000,000 deposit");
	}
	function test_juniorShareSweep() public {
		_boot();
		uint256[6] memory deps =
			[uint256(1_715_000e6), 2_153_847e6, 2_666_667e6, 3_275_000e6, 6_000_000e6, 12_000_000e6];
		int256[6] memory gains;
		int256[6] memory harms;
		uint256 s0 = vm.snapshotState();
		console2.log("JUNIOR SHARE SWEEP");
		for (uint256 i; i < 6; ++i) {
			uint256 notional = (12_000_000e6 + deps[i]) * 38 / 100 - 860_000e6;
			(Cycle memory atk, Cycle memory ctl, Stayers memory sAtk, Stayers memory sCtl) =
				_attackVsControl(deps[i], notional, 0, 0);
			vm.revertToState(s0);
			gains[i] = atk.net - ctl.net;
			harms[i] = int256(sCtl.exitPaid) - int256(sAtk.exitPaid);
			console2.log(
				string.concat(
					"  Junior share ",
					vm.toString(deps[i] * 10_000 / (deps[i] + 4_000_001e6)),
					" bps, deposit ",
					_usd(deps[i]),
					", LONG ",
					_usd(notional)
				)
			);
			console2.log("    attacker gain over the control (USDG):", _usdS(gains[i]));
			console2.log("    stayer lpJunior2 real exit, lower than the control by (USDG):", _usdS(harms[i]));
		}
		assertGt(gains[4], 0, "60% of Junior gains");
		assertGt(gains[3], 0, "45% of Junior gains");
		assertGt(gains[5], 0, "75% of Junior gains");
		assertLt(gains[0], 0, "30% of Junior loses");
		for (uint256 i = 1; i < 6; ++i) {
			assertGt(gains[i], gains[i - 1], "the gain grows with the attacker's share of Junior");
		}
		for (uint256 i; i < 6; ++i) {
			assertGt(harms[i], 0, "the stayer pays in every row");
		}
	}
	function _walkSeed(
		uint256 s0,
		uint256 seed
	) internal returns (int256 gain, int256 net) {
		vm.revertToState(s0);
		walkOn = true;
		walkSeed = seed;
		(Cycle memory atk, Cycle memory ctl,,) = _attackVsControl(DEP, NOTIONAL, 0, 0);
		gain = atk.net - ctl.net;
		net = atk.net;
		console2.log(
			string.concat(
				"  seed ",
				vm.toString(seed),
				": basket at open fill / settlement / close fill ",
				vm.toString(_priceAt(walkT0 + 2, 0)),
				" / ",
				vm.toString(_priceAt(walkT0 + 61, 0)),
				" / ",
				vm.toString(_priceAt(walkT0 + 63, 0)),
				"; gain over the control ",
				_usdS(gain),
				"; attacker net ",
				_usdS(net)
			)
		);
	}
	function _movingBasket(
		uint256 firstSeed
	) internal {
		_boot();
		uint256 s0 = vm.snapshotState();
		uint256 n = WALK_SEEDS / 2;
		int256[] memory gains = new int256[](n);
		int256 sum;
		int256 lo = type(int256).max;
		int256 hi = type(int256).min;
		uint256 wins;
		uint256 absWins;
		console2.log(
			string.concat(
				"MOVING BASKET: seeded random walks, sigma 0.17 bp per sqrt(second); seeds ",
				vm.toString(firstSeed),
				" to ",
				vm.toString(firstSeed + n - 1)
			)
		);
		for (uint256 k; k < n; ++k) {
			(int256 g, int256 net) = _walkSeed(s0, firstSeed + k);
			gains[k] = g;
			sum += g;
			if (g < lo) {
				lo = g;
			}
			if (g > hi) {
				hi = g;
			}
			if (g > 0) {
				++wins;
			}
			if (net > 0) {
				++absWins;
			}
		}
		int256 mean = sum / int256(n);
		uint256 ss;
		for (uint256 k; k < n; ++k) {
			int256 d = gains[k] - mean;
			ss += uint256(d * d);
		}
		uint256 sd = Math.sqrt(ss / (n - 1));
		console2.log("  seeds:", n);
		console2.log("  mean gain over the control (USDG):", _usdS(mean));
		console2.log("  standard deviation of the gain (USDG):", _usd(sd));
		console2.log("  minimum gain (USDG):", _usdS(lo));
		console2.log("  maximum gain (USDG):", _usdS(hi));
		console2.log("  seeds with a gain over the control:", wins);
		console2.log("  seeds where the attacker ends with more USDG than it brought:", absWins);
		assertGt(mean, 2000e6, "mean gain over the control > 2,000 USDG");
		assertEq(wins, n, "every seed gains over its control");
		assertEq(absWins, n, "in every seed the attacker ends with more USDG than it brought");
	}
	function test_movingBasket_seeds01to16() public {
		_movingBasket(1);
	}
	function test_movingBasket_seeds17to32() public {
		_movingBasket(WALK_SEEDS / 2 + 1);
	}
	function test_honestFlows() public {
		_boot();
		address[6] memory ts = [alice, bob, carol, dave, erin, frank];
		for (uint256 i; i < 6; ++i) {
			_setPrice(lastBasket);
			_close(ts[i], lastBasket);
			assertEq(_size(ts[i]), 0, "the seeded trader's position is closed");
			assertGt(_equity(ts[i]), 0, "the seeded trader keeps its collateral");
		}
		address lpJunior3 = makeAddr("lpJunior3");
		_setPrice(lastBasket);
		uint256 r2 = _lpRequestRedeem(juniorVault, lpJunior2, juniorVault.balanceOf(lpJunior2));
		uint256 rs = _lpRequestRedeem(seniorVault, lpSenior2, seniorVault.balanceOf(lpSenior2) / 2);
		uint256 d3 = _lpRequestDeposit(juniorVault, lpJunior3, 500_000e6);
		_settleEpoch();
		uint256 paid2 = _lpClaimRedeem(juniorVault, lpJunior2, r2);
		uint256 paidS = _lpClaimRedeem(seniorVault, lpSenior2, rs);
		uint256 shares3 = _lpClaimDeposit(juniorVault, lpJunior3, d3);
		console2.log("HONEST FLOWS: six closes, a Junior exit, a Senior partial exit, a Junior entry");
		console2.log("  lpJunior2 full exit paid (USDG):", _usd(paid2));
		console2.log("  lpSenior2 half exit paid (USDG):", _usd(paidS));
		console2.log("  lpJunior3 Junior shares received for 500,000 USDG:", shares3);
		assertGt(paid2, 0, "the Junior exit is paid");
		assertGt(paidS, 0, "the Senior exit is paid");
		assertGt(shares3, 0, "the Junior entry receives shares");
	}
}
contract VpiRefundLpExitWiderConfidenceTest is VpiRefundLpExitScenario {
	uint256 internal confDivisor = 40_000;
	function _pythData(
		uint256 basket8,
		uint256 publishTime,
		uint256 prevPublishTime
	) internal view override returns (bytes[] memory data) {
		data = super._pythData(basket8, publishTime, prevPublishTime);
		MockPyth.FeedUpdate[] memory u = abi.decode(data[0], (MockPyth.FeedUpdate[]));
		for (uint256 i; i < u.length; ++i) {
			uint64 conf = uint64(uint256(uint64(u[i].price.price)) / confDivisor);
			if (conf == 0) {
				conf = 1;
			}
			u[i].price.conf = conf;
		}
		data[0] = abi.encode(u);
	}
	function test_widerPythConfidence() public {
		_boot();
		uint256 s0 = vm.snapshotState();
		uint256[5] memory divisors = [uint256(40_000), 5000, 2000, 1250, 1000];
		string[5] memory levels = ["0.25", "2", "5", "8", "10"];
		int256[5] memory gains;
		int256[5] memory stayerChanges;
		uint256[5] memory finals;
		uint256[5] memory broughts;
		uint256[5] memory confHundredthsBp;
		console2.log("WIDER PYTH CONFIDENCE: the attack and its no-trade control at the same per-feed confidence");
		for (uint256 i; i < 5; ++i) {
			vm.revertToState(s0);
			confDivisor = divisors[i];
			(Cycle memory atk, Cycle memory ctl, Stayers memory sAtk, Stayers memory sCtl) =
				_attackVsControl(DEP, NOTIONAL, 0, 0);
			(, uint256 basketConfidence) = oracle.getLatestPoolReconcilePrice();
			gains[i] = atk.net - ctl.net;
			stayerChanges[i] = int256(sAtk.exitPaid) - int256(sCtl.exitPaid);
			finals[i] = atk.finalUsdg;
			broughts[i] = atk.brought;
			confHundredthsBp[i] = basketConfidence * 1_000_000 / engine.lastMarkPrice();
			console2.log(
				string.concat(
					"  per-feed confidence ", levels[i], " bp of price (raw price / ", vm.toString(divisors[i]), ")"
				)
			);
			console2.log(
				"    aggregate basket confidence the oracle accepted (hundredths of a bp of price):", confHundredthsBp[i]
			);
			console2.log("    attacker gain over the control (USDG):", _usdS(gains[i]));
			console2.log("    attacker net, attack (USDG):", _usdS(atk.net));
			console2.log("    attacker net, no-trade control (USDG):", _usdS(ctl.net));
			console2.log("    attacker trading leg, clearinghouse out - in (USDG):", _usdS(atk.tradingLeg));
			console2.log("    stayer lpJunior2 change vs the control (USDG):", _usdS(stayerChanges[i]));
		}
		for (uint256 i; i < 5; ++i) {
			assertGt(gains[i], 0, "the attack gains over its same-confidence control");
			assertGt(finals[i], broughts[i], "the attacker ends with more USDG than all the USDG it brought");
			assertLt(stayerChanges[i], 0, "the stayer's real exit is below its same-confidence control");
		}
		assertGe(confHundredthsBp[4], 995, "the last level is at the cap: aggregate confidence >= 9.95 bp of price");
	}
}
```

