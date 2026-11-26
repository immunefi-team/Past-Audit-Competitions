# \#99160 [SC-Low] A same-side open queued behind the account's own full close is rejected at commit, because admission previews it as an increase of the position that the queued close removes

**Submitted on Nov 12th 2026 at 07:58:00 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99160
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

Scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Affected: `packages/perps/src/router/OrderLiquidationBatchLogic.sol`
(`_validatedOpenBounty`, `_queuedPosition`), `packages/perps/src/CfdEngineLens.sol` (`previewOpenRevertCode`).

The Router rejects at commit an open whose planner preview returns a "commit-time rejectable" code while the cached
mark is fresh (`OrderLiquidationBatchLogic.sol:437-445`; the rejectable set is MUST_CLOSE_OPPOSING, POSITION_TOO_SMALL,
SKEW_TOO_HIGH, INSUFFICIENT_INITIAL_MARGIN, SOLVENCY_EXCEEDED, INVALID_SIZE_QUANTUM, LIQUIDATION_RESERVE_UNFUNDED,
VPI_REBATE_RESERVE_UNFUNDED, NUMERIC_DOMAIN_EXCEEDED, `CfdEnginePlanLib.sol:79-105`).

After the audit (MED-14, "Trader can be blocked from queueing an open behind their closing order"), admission projects
the live position through the account's earlier queued orders (`_queuedPosition`, `:367-402`). But that projection is
used ONLY for the side check (`:423-428`). The preview that decides the rejection is still computed on the LIVE
position:

```solidity
uint8 revertCode =
    host.engineLensForCommit().previewOpenRevertCode(account, side, sizeDelta, marginDelta, commitPrice);   // :431-432
```

For an open on the opposing side, the live preview returns MUST_CLOSE_OPPOSING, which the fix explicitly ignores
(`:436-437`), so the audited case works. For an open on the SAME side, behind a queued full close of that side, the
live preview plans the order as an INCREASE of a position the queued close will have removed. Every rejectable code
produced by the phantom combined position is then enforced: skew, initial margin, solvency headroom, liquidation-reserve
funding. The order is rejected although it is valid on the projected (flat) position and is accepted once the close has
executed.

The fix note for MED-14 states that admission "projects the live position through earlier queued orders for the same
account before checking the requested side" and that "an opposing open can be queued after a projected full close". The
same-side reopen (close a position and queue a fresh one of the same side, for example to reset the entry, re-size, or
re-margin) is the symmetric case and remains blocked.

### Relation to the AI-audit MED-14 (why this is not the known issue)

- **What MED-14 reported and what was fixed.** MED-14 ("Trader can be blocked from queueing an open behind their
  closing order") described the OPPOSING-side case: Alice is LONG, queues a full LONG close, then queues a SHORT open,
  and the commit is rejected as `MUST_CLOSE_OPPOSING` from the live position. Its disposition is "Addressed", with the
  fix scoped explicitly to that case: "An opposing open can be queued after a projected full close; a partial close that
  leaves opposing exposure still blocks it. The live-position preview's MUST_CLOSE_OPPOSING result is ignored only after
  the queued-position side check passes." The control test below
  (`test_Med14Residual_OpposingFlipBehindOwnFullCloseIsAdmitted`) confirms that case is fixed.
- **What this report shows.** The SAME-side case, which MED-14 neither described nor discussed. The fix uses the queued
  projection only for the side check (`OrderLiquidationBatchLogic.sol:423-428`); every other commit-time-rejectable code
  (skew, initial margin, solvency headroom, reserve funding) is still computed on the live position (`:431-445`), so the
  order is planned as an increase of a position that the queued close removes. This is a defect of the fix, not of the
  original report.
- **Not the disposition's accepted limitation.** The disposition accepts that an ADMITTED open can still fail at
  execution if the earlier close fails ("not a guarantee that the preceding close will execute"). Here the order is
  never admitted: it is rejected at commit although it is valid on the projected (flat) position and is accepted, unchanged,
  as soon as the close has executed.
- **The stated intent is projection.** The disposition describes the intended semantics as "Open admission projects the
  live position through earlier queued orders for the same account". For a same-side open behind a full close, admission
  does not project: it evaluates the live position.

## Impact

A trader cannot pre-queue the routine "close, then open again on the same side" sequence: the second commit reverts
with `OrderRouter__PredictableOpenInvalid` whenever the phantom combined position would breach skew, initial margin,
solvency or reserve funding — which is likely for a large position or an under-water one, exactly when a trader wants to
reset it. The trader must wait for the close to execute and commit again, losing the queue slot and the intended timing.
No value is lost.

## Recommendation

Preview the open against the projected position (`_queuedPosition`), not the live one: when the projection is flat,
plan the order as a fresh open (or skip the commit-time rejection, as is already done for MUST_CLOSE_OPPOSING) and leave
the authoritative checks to execution.

## Proof of Concept

## Proof of concept

Executable Foundry test against the scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, written and run on
2026-11-12 on the machine that wrote it; the run log is at the end. It uses the repository's own test stack
(`BasePerpTest`: real OrderRouter and its admission logic, real CfdEngineLens, Engine, Clearinghouse and HousePool, the
repository's MockPyth), so it must be saved inside the repository's test tree.

Save the file as `packages/perps/test/perps/accounting/close/ZzMed14SameSideReopen.t.sol` and run:

```bash
forge test --root packages/perps --match-path 'test/perps/accounting/close/ZzMed14SameSideReopen.t.sol' -vv
```

Expected: 2 tests pass.

### Step by step

1. Setup: release margin, skew and bounty parameters (maintenance 10 bps, initial 20 bps, max skew 40%, bounty 10 bps /
   1 USDG; price impact and carry off, irrelevant here), a live session, pool depth 1,002,000 USDG, so the skew cap is
   about 400,800 USDG of notional.
2. T1 commits a LONG of 3,000 lots (300,000 notional) through `router.commitOrder`; a keeper fills it.
3. T1 commits a full close of the 3,000 lots (`isClose = true`). It is queued.
4. T1 commits a new LONG of 3,000 lots behind that close. The commit reverts with
   `OrderRouter__PredictableOpenInvalid(4)` (4 = SKEW_TOO_HIGH). Admission projected the queue only for the side check;
   the rejecting preview planned the order as a 6,000-lot increase of the live position (600,000 > 400,800).
5. The keeper fills the queued close; the account is flat. The identical LONG of 3,000 lots now commits successfully: the
   rejection in step 4 was a false positive caused by the live-position preview.
6. Control (second test): behind the same queued full close, an opposing SHORT of 3,000 lots is admitted, as the audited
   MED-14 fix intends (`MUST_CLOSE_OPPOSING` from the live preview is skipped).

What the test simplifies: Pyth data is the repository's MockPyth (plumbing, not signatures) and the keeper is a test
address. Commits go through the repository's `LegacyOrderRouterHarness.commitOrder` wrapper, which builds a standard V3
request with the current configuration hash and maximal bounds; admission is the production logic.

### PoC file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// MED-14 residual: a SAME-side open queued behind the account's own full close is screened against the LIVE position.
//
// `_validatedOpenBounty` (router/OrderLiquidationBatchLogic.sol:404-455) projects the account's queued orders
// (`_queuedPosition`, :367-402) but uses the projection ONLY for the side check (:424-428). The rejecting preview,
// `engineLensForCommit().previewOpenRevertCode(account, side, sizeDelta, marginDelta, price)` (:431-432), plans the open
// against the live Engine position, so an open that follows a queued full close of the same side is planned as an
// INCREASE of a position that will no longer exist. Any commit-time-rejectable code produced by the phantom combined
// position (here SKEW_TOO_HIGH) rejects the order (:437-445). The opposing-side variant was the audited MED-14 and is
// fixed: the MUST_CLOSE_OPPOSING preview is skipped (:437). The same-side variant is not.
//
// Run: forge test --root packages/perps --match-path 'test/perps/accounting/close/ZzMed14SameSideReopen.t.sol' -vv

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

contract ZzMed14SameSideReopenTest is BasePerpTest {

    uint256 internal constant LOT_ = 1e20;
    address internal constant KEEPER0 = address(0xCAFE);
    address internal constant T1 = address(0x1414);
    uint256 internal constant LOTS = 3000; // 300,000 notional at 1.00; skew cap is 40% of about 1,000,000 of depth

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 0;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    function setUp() public override {
        super.setUp();
        vm.deal(KEEPER0, 100 ether);
        _fundTrader(T1, 100_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 days + 3 hours);
        _refresh();
        require(!engine.isFadWindow() && !engine.isOracleFrozen(), "setup must start in a live session");
    }

    function _pyth(
        uint256 publishTime
    ) internal returns (bytes[] memory updateData) {
        bytes32[] memory feedIds = new bytes32[](2);
        feedIds[0] = bytes32(uint256(1));
        feedIds[1] = bytes32(uint256(2));
        baseMockPyth.setAllUniquePrices(feedIds, int64(int256(1e8)), 0, int32(-8), publishTime, publishTime - 1);
        updateData = new bytes[](1);
        updateData[0] = abi.encode(uint256(1e8));
    }

    function _refresh() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _pyth(vm.getBlockTimestamp());
        vm.prank(KEEPER0);
        router.updateMarkPrice(data);
    }

    function _fillHead() internal {
        uint64 head = router.nextExecuteId();
        require(head != 0, "no head");
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(head);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory data = _pyth(uint256(pending.commitTime) + 1);
        vm.prank(KEEPER0);
        router.executeOrder(head, data);
    }

    function _commit(
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose
    ) internal returns (bool ok, bytes memory reason) {
        vm.prank(T1);
        try router.commitOrder(side, lots * LOT_, margin, 0, isClose) {
            ok = true;
        } catch (bytes memory r) {
            reason = r;
        }
    }

    /// The defect: LONG 3,000 open; full close queued; a fresh LONG 3,000 behind it is rejected at commit as
    /// SKEW_TOO_HIGH, because it is planned as a 6,000-lot increase of the live position.
    function test_Med14Residual_SameSideReopenBehindOwnFullCloseIsRejected() public {
        (bool ok,) = _commit(CfdTypes.Side.LONG, LOTS, 10_000e6, false);
        assertTrue(ok, "initial open commits");
        _fillHead();
        (uint256 size,,,,,,) = engine.positions(T1);
        assertEq(size, LOTS * LOT_, "position open");
        _refresh();

        (ok,) = _commit(CfdTypes.Side.LONG, LOTS, 0, true);
        assertTrue(ok, "full close commits");
        emit log_named_uint("pool depth (atoms)", pool.totalAssets());

        bytes memory reason;
        (ok, reason) = _commit(CfdTypes.Side.LONG, LOTS, 10_000e6, false);
        emit log_named_bytes("same-side reopen revert", reason);
        assertFalse(ok, "MED-14 residual: same-side reopen behind own full close is rejected");
        assertEq(
            keccak256(reason),
            keccak256(
                abi.encodeWithSelector(
                    IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector,
                    uint8(CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH)
                )
            ),
            "rejected as SKEW_TOO_HIGH from the phantom combined position"
        );

        // The same order is valid once the close has executed: the rejection was a false positive.
        _fillHead();
        (size,,,,,,) = engine.positions(T1);
        assertEq(size, 0, "close executed");
        _refresh();
        (ok, reason) = _commit(CfdTypes.Side.LONG, LOTS, 10_000e6, false);
        emit log_named_bytes("same order after the close executed (empty = ok)", reason);
        assertTrue(ok, "control: the identical open commits once the close has executed");
    }

    /// Control: the opposing-side flip behind the same full close (the audited MED-14 case) is admitted.
    function test_Med14Residual_OpposingFlipBehindOwnFullCloseIsAdmitted() public {
        (bool ok,) = _commit(CfdTypes.Side.LONG, LOTS, 10_000e6, false);
        assertTrue(ok, "initial open commits");
        _fillHead();
        _refresh();
        (ok,) = _commit(CfdTypes.Side.LONG, LOTS, 0, true);
        assertTrue(ok, "full close commits");
        bytes memory reason;
        (ok, reason) = _commit(CfdTypes.Side.SHORT, LOTS, 10_000e6, false);
        emit log_named_bytes("opposing flip revert (empty = ok)", reason);
        assertTrue(ok, "MED-14 fix: the opposing flip behind a queued full close is admitted");
    }

}
```

### Run log (2026-11-12, scope commit)

```
cmdline: forge test --root packages/perps --match-path test/perps/accounting/close/ZzMed14SameSideReopen.t.sol -vv  (MED-14 residual PoC)
Compiling 1 files with Solc 0.8.35
Solc 0.8.35 finished in 299.69s
Compiler run successful with warnings:
Ran 2 tests for test/perps/accounting/close/ZzMed14SameSideReopen.t.sol:ZzMed14SameSideReopenTest
[PASS] test_Med14Residual_OpposingFlipBehindOwnFullCloseIsAdmitted() (gas: 4437746)
Logs:
  opposing flip revert (empty = ok): 0x
[PASS] test_Med14Residual_SameSideReopenBehindOwnFullCloseIsRejected() (gas: 5061732)
Logs:
  pool depth (atoms): 1002000000000
  same-side reopen revert: 0x2ae052ed0000000000000000000000000000000000000000000000000000000000000004
  same order after the close executed (empty = ok): 0x
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 31.36ms (19.05ms CPU time)
Ran 1 test suite in 46.47ms (31.36ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
exit_code=0
```
