# \#99934 [SC-Critical] A trader-chosen execution deadline turns committed orders into a free option paid by LPs: the fix for pre-audit SC-M4 is incomplete

**Submitted on Nov 15th 2026 at 19:29:01 UTC by @xylz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99934
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

A committed order is meant to be binding, and its fill price is fixed at commit: the first Pyth print stamped after the commit second. But the trader chooses the order's deadline, down to one second, and an order that passes its deadline unfilled fails. Its owner clears it and gets the reserved bounty back.

So a trader decides after the fact whether its order executes. It commits a LONG on one account and a SHORT on another at a scheduled data release, each with a deadline of a few seconds. Once the next prints are public it executes the leg already in profit at the old print and lets the other expire. The LP pool pays the profit.

- Impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
- Severity: Critical
- Attacker needs: two funded trading accounts and its own `executeOrder` calls; no role and no open position.
- Measured with the v2.0.0 release risk and Router parameters and a Junior pool of 1,000,000 USDG, 300,000 USDG per leg, on fifteen scheduled releases:
  - Replay limits: four of six basket feeds are inferred, and I chose the threshold and timings on these releases.
  - Commit in the release second, deadlines of 2 and 3 seconds, no keeper inside them: three legs kept, all three in profit. The trader gains 1,116.30 USDG and the pool loses 1,476.31 USDG (test 4).
  - With a keeper in the replay, the trader gains 824.86 USDG and the pool loses 1,274.75 USDG (test 7). The figures weight keeper arrivals by the Arbitrum Sepolia test deployment's first 204 orders: 196 executed 4 seconds or more after commit, 8 after 3 seconds, none within 2.
  - A keeper that arrives within 2 seconds binds both legs, and the trader loses 21,210.89 USDG (test 7). Since 2026-11-07 that deployment's keeper does so for 82% of orders.

## Affected assets

- `OrderRouter.sol`
- `router/OrderLiquidationBatchLogic.sol`
- `OrderLifecycleBook.sol`
- `PletherOracle.sol`

All at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.

## Preconditions

- No keeper fills the leg the trader drops before that leg's deadline. The trader sets that margin: an order with a 1-second deadline can be filled only in its fill second (test 3).
- From the Router's event logs on the Arbitrum Sepolia v2.0.0 deployment I read the commit-to-execution time of every order executed to 2026-11-15T17:55Z (5,181). Until 2026-11-07T15:09Z none of 204 orders was executed within 2 seconds and 8 within 3 (median 6). Since 2026-11-07T17:22Z, 82% of 4,977 were executed within 2 seconds and 99% within 3. The terms say "Perps contracts are not upgradeable"; keepers are a service "operated by Plether".
- Orders still went unfilled after the speed-up. On 2026-11-12 three consecutive orders with 60-second deadlines, committed from 10:20:33Z to 10:20:52Z, expired unfilled and were cleared 15 to 32 seconds after their deadlines. The logs do not show why.
- Before it commits, the trader can read the keeper's recent speed on chain: each order's commit and execution blocks give its commit-to-fill time. When a keeper fills both legs in time, the trader loses about 1,414 USDG per release at 300,000 USDG per leg (test 7, keeper at 2 seconds).
- The terms say "Keepers are not trusted: anyone can execute orders, liquidations, price updates and protections, so harm caused by any caller acting as a keeper is in scope". They add that researchers "should consider any contract-permitted behavior, including deviations from the reference implementation". Here the trader executes its own order as a keeper, and a keeper that misses a 2-second deadline is contract-permitted behaviour.
- One account alone holds the option on one side: in test 1 the whole gain is on the LONG account. The second account adds the other side, because an account holds one side at a time (`router/OrderLiquidationBatchLogic.sol:426-427`).

## Root cause

1. The fill print is fixed by the commit second. The oracle takes the first print stamped after commit (`PletherOracle.sol:610-612`), and execution uses that same print until the order's deadline.
2. The deadline is the trader's. `executionWindowSeconds` may be any value from 1 up to the Router maximum (`router/OrderLiquidationBatchLogic.sol:287`), and the deadline is commit time plus that value (`OrderLifecycleBook.sol:287`).
3. Expiry is free. An expired order fails without a fill, and when its owner clears it the reserved bounty returns to the owner (`OrderRouter.sol:336-337`). If a keeper clears it, the owner loses at most the 0.2 USDG bounty (test 2).

The project states the property this breaks in two places:

- "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" (SECURITY.md:658-659);
- "Users cannot cancel a binding ordinary order." (ACCOUNTING_SPEC.md:1266).

A deadline shorter than a keeper's arrival is a cancellation path that only the trader holds.

The whitepaper says "finite settlement windows limit the option value given to the trader" (WHITEPAPER.md:1005-1006). In Plether's documents the settlement window is the oracle's post-commit price window, `orderSettlementWindow`, separate from the execution duration (CANONICAL_ENTRYPOINTS.md:71-72). It bounds which print fills an order, not who decides whether the order executes.

### Why this is a fix-incomplete report on SC-M4

The pre-audit reported this outcome as SC-M4 (audits/plether-pre-audit-report.pdf, pages 29 and 30). Titled "Risk Free Option in 60Sec Oracle Window", it has two opposite orders, the losing one cancelled, to "extract risk-free profits from liquidity providers". Its response is "Fixed in Plether-Fi/plether-core#24". The terms admit a report on a fixed finding "only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug".

SC-M4's option came from a keeper choosing a tick inside the 60-second window and from a `SOLVENCY_EXCEEDED` refund. At this commit the fill is pinned to the first print after commit (`PletherOracle.sol:610-612`), so no caller picks the tick. The order's lifetime is now the trader's (`router/OrderLiquidationBatchLogic.sol:287`): when SC-M4 was fixed, orders expired by the Router's own `maxOrderAge`. If SC-M4 is read as a different root cause, this is a new bug: no listed text treats a deadline chosen by the trader as a way to drop a committed order.

### Why this is not another listed item

- Cecuro HIGH-1 (audits/plether-ai-audit-report-2026-10-26.pdf, page 219) is about commit time: the block timestamp can trail a print the trader has already seen. The terms ask a new report on that root cause for "a proof of concept showing a repeatable, net-profitable extraction from LPs beyond this limitation". This report uses no timestamp lag: the fill print is unknown at commit, and the trader decides after it.
- The October 15 scan lists "Order execution accepts oracle updates older than the engine's cached mark" (audits/scan-plether-fi-plether-core-d4541b49.md:1196). Test 6 shows that behaviour, so a mark refresh is no defence. The attack does not depend on it: with a change that refuses such fills, tests 1 to 5 and 7 pass with identical figures. In their 60-second control, with the same commits and prints, the trader loses, so the gain comes from choosing which order executes.
- Pre-audit SC-I2, "Missing Order Cancellation Mechanism" (audits/plether-pre-audit-report.pdf, page 34), reports that the Router has no cancellation function. At this commit a deadline the trader chooses works as one (test 1).
- One limitation reads "execution remains dependent on fresh keeper infrastructure and oracle publication outside frozen windows" (SECURITY.md:1341). It says an order needs a keeper to execute, not that a trader may choose whether its own order executes.
- My report #97733 uses a different lever: a withdrawal after the fill print makes one order's VPI reserve funding fail. With the change that report recommends applied to the source, all seven tests below pass and no logged figure moves by more than 20 atoms, so that fix does not close this one.

## Attack path

1. In the second of a scheduled release, account A commits a LONG and account B a SHORT of the same size. A's deadline is 2 seconds and B's is 3: the queue is first-in first-out, so A's order must be terminal before B's can fill. Both fill at the print stamped one second after commit.
2. The trader reads the next print. If the basket fell by more than its threshold, A executes its own order in second 2; B's order expires and B clears it, and the bounty returns. If the basket rose, A's order expires and B executes its own order in second 3. If it did not move enough, both expire and both bounties return.
3. The trader closes the kept leg with an ordinary close order.

## Impact

Unless a keeper fills the dropped leg, the trader holds for free the right to enter at a stale price. The pool, the counterparty to every fill, pays for it.

Test 1 shows it on one step. With 100,000 USDG per leg and a basket that falls 50 bps two seconds after commit, the trader gains 374.74 USDG and pool assets fall by 414.74 USDG. The dropped leg costs its owner 0. With the maximum 60-second deadline the same two orders are both filled, and the trader loses 349.30 USDG.

Test 4 replays fifteen scheduled US and ECB releases between 2026-07-29 and 2026-11-10. The trader keeps a leg only if the decision print is more than 12 bps from the fill. With no keeper inside the deadlines of 2 and 3 seconds, three legs are kept, all in profit: the trader gains 1,116.30 USDG and the pool loses 1,476.31 USDG. In this replay an attempt that keeps no leg costs the trader 0 USDG. With the same commits and the 60-second deadline a keeper fills both legs, and the trader loses 21,210.87 USDG.

Test 7 puts a keeper into that replay. It arrives a fixed number of seconds after the commit and executes every order still pending, before the trader acts in that second.

| Keeper arrives | Of the deployment's first 204 orders | Of its orders since 2026-11-07 | Trader | Pool |
|---|---|---|---|---|
| 4 seconds or more after commit | 196 | 1% | gains 1,111.10 USDG | loses 1,476.31 USDG |
| 3 seconds after commit | 8 | 17% | loses 6,188.09 USDG | gains 3,663.44 USDG |
| within 2 seconds | 0 | 82% | loses 21,210.89 USDG | gains 14,005.75 USDG |

Weighted by the 204 measured orders, the trader gains 824.86 USDG and the pool loses 1,274.75 USDG over the fifteen releases. Against the keeper speeds since 2026-11-07 the attempts lose; the trader can read the keeper's recent speed before it commits.

Limits of the replay: EUR/USD and USD/JPY are the Pyth prints of those seconds; the four smaller basket components are inferred from the EUR/USD move of the same second. I chose the 12 bps threshold and the timing on these releases. A kept LONG is executed in the same second as the print it is decided on, and a kept SHORT one second later. Test 5 repeats the replay with deadlines of 4 and 5 seconds and no keeper. By the logic of test 7, that timing pays only against a keeper slower than 5 seconds (129 of the first 204 orders), and I do not rely on it.

## Severity

Critical. The pool's assets are LP funds, and the trader takes them with orders that cost it nothing when not kept and not filled by a keeper (tests 1 and 4). For the same commits and prints the pool gains 14,005.73 USDG when the orders bind and loses 1,476.31 USDG when the trader holds the deadline (test 4). With keeper arrivals weighted by the deployment's first 204 executions it still loses 1,274.75 USDG (test 7). On feasibility the terms say: "Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it."

## Recommended fix

Make the order's lifetime independent of the trader: accept at commit only the Router's full window, `maxExecutionWindowSeconds`, or a governance minimum that covers the keeper's horizon. The one-line change for the first option, in `router/OrderLiquidationBatchLogic.sol:287`:

```diff
-        if (bounds.executionWindowSeconds == 0 || bounds.executionWindowSeconds > host.maxExecutionWindowSeconds()) {
+        if (bounds.executionWindowSeconds != host.maxExecutionWindowSeconds()) {
             revert OrderRouter__InvalidExecutionWindow();
         }
```

With this change every short-deadline commit of the tests reverts with `OrderRouter__InvalidExecutionWindow`. It is enough while keepers fill inside the Router window: the leg the trader would drop is then filled before it expires. The 60-second control of tests 1, 4 and 5 is that case, and there the trader loses.


## Proof of Concept

Place the file below in `packages/perps/test/perps/OrderDeadlineFreeOption.t.sol` and run from `packages/perps`:

```
forge test --match-path test/perps/OrderDeadlineFreeOption.t.sol -vv
```

All seven tests pass on commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 with the repository's own configuration. No in-scope contract is modified. The mocks are the repository's `MockPyth` and its USDG fixture `MockSettlementToken`, both outside scope. Every order is committed with `OrderRouter.commitOrder(OrderRequest)` and executed with `OrderRouter.executeOrder`.

- Test 1: one price step, option world against binding world.
- Test 2: a keeper that arrives after the deadline cannot fill the order.
- Test 3: an order with a 1-second deadline can be filled only in its fill second.
- Test 4: fifteen releases, commit in the release second, deadlines of 2 and 3 seconds.
- Test 5: the same releases, commit one second earlier, deadlines of 4 and 5 seconds.
- Test 6: a mark refresh between commit and execution does not stop the fill.
- Test 7: the releases of test 4 with a keeper in the option world, arriving 4, 3 and 2 seconds after the commit.

Risk and Router parameters are the release values. The pool keeps three settings of the repository's test base that differ from the release. `maxSeniorExposureAssets` is unlimited (release 40,000,000 USDG). `maxSeniorShareBps` is 9,999 (release 8,000). The Senior / Junior seed is 1,000 USDG per tranche (release 1 USDG). The two caps cannot bind: Senior has no deposit beyond its 1,000 USDG seed, about 0.1% of the pool.

Expected output (amounts in settlement-token atoms, 6 decimals):

```
[PASS] test_1_OneStep_TheTraderKeepsTheWinnerAndTheLoserExpiresFree()
  option world (deadlines 3 s and 4 s)
    trader (both accounts), pool, treasury, keeper
  374744508
  -414735508
  39991000
  0
  binding world (deadline 60 s, keeper fills both at commit+7)
    trader (both accounts), pool, treasury, keeper
  -349302584
  189302584
  159600000
  400000
[PASS] test_2_AKeeperAfterTheDeadlineCannotFillTheOrder()
  owner's cost when a keeper clears the expired order (atoms) 200000
[PASS] test_3_AOneSecondOrderCanBeFilledOnlyInItsFillSecond()
[PASS] test_4_FifteenReleases_DeadlinesOfTwoAndThreeSeconds()
  release row / kept leg (1 LONG, 2 SHORT) 6 1
    trader, pool
  384537701
  -504540303
  release row / kept leg (1 LONG, 2 SHORT) 8 2
    trader, pool
  285431278
  -405434164
  release row / kept leg (1 LONG, 2 SHORT) 9 1
    trader, pool
  446335795
  -566335771
  legs kept / kept with a gain 3 3
  option world, fifteen releases, deadlines 2 s and 3 s
    trader (both accounts), pool, treasury, keeper
  1116304774
  -1476310238
  360005464
  0
  binding world, the same commits with the 60 s deadline
    trader (both accounts), pool, treasury, keeper
  -21210871015
  14005731436
  7199139579
  6000000
[PASS] test_5_FifteenReleases_DeadlinesOfFourAndFiveSeconds()
  release row / kept leg (1 LONG, 2 SHORT) 1 2
    trader, pool
  481119864
  -601120443
  release row / kept leg (1 LONG, 2 SHORT) 3 1
    trader, pool
  692164395
  -812158312
  release row / kept leg (1 LONG, 2 SHORT) 6 1
    trader, pool
  352957908
  -472954574
  release row / kept leg (1 LONG, 2 SHORT) 8 2
    trader, pool
  392659539
  -512668476
  release row / kept leg (1 LONG, 2 SHORT) 9 1
    trader, pool
  375048006
  -495041030
  release row / kept leg (1 LONG, 2 SHORT) 13 1
    trader, pool
  93850399
  -213848949
  legs kept / kept with a gain 6 6
  option world, fifteen releases, deadlines 4 s and 5 s
    trader (both accounts), pool, treasury, keeper
  2387800111
  -3107791784
  719991673
  0
  binding world, the same commits with the 60 s deadline
    trader (both accounts), pool, treasury, keeper
  -21105462658
  13900293180
  7199169478
  6000000
[PASS] test_6_AMarkRefreshBetweenCommitAndExecutionDoesNotStopTheFill()
[PASS] test_7_FifteenReleases_WithAKeeperInTheOptionWorld()
  keeper 4 s after commit (196 of the 204 orders took 4 s or more)
    trader (both accounts), pool, treasury, keeper
  1111104774
  -1476310238
  360005464
  5200000
  keeper 3 s after commit (8 of the 204 orders)
    trader (both accounts), pool, treasury, keeper
  -6188088090
  3663439905
  2519048185
  5600000
  keeper 2 s after commit (none of the 204 orders)
    trader (both accounts), pool, treasury, keeper
  -21210891741
  14005752162
  7199139579
  6000000
  releases where the keeper bound both legs, at 4 s / 3 s / 2 s 0 2 15
  weighted by the 204 measured orders: trader, pool
  824861916
  -1274751408
Suite result: ok. 7 passed; 0 failed; 0 skipped
```

The full file:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.33;

// The order deadline as a free option.
//
// An order's fill price is the first print stamped after its commit second, and execution accepts that print
// until the order's deadline. The trader chooses the order's deadline (executionWindowSeconds, 1 second or more). An order
// that passes its deadline fails as expired, and its owner gets the reserved bounty back when it clears the order.
// So a trader commits a LONG on one account and a SHORT on another just before prices move, waits for the next
// prints, executes the leg that is already in profit at the old print, and lets the other leg expire.
//
// Every test runs two worlds from one snapshot with the same commits and the same prints:
//   option world:  short deadlines; the trader executes the kept leg itself; the other leg expires.
//   binding world: the maximum 60-second deadline; a keeper fills both legs 7 seconds after commit.
// Test 7 puts a keeper into the option world: it arrives a fixed number of seconds after the commit and
// executes every order that is still pending, before the trader acts in that second.
//
// Parameters are the v2.0.0 release values (packages/perps/DEPLOYMENT.md): VPI factor 0.01, initial and maintenance
// margin 20 and 10 bps, confidence multiplier 2,500 bps, minimum open notional 1,000 USDG. The pool is the test
// base's 1,000,000 USDG of Junior, with the base's other pool settings. Every order is committed with
// OrderRouter.commitOrder(OrderRequest) and executed with OrderRouter.executeOrder. The mocks are the repository's
// MockPyth and its USDG fixture MockSettlementToken.
//
// Release rows (tests 4, 5 and 7): per-second basket values around fifteen scheduled US and ECB releases between
// 2026-07-29 and 2026-11-10. EUR/USD and USD/JPY are the Pyth prints of those seconds (Hermes). The four smaller
// components are inferred from the EUR/USD move of the same second. Each row is normalised so that the print of
// the commit second is 1.00000000, and every mock feed carries the basket value, so the basket equals it. Tests 1
// to 3 and 6 use no market data.
//
// Place in packages/perps/test/perps/ and run from packages/perps:
//   forge test --match-path test/perps/OrderDeadlineFreeOption.t.sol -vv
// Amounts are logged in settlement-token atoms (6 decimals): 374744508 is 374.744508 USDG.

import {OracleSynchronizationTestBase} from "./shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract OrderDeadlineFreeOptionPoC is OracleSynchronizationTestBase {
    address internal constant KEEPER = address(0xBEEF);
    // The two trading accounts the test base funds with 10,000 USDG each. One trader controls both.
    address internal constant LONG_ACCOUNT = address(0xA11CE);
    address internal constant SHORT_ACCOUNT = address(0xB0B);
    int256 internal constant THETA_BPS = 12; // keep a leg only if the decision print is this far from the fill

    struct Ev {
        int64 pf; // fill print: the first print stamped after the commit second
        uint64 cff; // its confidence
        int64 pd; // the print the trader decides on
        int64 pk; // the print the kept leg is closed at (stamped commit+8)
        uint64 ck; // its confidence
    }

    struct Tot {
        int256 trader; // both accounts of the trader
        int256 pool; // HousePool assets
        int256 treasury;
        int256 keeper;
    }

    uint256 internal nonce;
    bool internal releaseApplied;

    // ---- release parameters ----

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

    /// Router release settings through the admin delay, then back to the base setup's weekday and hour (a live
    /// market), one week later, and a mark refresh there.
    function _release() internal {
        if (releaseApplied) return;
        releaseApplied = true;
        uint256 t0 = vm.getBlockTimestamp();
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(rc);
        vm.warp(routerAdmin.routerConfigActivationTime());
        routerAdmin.finalizeRouterConfig();
        vm.warp(t0 + 7 days);
        vm.deal(LONG_ACCOUNT, 1 ether);
        vm.deal(SHORT_ACCOUNT, 1 ether);
        vm.deal(KEEPER, 1 ether);
        // A fresh engine mark in the commit second, at the commit-second price of every test (1.00000000).
        vm.prank(KEEPER);
        router.updateMarkPrice{value: 2 * FEE}(_print(uint64(vm.getBlockTimestamp()), 100_000_000, 10_000));
    }

    // ---- public entry points only ----

    /// One Pyth update for all six feeds: `price` with confidence `conf`, stamped `tick`.
    function _print(uint64 tick, int64 price, uint64 conf) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _synchronizationFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(price, conf, -8, tick, tick - 1));
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    /// OrderRouter.commitOrder: an open with the trader's own execution deadline, `window` seconds after commit.
    function _commitOpen(address trader, CfdTypes.Side side, uint32 window, uint256 size, uint256 margin)
        internal
        returns (uint64 id)
    {
        OrderV3Types.OrderRequest memory r;
        r.clientOrderId = bytes32(++nonce);
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = side == CfdTypes.Side.LONG ? 1 : type(uint256).max;
        r.bounds.submitBy = uint64(vm.getBlockTimestamp()) + 60;
        r.bounds.executionWindowSeconds = window;
        r.bounds.allowedExecutionModes = 7;
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        r.bounds.maxExecutionBountyAssets = type(uint256).max;
        r.bounds.maxExecutionNotionalAssets = type(uint256).max;
        r.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        r.bounds.maxActionChargeAssets = type(uint256).max;
        r.bounds.maxExplicitFeesAssets = type(uint256).max;
        r.bounds.maxPostPositionSize = type(uint256).max;
        r.bounds.maxPostLeverageBps = type(uint32).max;
        vm.prank(trader);
        id = router.commitOrder(r);
    }

    /// OrderRouter.executeOrder, called by `executor`.
    function _exec(address executor, uint64 id, bytes[] memory data) internal {
        vm.prank(executor);
        router.executeOrder{value: 2 * FEE}(id, data);
    }

    function _status(uint64 id) internal view returns (uint256) {
        return uint256(router.lifecycleBook().lifecycleStatus(id));
    }

    function _pending(uint64 id) internal view returns (bool) {
        return _status(id) == uint256(OrderV3Types.LifecycleStatus.Pending);
    }

    function _bal(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a);
    }

    function _size(address a) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    /// A full close committed at `commitAt` and filled by the keeper one second later at `price`.
    function _close(address trader, CfdTypes.Side side, uint256 size, uint64 commitAt, int64 price, uint64 conf)
        internal
    {
        if (vm.getBlockTimestamp() < commitAt) _advance(commitAt);
        OrderV3Types.OrderRequest memory r;
        r.clientOrderId = bytes32(++nonce);
        r.side = side;
        r.sizeDelta = size;
        r.isClose = true;
        r.targetPrice = side == CfdTypes.Side.LONG ? type(uint256).max : 1;
        r.bounds.submitBy = uint64(vm.getBlockTimestamp()) + 60;
        r.bounds.executionWindowSeconds = 60;
        r.bounds.allowedExecutionModes = 7;
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        r.bounds.maxExecutionBountyAssets = type(uint256).max;
        r.bounds.maxExecutionNotionalAssets = type(uint256).max;
        r.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        r.bounds.maxActionChargeAssets = type(uint256).max;
        r.bounds.maxExplicitFeesAssets = type(uint256).max;
        r.bounds.maxPostPositionSize = type(uint256).max;
        r.bounds.maxPostLeverageBps = type(uint32).max;
        vm.prank(trader);
        uint64 id = router.commitOrder(r);
        _advance(commitAt + 1);
        _exec(KEEPER, id, _print(commitAt + 1, price, conf));
        assertEq(_status(id), uint256(OrderV3Types.LifecycleStatus.Executed), "close executed");
        assertEq(_size(trader), 0, "flat after the close");
    }

    function _snapshotTotals() internal view returns (uint256[5] memory s) {
        s = [_bal(LONG_ACCOUNT), _bal(SHORT_ACCOUNT), _bal(KEEPER), _bal(PROTOCOL_TREASURY_ACCOUNT), pool.totalAssets()];
    }

    function _diff(uint256[5] memory s) internal view returns (Tot memory d) {
        d.trader = int256(_bal(LONG_ACCOUNT)) + int256(_bal(SHORT_ACCOUNT)) - int256(s[0]) - int256(s[1]);
        d.keeper = int256(_bal(KEEPER)) - int256(s[2]);
        d.treasury = int256(_bal(PROTOCOL_TREASURY_ACCOUNT)) - int256(s[3]);
        d.pool = int256(pool.totalAssets()) - int256(s[4]);
        assertEq(d.trader + d.keeper + d.treasury + d.pool, 0, "every atom is accounted for");
    }

    /// One LONG and one SHORT, on the trader's two accounts, committed in the same second. `e.pf` is their fill print.
    /// Option world: the first leg's deadline is `dA` seconds and the second leg's `dA + 1`, because the queue is
    /// first-in first-out and the first leg must be terminal before the second can fill. The trader looks at
    /// `e.pd`, executes the leg that is in profit, and lets the other expire. If neither is, both expire.
    /// Binding world: both deadlines are 60 seconds and a keeper fills both legs at commit+7.
    function _pair(Ev memory e, bool optionWorld, uint32 dA, uint256 size, uint256 margin)
        internal
        returns (Tot memory d, uint256 kept)
    {
        uint256[5] memory s = _snapshotTotals();
        uint64 c = uint64(vm.getBlockTimestamp());
        uint64 idA = _commitOpen(LONG_ACCOUNT, CfdTypes.Side.LONG, optionWorld ? dA : 60, size, margin);
        uint64 idB = _commitOpen(SHORT_ACCOUNT, CfdTypes.Side.SHORT, optionWorld ? dA + 1 : 60, size, margin);
        bytes[] memory fill = _print(c + 1, e.pf, e.cff);
        if (optionWorld) {
            int256 moveBps = ((int256(e.pd) - int256(e.pf)) * 10_000) / int256(e.pf);
            if (moveBps < -THETA_BPS) {
                // The basket fell: the LONG is in profit. Its owner executes it in the last second of its deadline.
                _advance(c + dA);
                _exec(LONG_ACCOUNT, idA, fill);
                assertEq(_status(idA), uint256(OrderV3Types.LifecycleStatus.Executed), "LONG filled by its owner");
                assertEq(engine.lastMarkTime(), c + 1, "at the print stamped commit+1");
                _advance(c + dA + 2);
                _exec(SHORT_ACCOUNT, idB, fill);
                assertEq(_status(idB), uint256(OrderV3Types.LifecycleStatus.Failed), "SHORT expired");
                assertEq(_size(SHORT_ACCOUNT), 0, "SHORT never filled");
                kept = 1;
                _close(LONG_ACCOUNT, CfdTypes.Side.LONG, size, c + 7, e.pk, e.ck);
            } else if (moveBps > THETA_BPS) {
                // The basket rose: the SHORT is in profit. The LONG ahead of it expires first.
                _advance(c + dA + 1);
                _exec(SHORT_ACCOUNT, idB, fill);
                assertEq(_status(idA), uint256(OrderV3Types.LifecycleStatus.Failed), "LONG expired");
                assertEq(_status(idB), uint256(OrderV3Types.LifecycleStatus.Executed), "SHORT filled by its owner");
                assertEq(_size(LONG_ACCOUNT), 0, "LONG never filled");
                kept = 2;
                _close(SHORT_ACCOUNT, CfdTypes.Side.SHORT, size, c + 7, e.pk, e.ck);
            } else {
                _advance(c + dA + 2);
                _exec(LONG_ACCOUNT, idA, fill);
                _exec(SHORT_ACCOUNT, idB, fill);
                assertEq(_status(idA), uint256(OrderV3Types.LifecycleStatus.Failed), "LONG expired");
                assertEq(_status(idB), uint256(OrderV3Types.LifecycleStatus.Failed), "SHORT expired");
            }
        } else {
            _advance(c + 7);
            _exec(KEEPER, idA, fill);
            _exec(KEEPER, idB, fill);
            assertEq(_status(idA), uint256(OrderV3Types.LifecycleStatus.Executed), "binding: LONG filled");
            assertEq(_status(idB), uint256(OrderV3Types.LifecycleStatus.Executed), "binding: SHORT filled");
            kept = 3;
            _close(LONG_ACCOUNT, CfdTypes.Side.LONG, size, c + 7, e.pk, e.ck);
            _close(SHORT_ACCOUNT, CfdTypes.Side.SHORT, size, c + 8, e.pk, e.ck);
        }
        d = _diff(s);
    }

    /// The option world of `_pair` with a keeper in it. The keeper arrives `keeperAt` seconds after the commit and
    /// executes every order that is still pending, in queue order, before the trader acts in that second. The trader
    /// follows the same rule as in `_pair`. `kept` is 1 if only the LONG ends open, 2 if only the SHORT, 3 if both
    /// (the keeper bound the leg the trader meant to drop), 0 if neither. Whatever is open is closed as in `_pair`.
    function _pairKeeper(Ev memory e, uint32 dA, uint32 keeperAt, uint256 size, uint256 margin)
        internal
        returns (Tot memory d, uint256 kept)
    {
        uint256[5] memory s = _snapshotTotals();
        uint64 c = uint64(vm.getBlockTimestamp());
        uint64 idA = _commitOpen(LONG_ACCOUNT, CfdTypes.Side.LONG, dA, size, margin);
        uint64 idB = _commitOpen(SHORT_ACCOUNT, CfdTypes.Side.SHORT, dA + 1, size, margin);
        bytes[] memory fill = _print(c + 1, e.pf, e.cff);
        int256 moveBps = ((int256(e.pd) - int256(e.pf)) * 10_000) / int256(e.pf);
        for (uint64 t = c + 1; t <= c + dA + 2; ++t) {
            _advance(t);
            if (t == c + keeperAt) {
                if (_pending(idA)) _exec(KEEPER, idA, fill);
                if (_pending(idB)) _exec(KEEPER, idB, fill);
            }
            if (t == c + dA && moveBps < -THETA_BPS && _pending(idA)) _exec(LONG_ACCOUNT, idA, fill);
            if (t == c + dA + 1 && moveBps > THETA_BPS && _pending(idB)) _exec(SHORT_ACCOUNT, idB, fill);
        }
        // Both deadlines have passed: the trader clears whatever nobody has cleared yet.
        if (_pending(idA)) _exec(LONG_ACCOUNT, idA, fill);
        if (_pending(idB)) _exec(SHORT_ACCOUNT, idB, fill);
        if (_size(LONG_ACCOUNT) != 0) kept += 1;
        if (_size(SHORT_ACCOUNT) != 0) kept += 2;
        if ((kept & 1) != 0) _close(LONG_ACCOUNT, CfdTypes.Side.LONG, size, c + 7, e.pk, e.ck);
        if ((kept & 2) != 0) _close(SHORT_ACCOUNT, CfdTypes.Side.SHORT, size, kept == 3 ? c + 8 : c + 7, e.pk, e.ck);
        d = _diff(s);
    }

    function _logTot(string memory tag, Tot memory t) internal pure {
        console2.log(tag);
        console2.log("  trader (both accounts), pool, treasury, keeper");
        console2.logInt(t.trader);
        console2.logInt(t.pool);
        console2.logInt(t.treasury);
        console2.logInt(t.keeper);
    }

    // ---------------------------------------------------------------------------------------------------------
    // 1. The mechanism on one price step: 100,000 per leg, the basket falls 50 bps two seconds after commit
    // ---------------------------------------------------------------------------------------------------------

    function test_1_OneStep_TheTraderKeepsTheWinnerAndTheLoserExpiresFree() public {
        _release();
        Ev memory e = Ev(100_000_000, 90_000, 99_500_000, 99_500_000, 90_000);
        uint256 shortBefore = _bal(SHORT_ACCOUNT);
        uint256 snap = vm.snapshotState();
        (Tot memory opt, uint256 kept) = _pair(e, true, 3, 100_000e18, 2000e6);
        uint256 shortAfter = _bal(SHORT_ACCOUNT);
        vm.revertToState(snap);
        (Tot memory bin,) = _pair(e, false, 3, 100_000e18, 2000e6);
        _logTot("option world (deadlines 3 s and 4 s)", opt);
        _logTot("binding world (deadline 60 s, keeper fills both at commit+7)", bin);

        assertEq(kept, 1, "the LONG was kept");
        assertEq(shortAfter, shortBefore, "the expired leg cost its owner nothing: the bounty came back");
        assertGt(opt.trader, 300e6, "option world: the trader gains");
        assertLt(opt.pool, -300e6, "option world: the pool pays");
        assertLt(bin.trader, 0, "binding world: the same two orders lose");
        assertGt(bin.pool, 0, "binding world: the pool gains");
    }

    // ---------------------------------------------------------------------------------------------------------
    // 2. After the deadline nobody can fill the order
    // ---------------------------------------------------------------------------------------------------------

    function test_2_AKeeperAfterTheDeadlineCannotFillTheOrder() public {
        _release();
        uint256 shortBefore = _bal(SHORT_ACCOUNT);
        uint64 c = uint64(vm.getBlockTimestamp());
        uint64 id = _commitOpen(SHORT_ACCOUNT, CfdTypes.Side.SHORT, 2, 100_000e18, 2000e6);
        _advance(c + 3);
        _exec(KEEPER, id, _print(c + 1, 100_000_000, 90_000));
        assertEq(_status(id), uint256(OrderV3Types.LifecycleStatus.Failed), "expired, not filled");
        assertEq(_size(SHORT_ACCOUNT), 0, "no position");
        console2.log("owner's cost when a keeper clears the expired order (atoms)", shortBefore - _bal(SHORT_ACCOUNT));
        assertLe(shortBefore - _bal(SHORT_ACCOUNT), 200_000, "at most the 0.2 USDG bounty, paid to the keeper who cleared it");
    }

    // ---------------------------------------------------------------------------------------------------------
    // 3. The trader sets how long a keeper has: a 1-second order can be filled only in its fill second
    // ---------------------------------------------------------------------------------------------------------

    function test_3_AOneSecondOrderCanBeFilledOnlyInItsFillSecond() public {
        _release();
        uint64 c = uint64(vm.getBlockTimestamp());
        uint256 snap = vm.snapshotState();
        uint64 id = _commitOpen(LONG_ACCOUNT, CfdTypes.Side.LONG, 1, 100_000e18, 2000e6);
        _advance(c + 1);
        _exec(KEEPER, id, _print(c + 1, 100_000_000, 90_000));
        assertEq(_status(id), uint256(OrderV3Types.LifecycleStatus.Executed), "filled in second commit+1");
        vm.revertToState(snap);
        id = _commitOpen(LONG_ACCOUNT, CfdTypes.Side.LONG, 1, 100_000e18, 2000e6);
        _advance(c + 2);
        _exec(KEEPER, id, _print(c + 1, 100_000_000, 90_000));
        assertEq(_status(id), uint256(OrderV3Types.LifecycleStatus.Failed), "one second later it has expired");
    }

    // ---------------------------------------------------------------------------------------------------------
    // 4. Fifteen scheduled releases, deadlines of 2 and 3 seconds
    //    Both legs are committed in the release second T. The fill is the print stamped T+1, the trader decides
    //    on the print stamped T+2, and the kept leg is closed at the print stamped T+8. 300,000 per leg.
    // ---------------------------------------------------------------------------------------------------------

    function _tightRow(uint256 i) internal pure returns (Ev memory e) {
        if (i == 0) return Ev(99994395, 21720, 100030806, 100170019, 21520); // FOMC decision, T = 2026-07-29 18:00:00 UTC (1785348000)
        if (i == 1) return Ev(99999828, 9648, 100092548, 100296010, 41801); // US payrolls, T = 2026-08-07 12:30:00 UTC (1786105800)
        if (i == 2) return Ev(100017653, 25407, 100115775, 99976203, 19563); // US CPI, T = 2026-08-12 12:30:00 UTC (1786537800)
        if (i == 3) return Ev(100000000, 8062, 99932497, 99663488, 14101); // US payrolls, T = 2026-09-04 12:30:00 UTC (1788525000)
        if (i == 4) return Ev(100002480, 4345, 100000819, 100005971, 2571); // ECB decision, T = 2026-09-10 12:15:00 UTC (1789042500)
        if (i == 5) return Ev(99997253, 3482, 99996894, 100005078, 2842); // ECB press conference, T = 2026-09-10 12:45:00 UTC (1789044300)
        if (i == 6) return Ev(100005602, 13733, 99864872, 99791731, 6601); // US CPI, T = 2026-09-11 12:30:00 UTC (1789129800)
        if (i == 7) return Ev(100011974, 16806, 99986829, 100010268, 10632); // FOMC decision, T = 2026-09-16 18:00:00 UTC (1789581600)
        if (i == 8) return Ev(99999824, 10338, 100181200, 100179922, 9204); // US payrolls, T = 2026-10-02 12:30:00 UTC (1790944200)
        if (i == 9) return Ev(100002181, 8807, 99861740, 99767184, 13440); // US CPI, T = 2026-10-14 12:30:00 UTC (1791981000)
        if (i == 10) return Ev(99995447, 11819, 100010940, 100042411, 10675); // FOMC decision, T = 2026-10-28 18:00:00 UTC (1793210400)
        if (i == 11) return Ev(100000998, 9046, 100002457, 100010194, 7033); // ECB decision, T = 2026-10-29 13:15:00 UTC (1793279700)
        if (i == 12) return Ev(100005356, 3828, 100007123, 100009538, 4668); // ECB press conference, T = 2026-10-29 13:45:00 UTC (1793281500)
        if (i == 13) return Ev(100000749, 8424, 99884771, 99877228, 13385); // US payrolls, T = 2026-11-06 13:30:00 UTC (1793971800)
        if (i == 14) return Ev(100011916, 9547, 99924959, 99790673, 14295); // US CPI, T = 2026-11-10 13:30:00 UTC (1794317400)
        revert();
    }

    function _replay(bool tight) internal returns (Tot memory opt, Tot memory bin, uint256 kept, uint256 keptGain) {
        _release();
        uint256 snap = vm.snapshotState();
        for (uint256 i; i < 15; ++i) {
            Ev memory e = tight ? _tightRow(i) : _wideRow(i);
            (Tot memory o, uint256 k) = _pair(e, true, tight ? 2 : 4, 300_000e18, 5000e6);
            vm.revertToState(snap);
            (Tot memory b,) = _pair(e, false, tight ? 2 : 4, 300_000e18, 5000e6);
            vm.revertToState(snap);
            if (k == 0) {
                assertEq(o.trader, 0, "an attempt that keeps no leg costs the trader nothing");
                assertEq(o.pool, 0, "and moves nothing in the pool");
            } else {
                kept++;
                if (o.trader > 0) keptGain++;
                console2.log("release row / kept leg (1 LONG, 2 SHORT)", i, k);
                console2.log("  trader, pool");
                console2.logInt(o.trader);
                console2.logInt(o.pool);
            }
            opt.trader += o.trader;
            opt.pool += o.pool;
            opt.treasury += o.treasury;
            opt.keeper += o.keeper;
            bin.trader += b.trader;
            bin.pool += b.pool;
            bin.treasury += b.treasury;
            bin.keeper += b.keeper;
        }
    }

    function test_4_FifteenReleases_DeadlinesOfTwoAndThreeSeconds() public {
        (Tot memory opt, Tot memory bin, uint256 kept, uint256 keptGain) = _replay(true);
        console2.log("legs kept / kept with a gain", kept, keptGain);
        _logTot("option world, fifteen releases, deadlines 2 s and 3 s", opt);
        _logTot("binding world, the same commits with the 60 s deadline", bin);
        assertEq(kept, 3, "three releases moved enough within two seconds");
        assertEq(keptGain, kept, "every kept leg gained");
        assertGt(opt.trader, 1000e6, "the trader gains");
        assertLt(opt.pool, -1000e6, "the pool pays");
        assertLt(bin.trader, 0, "bound to both fills, the trader loses");
        assertGt(bin.pool, 0, "and the pool gains");
    }

    // ---------------------------------------------------------------------------------------------------------
    // 5. The same fifteen releases, deadlines of 4 and 5 seconds
    //    Both legs are committed in second T-1. The fill is the print stamped T, the trader decides on the print
    //    stamped T+3, and the kept leg is closed at the print stamped T+7.
    // ---------------------------------------------------------------------------------------------------------

    function _wideRow(uint256 i) internal pure returns (Ev memory e) {
        if (i == 0) return Ev(99997330, 8775, 100085939, 100190163, 18858); // FOMC decision, T = 2026-07-29 18:00:00 UTC (1785348000)
        if (i == 1) return Ev(99998096, 9562, 100148332, 100247037, 24322); // US payrolls, T = 2026-08-07 12:30:00 UTC (1786105800)
        if (i == 2) return Ev(99992810, 11742, 100098699, 99991237, 10670); // US CPI, T = 2026-08-12 12:30:00 UTC (1786537800)
        if (i == 3) return Ev(99996573, 6563, 99694212, 99680421, 11684); // US payrolls, T = 2026-09-04 12:30:00 UTC (1788525000)
        if (i == 4) return Ev(99999451, 3660, 100000360, 100005782, 4715); // ECB decision, T = 2026-09-10 12:15:00 UTC (1789042500)
        if (i == 5) return Ev(99998866, 2483, 99999832, 100001282, 1775); // ECB press conference, T = 2026-09-10 12:45:00 UTC (1789044300)
        if (i == 6) return Ev(99999638, 9653, 99811509, 99797253, 6961); // US CPI, T = 2026-09-11 12:30:00 UTC (1789129800)
        if (i == 7) return Ev(99999670, 11633, 100029872, 100002071, 7833); // FOMC decision, T = 2026-09-16 18:00:00 UTC (1789581600)
        if (i == 8) return Ev(100005260, 8757, 100219861, 100220870, 9784); // US payrolls, T = 2026-10-02 12:30:00 UTC (1790944200)
        if (i == 9) return Ev(99996890, 10819, 99803936, 99784208, 17493); // US CPI, T = 2026-10-14 12:30:00 UTC (1791981000)
        if (i == 10) return Ev(100001142, 5896, 100044697, 100043697, 17619); // FOMC decision, T = 2026-10-28 18:00:00 UTC (1793210400)
        if (i == 11) return Ev(100003205, 5019, 100004291, 100006765, 3913); // ECB decision, T = 2026-10-29 13:15:00 UTC (1793279700)
        if (i == 12) return Ev(100000842, 2812, 100007303, 100009850, 4005); // ECB press conference, T = 2026-10-29 13:45:00 UTC (1793281500)
        if (i == 13) return Ev(100000614, 7283, 99863144, 99884460, 10778); // US payrolls, T = 2026-11-06 13:30:00 UTC (1793971800)
        if (i == 14) return Ev(100001659, 7003, 99882192, 99796870, 24849); // US CPI, T = 2026-11-10 13:30:00 UTC (1794317400)
        revert();
    }

    function test_5_FifteenReleases_DeadlinesOfFourAndFiveSeconds() public {
        (Tot memory opt, Tot memory bin, uint256 kept, uint256 keptGain) = _replay(false);
        console2.log("legs kept / kept with a gain", kept, keptGain);
        _logTot("option world, fifteen releases, deadlines 4 s and 5 s", opt);
        _logTot("binding world, the same commits with the 60 s deadline", bin);
        assertEq(kept, 6, "six releases moved enough within four seconds");
        assertEq(keptGain, kept, "every kept leg gained");
        assertGt(opt.trader, 2000e6, "the trader gains");
        assertLt(opt.pool, -2000e6, "the pool pays");
        assertLt(bin.trader, 0, "bound to both fills, the trader loses");
        assertGt(bin.pool, 0, "and the pool gains");
    }

    // ---------------------------------------------------------------------------------------------------------
    // 6. A mark refresh between commit and execution does not stop the fill at the older print
    // ---------------------------------------------------------------------------------------------------------

    function test_6_AMarkRefreshBetweenCommitAndExecutionDoesNotStopTheFill() public {
        _release();
        uint64 c = uint64(vm.getBlockTimestamp());
        uint64 id = _commitOpen(LONG_ACCOUNT, CfdTypes.Side.LONG, 3, 100_000e18, 2000e6);
        _advance(c + 2);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: 2 * FEE}(_print(c + 2, 99_500_000, 90_000));
        assertEq(engine.lastMarkTime(), c + 2, "the mark is now the print stamped commit+2");
        _advance(c + 3);
        _exec(LONG_ACCOUNT, id, _print(c + 1, 100_000_000, 90_000));
        assertEq(_status(id), uint256(OrderV3Types.LifecycleStatus.Executed), "filled at the print stamped commit+1");
        assertEq(_size(LONG_ACCOUNT), 100_000e18, "the position is open");
    }

    // ---------------------------------------------------------------------------------------------------------
    // 7. The fifteen releases of test 4 with a keeper in the option world
    //    Deadlines of 2 and 3 seconds. A keeper arrives K seconds after the commit and executes every order that
    //    is still pending, before the trader acts in that second. On the Arbitrum Sepolia deployment, the Router's
    //    logs show 204 orders executed until 2026-11-07T15:09Z: none within 2 seconds of its commit, 8 after
    //    3 seconds and 196 after 4 seconds or more. The last figures weight the replays by those counts.
    // ---------------------------------------------------------------------------------------------------------

    function _replayKeeper(uint32 keeperAt) internal returns (Tot memory tot, uint256 both) {
        uint256 snap = vm.snapshotState();
        for (uint256 i; i < 15; ++i) {
            (Tot memory o, uint256 k) = _pairKeeper(_tightRow(i), 2, keeperAt, 300_000e18, 5000e6);
            vm.revertToState(snap);
            if (k == 3) both++;
            tot.trader += o.trader;
            tot.pool += o.pool;
            tot.treasury += o.treasury;
            tot.keeper += o.keeper;
        }
    }

    function test_7_FifteenReleases_WithAKeeperInTheOptionWorld() public {
        _release();
        (Tot memory k4, uint256 both4) = _replayKeeper(4);
        (Tot memory k3, uint256 both3) = _replayKeeper(3);
        (Tot memory k2, uint256 both2) = _replayKeeper(2);
        _logTot("keeper 4 s after commit (196 of the 204 orders took 4 s or more)", k4);
        _logTot("keeper 3 s after commit (8 of the 204 orders)", k3);
        _logTot("keeper 2 s after commit (none of the 204 orders)", k2);
        console2.log("releases where the keeper bound both legs, at 4 s / 3 s / 2 s", both4, both3, both2);
        int256 trader = (196 * k4.trader + 8 * k3.trader) / 204;
        int256 pool = (196 * k4.pool + 8 * k3.pool) / 204;
        console2.log("weighted by the 204 measured orders: trader, pool");
        console2.logInt(trader);
        console2.logInt(pool);

        assertEq(both4, 0, "a keeper at 4 s finds both orders expired or already executed");
        assertGt(k4.trader, 1000e6, "keeper at 4 s: the trader gains");
        assertLt(k4.pool, -1000e6, "keeper at 4 s: the pool pays");
        assertLt(k3.trader, 0, "keeper at 3 s: it fills the second leg and the trader loses");
        assertEq(both2, 15, "keeper at 2 s: it binds both legs at every release");
        assertLt(k2.trader, 0, "keeper at 2 s: the trader loses");
        assertGt(trader, 0, "against the measured keeper speeds the attack pays");
        assertLt(pool, 0, "and the pool loses");
    }
}
```

