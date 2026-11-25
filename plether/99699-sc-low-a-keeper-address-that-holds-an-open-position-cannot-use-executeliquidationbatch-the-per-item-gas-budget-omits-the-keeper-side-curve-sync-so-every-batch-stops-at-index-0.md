# \#99699 [SC-Low] A keeper address that holds an open position cannot use executeLiquidationBatch: the per-item gas budget omits the keeper-side curve sync, so every batch stops at index 0

**Submitted on Nov 14th 2026 at 20:37:34 UTC by @EntropyResearch for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99699
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Category:** Insight — Code Optimizations and Enhancements (liquidation batch gas budgeting)
**Target:** `packages/perps/src/router/OrderLiquidationBatchLogic.sol` (`executeLiquidationBatch`), `packages/perps/src/CfdEngine.sol` (`_applyLiquidation`)
**Commit:** a82d4fefb02768e5b3bddcc97dee371eee24fea8

## Brief/Intro

`executeLiquidationBatch` gives every item a fixed gas budget that depends only on the liquidated account. When the caller (the keeper) holds an open position of its own, the engine performs extra keeper-side work on every liquidation that the budget does not include. The item runs out of gas, the batch emits `LiquidationBatchStopped(0)` and returns 0 without liquidating anyone, and this repeats for every batch that address sends. The single-account `executeLiquidation` has no item cap and still works, so no funds are at risk; the batch path is simply unusable for any liquidator that also trades from the same address.

## Vulnerability Details

1. Each batch item is called with a budget derived from the liquidated account only (`router/OrderLiquidationBatchLogic.sol:765-766, 777`):

```solidity
uint256 itemGas = engineGas + LIQUIDATION_BATCH_ROUTER_GAS
    + (host.pendingOrderCounts(account) * LIQUIDATION_BATCH_GAS_PER_ORDER);
...
try host.executeLiquidationBatchItem{gas: itemGas}(account, ..., msg.sender, riskOffCutoff)
```

   With `minEngineGas = 600_000` (`router/OrderRouterBase.sol:76`, not overridden in `DEPLOYMENT.md`) and `LIQUIDATION_BATCH_ROUTER_GAS = 250_000` (L114), an account without pending orders gets **850,000** gas.

2. The item's work also depends on the caller. When the keeper has a position, `CfdEngine._applyLiquidation` opens a terminal-curve mutation for the keeper and checkpoints its carry before crediting the bounty (`CfdEngine.sol:1331-1338`):

```solidity
bool syncKeeper = delta.keeperBountyAssets > 0 && keeper != delta.account && _positions[keeper].lots > 0;
bytes32 keeperExpectedOldHash;
if (syncKeeper) {
    keeperExpectedOldHash = _beginTerminalCurveMutation(keeper);
}
if (delta.keeperBountyAssets > 0 && keeper != delta.account) {
    _checkpointBountyRecipient(keeper, delta.price, publishTime);
}
```

3. An out-of-gas item reverts with empty data, and the batch stops at that index and returns (`OrderLiquidationBatchLogic.sol:796-799`):

```solidity
} catch (bytes memory revertData) {
    if (revertData.length == 0) {
        emit LiquidationBatchStopped(nextIndex);
        return nextIndex;
    }
```

Measured with the production Router, release risk parameters and cold storage per transaction (`--isolate`), same victim and same price in both runs (gas of the `executeLiquidationBatchItem` frame from the `-vvvv` trace):

| Scenario | Item gas | Item budget | Batch result | Batch tx gas |
|---|---|---|---|---|
| Keeper address without a position | 591,188 | 850,000 | liquidated, `nextIndex = 1` | 868,603 |
| **Keeper address holds a $50k position** | **OutOfGas after 835,228** (needs 912,362 uncapped) | 850,000 | **`LiquidationBatchStopped(0)`, `nextIndex = 0`, nobody liquidated** | 1,114,532 |
| Same keeper, single-account `executeLiquidation` (no cap) | 912,362 | — | liquidated | 1,154,850 |

The keeper-side sync adds about 321k gas (912,362 − 591,188), roughly 37% over the budget.

**Relation to known issues.** Cecuro LOW-52 (acknowledged) concerns the batch item budget and EIP-150 forwarding; the team noted that a finite item budget does not guarantee success under every state condition, and `SECURITY.md:1259-1261` documents that an empty-revert item stops the batch so a keeper can resume. This insight is narrower and deterministic: the full `itemGas` is forwarded, but the formula omits a cost that depends on the caller, so a keeper that holds a position fails on every item of every batch, and resuming cannot help. It is submitted as an Insight because the fix is local and the trigger is entirely on the keeper side.

## Impact Details

- No user or LP funds are at risk: positions are bound to `msg.sender`, so nobody can impose a position on another keeper, and the single-account path still liquidates.
- A liquidator that also trades or hedges from the same address (including a protocol-operated keeper that holds inventory) loses the batch path entirely: every `executeLiquidationBatch` call returns 0 after paying the transaction gas (≈1.1M in the PoC) and, on mainnet, the Pyth update fee. It must fall back to one transaction per account, which reduces liquidation throughput exactly when many accounts become liquidatable at once.
- The owner can raise `minEngineGas` (up to 5,000,000, `OrderRouterAdmin.sol:27`), but that inflates every Router budget to cover a keeper-specific cost.

## Recommendation

Budget the keeper-side work explicitly, for example add a fixed keeper-sync allowance to `itemGas` when `engine.positions(msg.sender)` is non-zero, or credit the keeper bounty without the keeper curve sync and carry checkpoint and settle the keeper's carry on its next own interaction. At minimum, document in `SECURITY.md` and the keeper guide that liquidator addresses must not hold positions.

## References

- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:114-116` (budget constants), `761-799` (`executeLiquidationBatch`)
- `packages/perps/src/router/OrderRouterBase.sol:76` (`minEngineGas = 600_000`); `packages/perps/src/OrderRouterAdmin.sol:25-27` (bounds)
- `packages/perps/src/CfdEngine.sol:1324-1340` (`_applyLiquidation`, keeper sync)
- `packages/perps/SECURITY.md:1259-1261`; Cecuro LOW-52 (acknowledged)

## Proof of Concept

## Proof of Concept

Runnable code is not required for Insights; a runnable test is included so that every number below can be reproduced.

### Conditions under which this insight is valuable

- Any liquidator address that also holds a position (trading, hedging or inventory, including a protocol-operated keeper) and uses `executeLiquidationBatch`. With release parameters every batch item sent by such an address needs about 912k gas against an 850k budget, so the batch stops at index 0 on every call.
- The value is liquidation throughput and keeper cost during volatile periods: the batch path is the efficient way to clear many liquidatable accounts, and it silently becomes unusable for these keepers (the call does not revert, it returns 0).

### How to run

No RPC, fork or environment variables are needed; the test runs in the repository's own Foundry suite at the commit in scope.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# copy the two files below into packages/perps/test/perps/poc/
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/perps/poc/KeeperPositionBatchInsight.t.sol' --isolate -vv
# per-item gas: run the same command with -vvvv and look at the executeLiquidationBatchItem frames
```

Tested with Foundry `forge 1.9.0` (solc 0.8.35 is selected by `foundry.toml`); the first via-IR compile takes roughly 10 minutes. `--isolate` runs every top-level call as its own transaction with cold storage, as for a real keeper transaction (Foundry 1.9 enables it by default; the flag makes the result independent of local configuration). The test uses the repository's `BasePerpTest` fixture (real `CfdEngine`, `MarginClearinghouse`, `HousePool`, `TrancheVault`s, `PletherOracle`, `OrderRouter`; only Pyth and USDG are mocked, so the Pyth update fee is zero here), release risk parameters, and production Router entrypoints only (`commitOrder`, `executeOrder`, `updateMarkPrice`, `executeLiquidationBatch`, `executeLiquidation`). The fixture deploys the router as `LegacyOrderRouterHarness`, a repository test utility that inherits the production `OrderRouter` without overriding any function, which is why that name appears in the trace. No in-scope contract is modified.

### What the tests show

- `test_insight_batchLiquidatesWhenKeeperHasNoPosition` — control: a $100k LONG with 450 USDG margin becomes liquidatable after one day of carry and a small adverse move; a keeper address without a position calls `executeLiquidationBatch([victim])`; the item uses 591,188 gas of its 850,000 budget and the victim is liquidated (`nextIndex = 1`).
- `test_insight_batchStopsAtIndexZeroWhenKeeperHasPosition` — identical victim, price and timing, but the keeper address first opened a $50k SHORT. The same batch call runs out of gas inside the item after 835,228 gas, emits `LiquidationBatchStopped(0)`, returns 0 and liquidates nobody, while the transaction still consumes 1,114,532 gas. The same keeper then liquidates the victim with single-account `executeLiquidation`, whose uncapped item needs 912,362 gas.

### Test output (`--isolate -vv`)

```
[PASS] test_insight_batchLiquidatesWhenKeeperHasNoPosition() (gas: 7979800)
Logs:
  STEP 1: victim opens a $100k LONG with 450 USDG margin; 1 day of carry; price moves just past liquidation
    liquidation price (1e8): 100169179
    victim pendingOrderCounts: 0
    item gas budget (minEngineGas + 250,000): 850000
    keeper position size: 0
  STEP 2: keeper calls executeLiquidationBatch([victim])
    returned nextIndex: 1
    LiquidationBatchStopped emitted: false
    victim liquidated: true
    batch transaction gas used: 868603
[PASS] test_insight_batchStopsAtIndexZeroWhenKeeperHasPosition() (gas: 12870805)
Logs:
  STEP 1: same victim and price; the keeper address also holds a $50k SHORT
    liquidation price (1e8): 100169179
    victim pendingOrderCounts: 0
    item gas budget (minEngineGas + 250,000): 850000
    keeper position size: 50000000000000000000000
  STEP 2: keeper calls executeLiquidationBatch([victim])
    returned nextIndex: 0
    LiquidationBatchStopped emitted: true
    victim liquidated: false
    batch transaction gas used: 1114532
  STEP 3 (control): the same keeper uses single-account executeLiquidation (no item gas cap)
    victim position size after single-path liquidation: 0
Suite result: ok. 2 passed; 0 failed; 0 skipped;
```

### Trace excerpt (`--isolate -vvvv`, arguments elided)

```
    ├─ [868603] LegacyOrderRouterHarness::executeLiquidationBatch(...)
    │   │   ├─ [591188] LegacyOrderRouterHarness::executeLiquidationBatchItem(...)
    │   │   ├─ emit LiquidationBatchItem(index: 0, account: victim: [0x131f15F1fD1024551542390614B6c7e210A911AF], result: 0, keeperBountyAssets: 50000000 [5e7], errorSelector: 0x00000000)
    ├─ [1114532] LegacyOrderRouterHarness::executeLiquidationBatch(...)
    │   │   ├─ [835228] LegacyOrderRouterHarness::executeLiquidationBatchItem(...)
    │   │   │   │   │   └─ ← [OutOfGas] EvmError: OutOfGas
    │   │   ├─ emit LiquidationBatchStopped(nextIndex: 0)
    ├─ [1154850] LegacyOrderRouterHarness::executeLiquidation(...)
    │   │   ├─ [912362] LegacyOrderRouterHarness::executeLiquidationBatchItem(...)
```

### `packages/perps/test/perps/poc/PocBase.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @notice Shared PoC base for the Immunefi Plether audit competition.
/// @dev Built on the repository's own `BasePerpTest` fixture (real CfdEngine, MarginClearinghouse, HousePool,
///      TrancheVaults, PletherOracle, OrderRouter stack). Only out-of-scope components are mocked: Pyth (`MockPyth`)
///      and USDG (`MockSettlementToken`). The fixture deploys the router as `LegacyOrderRouterHarness`, a test utility
///      that inherits the production `OrderRouter` unchanged; every call in these PoCs uses production selectors
///      (`commitOrder(OrderRequest)`, `executeOrder`, `executeLiquidation`, `updateMarkPrice`).
///      Risk parameters are the release values from packages/perps/DEPLOYMENT.md.
abstract contract PocBase is BasePerpTest {

    address internal attacker = makeAddr("attacker");
    address internal victim = makeAddr("victim");
    address internal keeper = makeAddr("keeper");

    uint256 private _pocClientOrderNonce;

    /// @dev Release parameters (DEPLOYMENT.md "Initial release parameters").
    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    /// @dev Commits through the production `OrderRouter.commitOrder(OrderRequest)` with a market target price.
    function _commit(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        // Market order: closes of LONG and opens of SHORT accept any price at or below the target, so use the maximum;
        // the other two cases accept any price at or above the target, so use 1 (OrderValidationLib.checkSlippage).
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        orderId = _commitWithTarget(trader, side, size, margin, target, isClose);
    }

    /// @dev Commits through the production `OrderRouter.commitOrder(OrderRequest)` with an explicit target price.
    function _commitWithTarget(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _orderRequest(side, size, margin, targetPrice, isClose);
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    /// @dev A fully bounded V3 request with permissive financial limits, as an integrator would build it.
    function _orderRequest(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("plether-poc", ++_pocClientOrderNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
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

    /// @dev A keeper executes `orderId` through the production Router with the first post-commit signed Pyth tick.
    function _execute(
        uint64 orderId,
        uint256 price
    ) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(keeper);
        router.executeOrder(orderId, updateData);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

}

/// @notice Sanity check that the base drives a full open/close through the production Router.
contract PocSanityTest is PocBase {

    function test_poc_sanity_open_close_via_router() public {
        _fundTrader(attacker, 10_000e6);

        uint64 openId = _commit(attacker, CfdTypes.Side.LONG, 100_000e18, 1_000e6, false);
        _execute(openId, 1e8);
        assertEq(_positionSize(attacker), 100_000e18, "open should fill");

        uint64 closeId = _commit(attacker, CfdTypes.Side.LONG, 100_000e18, 0, true);
        _execute(closeId, 1e8);
        assertEq(_positionSize(attacker), 0, "close should fill");
    }

}
```

### `packages/perps/test/perps/poc/KeeperPositionBatchInsight.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocBase} from "./PocBase.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice Insight PoC: a keeper address that holds an open position cannot use `executeLiquidationBatch`.
/// @dev Each batch item is called with a fixed budget `minEngineGas (600k) + 250k + 600k * pendingOrderCounts(account)`
///      (OrderLiquidationBatchLogic.sol:765-766, 777) that depends only on the liquidated account. When the caller holds
///      a position, CfdEngine._applyLiquidation also syncs the keeper's terminal curve and checkpoints its carry before
///      crediting the bounty (CfdEngine.sol:1331-1338). That extra work is not budgeted, the item runs out of gas, and
///      the batch emits LiquidationBatchStopped(0) and returns 0 without liquidating anyone. The single-account path
///      has no item cap and succeeds. Run with `--isolate` so every top-level call is its own transaction (cold
///      storage, as for a real keeper); Foundry 1.9 enables isolation by default.
contract KeeperPositionBatchInsight is PocBase {

    uint256 internal constant MARK = 1e8;
    uint256 internal constant SIZE = 100_000e18; // victim: $100k LONG
    uint256 internal constant MARGIN = 450e6;
    bytes32 internal constant STOPPED_TOPIC = keccak256("LiquidationBatchStopped(uint256)");

    function setUp() public override {
        super.setUp();
        _liveRefresh(MARK);
    }

    /// Control: the same victim, the same price, a keeper address without a position. The batch liquidates.
    function test_insight_batchLiquidatesWhenKeeperHasNoPosition() public {
        emit log("STEP 1: victim opens a $100k LONG with 450 USDG margin; 1 day of carry; price moves just past liquidation");
        uint256 price = _prepare(false);
        emit log_named_uint("  keeper position size", _positionSize(keeper));

        emit log("STEP 2: keeper calls executeLiquidationBatch([victim])");
        (uint256 nextIndex, bool stopped, uint256 txGas) = _runBatch(price);
        _logBatch(nextIndex, stopped, txGas);

        assertEq(nextIndex, 1, "batch processed the account");
        assertFalse(stopped, "no LiquidationBatchStopped");
        assertEq(_positionSize(victim), 0, "victim liquidated by the batch");
    }

    /// The same scenario after the keeper address opened a position of its own: every batch stops at index 0.
    function test_insight_batchStopsAtIndexZeroWhenKeeperHasPosition() public {
        emit log("STEP 1: same victim and price; the keeper address also holds a $50k SHORT");
        uint256 price = _prepare(true);
        emit log_named_uint("  keeper position size", _positionSize(keeper));

        emit log("STEP 2: keeper calls executeLiquidationBatch([victim])");
        (uint256 nextIndex, bool stopped, uint256 txGas) = _runBatch(price);
        _logBatch(nextIndex, stopped, txGas);

        assertEq(nextIndex, 0, "batch stopped at index 0");
        assertTrue(stopped, "LiquidationBatchStopped(0) emitted");
        assertEq(_positionSize(victim), SIZE, "victim was NOT liquidated");

        emit log("STEP 3 (control): the same keeper uses single-account executeLiquidation (no item gas cap)");
        bytes[] memory data = _liqData(price);
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(keeper, fee + 1 ether);
        vm.prank(keeper);
        IPerpsKeeper(address(router)).executeLiquidation{value: fee}(victim, data);
        emit log_named_uint("  victim position size after single-path liquidation", _positionSize(victim));
        assertEq(_positionSize(victim), 0, "single path liquidates");
    }

    // ---------------------------------------------------------------- scenario

    function _prepare(
        bool keeperHasPosition
    ) internal returns (uint256 liquidationPrice) {
        _fundTrader(victim, 600e6);
        uint64 openId = _commit(victim, CfdTypes.Side.LONG, SIZE, MARGIN, false);
        _execute(openId, MARK);
        assertEq(_positionSize(victim), SIZE, "victim open filled");

        if (keeperHasPosition) {
            _fundTrader(keeper, 5_000e6);
            uint64 keeperOpen = _commit(keeper, CfdTypes.Side.SHORT, 50_000e18, 2_000e6, false);
            _execute(keeperOpen, MARK);
            assertEq(_positionSize(keeper), 50_000e18, "keeper open filled");
        }

        vm.warp(vm.getBlockTimestamp() + 1 days);
        vm.roll(vm.getBlockNumber() + 1);
        liquidationPrice = _minimalLiquidationPrice() + 10_000;
        assertTrue(engineLens.isLiquidatableAt(victim, liquidationPrice, pool.totalAssets()), "victim liquidatable");
        emit log_named_uint("  liquidation price (1e8)", liquidationPrice);
        emit log_named_uint("  victim pendingOrderCounts", router.pendingOrderCounts(victim));
        emit log_named_uint("  item gas budget (minEngineGas + 250,000)", router.minEngineGas() + 250_000);
    }

    function _runBatch(
        uint256 price
    ) internal returns (uint256 nextIndex, bool stopped, uint256 txGas) {
        address[] memory accounts = new address[](1);
        accounts[0] = victim;
        bytes[] memory data = _liqData(price);
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(keeper, fee + 1 ether);

        vm.recordLogs();
        vm.prank(keeper);
        nextIndex = IPerpsKeeper(address(router)).executeLiquidationBatch{value: fee}(accounts, data);
        txGas = vm.lastCallGas().gasTotalUsed;
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].emitter == address(router) && logs[i].topics.length > 0 && logs[i].topics[0] == STOPPED_TOPIC) {
                stopped = true;
            }
        }
    }

    function _logBatch(
        uint256 nextIndex,
        bool stopped,
        uint256 txGas
    ) internal {
        emit log_named_uint("  returned nextIndex", nextIndex);
        emit log_named_string("  LiquidationBatchStopped emitted", stopped ? "true" : "false");
        emit log_named_string("  victim liquidated", _positionSize(victim) == 0 ? "true" : "false");
        emit log_named_uint("  batch transaction gas used", txGas);
    }

    // ---------------------------------------------------------------- helpers

    /// @dev Smallest LONG-adverse price (>= MARK) at which the victim is liquidatable, by bisection on the lens.
    function _minimalLiquidationPrice() internal view returns (uint256) {
        uint256 lo = MARK;
        uint256 hi = MARK * 102 / 100;
        uint256 depth = pool.totalAssets();
        require(engineLens.isLiquidatableAt(victim, hi, depth), "not liquidatable even at +2%");
        while (hi - lo > 1_000) {
            uint256 mid = (lo + hi) / 2;
            if (engineLens.isLiquidatableAt(victim, mid, depth)) {
                hi = mid;
            } else {
                lo = mid;
            }
        }
        return hi;
    }

    /// @dev Live liquidation payload (same shape as test/perps/shared/LiquidationBatchFixture.sol).
    function _liqData(
        uint256 price
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 nowTs = vm.getBlockTimestamp();
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, nowTs, 0));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    /// @dev Live signed basket refresh through the Router.
    function _liveRefresh(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, nowTs, previous));
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        router.updateMarkPrice(updateData);
    }

}
```
