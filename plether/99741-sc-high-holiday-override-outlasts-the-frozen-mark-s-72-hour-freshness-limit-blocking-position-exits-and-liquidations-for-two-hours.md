# \#99741 [SC-High] Holiday override outlasts the frozen mark's 72-hour freshness limit, blocking position exits and liquidations for two hours

**Submitted on Nov 15th 2026 at 03:32:38 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99741
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief / Introduction

I found a liveness failure when a whole-UTC-day holiday override overlaps the expiration of the Engine's frozen-market mark. The default frozen-mark freshness limit is exactly **three days**. The holiday calendar can keep the Oracle in frozen mode for longer than that, but the frozen Oracle has only one permitted price source: the Engine's previously accepted mark. It does not read newly supplied signed Pyth updates while the override remains active.

If the last accepted mark comes from the Friday FX close and an extended Monday holiday is configured, that mark becomes more than 72 hours old before Monday's UTC-day override finishes. At that point, every path requiring the frozen basket rejects the cached mark with `PletherOracle__StalePrice`. Fresh price updates submitted to the Router do not recover the system because the Oracle returns from `_frozenBasket()` before examining those updates. The cached mark's timestamp cannot advance through a successful frozen refresh either, because it is the same old mark that the Oracle continues to return.

I reproduced a **7,200-second (two-hour) window** in a Christmas-Monday example. I opened a **600,000 USDG SHORT**, configured the December 25, 2028 holiday through the normal timelocked calendar path, and retained a mark published at **Friday, December 22, 2028, 21:59:59 UTC**. From **Monday, December 25, 22:00 UTC until Tuesday, December 26, 00:00 UTC**, the configured frozen period continues after the cached mark has expired.

During this interval I supplied fresh Pyth test updates on every attempt. Mark refresh, an ordinary full close, liquidation, a withdrawal of free margin from an open-position account, and direct and Router-mediated LP epoch settlement all fail. The failing operation is the age check on the **old stored mark**, not the age of the newly submitted update.

I also checked that this is not a rejected liquidation of a healthy position. At a fresh basket price of **0.94000000** against the SHORT entered at **1.00000000**, the Engine's liquidation lens reports the account as liquidatable. Liquidation still reverts during the stale frozen window. Immediately after the override expires, the same fresh price is accepted and permissionless liquidation succeeds, reducing the position size from **600,000 USDG to zero**.

All four tests passed against the pinned competition commit. I ran twelve separate ten-minute checkpoints from **22:01 through 23:51 UTC** and got the same failures at every checkpoint. No audited production contracts were modified.

## Vulnerability Details

### 1. The holiday override forces the Oracle into frozen mode for the full UTC day

`MarketCalendarLib.marketStatus()` forces both the FAD regime and the frozen Oracle regime when the current UTC day is marked as a holiday:

```solidity
if (todayOverride) {
    return (true, true);
}
```

The date override has no intraday ending time. It remains active until the UTC calendar advances to the next day. For a Monday holiday, the setting is therefore still active at 22:00 and 23:59 UTC even if the underlying feed becomes available before midnight.

The calendar update is a normal supported governance action, not a compromised-admin action. In my test I apply it using the repository's `_setCalendarConfig()` helper, which proposes and finalizes the configuration through the Engine's ordinary 72-hour timelock. I do not bypass governance authorization or directly alter storage.

### 2. Frozen-mode freshness is limited to exactly 72 hours

`CfdEngine.sol` ships with:

```solidity
uint256 public fadMaxStaleness = 3 days;
```

`OracleFreshnessPolicyLib.getPolicy()` applies this value to frozen close execution, mark refresh, liquidation and pool reconciliation. The limit is **259,200 seconds**. `OracleFreshnessPolicyLib.isStale()` permits a mark exactly at the boundary but rejects it when:

```solidity
currentTimestamp - oraclePublishTime > maxStaleness
```

This is a strict boundary: the last Friday mark is accepted at Monday 21:59:59 UTC, but it is rejected one second later.

### 3. `_resolveCurrentBasket()` refuses fresh signed data during the override

`PletherOracle._resolveCurrentBasket()` contains the decisive branch:

```solidity
if (policy.oracleFrozen) {
    return _frozenBasket(mode, policy.maxStaleness);
}
```

It takes this branch before parsing `pythUpdateData`, so supplying valid, freshly published price updates does not change the result.

`_frozenBasket()` uses only the old Engine mark and timestamp:

```solidity
basket.price = engine.lastMarkPrice();
basket.publishTime = engine.lastMarkTime();
if (basket.price == 0 || basket.publishTime == 0) {
    revert PletherOracle__NoAcceptedBasket();
}
if (OracleFreshnessPolicyLib.isStale(basket.publishTime, maxStaleness, block.timestamp)) {
    revert PletherOracle__StalePrice(
        mode, bytes32(0), basket.publishTime, maxStaleness, block.timestamp
    );
}
```

A keeper cannot refresh away the condition by calling `updateMarkPrice()` again. The same stale cached timestamp is re-read and rejected. This makes the failure self-sustaining for as long as the holiday override remains active.

### 4. The frozen staleness check is shared by the affected operations

In my reproduction, the same expiration disables the following real production entrypoints:

| Operation | Result while the override is active and Friday's mark is stale |
|---|---|
| `router.updateMarkPrice(fresh)` | `PletherOracle__StalePrice` (`0xf4a25e0f`) |
| `router.executeOrder(closeId, fresh)` | `PletherOracle__StalePrice` (`0xf4a25e0f`); the SHORT stays open |
| `router.executeLiquidation(ALICE, fresh)` | `PletherOracle__StalePrice` (`0xf4a25e0f`) even though the SHORT is liquidatable at the fresh price |
| `clearinghouse.withdraw(ALICE, 1e6)` | `CfdEngine__MarkPriceStale` (`0xe4c92389`) while the account still has an open position |
| `pool.settleLpEpoch(0, 0)` | `HousePool__MarkPriceStale` |
| `router.settleLpEpoch(fresh)` | `PletherOracle__StalePrice` (`0xf4a25e0f`) |

These failures block position exits, liquidation and settlement of pending LP epochs requiring a valid current mark. They do not imply that a funded LP claim already available for collection is blocked, or that a completely flat trader is unable to withdraw. The affected operations are the price-dependent ones identified above.

### 5. The calendar/freshness boundary is incompatible with the extended holiday closure

Here is the exact timeline used in the PoC:

| UTC time | Cached mark age | Behavior |
|---|---:|---|
| Fri 22 Dec 2028, 21:59:59 | 0 seconds | Final accepted pre-closure mark at `1.00000000` |
| Mon 25 Dec 2028, 20:59:59 | 255,600 seconds | Frozen mark still usable; full close succeeds in the control |
| Mon 25 Dec 2028, 21:59:59 | 259,200 seconds | Last permitted instant under the three-day maximum |
| Mon 25 Dec 2028, 22:00:00 | 259,201 seconds | Mark becomes stale, but Monday's holiday override remains active |
| Mon 25 Dec 2028, 22:01:00 | 259,261 seconds | Fresh signed test updates supplied; operations still revert |
| Mon 25 Dec 2028, 22:01–23:51 | Over the maximum | All twelve ten-minute failure checkpoints reproduce |
| Tue 26 Dec 2028, 00:00:02 | Live mode | Fresh mark accepted; eligible SHORT liquidation succeeds |

The incompatibility comes from two individually allowed settings: a **three-day frozen freshness limit** and a **full-UTC-day holiday freeze**. There is no check linking them, so a valid calendar override can extend beyond the lifetime of the only mark the Oracle permits during that override.

The PoC does not rely on a forged Pyth signature or a malicious administrator. It exercises the normal holiday configuration, and it passes current timestamps and updated basket prices through the test suite's permitted Pyth mock. In production, a comparable freeze requires an extended holiday/provider closure that leaves the final accepted mark old enough while fresh signed data is available before the override ends. My test proves the contract response to that state; the exact publication hours of Pyth feeds for a future December 2028 holiday are an external operational precondition, not something the test can establish.

### 6. A genuinely liquidatable position is blocked, not merely a liquidation call that would fail anyway

I keep Alice's SHORT open at the release margin configuration and pass a fresh basket price of **0.94000000**. Plether's SHORT position is exposed to a decline in the FX basket; the corresponding notional movement is approximately **36,000 USDG against the trader** on a 600,000 USDG position, before fees and other accounting adjustments.

Before each simulated liquidation, the test asserts:

```solidity
assertTrue(
    engineLens.isLiquidatableAt(ALICE, ADVERSE_PRICE, pool.totalAssets()),
    "short is liquidatable at fresh Pyth price"
);
```

The lens returns `true`, yet `router.executeLiquidation()` reverts on the stale frozen basket. The position remains at its original size across all twelve checkpoints. When the calendar changes back to live mode at Tuesday midnight, `router.updateMarkPrice()` accepts the adverse price and the Router's actual `executeLiquidation()` removes the position. That establishes an execution-time liquidation liveness failure under the reproduced conditions, not just an oracle-view discrepancy.

## Impact Details

The demonstrated impact is **temporary freezing of funds for at least one hour**. An account with an open position cannot reduce its exposure through the Router or withdraw free settlement, even though it can present fresh signed prices. LP withdrawals awaiting epoch reconciliation cannot be funded while settlement requires the expired mark. At the same time, liquidations that should be eligible at the current price are rejected, leaving risk exposure open until the calendar exits frozen mode.

The freeze lasts **two hours** in the winter UTC-day example: from Monday 22:00 UTC until Tuesday 00:00 UTC. It does not require a dishonest keeper, a token shortfall or a separate attack transaction. Once the holiday override and last-accepted-mark conditions occur, the stale checks fail deterministically until the mode changes or the configured limit is increased.

The existing governance freshness configuration can be changed through a 72-hour timelock, but the calendar setup does not require a corresponding increase to `fadMaxStaleness`, and the stale period cannot be repaired during the two-hour gap by a fresh Pyth update or an ordinary price-refresh call.

**The impact I am claiming is the verified temporary inability to access or settle funds through price-dependent paths, with eligible liquidation also blocked.** I am not claiming a demonstrated theft, protocol insolvency, or realized LP bad debt. The PoC's post-reopening liquidation confirms the exposure that remained unresolved during the frozen interval.

## Why this is not simply the disclosed holiday-calendar limitation

`SECURITY.md` notes that unscheduled FX provider closures and holiday hours require governance-configured UTC-day overrides. It also documents the general rule that frozen markets use a relaxed but finite stale-mark window. I am not reporting either fact as a separate issue.

The failure here is their **unvalidated combination**: using the documented UTC-day override can keep the protocol in frozen mode after the default frozen mark has expired, at which point the normal fallback that is intended to keep closes and liquidations available rejects every price it can use. Supplying a fresh price does not help because frozen mode ignores it, and replaying the cached mark cannot advance its publication timestamp.

This is materially different from an ordinary stale oracle when no current price exists: the final PoC supplies fresh signed test prices during the entire lockout and shows that they are refused solely because the calendar still says frozen. The earlier known stale-frozen test establishes the safety behavior when a mark is too old; my finding is that a supported holiday configuration can drive the system into that safety failure **for two hours while fresh data is available**.

## Recommendation

I recommend enforcing compatibility between the holiday calendar and frozen-mark freshness. Any of these approaches can resolve the demonstrated gap:

1. Make `fadMaxStaleness` long enough for the longest permitted frozen interval, including a multi-day closure and the entire UTC-day holiday overhang. Validate this relationship whenever calendar and freshness settings are finalized.
2. Allow a narrowly defined recovery from frozen mode using a fully verified set of fresh signed Pyth updates, with the same integrity, freshness, confidence and divergence checks required on the live path. Do not silently accept unverified data during real feed outages.
3. Make holiday overrides end at the actual provider-market reopen boundary rather than requiring the Oracle to remain frozen until UTC midnight when the feed has resumed.

The essential requirement is that a legitimate market-calendar override must not produce an interval in which **the cached mark is forbidden for being too old and any fresh mark is forbidden because the calendar is still frozen**.

## References

- [PletherOracle.sol — `_frozenBasket()` and `_resolveCurrentBasket()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L758)
- [OracleFreshnessPolicyLib.sol — frozen-mode limits and strict stale check](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OracleFreshnessPolicyLib.sol#L52-L96)
- [CfdEngine.sol — default `fadMaxStaleness = 3 days`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L136-L142)
- [MarketCalendarLib.sol — UTC-day override forces frozen mode](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L30-L60)
- [CfdEngine.sol — original mark timestamp is retained](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1080-L1093)
- [SECURITY.md — documented calendar overrides](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1332-L1343)
- [BasePerpTest.sol — historical Pyth helper and timelocked calendar config](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/BasePerpTest.sol#L297-L328)
- [Plether Immunefi competition — assets and impacts](https://immunefi.com/audit-competition/plether/scope/)


## Proof of Concept

## Proof of Concept

I reproduced the issue against commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` in the repository's `packages/perps` Foundry test suite. The PoC uses the real audited contracts. The only simulated external components are USDG and Pyth, as permitted by the competition. A normal calendar configuration is finalized through the provided timelock helper. No source file inside `packages/perps/src` is modified.

The setup uses a **2,000,000 USDG Junior pool**, opens a **600,000 USDG SHORT** with **25,200 USDG opening margin**, keeps the released frozen freshness limit of **259,200 seconds**, and sets a Monday holiday override. The historical order helper advances the block and supplies an acceptable post-commit Pyth observation. The last live mark is accepted on Friday at 21:59:59 UTC. I then run the control and failure scenarios on independent test states.

## Reproduction steps

1. Check out the exact audited commit and make sure Foundry and the repository dependencies are available.
2. Save the first Solidity block below as `packages/perps/test/perps/insight/Cal_HolidayStaleStrict_PoC.t.sol`.
3. Save the second block as `packages/perps/test/perps/insight/Cal_HolidayStaleFrozenFreeze_PoC.t.sol`.
4. To avoid the large test-suite compiler termination encountered in Codespaces, copy both files into an isolated `test/perps/holiday_only/` directory as shown below.
5. Run both test contracts and compare their output to the verified logs.

```bash
cd /workspaces/audit-comp-plether
export PATH="$HOME/.foundry/bin:$PATH"

test "$(git rev-parse HEAD)" = "a82d4fefb02768e5b3bddcc97dee371eee24fea8"

mkdir -p packages/perps/test/perps/holiday_only holiday_stale_evidence
cp packages/perps/test/perps/insight/Cal_HolidayStaleStrict_PoC.t.sol \
   packages/perps/test/perps/holiday_only/Cal_HolidayStaleStrict_PoC.t.sol
cp packages/perps/test/perps/insight/Cal_HolidayStaleFrozenFreeze_PoC.t.sol \
   packages/perps/test/perps/holiday_only/Cal_HolidayStaleFrozenFreeze_PoC.t.sol

set +e
FOUNDRY_TEST="test/perps/holiday_only" forge test \
  --root packages/perps --offline -j 1 --skip script \
  --match-contract '^Cal_HolidayStaleStrict_PoC$' -vv \
  2>&1 | tee holiday_stale_evidence/STRICT_LIQUIDATION_CORRECTED.log
STRICT=${PIPESTATUS[0]}

FOUNDRY_TEST="test/perps/holiday_only" forge test \
  --root packages/perps --offline -j 1 --skip script \
  --match-contract '^Cal_HolidayStaleFrozenFreeze_PoC$' -vv \
  2>&1 | tee holiday_stale_evidence/FREEZE_LIQUIDATION_CORRECTED.log
FREEZE=${PIPESTATUS[0]}

printf 'STRICT=%s FREEZE=%s\n' "$STRICT" "$FREEZE"
grep -E '\[PASS\]|\[FAIL\]|Suite result|mark age|freeze seconds|10-minute checks|POST REOPEN' \
 holiday_stale_evidence/STRICT_LIQUIDATION_CORRECTED.log \
 holiday_stale_evidence/FREEZE_LIQUIDATION_CORRECTED.log

git diff HEAD -- packages/perps/src
```

## Full Solidity PoC 1 — strict control and exact stale-price selectors

**File:** `packages/perps/test/perps/insight/Cal_HolidayStaleStrict_PoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

abstract contract CalHolidayBase is BasePerpTest {
    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xBEE9E2);

    uint256 internal constant PRICE = 100_000_000;
    // Plether's SHORT loses when the raw FX-basket price falls.
    uint256 internal constant ADVERSE_PRICE = 94_000_000;

    uint256 internal constant SIZE = 600_000e18;
    uint256 internal constant MARGIN = 25_200e6;

    // Friday, 22 December 2028 at 21:59:59 UTC.
    uint256 internal constant FRIDAY_MARK = 1_861_135_199;
    // Monday, 25 December 2028 at 00:00:00 UTC.
    uint256 internal constant HOLIDAY = 1_861_315_200;
    // Monday, 25 December 2028 at 20:59:59 UTC.
    uint256 internal constant MONDAY_BEFORE = 1_861_390_799;
    // Monday, 25 December 2028 at 22:00:00 UTC.
    uint256 internal constant FIRST_STALE = 1_861_394_400;
    // Tuesday, 26 December 2028 at 00:00:00 UTC.
    uint256 internal constant TUESDAY_REOPEN = 1_861_401_600;

    uint256 internal nonce = 1;

    function _riskParams()
        internal
        pure
        override
        returns (CfdTypes.RiskParams memory)
    {
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

    function _initialJuniorDeposit()
        internal
        pure
        override
        returns (uint256)
    {
        return 2_000_000e6;
    }

    function setUp() public virtual override {
        super.setUp();

        assertEq(engine.fadMaxStaleness(), 3 days, "released frozen freshness limit");

        // Configure a holiday through the same timelocked Engine path.
        ICfdEngineAdminHost.EngineCalendarConfig memory calendar =
            _engineCalendarConfig();
        calendar.fadDayTimestamps = new uint256[](1);
        calendar.fadDayTimestamps[0] = HOLIDAY;
        _setCalendarConfig(calendar);

        // Open the position while the Friday market is still live.
        _at(FRIDAY_MARK - 2 hours);
        router.updateMarkPrice(_signed(PRICE, block.timestamp, block.timestamp - 1));

        _fundTrader(ALICE, 1_200_000e6);

        uint64 id = _commit(false);
        bytes[] memory openingTicks = _mockPythUpdateData(PRICE);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, openingTicks);

        console2.log("OPENING LIFECYCLE STATUS", uint256(uint8(r.status)));
        console2.log("OPENING PENDING REASON", uint256(uint8(r.pendingReason)));
        console2.log("OPENING TERMINAL REASON", uint256(uint8(r.terminalReason)));

        assertEq(
            uint8(r.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "short opened"
        );
        assertEq(_size(), SIZE, "600k short opened");

        // The last accepted mark before the extended closure.
        _at(FRIDAY_MARK);
        router.updateMarkPrice(_signed(PRICE, FRIDAY_MARK, FRIDAY_MARK - 1));
        assertEq(engine.lastMarkTime(), FRIDAY_MARK, "Friday mark cached");
        assertFalse(engine.isOracleFrozen(), "Friday market still live");
    }

    function _at(uint256 timestamp) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _size() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(ALICE);
    }

    function _signed(
        uint256 price,
        uint256 publish,
        uint256 previous
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);

        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(int64(uint64(price)), 0, -8, publish, previous)
            );
        }

        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _request(bool close)
        internal
        returns (OrderV3Types.OrderRequest memory req)
    {
        uint32 window = uint32(router.maxExecutionWindowSeconds());

        OrderV3Types.ExecutionBounds memory b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
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
        });

        req = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(nonce++),
            side: CfdTypes.Side.SHORT,
            sizeDelta: SIZE,
            marginDelta: close ? 0 : MARGIN,
            targetPrice: close ? 1 : PRICE,
            isClose: close,
            bounds: b
        });
    }

    function _commit(bool close) internal returns (uint64 id) {
        // Construct the request before vm.prank: _request calls the Router.
        OrderV3Types.OrderRequest memory request = _request(close);
        vm.prank(ALICE);
        id = router.commitOrder(request);
    }

    function _frozenCloseBeforeExpiry() internal {
        _at(MONDAY_BEFORE);
        assertTrue(engine.isOracleFrozen(), "holiday frozen");
        assertLt(
            block.timestamp - engine.lastMarkTime(),
            engine.fadMaxStaleness(),
            "mark still within 72-hour limit"
        );

        router.updateMarkPrice(_signed(PRICE, block.timestamp, block.timestamp - 1));
        assertEq(
            engine.lastMarkTime(),
            FRIDAY_MARK,
            "frozen refresh does not renew timestamp"
        );

        uint64 id = _commit(true);
        uint256 committedAt = block.timestamp;
        _at(committedAt + 1);

        OrderV3Types.ExecutionResult memory r = router.executeOrder(
            id,
            _signed(PRICE, block.timestamp, block.timestamp - 1)
        );

        assertEq(
            uint8(r.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "frozen full close executes"
        );
        assertEq(_size(), 0, "position closed before mark expiry");
    }

    function _expectFrozenFailures(uint256 current) internal {
        _at(current);
        assertTrue(engine.isOracleFrozen(), "holiday override remains active");
        assertGt(
            current - engine.lastMarkTime(),
            engine.fadMaxStaleness(),
            "cached mark has exceeded staleness limit"
        );

        bytes[] memory fresh = _signed(ADVERSE_PRICE, current, current - 1);

        // Fresh signed data cannot update the mark in the frozen branch.
        vm.expectPartialRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        router.updateMarkPrice(fresh);

        // The SHORT is underwater at this simulated fresh price.
        assertTrue(
            engineLens.isLiquidatableAt(ALICE, ADVERSE_PRICE, pool.totalAssets()),
            "short is liquidatable at fresh Pyth price"
        );

        // An otherwise eligible liquidation reverts before it can execute.
        vm.prank(KEEPER);
        vm.expectPartialRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        router.executeLiquidation(ALICE, fresh);

        // An open-position account cannot withdraw even free margin.
        vm.prank(ALICE);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__MarkPriceStale.selector);
        clearinghouse.withdraw(ALICE, 1e6);

        // Both direct and Router-mediated LP settlement fail.
        vm.expectRevert(IHousePool.HousePool__MarkPriceStale.selector);
        pool.settleLpEpoch(0, 0);

        vm.expectPartialRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        router.settleLpEpoch(fresh);

        assertEq(_size(), SIZE, "SHORT remains open");
        assertEq(engine.lastMarkTime(), FRIDAY_MARK, "mark still Friday timestamp");
    }
}

contract Cal_HolidayStaleStrict_PoC is CalHolidayBase {
    function test_Strict_ControlBeforeAgeOut() public {
        _frozenCloseBeforeExpiry();
    }

    function test_Strict_StaleSelectors() public {
        _at(FIRST_STALE + 59);
        uint64 id = _commit(true);
        uint256 commitTime = block.timestamp;
        _at(commitTime + 1);

        bytes[] memory fresh = _signed(
            ADVERSE_PRICE,
            block.timestamp,
            block.timestamp - 1
        );

        vm.expectPartialRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        router.executeOrder(id, fresh);
        _expectFrozenFailures(block.timestamp);

        console2.log("executeOrder reverted, selector:");
        console2.logBytes4(IPletherOracle.PletherOracle__StalePrice.selector);
        console2.log("withdraw reverted, selector:");
        console2.logBytes4(ICfdEngineTypes.CfdEngine__MarkPriceStale.selector);
        console2.log("mark age (s)", block.timestamp - engine.lastMarkTime());
        console2.log("fadMaxStaleness", engine.fadMaxStaleness());
    }
}
```

## Full Solidity PoC 2 — twelve-interval lockout and recovery liquidation

**File:** `packages/perps/test/perps/insight/Cal_HolidayStaleFrozenFreeze_PoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CalHolidayBase} from "./Cal_HolidayStaleStrict_PoC.t.sol";
import {console2} from "forge-std/console2.sol";

contract Cal_HolidayStaleFrozenFreeze_PoC is CalHolidayBase {
    function test_Control_FrozenCloseWorksBeforeMarkAgesOut() public {
        _frozenCloseBeforeExpiry();
    }

    function test_PoC_TotalFreezeAfterPythReopens() public {
        assertEq(
            FIRST_STALE - FRIDAY_MARK,
            259_201,
            "stale at first second past limit"
        );

        uint256 count;
        // Monday 22:01, 22:11, ... 23:51 UTC.
        for (uint256 i; i < 12; ++i) {
            _expectFrozenFailures(FIRST_STALE + 60 + i * 10 minutes);
            ++count;
        }

        assertEq(count, 12, "all 12 checks performed");
        assertEq(
            TUESDAY_REOPEN - FIRST_STALE,
            7200,
            "two-hour calendar/staleness gap"
        );

        console2.log("mark age at Pyth reopen (s)", FIRST_STALE - FRIDAY_MARK);
        console2.log("10-minute checks fully frozen while Pyth publishes", count);
        console2.log("freeze seconds", TUESDAY_REOPEN - FIRST_STALE);

        _at(TUESDAY_REOPEN + 2);
        assertFalse(engine.isOracleFrozen(), "holiday override ended");

        // The same fresh mark is now accepted in live mode.
        router.updateMarkPrice(
            _signed(ADVERSE_PRICE, block.timestamp, block.timestamp - 1)
        );
        assertEq(engine.lastMarkTime(), block.timestamp, "fresh live mark accepted");
        assertEq(_size(), SIZE, "SHORT survived two-hour lockout");

        assertTrue(
            engineLens.isLiquidatableAt(ALICE, ADVERSE_PRICE, pool.totalAssets()),
            "short liquidatable at Tuesday reopening"
        );
        console2.log("POST REOPEN: short liquidatable", true);
        console2.log("POST REOPEN: position size before liquidation", _size());

        bytes[] memory liquidationTicks = _signed(
            ADVERSE_PRICE,
            block.timestamp,
            block.timestamp - 1
        );
        vm.prank(KEEPER);
        router.executeLiquidation(ALICE, liquidationTicks);

        assertEq(_size(), 0, "actual liquidation executes after reopening");
        console2.log("POST REOPEN: remaining position size", _size());
    }
}
```

## Verified test output

This is the actual Foundry output from the successful run on the audited commit, after correcting the SHORT's adverse price and adding the post-reopening liquidation checks.

### Strict test suite

```text
[PASS] test_Strict_ControlBeforeAgeOut() (gas: 1877214)
[PASS] test_Strict_StaleSelectors() (gas: 2046960)
  mark age (s) 259261
  fadMaxStaleness 259200
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 53.99ms (12.93ms CPU time)
```

The strict test also checked these revert selectors:

```text
executeOrder reverted, selector: 0xf4a25e0f
withdraw reverted, selector: 0xe4c92389
```

### Full two-hour freeze and recovery liquidation suite

```text
Ran 2 tests for test/perps/holiday_only/Cal_HolidayStaleFrozenFreeze_PoC.t.sol:Cal_HolidayStaleFrozenFreeze_PoC
[PASS] test_Control_FrozenCloseWorksBeforeMarkAgesOut() (gas: 1877100)
Logs:
  OPENING LIFECYCLE STATUS 2
  OPENING PENDING REASON 0
  OPENING TERMINAL REASON 1

[PASS] test_PoC_TotalFreezeAfterPythReopens() (gas: 11117039)
Logs:
  OPENING LIFECYCLE STATUS 2
  OPENING PENDING REASON 0
  OPENING TERMINAL REASON 1
  mark age at Pyth reopen (s) 259201
  10-minute checks fully frozen while Pyth publishes 12
  freeze seconds 7200
  POST REOPEN: short liquidatable true
  POST REOPEN: position size before liquidation 600000000000000000000000
  POST REOPEN: remaining position size 0

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 67.99ms (39.64ms CPU time)
```

### Final verification

```text
STRICT: 0
FREEZE: 0
PRODUCTION CONTRACT DIFF: (empty)
SUCCESS: ALL FOUR STRENGTHENED POC TESTS PASSED
```

Both controls prove normal liveness before the cached mark crosses the permitted age. The strict test confirms the exact errors after the threshold. The main PoC checks the frozen rejection twelve times, confirms that the SHORT is already liquidatable at the freshly supplied adverse price, then executes its liquidation immediately after the UTC-day override expires.

**The reproduced failure depends on the documented extended-closure state in which Friday's accepted mark remains the last mark into late Monday. The PoC demonstrates contract behavior under that state; it does not independently establish Pyth's actual December 2028 feed-publishing schedule.**

