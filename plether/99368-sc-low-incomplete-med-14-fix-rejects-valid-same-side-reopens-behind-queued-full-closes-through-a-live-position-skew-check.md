# \#99368 [SC-Low] Incomplete MED-14 fix rejects valid same-side reopens behind queued full closes through a live-position skew check

**Submitted on Nov 13th 2026 at 11:51:43 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99368
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description


## Brief / Introduction

While reviewing Cecuro's MED-14 remediation, I found that the Router still rejects a valid close-and-reopen sequence when the reopening order uses the **same side** as the position being closed. The remediation projects an account's earlier queued orders when checking whether the requested side is allowed. However, the subsequent predictable-open validation still calls `previewOpenRevertCode()` against the **currently live position**, rather than the position that will exist when the queued close has executed.

That distinction matters in the global FIFO queue. In my reproduction, Alice holds a 300,000-unit LONG and queues a full close. She then attempts to queue a new 300,000-unit LONG behind it. The projected position at that second order is flat, so the new LONG should be evaluated as a fresh 300,000-unit position. Instead, the commit-time preview merges the request with Alice's still-live 300,000-unit LONG, treats it as a 600,000-unit increase, and reverts with `OrderRouter__PredictableOpenInvalid(SKEW_TOO_HIGH)` under the 40% release skew cap.

I reproduced this through the canonical V3 `commitOrder(OrderV3Types.OrderRequest)` interface and added three independent controls. The same-side reopen succeeds if the queued close executes first; an opposite-side reopen can already be queued behind that close; and a *real* live-position increase to 600,000 units is correctly rejected. These differences isolate the failure to commit-time handling of queued exposure, not to the underlying risk limit.

**Selected impact:** Security Best practices Insight. The reproducible issue is incorrect admission of a supported FIFO trading workflow. I am reporting the concrete order-rejection defect and its operational consequence, without inferring unrelated asset loss or liquidation behavior.

## Vulnerability Details

### The remediation checks the projected side but not the projected risk state

The affected code is [`OrderLiquidationBatchLogic.sol`, `_validatedOpenBounty()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L403-L455).

The remediation first obtains the account's projected position:

```solidity
QueuedPositionView memory queuedPosition = _queuedPosition(host, engine, account);
uint8 opposingCode = uint8(CfdEnginePlanTypes.OpenRevertCode.MUST_CLOSE_OPPOSING);
if (queuedPosition.exists && queuedPosition.side != side) {
    revert OrderRouter__PredictableOpenInvalid(opposingCode);
}
```

This correctly recognizes that an earlier full close can leave the account without a position. However, immediately afterward the code calls:

```solidity
uint8 revertCode =
    host.engineLensForCommit().previewOpenRevertCode(account, side, sizeDelta, marginDelta, commitPrice);
```

`previewOpenRevertCode()` reads the actual Engine position for `account`; it is not supplied the `queuedPosition` computed above. The Router suppresses the live-position result only when it equals `MUST_CLOSE_OPPOSING`. A fresh mark does **not** suppress other predictable failure codes, including `SKEW_TOO_HIGH`:

```solidity
if (commitMarkTime != 0 && revertCode != opposingCode) {
    // ... fresh-mark and classification checks ...
    if (OrderFailurePolicyLib.isPredictablyInvalidOpen(failureCategory)) {
        revert OrderRouter__PredictableOpenInvalid(revertCode);
    }
}
```

The result is an inconsistent application of the same queued-position projection: the Router knows the earlier order will close the position for the *side* check, but still uses the old position for the *size and risk* check. In this test, that discrepancy incorrectly turns an otherwise valid reopen into a purported skew violation.

### Reproduced FIFO sequence

The test uses the release-style risk parameters, including a **40% maximum skew ratio**, and the project's approximately **1,000,000 USDG** seeded LP fixture. The account funds its own orders and uses the standard Router entry points; no production contract storage is changed.

| Stage | Live account position | Earlier queued order | Request being assessed | Correct interpretation |
|---|---|---|---|---|
| Initial open completed | LONG 300,000 | None | — | Valid existing position |
| Full close committed | LONG 300,000 | Close LONG 300,000 | — | Account projects flat after close |
| Same-side reopen attempted | LONG 300,000 | Full close ahead in FIFO | Open LONG 300,000 | Fresh LONG 300,000 should be admissible |
| Actual precheck | LONG 300,000 | Full close ignored in preview | Open LONG 300,000 | **Wrongly treated as LONG 600,000** |

The second open is rejected at **commit time**. It is not merely left in the queue to fail at execution. The test checks that the commit sequence does not advance and the earlier full close remains pending.

### Why this is an incomplete MED-14 fix, not the previously disclosed case

Cecuro MED-14, **“Trader can be blocked from queueing an open behind their closing order,”** reported the same broad class of inconsistency for **opposite-side** opens and was marked addressed in remediation commit [`5029cde91b87d24865fc8d6c8683686989ef45ac`](https://github.com/Plether-Fi/plether-core/commit/5029cde91b87d24865fc8d6c8683686989ef45ac).

The remediation addresses the specific `MUST_CLOSE_OPPOSING` result: an earlier queued full close now permits a new opposite-side open, as my control test verifies. It does not apply the projected-position logic to other live-state rejection codes. The **same-side reopen following a queued full close** therefore remains incorrectly rejected with `SKEW_TOO_HIGH` at the frozen competition commit.

I checked this behavior against [`ACCOUNTING_SPEC.md` — Open projection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1086-L1102), which requires opens to match positions projected through earlier queued orders and preserves revalidation at execution. The defect is specifically in early commit rejection; it does not require removing the actual execution-time risk checks.

The [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/) expressly permits reports on incomplete fixes to previously fixed or acknowledged audit findings. This report identifies the residual code path, the different affected order sequence, and a differential reproduction against the addressed opposite-side case.

## Impact Details

A trader using the documented FIFO order model cannot reliably stage a **same-side close-and-reopen** sequence when the live-state preview would classify the new order as predictably invalid, despite a prior queued close eliminating the exposure before the new open reaches the front of the queue.

In my test, the account has no way to commit the valid reopening order while the close is still pending and the cached price is fresh. The trader must instead let the close execute and then submit the new open, or rely on the admission check no longer taking that fresh-mark rejection path. This removes the ability to commit a two-order roll ahead of time, despite the order queue being designed to support dependent histories.

This affects workflows such as closing and reopening on the same side with a new size or fresh margin arrangement, particularly where the *current-plus-requested* exposure exceeds the skew cap while the *post-close* exposure would comply. It can also cause a valid replacement order to miss the intended execution sequence or require additional manual interaction. These are operational effects of the rejected commit; the PoC does not measure a particular market-loss amount.

The security-relevant observation is the mismatch between the Router's queue-state model and its pre-execution admission controls. The failure is deterministic under ordinary release parameters, can be reproduced without privileged access, and remains present despite the MED-14 remediation.

### Suggested remediation

Apply the same projected account state to **all state-dependent commit-time open checks**, not only side compatibility. If the projected state after preceding queued orders is flat, evaluate the new open as a potential fresh position rather than as an increase to the live position. An alternative is to avoid hard-rejecting state-dependent predictable-invalid codes when prior pending orders change the account's relevant exposure, while retaining unconditional input/domain checks.

Execution must continue to revalidate the real state and risk constraints when the order reaches the FIFO head. If an earlier close fails, its dependent reopen must not bypass skew, margin, or solvency rules. A regression test should cover same-side and opposite-side reopens behind full and partial closes, alongside genuine in-place position increases.

## References

- [Affected function: `OrderLiquidationBatchLogic.sol::_validatedOpenBounty`, lines 403–455](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L403-L455)
- [Queued-position calculation: `OrderLiquidationBatchLogic.sol::_queuedPosition`, lines 367–400](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L367-L400)
- [Live-state preview: `CfdEngineLens.sol::previewOpenRevertCode`, lines 145–153](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineLens.sol#L145-L153)
- [Accounting specification: *Open projection*, lines 1086–1102](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1086-L1102)
- [Cecuro audit: MED-14, “Trader can be blocked from queueing an open behind their closing order”](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [MED-14 remediation commit `5029cde` on Plether core](https://github.com/Plether-Fi/plether-core/commit/5029cde91b87d24865fc8d6c8683686989ef45ac)
- [Plether competition scope: known issues and incomplete-fix exception](https://immunefi.com/audit-competition/plether/scope/)



## Proof of Concept


## Reproduction environment

- **Audited commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- **Test location:** `packages/perps/test/perps/insight/Med14IncompleteFixV3Final.t.sol`
- **Target:** `packages/perps/src/router/OrderLiquidationBatchLogic.sol`
- **Toolchain:** repository's Foundry suite, Solidity `0.8.35`
- **Test fixture:** `BasePerpTest` with mocked external Pyth price data and mock USDG. The fixture deploys a test Router harness, but this PoC calls the **canonical production-format V3** `commitOrder(OrderV3Types.OrderRequest)` overload; it does not use the harness's legacy five-argument order adapter.

I reproduced the issue in four tests, each starting from a clean Foundry setup:

1. **Same-side queued reopen, bug case.** Alice funds and opens a 300,000-unit LONG at a price of `1e8`, then commits a full LONG close without executing it. She attempts to queue a new LONG of 300,000 units with 2,000 USDG margin. The Router rejects the commit with `OrderRouter__PredictableOpenInvalid(SKEW_TOO_HIGH)` even though the account projects flat after the earlier close. The close remains in the queue and no new order is recorded.
2. **Opposite-side control.** Starting from the same 300,000 LONG and queued full close, Alice queues a 300,000 SHORT instead. The second order is accepted. Both the close and the new SHORT execute successfully in order.
3. **Post-close validity control.** Alice executes the queued LONG close first, then submits the *same* 300,000 LONG request, which passes the Engine preview, commits, and executes. This isolates the premature rejection to the queued-state precheck.
4. **Genuine-skew rejection control.** Alice opens a 300,000 LONG with **no** close queued, then attempts to increase it by another 300,000. The Router correctly rejects the true 600,000-unit exposure with `SKEW_TOO_HIGH`, confirming that the skew limit itself works.

### Reproduction command

Save the source below as `packages/perps/test/perps/insight/Med14IncompleteFixV3Final.t.sol` in the repository at the frozen commit, then execute:

```bash
cd /workspaces/audit-comp-plether/packages/perps
export PATH="$HOME/.foundry/bin:$PATH"

test "$(git rev-parse HEAD)" = "a82d4fefb02768e5b3bddcc97dee371eee24fea8" || {
    echo "Wrong audited commit" >&2
    exit 1
}

export FOUNDRY_TEST=test
export FOUNDRY_OUT=out-med14-final
export FOUNDRY_CACHE_PATH=cache-med14-final

forge test --offline \
    --match-path 'test/perps/insight/Med14IncompleteFixV3Final.t.sol' \
    -vvv 2>&1 | tee med14-v3-final.log

git diff -- src/
```

### Verified Foundry results

I ran the four tests against commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. All four passed; the bug case is a passing test because it **asserts the unexpected revert**, while the controls assert successful execution or a justified rejection as appropriate.

```text
Ran 4 tests for test/perps/insight/Med14IncompleteFixV3Final.t.sol:Med14IncompleteFixV3FinalTest
[PASS] test_Control_OppositeSideQueuedReopenExecutes() (gas: 6092320)
Logs:
  MED14 CONTROL: opposite-side V3 reopen executed

[PASS] test_Control_RealLiveIncreaseCorrectlyRejected() (gas: 3019743)
Logs:
  MED14 CONTROL: true same-side increase rejected

[PASS] test_Control_SameSideReopenExecutesAfterClose() (gas: 6283788)
Logs:
  MED14 CONTROL: identical V3 LONG reopened after close

[PASS] test_PoC_SameSideReopenBehindFullCloseFalselyRejected() (gas: 3915034)
Logs:
  MED14: live LONG size 300000000000000000000000
  MED14: rejected planned increase 600000000000000000000000
  MED14: prior full close order 2
  MED14: same-side reopen rejected with SKEW_TOO_HIGH

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 103.32ms (80.98ms CPU time)
```

`git diff -- src/` produced no production source changes in the run.

### Complete Solidity PoC

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {console2} from "forge-std/console2.sol";

/// @notice MED-14 remediation regression: a queued full close is honored for the
///         prospective side check, but not for live-state open preview checks.
///         All commits use the canonical production V3 OrderRequest interface.
contract Med14IncompleteFixV3FinalTest is BasePerpTest {
    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xCAFE);

    uint256 internal constant SIZE = 300_000e18;
    uint256 internal constant OPEN_MARGIN = 2_000e6;
    uint256 internal constant MARK_PRICE = 1e8;
    uint256 private nextClientOrderId = 1;

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

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        uint256 target =
            ((side == CfdTypes.Side.LONG) == isClose)
                ? type(uint256).max
                : 1;

        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(nextClientOrderId++),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(window),
                executionWindowSeconds: window,
                allowedExecutionModes: 7,
                expectedConfigHash:
                    router.lifecycleBook().currentExecutionConfigHash(),
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

    function _commit(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request =
            _request(side, size, margin, isClose);

        vm.prank(ALICE);
        orderId = router.commitOrder(request);
        assertGt(orderId, 0, "new V3 order was not recorded");
    }

    function _execute(uint64 orderId) internal {
        bytes[] memory updates = _mockPythUpdateData(MARK_PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(orderId, updates);
        assertEq(
            uint8(result.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "V3 order was not executed"
        );
    }

    function _openLongAndQueueFullClose()
        internal
        returns (uint64 closeId)
    {
        _fundTrader(ALICE, 50_000e6);

        uint64 firstOpenId =
            _commit(CfdTypes.Side.LONG, SIZE, OPEN_MARGIN, false);
        _execute(firstOpenId);

        (uint256 currentSize,,,,,,) = engine.positions(ALICE);
        assertEq(currentSize, SIZE, "Alice must hold live LONG");

        closeId = _commit(CfdTypes.Side.LONG, SIZE, 0, true);
        assertEq(
            router.accountHeadOrderId(ALICE),
            closeId,
            "full close must remain queued"
        );

        (currentSize,,,,,,) = engine.positions(ALICE);
        assertEq(currentSize, SIZE, "queued close is not executed yet");
    }

    function _assertSkewRevertCode() internal view {
        uint8 actual = engineLens.previewOpenRevertCode(
            ALICE,
            CfdTypes.Side.LONG,
            SIZE,
            OPEN_MARGIN,
            MARK_PRICE
        );
        assertEq(
            actual,
            uint8(CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH),
            "live-state preview must see 600k LONG"
        );
    }

    function _expectSameSideReject() internal {
        OrderV3Types.OrderRequest memory reopen =
            _request(CfdTypes.Side.LONG, SIZE, OPEN_MARGIN, false);

        vm.startPrank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector,
                uint8(CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH)
            )
        );
        router.commitOrder(reopen);
        vm.stopPrank();
    }

    /// A full close is already pending; a fresh same-side 300k open is falsely
    /// classified as an increase to 600k and rejected at commit time.
    function test_PoC_SameSideReopenBehindFullCloseFalselyRejected()
        public
    {
        uint64 closeId = _openLongAndQueueFullClose();
        _assertSkewRevertCode();

        uint64 expectedNextId = router.nextCommitId();
        _expectSameSideReject();

        assertEq(
            router.nextCommitId(),
            expectedNextId,
            "rejected V3 request should not be queued"
        );
        assertEq(
            router.accountHeadOrderId(ALICE),
            closeId,
            "original full close should remain pending"
        );

        console2.log("MED14: live LONG size", SIZE);
        console2.log("MED14: rejected planned increase", SIZE * 2);
        console2.log("MED14: prior full close order", uint256(closeId));
        console2.log("MED14: same-side reopen rejected with SKEW_TOO_HIGH");
    }

    /// Control A: the original MED-14 fix lets the opposite-side order queue
    /// behind the full close. Both orders then execute in FIFO order.
    function test_Control_OppositeSideQueuedReopenExecutes() public {
        uint64 closeId = _openLongAndQueueFullClose();

        uint64 shortOpenId =
            _commit(CfdTypes.Side.SHORT, SIZE, OPEN_MARGIN, false);
        assertGt(shortOpenId, closeId);

        _execute(closeId);
        (uint256 afterClose,,,,,,) = engine.positions(ALICE);
        assertEq(afterClose, 0, "the queued full close must execute");

        _execute(shortOpenId);
        (uint256 finalSize,,,, CfdTypes.Side finalSide,,) =
            engine.positions(ALICE);
        assertEq(finalSize, SIZE, "fresh short did not execute");
        assertEq(
            uint8(finalSide),
            uint8(CfdTypes.Side.SHORT),
            "post-close side is not SHORT"
        );

        console2.log("MED14 CONTROL: opposite-side V3 reopen executed");
    }

    /// Control B: the same LONG 300k request executes if submitted only after
    /// the queued close has finished. This proves the proposed open is valid.
    function test_Control_SameSideReopenExecutesAfterClose() public {
        uint64 closeId = _openLongAndQueueFullClose();
        _execute(closeId);

        (uint256 afterClose,,,,,,) = engine.positions(ALICE);
        assertEq(afterClose, 0, "first close did not execute");

        uint8 freshCode = engineLens.previewOpenRevertCode(
            ALICE,
            CfdTypes.Side.LONG,
            SIZE,
            OPEN_MARGIN,
            MARK_PRICE
        );
        assertEq(
            freshCode,
            uint8(CfdEnginePlanTypes.OpenRevertCode.OK),
            "fresh-position preview should accept same-side reopen"
        );

        uint64 reopenId =
            _commit(CfdTypes.Side.LONG, SIZE, OPEN_MARGIN, false);
        _execute(reopenId);

        (uint256 finalSize,,,, CfdTypes.Side finalSide,,) =
            engine.positions(ALICE);
        assertEq(finalSize, SIZE, "same-side reopen failed after close");
        assertEq(uint8(finalSide), uint8(CfdTypes.Side.LONG));

        console2.log("MED14 CONTROL: identical V3 LONG reopened after close");
    }

    /// Control C: without an earlier queued close, the exact 600k exposure
    /// would really violate the skew cap, so the rejection is justified here.
    function test_Control_RealLiveIncreaseCorrectlyRejected() public {
        _fundTrader(ALICE, 50_000e6);
        uint64 firstId =
            _commit(CfdTypes.Side.LONG, SIZE, OPEN_MARGIN, false);
        _execute(firstId);
        assertEq(router.accountHeadOrderId(ALICE), 0);

        _assertSkewRevertCode();
        _expectSameSideReject();

        console2.log("MED14 CONTROL: true same-side increase rejected");
    }
}
```

