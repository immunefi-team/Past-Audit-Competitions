# \#99011 [SC-Medium] Open admission and the LP firewall spend cash already owed to an open winner, leaving trader claims larger than all pool assets

**Submitted on Nov 11th 2026 at 12:43:01 UTC by @Nescient for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99011
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Protocol insolvency

## Description

# Description

## Brief/Intro

Plether bounds every trader payout and backs it with LP cash: every risk-increasing open must leave pool cash above the largest directional maximum payout plus a buffer, and LP withdrawals must leave the same reserve. Both checks ignore profit that an open winner has already earned and can withdraw at any time, even though the engine measures that profit itself. The same cash therefore backs two payouts: the winner's existing gain and the maximum payout of a newly admitted opposite position.

After the winner is paid, the pool is short by the double-pledged amount. If the price then moves toward the admitted trader, that trader's payout exceeds every USDG the pool holds. In the PoC, with release parameters and every position closed, an honest trader's claim is 417,673.56 USDG and the pool holds 415,782.07. The protocol owes more than it has. Because claim settlement is all-or-nothing, the trader cannot withdraw any of the 417,673.56 USDG. Only owner action can make it payable.

## Vulnerability Details

The protocol's solvency quantities (`SECURITY.md:926-935`):

```text
P = HousePool.totalAssets()
C = totalTraderClaimBalance
E = max(P - C, 0)
L = max(bullMaxProfit, bearMaxProfit)
B = ceil(L * settlementBufferBps / 10_000)
```

Two gates protect `L`:

1. **Open admission** ("buffered entry solvency", `SECURITY.md:339`): every risk-increasing open requires post-op `E >= L + B`. `CfdEnginePlanLib._applyOpenSettlementBufferAfterPlan` implements it at L755-L772:

```solidity
SolvencyAccountingLib.SolvencyState memory currentState = SolvencyAccountingLib.buildSolvencyState(
    snap.poolCashAssets,
    SolvencyAccountingLib.getMaxLiability(snap.longSide.maxProfitAssets, snap.shortSide.maxProfitAssets),
    snap.totalTraderClaimBalanceAssets
);
// ...
satisfied = SolvencyAccountingLib.hasRequiredSettlementBuffer(
    result.effectiveAssetsAfterAssets, result.maxLiabilityAfterAssets, snap.settlementBufferBps
);
```

2. **LP withdrawal firewall** (`SECURITY.md:342`): redemption funding is limited to free cash above a base reserve of `C + L + B`.

Neither gate counts the marked profit of open winners:

- `E` is physical cash minus recorded claims. An open winner's gain becomes a claim only when the winner closes, so until then the gates treat it as free backing.
- `L` uses each side's entry-based endpoint payout (the long side's entry cost, the short side's `lots * CAP_PRICE - entryCost`). It does not change when the mark moves.

Taking `max(long, short)` is only safe if both sides settle at the same terminal price. A winner who can close now at the current mark breaks that assumption: the winner is paid at today's price, and the opposite side's maximum payout stays reachable at the other endpoint.

The engine already knows the amount. `engine.terminalNavSnapshot().terminalLpPriceDeltaAssets` reports what the book owes traders at the current mark. In the PoC it is below −390,000 USDG when the opposite open is admitted. Neither gate reads it.

The accounting spec says admission "must not count unrealized trader losses as spendable assets" (`ACCOUNTING_SPEC.md:250`), but it has no matching rule that reserves unrealized trader gains. The implementation counts the gain as free cash twice: once at admission and again at the LP firewall.

### Sequence

All parameters are the release values in `DEPLOYMENT.md`. Every trade goes through the production `OrderRouter.commitOrder` and a permissionless `executeOrder`. Every mark comes from the permissionless `OrderRouter.updateMarkPrice`. Each mark step is at most 2% and one market day apart.

1. The pool holds 1,000,000 USDG of Junior capital.
2. The basket falls from 1.00 to 0.6667 over 21 market days. Alice opens a 599,800-lot SHORT, which is inside the 40% skew limit at that price.
3. A Junior LP files an ordinary redemption request.
4. The basket rises to 1.3333 over 36 market days. The engine now measures more than 390,000 USDG owed to Alice.
5. Bob opens a 599,800-lot LONG at 1.3333. Its endpoint payout makes `L` about 799,733 USDG. Admission compares that with about 1,000,000 of cash, of which about 400,000 is already Alice's, and accepts.
6. The LP epoch settles. The firewall funds 195,871.48 USDG of the redemption and keeps only `L + B`, again counting Alice's gain as free cash.
7. Alice closes at the same price, 1.3333. `E` falls 389,404.31 USDG below `L`, and degraded mode latches. That gap is exactly the cash both gates counted twice.
8. The basket falls back over 37 market days to 0.6314. Bob's position is profitable, so liquidation reverts with `CfdEngine__PositionIsSolvent`. Bob closes.
9. With every position closed, Bob's claim is 417,673.56 USDG and the pool holds 415,782.07 USDG. The pool is short by 1,891.50 USDG.

The whole path takes 130 calendar days. If Bob closes lower, the shortfall grows by about 5,998 USDG per 0.01 of further decline (599,800 lots × 0.01). It can grow to about the double-pledged 389,404 USDG.

Had both gates reserved Alice's gain, effective assets at step 5 would have been about 600,000 USDG. Bob's open would then be limited to roughly 449,000 lots instead of 599,800, the LP firewall would have kept Alice's cash, and Alice's close could not have left `E` below `L`.

### Why this is not the documented "not pathwise" limitation

The design accepts that `max(long, short)` is not insurance against every future price path. That limitation concerns gains that may appear later. Here the gain already existed, was authenticated by the oracle and was measured by the engine before Bob was admitted and before the LP was paid. Step 7 happens at an unchanged price: the deficit comes from both gates spending cash that was already owed, not from a price move.

## Impact Details

The selected impact is **Protocol insolvency**.

The protocol's own runbook defines the state: "actual negative terminal LP equity or trader payout claims that physical cash cannot currently service" (`SECURITY.md:1501-1502`). The PoC reaches it with every position closed:

- **Liabilities exceed assets.** Trader claims are 417,673.56 USDG, and pool assets are 415,782.07 USDG.
- **Nothing is left to collect.** No position is open, all collectible carry has been realized, and no receivable remains.
- **The honest trader receives nothing.** `CfdEngine._settleTraderClaimBalance` reverts whenever `pool.totalAssets() < totalTraderClaimBalanceAssets` (`CfdEngine.sol:224`), and claims cannot be paid in part. Bob cannot withdraw any of his 417,673.56 USDG, even though the pool holds 99.5% of it.
- **Nobody but the owner can fix it.**
  - A USDG transfer to the pool does not help, because `HousePool.totalAssets()` returns `min(balance, accountedAssets)` (`HousePool.sol:426-429`).
  - While degraded mode is latched, new LP epochs revert with `HousePool__DegradedMode`, and new opens revert at commitment with `OrderRouter__DegradedMode` (`OrderLiquidationBatchLogic.sol:327-329`).
  - Every path to payment starts with the owner: a recapitalization through `CfdEngine.recapitalizePool` (`CfdEngine.sol:632`), or clearing degraded mode, which `clearDegradedMode` allows once no position is open because `L` is then zero. Clearing pays nothing by itself: the claim stays unpayable until new cash arrives.
- **The LP firewall doubles the deficit.** It let 195,871.48 USDG leave against cash owed to Alice. The PoC's second test shows a gap of 194,308.67 without the redemption and 389,404.31 with it.

No privileged role is involved. Alice, Bob and the LP each make ordinary user calls, and none of them needs to act maliciously. Any large opposite-side open while winners hold marked gains hits the same gap. The losses fall on traders whose admitted payouts the protocol promised to back.

On feasibility: the realized insolvency needs a large round trip in the basket. The contest scope states that feasibility limitations do not change a finding's impact or severity, only the amount paid. The double pledge itself needs no extreme move: the gap in step 7 appears at an unchanged price, and this root was confirmed as Medium in report #97605 on a 5% move. This report shows its Critical impact.

## Recommendation

1. In `_applyOpenSettlementBufferAfterPlan`, subtract the aggregate positive marked price PnL of open positions from effective assets before applying `E >= L + B`. Do not net it against unrealized losses, matching the existing rule that unrealized losses are not spendable.
2. Apply the same reserve in the LP withdrawal firewall, so the base reserve becomes `C + L + B + markedGains`.
3. Add winner-first settlement permutations, including an LP redemption between admission and the winner's close, to the solvency invariant suite.

## References

- Admission check: [`CfdEnginePlanLib.sol` L755-L772](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L755-L772)
- All-or-nothing claim settlement: [`CfdEngine.sol` L219-L230](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L219-L230) and [`settleTraderClaim` L559](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L559)
- Owner-only recapitalization: [`CfdEngine.sol` L632](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632)
- Degraded-mode latch: [`CfdEngine.sol` L1351-L1368](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1368)
- Pool assets ignore donations: [`HousePool.sol` L426-L429](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L426-L429)
- Commit-time degraded rejection: [`OrderLiquidationBatchLogic.sol` L327-L329](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L327-L329)
- Promises in `SECURITY.md`: [bounded payouts and headroom, L16 and L25](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L16-L25), [buffered entry solvency L339](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L339), [withdrawal firewall L342](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L342), [terminal-deficit definition L1499-L1502](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1499-L1502)
- [`ACCOUNTING_SPEC.md` admission rule L250](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L250)
- [`DEPLOYMENT.md` release parameters](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L125-L156)

## Proof of Concept

## Proof of Concept

Only Pyth and USDG are mocked. Every trade is a production V3 `OrderRouter.commitOrder(OrderRequest)` by the trader, followed by a permissionless `OrderRouter.executeOrder`. Marks use the permissionless `OrderRouter.updateMarkPrice`. Risk parameters are the release values. The execution fee, the 25 bps settlement buffer, `CAP_PRICE` (2.00) and the 60-second execution window are the unchanged defaults.

The PoC has two tests:

- `test_PoC_ProtocolInsolvencyAfterAllPositionsClose` runs steps 1-9 and asserts:
  1. all positions are closed and claims exceed pool assets;
  2. `settleTraderClaim` reverts;
  3. LP epoch settlement reverts;
  4. new opens revert.
- `test_LpFirewallAlsoIgnoresMarkedGain` isolates the firewall. It compares the post-winner gap with and without the matured LP redemption.

Save the following file as `packages/perps/test/perps/accounting/cash-priority/MarkedWinnerProtocolInsolvency.t.sol` in the frozen repository:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {ProtocolLensViewTypes} from "@plether/perps/interfaces/ProtocolLensViewTypes.sol";

/// @notice Open admission and the LP cash firewall both ignore marked profit owed to an open winner. The same pool
///         cash backs the winner's gain and a newly admitted opposite position. After the winner closes, the
///         admitted trader's later payout exceeds every USDG the pool holds, with no position left open.
/// @dev Every trade is a production V3 `OrderRouter.commitOrder` by the trader, followed by a permissionless
///      `OrderRouter.executeOrder`. Marks use the permissionless `OrderRouter.updateMarkPrice`. Only Pyth and USDG are
///      mocked. Risk parameters are the release values from DEPLOYMENT.md. Each mark step moves at most 2% and is
///      published one market day after the previous one.
contract MarkedWinnerProtocolInsolvencyPoC is BasePerpTest {

    address internal constant WINNER = address(0xA11CE);
    address internal constant ADMITTED_TRADER = address(0xB0B);
    uint256 internal constant START_PRICE = 1e8;
    uint256 internal constant SIZE = 599_800e18;
    uint256 internal constant LOW_ENTRY = 66_666_666;
    uint256 internal constant HIGH_ENTRY = 133_333_334;

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
        return 998_000e6;
    }

    /// @notice End state: all positions closed, trader claims exceed pool assets, and the honest trader can withdraw
    ///         none of the claim. While degraded mode stays latched, no new LP capital or trading can enter.
    function test_PoC_ProtocolInsolvencyAfterAllPositionsClose() public {
        uint256 pathStart = engine.lastMarkTime();
        (uint256 gap, uint256 lpFunded) = _setUpGap();
        uint256 priorPrice = HIGH_ENTRY;
        uint256 downSteps;
        uint256 closePrice;

        for (uint256 day = 1; day <= 120; ++day) {
            uint256 price = _twoPercentDown(priorPrice);
            _warpOneDayAndPublish(price);
            ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(ADMITTED_TRADER, SIZE, price);
            if (preview.traderClaimBalanceAssets > pool.totalAssets()) {
                downSteps = day;
                closePrice = price;
                break;
            }
            priorPrice = price;
        }
        assertGt(closePrice, 0, "path must reach an unpayable close");

        // The remaining position is profitable, so liquidation cannot turn it into pool cash.
        bytes[] memory liquidationData = _mockPythUpdateData(closePrice);
        vm.expectPartialRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        router.executeLiquidation(ADMITTED_TRADER, liquidationData);

        // The honest trader closes at the first price where the claim exceeds the pool.
        _trade(ADMITTED_TRADER, CfdTypes.Side.LONG, SIZE, 0, true, closePrice);

        uint256 claim = engine.traderClaimBalanceAssets(ADMITTED_TRADER);
        uint256 totalClaims = engine.totalTraderClaimBalanceAssets();
        uint256 poolAssets = pool.totalAssets();

        // 1. Insolvency: every position is closed and the pool owes more USDG than it holds.
        assertEq(_positionSize(WINNER), 0, "winner is closed");
        assertEq(_positionSize(ADMITTED_TRADER), 0, "honest trader is closed");
        assertEq(engine.unsettledCarryAssets(ADMITTED_TRADER), 0, "close realized all collectible carry");
        assertEq(claim, totalClaims, "the honest trader owns the only claim");
        assertGt(totalClaims, poolAssets, "trader claims exceed all pool assets");
        assertTrue(engine.degradedMode(), "engine is in degraded mode");

        // 2. The whole claim is frozen, not only the shortfall: settlement is all-or-nothing.
        vm.expectPartialRevert(ICfdEngineTypes.CfdEngine__InsufficientPoolLiquidity.selector);
        vm.prank(ADMITTED_TRADER);
        engine.settleTraderClaim(ADMITTED_TRADER);

        // 3. No new LP capital can be settled into the pool.
        bytes[] memory lpData = _mockPythUpdateData(closePrice);
        vm.expectPartialRevert(IHousePool.HousePool__DegradedMode.selector);
        router.settleLpEpoch(lpData);

        // 4. No new trading can start.
        address newcomer = address(0xD00D);
        _fundTrader(newcomer, 10_000e6);
        OrderV3Types.OrderRequest memory nextOpen = _request(CfdTypes.Side.SHORT, 10_000e18, 5_000e6, false);
        vm.expectPartialRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        vm.prank(newcomer);
        router.commitOrder(nextOpen);

        emit log_named_decimal_uint("double-pledged gap after winner close", gap, 6);
        emit log_named_decimal_uint("LP redemption funded before winner close", lpFunded, 6);
        emit log_named_decimal_uint("close price", closePrice, 8);
        emit log_named_uint("decline market steps", downSteps);
        emit log_named_uint("calendar days", (engine.lastMarkTime() - pathStart) / 1 days);
        emit log_named_decimal_uint("honest trader claim (frozen)", claim, 6);
        emit log_named_decimal_uint("pool assets", poolAssets, 6);
        emit log_named_decimal_uint("claims minus pool assets", totalClaims - poolAssets, 6);
    }

    /// @notice The LP firewall reserves C + L + B but not the winner's marked gain, so a matured redemption removes
    ///         cash the winner is owed. The removed cash adds to the double-pledged gap.
    function test_LpFirewallAlsoIgnoresMarkedGain() public {
        _setUpBeforeLp();
        uint256 branch = vm.snapshotState();

        _trade(WINNER, CfdTypes.Side.SHORT, SIZE, 0, true, HIGH_ENTRY);
        uint256 gapWithoutRedemption = _currentGap();
        assertTrue(vm.revertToState(branch));

        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory beforeLp =
            engineProtocolLens.getProtocolAccountingSnapshot();
        uint256 poolBefore = pool.totalAssets();
        router.settleLpEpoch(_mockPythUpdateData(HIGH_ENTRY));
        uint256 funded = poolBefore - pool.totalAssets();
        uint256 buffer = (beforeLp.maxLiabilityAssets * 25 + 9999) / 10_000;
        assertGe(pool.totalAssets(), beforeLp.maxLiabilityAssets + buffer, "firewall keeps only L plus 25 bps");

        _trade(WINNER, CfdTypes.Side.SHORT, SIZE, 0, true, HIGH_ENTRY);
        uint256 gapWithRedemption = _currentGap();
        assertGt(gapWithRedemption, gapWithoutRedemption + funded - 3_000e6, "redeemed cash adds to the gap");

        emit log_named_decimal_uint("gap without LP redemption", gapWithoutRedemption, 6);
        emit log_named_decimal_uint("LP redemption funded", funded, 6);
        emit log_named_decimal_uint("gap with LP redemption", gapWithRedemption, 6);
    }

    function _setUpGap() internal returns (uint256 gap, uint256 lpFunded) {
        _setUpBeforeLp();

        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory beforeLp =
            engineProtocolLens.getProtocolAccountingSnapshot();
        uint256 poolBefore = pool.totalAssets();
        router.settleLpEpoch(_mockPythUpdateData(HIGH_ENTRY));
        lpFunded = poolBefore - pool.totalAssets();
        uint256 buffer = (beforeLp.maxLiabilityAssets * 25 + 9999) / 10_000;
        assertGt(lpFunded, 190_000e6, "redemption takes the cash above the endpoint envelope");
        assertGe(pool.totalAssets(), beforeLp.maxLiabilityAssets + buffer, "firewall keeps only L plus buffer");

        _trade(WINNER, CfdTypes.Side.SHORT, SIZE, 0, true, HIGH_ENTRY);

        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory afterWinner =
            engineProtocolLens.getProtocolAccountingSnapshot();
        assertTrue(engine.degradedMode(), "paying the winner exposes the double-pledged cash");
        gap = afterWinner.maxLiabilityAssets - afterWinner.effectiveSolvencyAssets;
    }

    function _setUpBeforeLp() internal {
        assertEq(pool.totalAssets(), 1_000_000e6, "release-sized pool");
        _fundTrader(WINNER, 25_000e6);
        _fundTrader(ADMITTED_TRADER, 25_000e6);

        _walkPriceDown(START_PRICE, LOW_ENTRY);
        _trade(WINNER, CfdTypes.Side.SHORT, SIZE, 20_000e6, false, LOW_ENTRY);

        // An ordinary Junior LP exit request, matured before the winner closes.
        uint256 redeemShares = juniorVault.maxRequestRedeem(address(this));
        juniorVault.requestRedeem(redeemShares, address(this), address(this));

        _walkPriceUp(LOW_ENTRY, HIGH_ENTRY);
        int256 markedLpDelta = engine.terminalNavSnapshot().terminalLpPriceDeltaAssets;
        assertLt(markedLpDelta, -390_000e6, "the engine itself measures about 400,000 owed to the winner");

        _trade(ADMITTED_TRADER, CfdTypes.Side.LONG, SIZE, 20_000e6, false, HIGH_ENTRY);
        assertFalse(engine.degradedMode(), "admission accepts the opposite position against the winner's cash");
    }

    function _walkPriceDown(
        uint256 fromPrice,
        uint256 targetPrice
    ) internal {
        uint256 price = fromPrice;
        while (price > targetPrice) {
            uint256 next = _twoPercentDown(price);
            if (next < targetPrice) {
                next = targetPrice;
            }
            assertLe((price - next) * 10_000, price * 200, "down step is at most two percent");
            _warpOneDayAndPublish(next);
            price = next;
        }
    }

    function _walkPriceUp(
        uint256 fromPrice,
        uint256 targetPrice
    ) internal {
        uint256 price = fromPrice;
        while (price < targetPrice) {
            uint256 next = ((price * 102) / 100) & ~uint256(1);
            if (next > targetPrice) {
                next = targetPrice;
            }
            assertLe((next - price) * 10_000, price * 201, "up step is at most two percent");
            _warpOneDayAndPublish(next);
            price = next;
        }
    }

    function _twoPercentDown(
        uint256 price
    ) internal pure returns (uint256 next) {
        next = (price * 98 + 99) / 100;
        if (next & 1 == 1) {
            ++next;
        }
    }

    function _warpOneDayAndPublish(
        uint256 price
    ) internal {
        vm.warp(vm.getBlockTimestamp() + 1 days);
        while (engine.isOracleFrozen()) {
            vm.warp(vm.getBlockTimestamp() + 1 days);
        }
        _refreshMarkThroughRouter(price);
    }

    function _currentGap() internal view returns (uint256 gap) {
        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory accounting =
            engineProtocolLens.getProtocolAccountingSnapshot();
        gap = accounting.maxLiabilityAssets > accounting.effectiveSolvencyAssets
            ? accounting.maxLiabilityAssets - accounting.effectiveSolvencyAssets
            : 0;
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint64 orderId = _commit(account, side, size, margin, isClose);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _mockPythUpdateData(price));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, isClose);
        orderId = router.nextCommitId();
        vm.prank(account);
        router.commitOrder(request);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
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
    }

    function _refreshMarkThroughRouter(
        uint256 price
    ) internal {
        uint256 nowTime = vm.getBlockTimestamp();
        if (nowTime <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
            nowTime = vm.getBlockTimestamp();
        }
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, nowTime, previous));
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        router.updateMarkPrice(updateData);
        assertEq(engine.lastMarkPrice(), price, "public Pyth refresh installs mark");
        assertEq(engine.lastMarkTime(), nowTime, "publish time is read back from engine");
    }

}
```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/accounting/cash-priority/MarkedWinnerProtocolInsolvency.t.sol -vv
```

Expected output:

```text
Ran 2 tests for test/perps/accounting/cash-priority/MarkedWinnerProtocolInsolvency.t.sol:MarkedWinnerProtocolInsolvencyPoC
[PASS] test_LpFirewallAlsoIgnoresMarkedGain() (gas: 13314823)
Logs:
  gap without LP redemption: 194308.674619
  LP redemption funded: 195871.483555
  gap with LP redemption: 389404.313272

[PASS] test_PoC_ProtocolInsolvencyAfterAllPositionsClose() (gas: 22447959)
Logs:
  double-pledged gap after winner close: 389404.313272
  LP redemption funded before winner close: 195871.483555
  close price: 0.63139876
  decline market steps: 37
  calendar days: 130
  honest trader claim (frozen): 417673.563386
  pool assets: 415782.067971
  claims minus pool assets: 1891.495415

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

