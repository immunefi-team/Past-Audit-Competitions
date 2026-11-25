# \#99881 [SC-Low] MED-14 fix is incomplete: a valid same-side re-open queued behind a full close is rejected under a fresh mark

**Submitted on Nov 15th 2026 at 16:17:25 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99881
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# MED-14 fix is incomplete: a valid same-side re-open queued behind a full close is rejected under a fresh mark

**Impact (paid title):** Contract fails to deliver promised returns, but doesn't lose value
**Severity:** Low
**Relation:** Incomplete fix of Cecuro **MED-14** ("Trader can be blocked from queueing an open behind their closing order"), marked *Addressed* in PR #126 (`88ef3954`).
**Status:** Reproduced (Foundry PoC, release risk params, 2/2 tests pass).

---

## Affected code (HEAD = `a82d4fefb02768e5b3bddcc97dee371eee24fea8`)

- **Root:** `packages/perps/src/router/OrderLiquidationBatchLogic.sol:423-446` (`_validatedOpenBounty`)
  - `423-428` queued-position projection + **side-only** test
  - `430-432` live-position preview (`previewOpenRevertCode`)
  - `437-445` fresh-mark predictable-open rejection, skipped **only** for `MUST_CLOSE_OPPOSING`
  - `367-402` `_queuedPosition` seeds from the **live** engine position (`372-377`) then applies the account pending orders
- **Snapshot is the live position:** `packages/perps/src/CfdEngineLens.sol:145-153` (`previewOpenRevertCode` -> `_previewOpen`) and `255-266` (snapshot built from `engine.pool().totalAssets()` + live `engine.positions`, line `262`)
- **Combined-position checks that fire:** `packages/perps/src/libraries/CfdEnginePlanLib.sol:443-480` (skew on live side OI + `order.sizeDelta`; `SKEW_TOO_HIGH` at `478`), `431-437` (`INSUFFICIENT_INITIAL_MARGIN`), `421-424` (`SOLVENCY_EXCEEDED`)
- **Classifier:** `CfdEnginePlanLib.sol:82-94` marks `SKEW_TOO_HIGH`, `INSUFFICIENT_INITIAL_MARGIN`, `SOLVENCY_EXCEEDED` (and `MUST_CLOSE_OPPOSING`) `CommitTimeRejectable`; `OrderFailurePolicyLib.sol:13-17` rejects any `CommitTimeRejectable` at commit
- **Error:** `IOrderRouterErrors.sol:163` `OrderRouter__PredictableOpenInvalid(uint8 code)`; codes from `CfdEnginePlanTypes.sol:166-191` (`SKEW_TOO_HIGH = 4`, `INSUFFICIENT_INITIAL_MARGIN = 6`)
- **Seed (prior report):** `ai_audit_2026-10-26.txt:1691-1750` (MED-14 body + team "Addressed" response + locations + remediation)

---

## Summary

MED-14's root cause: commit-time predictable-open rejection is computed from the **live** engine position instead of the router's **queued-position projection**, so a valid open staged behind an earlier queued close is rejected while the cached mark is fresh. PR #126 "Addressed" this only for the **opposite-side** open:

1. it added `_queuedPosition(...)` + a **side-only** test (`L424-428`): if the *projected* position is on the opposite side, revert `MUST_CLOSE_OPPOSING`; and
2. it made the live-preview rejection skip **exactly one** code -- `MUST_CLOSE_OPPOSING` -- via `revertCode != opposingCode` (`L437`).

Everything else is unchanged: `previewOpenRevertCode` still builds its snapshot from the **live** position (`CfdEngineLens.sol:262`). So for a trader who is live `LONG S`, queues a **full close** of `S`, then queues a **valid same-side `LONG X` re-open**:

- `_queuedPosition` projects **flat** (the close removes `S`), so the side test at `L426` passes; but
- `previewOpenRevertCode` plans `X` as an **increase on the still-live `S`** (`S + X`) and returns `SKEW_TOO_HIGH` / `INSUFFICIENT_INITIAL_MARGIN` / `SOLVENCY_EXCEEDED` -- none is `MUST_CLOSE_OPPOSING`, so `L437` does **not** skip them; and
- with a fresh mark (`L439` `!isStale`), `L442-444` reverts `OrderRouter__PredictableOpenInvalid`.

The order is valid: once the queued close executes, the position is flat and `X` opens cleanly. The same order is also admitted the moment the mark goes stale -- admission "depends on mark freshness rather than queued exposure", exactly as MED-14 described. The auditor's primary remediation ("base commit-time predictable-open rejection on the router's queued-position projection instead of the live engine position") was **not** implemented; only the narrower `MUST_CLOSE_OPPOSING` skip was.

## Why the fix is incomplete -- original MED-14 vs HEAD

**Baseline `6d6622f2` (pre-fix):** no projection; the live-preview code is rejected directly under a fresh mark (`if (commitMarkTime != 0) { if(!isStale) if(isPredictablyInvalidOpen(cat)) revert(revertCode); }`). MED-14 observed this rejects `MUST_CLOSE_OPPOSING` for an opposite-side open behind a full close.

**HEAD `a82d4fef` (`OrderLiquidationBatchLogic.sol:423-446`) -- only material changes:** (a) added `_queuedPosition` + the `L426` side-only test; (b) added `&& revertCode != opposingCode` to the `L437` gate. The projected (flat) position is **never substituted into the lens snapshot** (`CfdEngineLens.sol:262` still reads `engine.positions`), so the MED-14 root cause -- admission driven by the live position, not the queued projection -- remains reachable on the **same-side** branch via `SKEW_TOO_HIGH` / `INSUFFICIENT_INITIAL_MARGIN` / `SOLVENCY_EXCEEDED`. Internal inconsistency: the side test certifies the account will be flat at the FIFO head, yet the next check rejects the open because the live `S` (which that projection just said will be gone) is still counted.

## Unprivileged call sequence (release config: maxSkewRatio 0.4e18, maint 10bps, init 20bps, vpi 0.01e18)

All calls by the trader (unprivileged); the close is executed by a permissionless keeper.

1. Trader holds a live `LONG S`, notional ~`0.30x` net pool depth (openable: `0.30 < 0.40`). Cached mark fresh.
2. `router.commitOrder(close LONG, sizeDelta = S)` -> admitted (full close queued).
3. `router.commitOrder(open LONG, sizeDelta = X, margin)` with `X` ~ `0.25x` depth (valid alone: `0.25 < 0.40`):
   - `_queuedPosition` projects flat -> `L426` side test passes;
   - `previewOpenRevertCode` plans `S + X` ~ `0.55x` depth on the **live** position -> `SKEW_TOO_HIGH` (code `4`);
   - fresh mark -> `revert OrderRouter__PredictableOpenInvalid(4)`. **The promised re-open is denied.**
4. Proof it was valid: a keeper calls `router.executeOrder(closeId, pythData)` -> the close executes, position flat -> the **identical** `open LONG X` now commits successfully. Equivalently, once `lastMarkTime` ages past `maxStaleness` (~60s), step 3 is admitted unchanged.

**Variant (same root, `INSUFFICIENT_INITIAL_MARGIN`, code `6`):** a live `LONG S` whose equity sits between maintenance and its own initial margin (healthy to hold after an adverse drift) + a correctly-margined fresh `LONG X` behind the full close: combined `S + X` fails the initial-margin check (`CfdEnginePlanLib.sol:431-437`) although `X` alone is adequately margined. **Reverse corollary:** an open that only passes *because of* the live position's margin/skew headroom is admitted and then deterministically fails at execution after the close clears, wasting the open-order bounty.

### Harm / numbers at release config
No funds lost. The documented FIFO close/open workflow (the README / DEPOSIT_FREE_CLOSE support MED-14 cited) cannot be staged for same-side re-opens (roll / re-margin / re-enter) while the mark is fresh; the trader must wait for the close to execute (up to the ~60s window) or for the mark to go stale, losing back-to-back execution. Reverse corollary forfeits the open-order bounty (1 bp, floor 0.01, cap 0.20 USDG at release). The auditor rated the opposite-side sibling **Medium** (blocked-from-queueing / griefing); the honest fit for this same-side residual is **Low -- "Contract fails to deliver promised returns, but doesn't lose value."**

## Known-issue / duplicate separation

- **Not MED-14 as written / as fixed.** MED-14 and its "Addressed" response are explicitly scoped to the **opposite-side** open ("An opposing open can be queued after a projected full close ... the live-position preview's `MUST_CLOSE_OPPOSING` result is ignored only after the queued-position side check passes"). This is the **same-side** re-open, rejected via `SKEW_TOO_HIGH` / `INSUFFICIENT_INITIAL_MARGIN`, which the fix does not address. Reportable because the fix at `a82d4fef` is shown **incomplete**.
- **MED-14 is not pinned.** The pinned list (MED-4/7/13/16+LOW-57/22/24+29/26/30, HIGH-1; SC-M4/M6/Q3/Q4/I2; SCAN-*) does not include MED-14.
- **Not MED-16 / LOW-57 (pinned).** Those concern **stale-mark admission** (a free option admitted *because* the mark is stale). This is the opposite direction: a **false rejection under a FRESH mark**. Stale-mark admission is used in the PoC only as a contrast, not as the reported defect.
- **Distinct from already-submitted/claimed Plether reports.** Different contract/root from A (`#99680`, PositionProtectionBook create/replace reverts after a fill), B (same-batch recap HWM collapse -- rejected), C (`#99689`, recap remainder strands in `unassignedAssets`/HousePool), and claimed G2/L2/I1 + insights I4/I5/I7/I9. This root is `OrderRouter`/`OrderLiquidationBatchLogic._validatedOpenBounty` commit-time open admission.

## Proof of Concept

## Proof of Concept

**Lane:** `D:\Pentest\bb\plether\pocw4`  **Test:** `test/Med14SameSideReopen.t.sol` (inherits `harn/BasePerpTest.sol`; release `_riskParams()`; `packages/perps/src` untouched).

- `test_FreshMarkRejectsValidSameSideReopen_Skew` -- live `LONG S` (`0.30x` depth); queue full close (admitted); queue same-side `LONG X` (`0.25x` depth) -> **reverts `OrderRouter__PredictableOpenInvalid(4)` (`SKEW_TOO_HIGH`)** under a fresh mark; then execute the queued close (keeper) -> flat -> the **identical** re-open **commits successfully**.
- `test_StaleMarkAdmitsTheSameReopen` -- same setup; after the mark ages past `maxStaleness`, the identical re-open is admitted (pending 1->2), proving admission flips purely on mark freshness.

**Trace evidence (commit at step 3):**
```
VM::expectRevert(OrderRouter__PredictableOpenInvalid(4))
  CfdEngineLens::previewOpenRevertCode(0x..A11cE, 0 /*LONG*/, 250500000000000000000000 /*X=0.25x depth*/, 50000000000, 100000000) [staticcall]
  <- [Revert] OrderRouter__PredictableOpenInvalid(4)
<- [Revert] OrderRouter__PredictableOpenInvalid(4)
```
RawSnapshot confirms release params: `maxSkewRatio 4e17`, `maintMarginBps 10`, `initMarginBps 20`, `vpiFactor 1e16`; `poolAssets 1.002e12`.

**Run:**
```
cd /d/Pentest/bb/plether/pocw4
C:\Users\pcher\.foundry\bin\forge.exe test --offline --match-path test/Med14SameSideReopen.t.sol -vv
```
**Result:**
```
Ran 2 tests for test/Med14SameSideReopen.t.sol:Med14SameSideReopen
[PASS] test_FreshMarkRejectsValidSameSideReopen_Skew() (gas: 6054058)
[PASS] test_StaleMarkAdmitsTheSameReopen() (gas: 4460790)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 22.31ms
```

## Remediation

Implement the auditor's primary MED-14 remediation: substitute the **queued-position projection** into the lens snapshot used for commit-time predictable-open rejection (not just the side test), or skip the fresh-mark predictable-open rejection whenever earlier queued orders change the account's exposure before this open reaches the FIFO head -- not only for `MUST_CLOSE_OPPOSING` but for the skew / initial-margin / solvency codes that also depend on the live position being present. Execution already re-checks all risk against the real post-close state.

### Full test source: `pocw4/test/Med14SameSideReopen.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "harn/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title MED-14 incomplete-fix: same-side re-open queued behind a full close is rejected under a fresh mark.
/// @notice Cecuro MED-14 ("Trader can be blocked from queueing an open behind their closing order") was
///         "Addressed" in PR #126 only by (a) projecting the live position through earlier queued orders for a
///         SIDE test and (b) ignoring the live-position preview's MUST_CLOSE_OPPOSING result. Every other
///         commit-time-rejectable code is still produced from the LIVE engine position
///         (CfdEngineLens.sol:262 -> CfdEnginePlanLib.sol:458-480 / 431-437). So a trader who queues a full
///         close and then queues a VALID same-side re-open is screened as if the re-open were an INCREASE on
///         the still-live position (S + X), and is rejected (SKEW_TOO_HIGH / INSUFFICIENT_INITIAL_MARGIN)
///         whenever the cached mark is fresh -- exactly the documented FIFO close/open workflow MED-14 was
///         supposed to restore. Root: router/OrderLiquidationBatchLogic.sol:423-446.
///
/// @dev Local stand (MockPyth), RELEASE risk params (_riskParams below), packages/perps/src untouched.
contract Med14SameSideReopen is BasePerpTest {
    address internal constant ALICE = address(0xA11CE);

    uint256 internal constant PRICE = 1e8;

    uint256 internal nonce;

    // -------------------------------------------------------------------------------------------------
    // RELEASE PARAMETERS (script/DeployPerpsArbitrumSepolia.s.sol _riskParams): vpi 0.01e18, maxSkew 0.4e18,
    // maint 10 bps, init 20 bps, fad 300 bps, baseCarry 500, minBounty 1e6, bountyBps 10, keeperShare 5000.
    // -------------------------------------------------------------------------------------------------
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
        _setRiskParams(_riskParams());
        _setMark(PRICE);
    }

    // -------------------------------------------------------------------------------------------------
    // Helpers (mirrors of the proven PlannerMixLane helpers)
    // -------------------------------------------------------------------------------------------------
    function _feedTicks(
        uint256 price
    ) internal view returns (MockPyth.FeedUpdate[] memory ticks) {
        bytes32[] memory ids = _basePythFeedIds();
        ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 publishTime = vm.getBlockTimestamp();
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previous));
        }
    }

    function _priceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        updateData = new bytes[](1);
        updateData[0] = abi.encode(_feedTicks(price));
    }

    /// @dev Permissionless mark refresh through the production Router path (makes lastMarkTime fresh).
    function _setMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        uint256 publishTime = vm.getBlockTimestamp();
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, publishTime);
        router.updateMarkPrice(_priceData(price));
    }

    function _orderRequest(
        CfdTypes.Side side,
        bool close,
        uint256 size,
        uint256 margin
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        ++nonce;
        request.clientOrderId = keccak256(abi.encode("med14-same-side", nonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = close;
        request.targetPrice = (side == CfdTypes.Side.LONG) == close ? 2e8 : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 600),
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
        address account,
        CfdTypes.Side side,
        bool close,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory request = _orderRequest(side, close, size, margin);
        vm.prank(account);
        id = router.commitOrder(request);
    }

    /// @dev Rounds a target notional (USDG, 6dec) into a canonical quantum-aligned size at PRICE = 1e8.
    ///      notionalAssets = (size / SIZE_QUANTUM) * price; at price 1e8, size = notionalAssets * 1e12.
    function _sizeForNotional(
        uint256 notionalAssets
    ) internal pure returns (uint256 size) {
        size = notionalAssets * 1e12;
        size = (size / CfdTypes.SIZE_QUANTUM) * CfdTypes.SIZE_QUANTUM;
    }

    // -------------------------------------------------------------------------------------------------
    // MAIN: a valid same-side re-open queued behind a full close is rejected under a FRESH mark,
    //       then admitted once the queued close executes -- proving the order was valid all along.
    // -------------------------------------------------------------------------------------------------
    function test_FreshMarkRejectsValidSameSideReopen_Skew() public {
        uint256 depth = pool.totalAssets(); // net pool depth seed (trader claims start at 0)

        // Live LONG S ~ 0.30x depth: openable on its own (skew 0.30 < cap 0.40).
        uint256 liveSize = _sizeForNotional((depth * 30) / 100);
        // Same-side re-open X ~ 0.25x depth: valid on its own (skew 0.25 < 0.40),
        // but S + X ~ 0.55x depth > 0.40x cap -> SKEW_TOO_HIGH when screened against the LIVE position.
        uint256 reopenSize = _sizeForNotional((depth * 25) / 100);

        _fundTrader(ALICE, 3_000_000e6);
        _open(ALICE, CfdTypes.Side.LONG, liveSize, 50_000e6, PRICE); // seed the live LONG directly in the engine
        _setMark(PRICE); // fresh mark (lastMarkTime == now)

        (uint256 sz0,,,, CfdTypes.Side side0,,) = engine.positions(ALICE);
        assertEq(sz0, liveSize, "precondition: ALICE holds the live LONG");
        assertEq(uint256(side0), uint256(CfdTypes.Side.LONG), "precondition: side LONG");
        assertTrue(engine.lastMarkTime() != 0, "precondition: cached mark present");

        // Step 1: queue a FULL close of the live LONG. Admitted.
        uint64 closeId = _commit(ALICE, CfdTypes.Side.LONG, true, liveSize, 0);
        assertEq(router.pendingOrderCounts(ALICE), 1, "full close queued");

        // Step 2: queue a VALID same-side (LONG) re-open behind the close. The queued-position projection is
        // flat (the close removes S), so the L426 side test passes -- but previewOpenRevertCode still plans
        // S + X on the LIVE position and returns SKEW_TOO_HIGH, which L437 does NOT skip (only MUST_CLOSE_
        // OPPOSING is skipped). With a fresh mark, L439-444 rejects the valid order.
        OrderV3Types.OrderRequest memory reopen = _orderRequest(CfdTypes.Side.LONG, false, reopenSize, 50_000e6);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector,
                uint8(CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH)
            )
        );
        vm.prank(ALICE);
        router.commitOrder(reopen);

        // Still only the close is queued; the promised FIFO close/open staging was denied.
        assertEq(router.pendingOrderCounts(ALICE), 1, "re-open was rejected, not queued");

        // Step 3 (control): execute the queued close through the permissionless keeper path. ALICE goes flat.
        router.executeOrder(closeId, _mockPythUpdateData(PRICE));
        (uint256 szAfterClose,,,,,,) = engine.positions(ALICE);
        assertEq(szAfterClose, 0, "queued close executed -> flat position");
        assertEq(router.pendingOrderCounts(ALICE), 0, "no orders pending after close");

        // Step 4 (control): the IDENTICAL re-open now commits successfully under a still-fresh mark. The order
        // was valid all along; the fresh-mark screening in Step 2 rejected a promised action.
        OrderV3Types.OrderRequest memory reopenAgain =
            _orderRequest(CfdTypes.Side.LONG, false, reopenSize, 50_000e6);
        vm.prank(ALICE);
        uint64 reopenId = router.commitOrder(reopenAgain);
        assertTrue(reopenId != 0, "re-open admitted once the earlier close executed");
        assertEq(router.pendingOrderCounts(ALICE), 1, "re-open now queued");
    }

    // -------------------------------------------------------------------------------------------------
    // SUPPLEMENTARY: admission of the exact same order flips purely on cached-mark freshness -- exactly
    // MED-14's observation that admission "depends on mark freshness rather than queued exposure".
    // (The stale-mark admission itself is the separately-pinned free-option behavior; it is used here only
    //  as a contrast that proves the fresh-mark rejection is not a genuine validity check.)
    // -------------------------------------------------------------------------------------------------
    function test_StaleMarkAdmitsTheSameReopen() public {
        uint256 depth = pool.totalAssets();
        uint256 liveSize = _sizeForNotional((depth * 30) / 100);
        uint256 reopenSize = _sizeForNotional((depth * 25) / 100);

        _fundTrader(ALICE, 3_000_000e6);
        _open(ALICE, CfdTypes.Side.LONG, liveSize, 50_000e6, PRICE);
        _setMark(PRICE);

        _commit(ALICE, CfdTypes.Side.LONG, true, liveSize, 0); // full close queued
        assertEq(router.pendingOrderCounts(ALICE), 1, "full close queued");

        // Let the cached mark go stale (do NOT refresh it).
        IPletherOracle.PolicySnapshot memory policy = router.pletherOracle().getOrderExecutionPolicy(false);
        vm.warp(vm.getBlockTimestamp() + uint256(policy.maxStaleness) + 2);

        // Same order the fresh-mark path rejected is now admitted, because L439 isStale() short-circuits the
        // predictable-open rejection entirely.
        OrderV3Types.OrderRequest memory reopen = _orderRequest(CfdTypes.Side.LONG, false, reopenSize, 50_000e6);
        vm.prank(ALICE);
        uint64 id = router.commitOrder(reopen);
        assertTrue(id != 0, "stale mark admits the identical re-open");
        assertEq(router.pendingOrderCounts(ALICE), 2, "both close and re-open now queued");
    }
}

```
