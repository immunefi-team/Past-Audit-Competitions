# \#98170 [SC-Low] Commit-time open preflight ignores the account's own queued opens, causing false rejections of valid increases and false acceptances that forfeit the execution bounty

**Submitted on Nov 5th 2026 at 17:52:34 UTC by @d4rkp3n for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98170
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
When a trader commits an open or increase, the Router runs a planner preflight. It is meant to reject orders that are predictably invalid ("CommitTimeRejectable"). It already projects the account's position through its own earlier queued orders, but uses that projection only for the opposing-side check. Every other check (initial margin, skew, solvency) runs against the live engine state, so the account's own FIFO-ahead queued opens are ignored. Result:
- **False reject:** a valid increase is refused unless it is over-collateralised as a standalone position.
- **False accept:** an increase that will predictably fail once the queued order fills is accepted. It then fails terminally at execution, and the trader's reserved execution bounty goes to the executor.

The protocol therefore does not deliver its documented commit-time guarantee. Losses are capped at the opening bounty (≤ 0.20 USDG per order with release parameters), and no protocol value is lost.

## Vulnerability Details

### Background: relevant mechanics
- **Delayed orders.** Orders are committed, then executed later by any keeper in one global FIFO queue, with a Pyth price. Committed orders are binding and cannot be cancelled by the user. Up to `maxPendingOrders = 5` orders per account can be pending (DEPLOYMENT.md).
- **Bounty.** At commit, the Clearinghouse reserves an execution bounty from the trader's own settlement balance (`BountyKind.Order`):
  - Amount = `notional * openOrderExecutionBountyBps / 10_000`, clamped to `[minOpenOrderExecutionBountyAssets, maxOpenOrderExecutionBountyAssets]`.
  - Release values (OrderRouterBase constructor, not overridden in DEPLOYMENT.md) are 1 bps, 0.01 USDG and 0.20 USDG.
  - Per SECURITY.md (Queue failure handling), "ordinary open failure pays the executor from its reserved bounty".
- **Commit-time preflight** (`OrderLiquidationBatchLogic`, ~L421-445):

```solidity
// Project the live position through earlier account orders before checking the requested side.
QueuedPositionView memory queuedPosition = _queuedPosition(host, engine, account);
if (queuedPosition.exists && queuedPosition.side != side) revert OrderRouter__PredictableOpenInvalid(opposingCode);
uint8 revertCode =
    host.engineLensForCommit().previewOpenRevertCode(account, side, sizeDelta, marginDelta, commitPrice);
...
if (OrderFailurePolicyLib.isPredictablyInvalidOpen(failureCategory)) revert OrderRouter__PredictableOpenInvalid(revertCode);
```

- **Promised behaviour.** The code documentation states it:
  - `OpenFailurePolicyCategory.CommitTimeRejectable`: "Current inputs or state allow the router to reject the order at commit time."
  - `OrderRouter__PredictableOpenInvalid`: "Commit-time planner preflight identified a predictably invalid open/increase."
  - `CfdEnginePlanLib.getOpenFailurePolicyCategory` lists skew, initial-margin and solvency as commit-time rejectable.
- **Not a known issue.** Neither SECURITY.md (Known Limitations, Router and keeper limitations) nor DEPLOYMENT.md nor the program's known-issues list documents that this preflight ignores the account's own queued opens. This is a different root cause from Cecuro MED-16/LOW-57 (a stale cached price at commit).

### Root cause
`_queuedPosition` already computes the account's position as it will be once its earlier queued orders have executed. Only the side of that projection is used. `previewOpenRevertCode` reads the live position, margin and side OI, so any state-dependent CommitTimeRejectable code is evaluated against the wrong state whenever the account has same-side opens ahead in the FIFO queue. Those orders are binding and execute strictly first, so the state they create is known at commit time.

All three cases below are proven in the PoC (`terminalReason 7` = the open was invalidated at execution):

| # | Code | Direction | Setup | Result |
|---|---|---|---|---|
| 1 | INSUFFICIENT_INITIAL_MARGIN (6) | false reject | A = LONG 50,000 tokens / 20,000 USDG queued; B = +1,000 tokens | B rejected for 2, 5 and 10 USDG margin, with 20 USDG the minimum accepted. With A live, B is accepted with 2 USDG and executes (size 51,000). |
| 2 | SKEW_TOO_HIGH (4) | false accept | A takes skew to 90% of `maxSkewRatio`; B = +20% of the limit | B accepted. With A live, the same B is rejected with code 4 (control). Execution: A fills, B fails terminally (terminalReason 7), and B's bounty goes to the executor. |
| 3 | SOLVENCY_EXCEEDED (7) | false accept | Third parties: long = short = 90% of pool depth (skew ≈ 0); A and B = +7% long each | B accepted. With A live, B is rejected with code 7 (control). Execution: A fills, B fails terminally (terminalReason 7), and B's bounty goes to the executor. |

Other codes:
- LIQUIDATION_RESERVE_UNFUNDED (9) is not affected, because each open funds its own reserve.
- POSITION_TOO_SMALL is unreachable above the 1,000 USDG opening minimum.
- VPI_REBATE_RESERVE_UNFUNDED was not tested.

## Impact Details
- **Selected impact: "Contract fails to deliver promised returns, but doesn't lose value" (Low).** The documented commit-time rejection of predictably invalid opens and increases does not hold for accounts with queued same-side opens:
  - **False reject:** valid scale-in orders are blocked unless each increment carries standalone margin. In the PoC that is 20 USDG instead of 2 USDG, i.e. 10x, and the extra capital stays locked until the order executes.
  - **False accept:** orders are accepted that will predictably fail.
- **Precise economic effect:**
  - Each false accept costs the trader the reserved opening bounty, paid to whoever executes the order. That is `max(0.01, min(0.20, notional * 1bps))` USDG, i.e. at most 0.20 USDG per order. With 5 pending orders per account, at most 4 orders queued behind the first can be falsely accepted at once, so ≤ 0.80 USDG per account per queue cycle.
  - Margin is fully returned and no position is opened. No LP or protocol funds are affected, and there is no insolvency.
- **Who can trigger it:** only the account owner's own commits create the condition. A keeper cannot cause it, it only collects the bounty it would also collect on success. Severity is not claimed above Low; if the triager finds it fits better, Insight is acceptable.
- **Scope caveat:** global checks also cannot see *other* accounts' queued orders, which is inherent. The defect reported here is narrower: the Router already projects the account's own queue, but passes only the side, not the size and margin, into the preflight.

## Recommendation
- **Preferred fix:** evaluate the commit-time preflight against the account state projected through its earlier queued orders. That means the projected size and margin, plus the account's own contribution to side OI, skew and max-profit liability.
- **Conservative alternative:** when the account has queued same-side opens, enforce at commit only the codes that depend solely on the new order, and defer the state-dependent ones (initial margin, skew, solvency) to execution.
- Add a regression test equivalent to the PoC above.

## References
- Commit in scope: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol` (commit-time preflight, ~L421-445)
- `packages/perps/src/CfdEngineLens.sol` (`previewOpenRevertCode`)
- `CfdEnginePlanLib.getOpenFailurePolicyCategory`, `OpenFailurePolicyCategory.CommitTimeRejectable`, `OrderRouter__PredictableOpenInvalid` NatSpec
- `packages/perps/SECURITY.md` (Queue failure handling), `packages/perps/DEPLOYMENT.md` (release parameters)

## Proof of Concept

## Proof of Concept

### Step-by-step (case 1, false reject)
1. Trader commits order A: LONG 50,000 tokens with 20,000 USDG margin. A sits in the FIFO queue (binding, not cancellable).
2. Trader commits order B: LONG +1,000 tokens. The preflight evaluates B against the live (empty) position, so B is rejected with `PredictableOpenInvalid(6)` for 2, 5 and 10 USDG margin; 20 USDG is the minimum accepted.
3. A is executed by a keeper. The same B is now accepted with 2 USDG and executes (size 51,000), proving the rejected order was valid in FIFO order.

### Step-by-step (cases 2 and 3, false accept)
1. Trader commits A, which pushes skew to 90% of `maxSkewRatio` (case 2), or pushes long max-profit liability close to pool depth on a balanced book (case 3).
2. Trader commits B (+20% of the skew limit / +7% long). The preflight ignores A, so B is accepted.
3. Control: if A were already live, the same B is rejected at commit with code 4 / code 7.
4. A keeper executes A, then B. B fails terminally (`terminalReason 7`), no position is opened, margin is returned, and the trader's reserved execution bounty (up to 0.20 USDG) is paid to the executor.

### Run
```bash
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
cd packages/perps && mkdir -p test/poc/llm   # save the file below as test/poc/llm/QueuedCommitPoc.t.sol
forge test --match-contract QueuedCommitPoc -vvv
```

- **Expected result:** `3 passed; 0 failed`. Key logs:
  - `test_queuedOpenMarginIgnoredAtCommit`: `min margin queued: 20000000`, `min margin live: 2000000`, `size after B: 51000000000000000000000`.
  - `test_queuedOpenSkewFalseAccept`: B accepted; `control: rejected` with `0x2ae052ed…04` (`PredictableOpenInvalid(4)`); the `executeOrder(b)` result has status 3 and terminalReason 7. `keeper bounty gained from B: 400000` is cumulative, measured before A's execution: A's 0.20 USDG plus B's 0.20 USDG.
  - `test_queuedOpenSolvencyFalseAccept`: `control: rejected` with `PredictableOpenInvalid(7)`; B ends with terminalReason 7 and the position stays at A's size.
- **Verified:** PASS on a fresh clone at the commit above with pristine `src/`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;
import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "../../../src/CfdTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

contract QueuedCommitPoc is BasePerpTest {
    address eve = address(0xE7E);
    function _mark(uint256 p) internal {
        vm.prank(address(router));
        engine.updateMarkPrice(p, SafeCast.toUint64(vm.getBlockTimestamp()));
    }
    function extCommit(CfdTypes.Side s, uint256 sz, uint256 m) external returns (uint64) {
        vm.prank(eve); return router.commitOrder(s, sz, m, 0, false);
    }
    function _probe(string memory tag) internal returns (uint256 firstOk) {
        uint256[6] memory ms = [uint256(2e6), 5e6, 10e6, 20e6, 50e6, 100e6];
        uint256 snap = vm.snapshotState();
        firstOk = type(uint256).max;
        for (uint256 i; i < 6; ++i) {
            vm.revertToState(snap);
            try this.extCommit(CfdTypes.Side.LONG, 10e20, ms[i]) returns (uint64) { if (firstOk == type(uint256).max) firstOk = ms[i]; emit log_named_uint(string.concat(tag, " ok margin"), ms[i]); }
            catch (bytes memory e) { emit log_named_uint(string.concat(tag, " rej margin"), ms[i]); emit log_bytes(e); }
        }
        vm.revertToState(snap);
    }
    function test_queuedOpenMarginIgnoredAtCommit() public {
        _fundTrader(eve, 50_000e6);
        vm.warp(vm.getBlockTimestamp() + 10);
        _mark(1e8);
        vm.prank(eve); uint64 a = router.commitOrder(CfdTypes.Side.LONG, 500e20, 20_000e6, 0, false);
        uint256 q = _probe("queued");
        vm.warp(vm.getBlockTimestamp() + 2);
        router.executeOrder(a, _mockPythUpdateData(1e8));
        (uint256 s1, uint256 m1,,,,,) = engine.positions(eve);
        emit log_named_uint("size after A", s1); emit log_named_uint("margin after A", m1);
        vm.warp(vm.getBlockTimestamp() + 1); _mark(1e8);
        uint256 l = _probe("live");
        emit log_named_uint("min margin queued", q); emit log_named_uint("min margin live", l);
        // The $2-margin increase is genuinely executable once A is live
        uint64 b = this.extCommit(CfdTypes.Side.LONG, 10e20, 2e6);
        vm.warp(vm.getBlockTimestamp() + 2);
        emit log_bytes(abi.encode(router.executeOrder(b, _mockPythUpdateData(1e8))));
        (uint256 s2,,,,,,) = engine.positions(eve);
        emit log_named_uint("size after B", s2);
        assertEq(s2, 510e20, "B executed");
        assertGt(q, l, "commit-time false reject while A queued");
    }

    function _lots(uint256 notional6) internal pure returns (uint256) { return (notional6 / 100e6) * 1e20; }

    /// False ACCEPT: B passes the commit-time skew check against live skew, but A (queued, FIFO-ahead, same account)
    /// pushes skew so B fails at execution with SKEW_TOO_HIGH and its execution bounty is paid to the keeper.
    function test_queuedOpenSkewFalseAccept() public {
        uint256 depth = pool.totalAssets();
        uint256 aN = depth * 4 / 10 * 9 / 10; uint256 bN = depth * 4 / 10 * 2 / 10;
        _fundTrader(eve, aN + bN);
        vm.warp(vm.getBlockTimestamp() + 10); _mark(1e8);
        vm.prank(eve); uint64 a = router.commitOrder(CfdTypes.Side.LONG, _lots(aN), aN * 3 / 10, 0, false);
        // B alone would also be rejected if A were live; here it is accepted.
        vm.prank(eve); uint64 b = router.commitOrder(CfdTypes.Side.LONG, _lots(bN), bN * 3 / 10, 0, false);
        emit log_named_uint("B accepted id", b);
        uint256 keeperBefore = clearinghouse.balanceAssets(address(this));
        uint256 eveBefore = _settlementBalance(eve);
        vm.warp(vm.getBlockTimestamp() + 2);
        router.executeOrder(a, _mockPythUpdateData(1e8));
        (uint256 sA,,,,,,) = engine.positions(eve);
        assertEq(sA, _lots(aN), "A executed");
        // Control: with A live, the same B is rejected at commit (skew = 4)
        vm.warp(vm.getBlockTimestamp() + 1); _mark(1e8);
        try this.extCommit(CfdTypes.Side.LONG, _lots(bN), bN * 3 / 10) returns (uint64) { emit log("control: accepted"); }
        catch (bytes memory e) { emit log("control: rejected"); emit log_bytes(e); }
        vm.warp(vm.getBlockTimestamp() + 2);
        emit log_bytes(abi.encode(router.executeOrder(b, _mockPythUpdateData(1e8))));
        (uint256 sB,,,,,,) = engine.positions(eve);
        assertEq(sB, _lots(aN), "B failed at execution");
        uint256 keeperGain = clearinghouse.balanceAssets(address(this)) - keeperBefore;
        emit log_named_uint("keeper bounty gained from B", keeperGain);
        emit log_named_int("eve settlement delta (incl A fees)", int256(_settlementBalance(eve)) - int256(eveBefore));
        assertGt(keeperGain, 0, "bounty forfeited");
    }

    function _openNow(address who, CfdTypes.Side s, uint256 n6) internal {
        _fundTrader(who, n6 / 2);
        vm.warp(vm.getBlockTimestamp() + 1); _mark(1e8);
        vm.prank(who); uint64 id = router.commitOrder(s, _lots(n6), n6 * 3 / 10, 0, false);
        vm.warp(vm.getBlockTimestamp() + 2);
        router.executeOrder(id, _mockPythUpdateData(1e8));
        (uint256 sz,,,,,,) = engine.positions(who);
        require(sz > 0, "setup open failed");
    }

    /// SOLVENCY: balanced book (skew ~0) with max liability near pool depth. A (queued) + B push long max-profit
    /// above effective assets; B alone passes the commit-time solvency check.
    function test_queuedOpenSolvencyFalseAccept() public {
        uint256 d = pool.totalAssets();
        address c = address(0xC0); address dv = address(0xD0);
        _openNow(c, CfdTypes.Side.LONG, d * 35 / 100);
        _openNow(dv, CfdTypes.Side.SHORT, d * 35 / 100 * 2);
        _openNow(address(0xC1), CfdTypes.Side.LONG, d * 35 / 100);
        _openNow(address(0xD1), CfdTypes.Side.SHORT, d * 20 / 100);
        _openNow(address(0xC2), CfdTypes.Side.LONG, d * 20 / 100); // L = S = 0.9 d
        uint256 n = d * 7 / 100;
        _fundTrader(eve, n);
        vm.warp(vm.getBlockTimestamp() + 1); _mark(1e8);
        vm.prank(eve); uint64 a = router.commitOrder(CfdTypes.Side.LONG, _lots(n), n * 3 / 10, 0, false);
        vm.prank(eve); uint64 b = router.commitOrder(CfdTypes.Side.LONG, _lots(n), n * 3 / 10, 0, false);
        uint256 keeperBefore = clearinghouse.balanceAssets(address(this));
        vm.warp(vm.getBlockTimestamp() + 2);
        router.executeOrder(a, _mockPythUpdateData(1e8));
        (uint256 sA,,,,,,) = engine.positions(eve);
        assertEq(sA, _lots(n), "A executed");
        vm.warp(vm.getBlockTimestamp() + 1); _mark(1e8);
        try this.extCommit(CfdTypes.Side.LONG, _lots(n), n * 3 / 10) returns (uint64) { emit log("control: accepted"); }
        catch (bytes memory e) { emit log("control: rejected"); emit log_bytes(e); }
        vm.warp(vm.getBlockTimestamp() + 2);
        emit log_bytes(abi.encode(router.executeOrder(b, _mockPythUpdateData(1e8))));
        (uint256 sB,,,,,,) = engine.positions(eve);
        assertEq(sB, _lots(n), "B failed at execution");
        emit log_named_uint("keeper bounty gained from B", clearinghouse.balanceAssets(address(this)) - keeperBefore);
    }
}
```
