# \#99646 [SC-Low] Liquidation batches repeatedly stop at index zero when the keeper holds a position because the per-item gas budget omits keeper-side accounting

**Submitted on Nov 14th 2026 at 16:26:08 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99646
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief / Introduction

I found that the gas allowance used by `executeLiquidationBatch()` does not cover all the work that a liquidation may perform when the calling keeper also has an open trading position. The Router builds its per-item budget using the configured Engine gas minimum, a fixed Router allowance, and the pending-order count of the **account being liquidated**. It does not account for the additional carry realization and terminal-NAV synchronization that the Engine performs on the **keeper's own account** before paying the liquidation bounty.

This makes the batch outcome depend on whether the keeper holds a position, even where the liquidation candidates and price data are identical. With the release test configuration and two eligible LONG accounts, a keeper with an open position causes the first item to run out of its allotted gas. The Router catches the empty revert and returns `nextIndex = 0` without processing either account. Retrying the returned cursor repeats the failure. The same keeper can liquidate those accounts individually, and a keeper without an open position can batch-liquidate both.

I reproduced this using the production liquidation entrypoints on the audited commit. The original three-test PoC passed, a separate three-test control **without any manual `vm.cool()` calls** passed, and a two-test gas-budget control established causality: at `minEngineGas = 600_000` the batch stops at zero, while at `1_200_000` it processes both items. All eight tests passed with no changes to the production contracts.

## Vulnerability Details

### 1. The per-account batch allowance ignores the keeper's position state

`OrderLiquidationBatchLogic.executeLiquidationBatch()` dispatches each candidate using an external Router self-call with a fixed gas allowance:

```solidity
uint256 engineGas = host.minEngineGas();

while (nextIndex < accountCount) {
    address account = accounts[nextIndex];
    uint256 itemGas = engineGas + LIQUIDATION_BATCH_ROUTER_GAS
        + (host.pendingOrderCounts(account) * LIQUIDATION_BATCH_GAS_PER_ORDER);
    // ...
    try host.executeLiquidationBatchItem{gas: itemGas}(
        account,
        snapshot.longPrice,
        snapshot.shortPrice,
        snapshot.markPrice,
        snapshot.publishTime,
        msg.sender,
        riskOffCutoff
    ) returns (uint256 outcome) {
        // ...
    } catch (bytes memory revertData) {
        if (revertData.length == 0) {
            emit LiquidationBatchStopped(nextIndex);
            return nextIndex;
        }
        // ...
    }
}
```

**Source:** `packages/perps/src/router/OrderLiquidationBatchLogic.sol:761–803`.

With the audited release test defaults, the first victim has no pending orders and the item allowance is:

```text
minEngineGas                               600,000
LIQUIDATION_BATCH_ROUTER_GAS               250,000
pendingOrderCounts(victim) × 600,000             0
-------------------------------------------------
Requested gas for the item self-call      850,000
```

The EIP-150 forwarding allowance and retained Router tail at lines 768–775 protect the outer call from running short of gas. They do not increase the work budget inside `executeLiquidationBatchItem()`. A state-dependent Engine operation can therefore exhaust that item budget even while the outer transaction has ample gas remaining.

### 2. Keeper-side bounty accounting adds work inside the capped item

In `CfdEngine._applyLiquidation()`, the Engine handles both the liquidated account and the caller receiving the keeper bounty:

```solidity
bytes32 expectedOldHash = _beginTerminalCurveMutation(delta.account);
_realizeCarryFromSettlement(delta.account, _positions[delta.account]);
bool syncKeeper = delta.keeperBountyAssets > 0
    && keeper != delta.account && _positions[keeper].lots > 0;
bytes32 keeperExpectedOldHash;
if (syncKeeper) {
    keeperExpectedOldHash = _beginTerminalCurveMutation(keeper);
}
if (delta.keeperBountyAssets > 0 && keeper != delta.account) {
    _checkpointBountyRecipient(keeper, delta.price, publishTime);
}
keeperBountyAssets = settlementSidecar.executeLiquidation(delta, publishTime, keeper);
// ...
_endTerminalCurveMutation(delta.account, expectedOldHash);
if (syncKeeper) {
    _endTerminalCurveMutation(keeper, keeperExpectedOldHash);
}
```

**Source:** `packages/perps/src/CfdEngine.sol:1324–1348`.

For a flat keeper, the Engine does not enter the keeper-position branch. For a keeper with an open position, it authenticates the existing terminal curve, checkpoints the keeper as a bounty recipient, potentially realizes indexed carry through settlement-account and pool mutations, and synchronizes the keeper's terminal NAV state. The additional path includes `CfdEngine._checkpointBountyRecipient()` (lines 203–216) and `_realizeCarryFromSettlement()` (lines 1407–1449).

The victim's pending-order count is unchanged by any of this. As a result, the Router gives both keepers the same nominal per-item gas allowance despite the different Engine execution costs.

### 3. An empty item revert stops the entire batch without reaching the second account

The Router explicitly distinguishes an empty revert from other errors:

```solidity
} catch (bytes memory revertData) {
    if (revertData.length == 0) {
        emit LiquidationBatchStopped(nextIndex);
        return nextIndex;
    }
    bytes4 selector = revertData.length >= 4 ? bytes4(revertData) : bytes4(0);
    emit LiquidationBatchItem(nextIndex, account, _liquidationBatchResult(selector), 0, selector);
}
```

An out-of-gas revert in the capped self-call normally produces no revert payload. The Router correctly preserves state isolation, but the current gas budget makes the same first candidate repeatedly fail for the same active keeper. `nextIndex` remains zero, so following the documented resume procedure simply repeats the same two-account batch and makes no progress.

The batch transaction itself does not revert: it returns successfully with `LiquidationBatchStopped(0)`. This distinction matters to keepers that rely on the returned cursor to continue processing a liquidation list.

### 4. Matched controls isolate the keeper-side gas-budget failure

I used two liquidatable LONG accounts, `V1` and `V2`, and compared three execution paths:

| Execution path | Result |
|---|---|
| Keeper with an open position: `executeLiquidationBatch([V1, V2])` | Returns `nextIndex = 0` and processes no accounts, three times |
| Same keeper: `executeLiquidation(V1)` followed by `executeLiquidation(V2)` | Both positions are liquidated |
| Separate keeper without an open position: `executeLiquidationBatch([V1, V2])` | Returns `nextIndex = 2`; both positions are liquidated |

I also removed the original PoC's manual storage-cooling calls and ran the same three tests with Foundry isolation. The active-keeper batch still stopped at index zero while both controls succeeded. This rules out `vm.cool()` as a necessary cause of the observed failure.

Finally, I changed only the test fixture's configured `minEngineGas` and held the position and victim setup constant:

| `minEngineGas` | `nextIndex` | Stopped | Items completed |
|---:|---:|---|---:|
| `600,000` | `0` | `true` | `0` |
| `1,200,000` | `2` | `false` | `2` |

This control identifies the per-item gas allowance as the reason the first liquidation does not finish. The higher value is a **diagnostic test configuration**, not a privileged step needed by the keeper to trigger the defect. The defective path uses the ordinary release budget.

## Impact Details

The failure degrades the permissionless liquidation-batch mechanism for a legitimate keeper that also trades on Plether. The two victims are eligible for liquidation, their positions remain open after each stopped batch, and every retry consumes transaction gas without finalizing an item or advancing the cursor.

A keeper or market maker that maintains its own open CFD position can therefore lose access to batching exactly when it needs to process multiple liquidations. In a time-sensitive liquidation period, repeated zero-progress calls consume keeper resources and reduce the effective throughput available from that keeper. A client that simply follows `nextIndex` cannot recover by retrying; it must switch to a different execution path or use an account without an open position.

I demonstrated the effect using ordinary keeper and trader accounts, rather than privileged Router access or state manipulation. The PoC shows a reproducible service failure of the **batch entrypoint**, not a failure of liquidation eligibility or price validation. It also shows that single-account liquidation and batching by a flat keeper remain operational. I am reporting the actual keeper-specific batch liveness and gas-waste impact, not claiming that every keeper is prevented from liquidating positions or that trader funds are frozen protocol-wide.

I selected **Medium — Griefing**, because the result is repeatable failed work and wasted keeper execution resources in a permissionless risk-management path, rather than theft of trader principal or a permanent freeze.

## Suggested Remediation

The per-item execution allowance should cover both the liquidated account and the additional bounty-recipient work when the keeper has an open position. The immediate correction can add a measured allowance for keeper-side carry checkpointing, settlement, and terminal-NAV synchronization, while retaining the existing EIP-150 allowance and outer gas tail.

A more robust design is to isolate or amortize keeper bounty-recipient accounting outside the tightly gas-capped candidate frame, without weakening atomic liquidation accounting. The batch should also expose a clearly actionable failure status so an operator does not repeatedly submit the same cursor under an unchanged, insufficient budget.

I would add regression tests using an active-position keeper and a flat keeper, run them with per-call gas isolation, and exercise both the normal release budget and an increased budget. The eight-test reproduction below gives a direct basis for those regressions.

## References

- [OrderLiquidationBatchLogic — fixed item budget and empty-revert handling](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L761-L809)
- [CfdEngine — keeper checkpoint and carry realization](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L203-L216)
- [CfdEngine — liquidation and bounty recipient mutation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1324-L1348)
- [CfdEngine — carry realization and pool inflow](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1407-L1449)
- [Plether audit competition scope](https://immunefi.com/audit-competition/plether/scope/)


## Proof of Concept

## Proof of Concept


I ran the tests against the frozen audited commit using the existing `BasePerpTest` fixture and real Router/Engine liquidation logic. Only the external Pyth price feed and test USDG infrastructure are mocked through the repository fixture. I did not alter any production contract under `packages/perps/src`.

**Repository:** `https://github.com/immunefi-team/audit-comp-plether`  
**Commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`  
**Original test file:** `packages/perps/test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall.t.sol`  
**No-cooling control:** `packages/perps/test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall_NoCool.t.sol`  
**Gas-budget control:** `packages/perps/test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall_BudgetControl.t.sol`  
**Original PoC SHA-256:** `c4db0c1ae94dd85467356d7294f63bd687927339e5e04912091d372299270246`

## Reproduction Steps

1. Deploy the normal perps test fixture, fund a maker and two liquidation candidates, and open a sufficiently collateralized SHORT for the maker.
2. Open `100,000e18` LONG positions for `V1` and `V2` and a healthy `10,000e18` LONG position for the keeper.
3. Advance approximately one hour and move the mocked basket price to `1.004e8`, making the two victim accounts liquidatable.
4. Have the position-holding keeper call `executeLiquidationBatch([V1, V2])` three times, resubmitting the same cursor. Confirm `nextIndex = 0`, `LiquidationBatchStopped(0)`, zero `LiquidationBatchItem` events, and both victim positions still open.
5. In separate matched tests, have the same keeper liquidate each victim individually and have a flat keeper batch both victims. Confirm those operations succeed.
6. Repeat the three tests with the manual `vm.cool()` invocations removed; confirm that the batch stall remains.
7. Finally, compare the release `minEngineGas` of `600,000` with a test-only `1,200,000` setting. Confirm that only the higher-budget version processes both accounts.

## Commands

Run from the repository root:

```bash
cd /workspaces/audit-comp-plether
export PATH="$HOME/.foundry/bin:$PATH"

git rev-parse HEAD
# Expected: a82d4fefb02768e5b3bddcc97dee371eee24fea8

sha256sum packages/perps/test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall.t.sol
# Expected: c4db0c1ae94dd85467356d7294f63bd687927339e5e04912091d372299270246

forge test --root packages/perps --offline --isolate \
  --match-path 'test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall.t.sol' -vv

forge test --root packages/perps --offline --isolate \
  --match-path 'test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall_NoCool.t.sol' -vv

forge test --root packages/perps --offline --isolate \
  --match-path 'test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall_BudgetControl.t.sol' \
  --match-test '^test_budgetControl_' -vv

git diff HEAD -- packages/perps/src
# Expected: no production contract diff
```

## Observed Console Output — Original PoC

```text
Ran 3 tests for test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall.t.sol:Chase_wholerevert_KeeperPositionLiqBatchStall
[PASS] test_batchStallsAtIndexZeroForPositionHoldingKeeper() (gas: 26097282)
Logs:
  attempt / nextIndex / items 0 0 0
  attempt / nextIndex / items 1 0 0
  attempt / nextIndex / items 2 0 0

[PASS] test_control_flatKeeperBatchSucceeds() (gas: 22674905)
Logs:
  flat keeper nextIndex / items 2 2

[PASS] test_control_singlesSucceedForSameKeeper() (gas: 23402547)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 153.05ms (209.87ms CPU time)

Ran 1 test suite in 183.50ms (153.05ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

## Observed Console Output — No-`vm.cool()` Control

```text
Ran 3 tests for test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall_NoCool.t.sol:Chase_wholerevert_KeeperPositionLiqBatchStall_NoCool
[PASS] test_batchStallsAtIndexZeroForPositionHoldingKeeper() (gas: 25907095)
Logs:
  attempt / nextIndex / items 0 0 0
  attempt / nextIndex / items 1 0 0
  attempt / nextIndex / items 2 0 0

[PASS] test_control_flatKeeperBatchSucceeds() (gas: 22610789)
Logs:
  flat keeper nextIndex / items 2 2

[PASS] test_control_singlesSucceedForSameKeeper() (gas: 23267478)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 180.31ms (273.80ms CPU time)

Ran 1 test suite in 191.69ms (180.31ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

## Observed Console Output — Matched Gas-Budget Control

```text
Ran 2 tests for test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall_BudgetControl.t.sol:Chase_wholerevert_KeeperPositionLiqBatchStall_BudgetControl
[PASS] test_budgetControl_Higher1200kProcessesBoth() (gas: 23422658)
Logs:
  1200k budget: next / stopped / items 2 false 2

[PASS] test_budgetControl_Original600kStillStalls() (gas: 23457150)
Logs:
  600k budget: next / stopped / items 0 true 0

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 132.66ms (206.04ms CPU time)

Ran 1 test suite in 139.73ms (132.66ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

## Complete Solidity PoC Source — Original 221-Line Test

The following is the original PoC used for the first three passing tests, reproduced byte-for-byte.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Lead "wholerevert" (executeLiquidationBatch). The fixed per-item budget
///         `minEngineGas (600k, release default) + LIQUIDATION_BATCH_ROUTER_GAS (250k) + 600k * pendingOrders`
///         (OrderLiquidationBatchLogic.sol:763-764) does not cover the keeper-side work done inside every item:
///         `engine.liquidatePosition(..., keeper)` credits the bounty to msg.sender and, when the keeper holds an open
///         position, `_checkpointBountyRecipient` realizes the keeper's carry and re-syncs its terminal NAV curve
///         (CfdEngine.sol:1336-1337, 203-216, 1407-1449). With realistic (cold, per-transaction) storage access the first item
///         then needs ~872k gas, runs out of gas, empty-reverts, and the batch returns `nextIndex = 0` with
///         LiquidationBatchStopped(0) (OrderLiquidationBatchLogic.sol:791-794). Every retry from `nextIndex` stalls
///         identically. The same keeper liquidates the same accounts one by one with executeLiquidation.
///         Run with `--isolate` (real-transaction cold/warm accounting); the test also cools protocol storage so the
///         default runner reproduces it.
contract Chase_wholerevert_KeeperPositionLiqBatchStall is BasePerpTest {

    address internal constant KEEPER = address(0xCAFE); // holds a small, healthy position
    address internal constant KEEPER2 = address(0xCAFF); // flat keeper (control)
    address internal constant EXEC = address(0xE1EC);
    address internal constant MAKER = address(0x3A3E);
    address internal constant V1 = address(0x1001);
    address internal constant V2 = address(0x1002);

    uint256 internal constant LIQ_PRICE = 1.004e8;

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

    function _priceData(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _openNow(
        address a,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        {
            bytes[] memory u = _priceData(price);
            vm.prank(EXEC);
            router.updateMarkPrice(u);
        }
        uint64 id;
        for (uint256 k; k < 60; k++) {
            vm.prank(a);
            try router.commitOrder(side, size, margin, 0, false) returns (uint64 got) {
                id = got;
                break;
            } catch {
                margin = margin * 105 / 100;
            }
        }
        require(id != 0, "commit never ok");
        bytes[] memory upd = _mockPythUpdateData(price);
        vm.prank(EXEC);
        router.executeOrder(id, upd);
        require(_size(a) != 0, "open failed");
    }

    function _size(
        address a
    ) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _setup() internal {
        _fundTrader(MAKER, 50_000e6);
        _fundTrader(V1, 5_000e6);
        _fundTrader(V2, 5_000e6);
        _fundTrader(KEEPER, 5_000e6);
        _openNow(MAKER, CfdTypes.Side.SHORT, 300_000e18, 30_000e6, 1e8);
        _openNow(V1, CfdTypes.Side.LONG, 100_000e18, 210e6, 1e8);
        _openNow(V2, CfdTypes.Side.LONG, 100_000e18, 210e6, 1e8);
        // The keeper also trades: one small, well-collateralized position.
        _openNow(KEEPER, CfdTypes.Side.LONG, 10_000e18, 500e6, 1e8);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.roll(vm.getBlockNumber() + 100);
    }

    /// @dev Models the start of a fresh transaction for the default (non-isolated) runner.
    function _coolProtocol() internal {
        address[20] memory t = [
            address(engine),
            address(terminalNavBook),
            address(pool),
            address(clearinghouse),
            address(router),
            address(pletherOracle),
            address(baseMockPyth),
            address(settlementToken),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(routerAdmin),
            address(engineAdmin),
            address(seniorVault),
            address(juniorVault),
            address(engine.planningReader()),
            address(engine.settlementSidecar()),
            address(router.lifecycleBook()),
            address(router.positionProtectionBook()),
            address(router.liquidationBatchSidecar()),
            address(engineLens)
        ];
        for (uint256 i; i < t.length; i++) {
            vm.cool(t[i]);
        }
    }

    function _batch(
        address keeper,
        address[] memory accounts
    ) internal returns (uint256 next, bool stopped, uint256 items) {
        bytes[] memory upd = _priceData(LIQ_PRICE);
        _coolProtocol();
        vm.recordLogs();
        vm.prank(keeper);
        next = IPerpsKeeper(address(router)).executeLiquidationBatch{gas: 25_000_000}(accounts, upd);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 itemTopic = keccak256("LiquidationBatchItem(uint256,address,uint8,uint256,bytes4)");
        bytes32 stopTopic = keccak256("LiquidationBatchStopped(uint256)");
        for (uint256 i; i < logs.length; i++) {
            if (logs[i].topics[0] == itemTopic) {
                ++items;
            }
            if (logs[i].topics[0] == stopTopic) {
                stopped = true;
            }
        }
    }

    function _accounts() internal pure returns (address[] memory a) {
        a = new address[](2);
        a[0] = V1;
        a[1] = V2;
    }

    /// @dev True when the runner charges every top-level call as its own transaction (forge test --isolate).
    function _isolated() internal view returns (bool) {
        engine.lastMarkPrice();
        uint256 g0 = gasleft();
        engine.lastMarkPrice();
        return g0 - gasleft() > 2000;
    }

    /// Attack/defect: a keeper with an open position gets nextIndex = 0 on every batch; nobody is liquidated.
    function test_batchStallsAtIndexZeroForPositionHoldingKeeper() public {
        _setup();
        if (!_isolated()) {
            console2.log("SKIP: run with --isolate for real per-transaction cold-storage gas");
            vm.skip(true);
        }
        assertTrue(engineLens.previewLiquidation(V1, LIQ_PRICE).liquidatable, "V1 liquidatable");
        assertTrue(engineLens.previewLiquidation(V2, LIQ_PRICE).liquidatable, "V2 liquidatable");

        for (uint256 attempt; attempt < 3; attempt++) {
            (uint256 next, bool stopped, uint256 items) = _batch(KEEPER, _accounts());
            console2.log("attempt / nextIndex / items", attempt, next, items);
            assertEq(next, 0, "batch must stop before the first item");
            assertTrue(stopped, "LiquidationBatchStopped(0) emitted");
            assertEq(items, 0, "no item was attempted to completion");
            assertGt(_size(V1), 0, "V1 still open");
            assertGt(_size(V2), 0, "V2 still open");
        }
    }

    /// Control 1: the same keeper liquidates the same accounts individually in the same state.
    function test_control_singlesSucceedForSameKeeper() public {
        _setup();
        bytes[] memory upd = _priceData(LIQ_PRICE);
        _coolProtocol();
        vm.prank(KEEPER);
        router.executeLiquidation(V1, upd);
        upd = _priceData(LIQ_PRICE);
        _coolProtocol();
        vm.prank(KEEPER);
        router.executeLiquidation(V2, upd);
        assertEq(_size(V1), 0, "V1 liquidated singly");
        assertEq(_size(V2), 0, "V2 liquidated singly");
    }

    /// Control 2: a flat keeper's batch liquidates both accounts.
    function test_control_flatKeeperBatchSucceeds() public {
        _setup();
        (uint256 next, bool stopped, uint256 items) = _batch(KEEPER2, _accounts());
        console2.log("flat keeper nextIndex / items", next, items);
        assertEq(next, 2, "both attempted");
        assertFalse(stopped, "not stopped");
        assertEq(_size(V1), 0, "V1 liquidated");
        assertEq(_size(V2), 0, "V2 liquidated");
    }
}
```

## Complete No-Cooling and Budget-Control Reproduction

The following commands derive the two control test files from the original PoC. They change only test code and leave the original Solidity file intact.

### Control A — no manual storage cooling

```bash
cd /workspaces/audit-comp-plether
SRC="packages/perps/test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall.t.sol"
DST="packages/perps/test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall_NoCool.t.sol"

sed \
  -e 's/contract Chase_wholerevert_KeeperPositionLiqBatchStall is/contract Chase_wholerevert_KeeperPositionLiqBatchStall_NoCool is/' \
  -e '/^[[:space:]]*_coolProtocol();[[:space:]]*$/d' \
  "$SRC" > "$DST"

forge test --root packages/perps --offline --isolate \
  --match-path 'test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall_NoCool.t.sol' -vv
```

### Control B — baseline-versus-increased gas allowance

```bash
cd /workspaces/audit-comp-plether
python3 - <<'PYTHON'
from pathlib import Path

folder = Path('packages/perps/test/perps/insight')
src = folder / 'Chase_wholerevert_KeeperPositionLiqBatchStall_NoCool.t.sol'
dst = folder / 'Chase_wholerevert_KeeperPositionLiqBatchStall_BudgetControl.t.sol'
s = src.read_text()
old_contract = 'contract Chase_wholerevert_KeeperPositionLiqBatchStall_NoCool is BasePerpTest {'
new_contract = 'contract Chase_wholerevert_KeeperPositionLiqBatchStall_BudgetControl is BasePerpTest {'
assert s.count(old_contract) == 1
old_import = 'import {BasePerpTest} from "../BasePerpTest.sol";'
assert s.count(old_import) == 1
s = s.replace(old_import, old_import + '\nimport {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";', 1)
s = s.replace(old_contract, new_contract, 1)

new_tests = r'''
    function _budgetControlConfigure(uint256 budget) internal {
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.minEngineGas = budget;
        _setRouterConfig(cfg);
        assertEq(router.minEngineGas(), budget, "budget not installed");
    }

    function test_budgetControl_Original600kStillStalls() public {
        _budgetControlConfigure(600_000);
        _setup();
        assertTrue(_isolated(), "run with --isolate");
        (uint256 next, bool stopped, uint256 items) = _batch(KEEPER, _accounts());
        console2.log("600k budget: next / stopped / items", next, stopped, items);
        assertEq(next, 0, "expected stall at 600k");
        assertTrue(stopped, "expected stop event");
        assertEq(items, 0, "expected zero processed items");
    }

    function test_budgetControl_Higher1200kProcessesBoth() public {
        _budgetControlConfigure(1_200_000);
        _setup();
        assertTrue(_isolated(), "run with --isolate");
        (uint256 next, bool stopped, uint256 items) = _batch(KEEPER, _accounts());
        console2.log("1200k budget: next / stopped / items", next, stopped, items);
        assertEq(next, 2, "expected both items to complete");
        assertFalse(stopped, "batch must not stop");
        assertEq(items, 2, "expected two completed items");
        assertEq(_size(V1), 0, "V1 not liquidated");
        assertEq(_size(V2), 0, "V2 not liquidated");
    }
'''
assert s.rstrip().endswith('}')
s = s.rstrip()[:-1] + new_tests + '\n}\n'
assert 'function test_budgetControl_Higher1200kProcessesBoth' in s
dst.write_text(s)
print('Created', dst)
PYTHON

forge test --root packages/perps --offline --isolate \
  --match-path 'test/perps/insight/Chase_wholerevert_KeeperPositionLiqBatchStall_BudgetControl.t.sol' \
  --match-test '^test_budgetControl_' -vv

git diff HEAD -- packages/perps/src
```

## What My Tests Establish

These controls separate the candidate accounts' liquidation eligibility from the batch item's gas allowance. The same eligible victims can be processed individually or by a flat keeper, and an active-position keeper's batch succeeds after only increasing its per-item gas allowance. The original failure persists even without manual storage cooling. The issue is thus an underbudgeted, keeper-state-dependent execution path in the permissionless liquidation batch.

