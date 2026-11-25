# \#99453 [SC-Insight] Repeated curve and carry work adds 51,574 gas to live-position withdrawals

**Submitted on Nov 13th 2026 at 18:16:17 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99453
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro
A successful withdrawal from a healthy live position authenticates and synchronizes its terminal NAV curve while the clearinghouse realizes carry before debiting free settlement. Immediately after that debit, CfdEngine.checkWithdraw authenticates and synchronizes the same curve again, and its sidecar repeats a same-timestamp carry checkpoint. A minimal isolated optimization saved 51,574 gas on both public withdrawal APIs while preserving the token transfer and terminal-curve state. This is a code-optimization Insight; no fund loss or freeze is claimed.

## Vulnerability Details
MarginClearinghouse._withdraw calls _checkpointCarryBeforeMarginChange before reducing settlementBalances[account], then calls the Engine's checkWithdraw after the debit (MarginClearinghouse.sol:450-465). The pre-debit hook invokes CfdEngine.realizeCarryBeforeMarginChange, which authenticates the account's terminal curve, realizes carry, and synchronizes the curve (CfdEngine.sol:535-550).

For a live position, checkWithdraw starts and ends another terminal-curve mutation around settlementSidecar.validateWithdraw (CfdEngine.sol:730-747). The sidecar calls settlementRealizeCarry again before the post-debit health checks (CfdEngineSettlementSidecar.sol:255-284). Between the two Engine calls, the clearinghouse only debits free settlement. TerminalNavBookV2 forms the curve from position and pledge/claim data, not free settlement, so the second authenticate/sync sees the same curve. The first carry hook already advanced the account's carry checkpoint at this block timestamp, making the second call zero-due in the successful-path fixture.

The post-debit freshness, degraded-mode, carry-arrears, collateral, and pending-order-floor checks must remain. Only the repeated authentication/synchronization and carry checkpoint are candidates for removal.

## Impact Details
On identical pinned-revision Foundry harnesses, a 1 USDG withdraw call used 191,510 gas before the focused change and 139,936 gas after it: 51,574 gas saved (26.93%). withdrawMargin(1e6) used 190,183 versus 138,609 gas: also 51,574 saved (27.12%). Both calls transferred exactly 1 USDG and preserved the terminal-curve hash and book version; the Engine/curve parity assertion passed. These are local test gas figures, not a deployed-transaction measurement. The improvement applies to healthy live-position accounts withdrawing free settlement.

## Mitigation
Consolidate the pre-debit carry/curve checkpoint and post-debit eligibility check. In the pinned call graph, a focused variant retains the flat-account early return and all sidecar health checks but omits the second begin/end terminal-curve mutation in checkWithdraw and the repeated settlementRealizeCarry call in validateWithdraw. Keep the first authenticated sync. If a future clearinghouse path can call checkWithdraw without the prehook, make that precondition explicit or retain a fallback checkpoint.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L446-L476
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L535-L550
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L730-L747
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L299

## Proof of Concept

## Proof of Concept / condition of value

The following Foundry harness extends the repository's BasePerpTest. Save it as test/perps/spec/trader/WithdrawGas116.t.sol in a disposable checkout of pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;
import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";
contract WithdrawGas116Test is BasePerpTest {
    address private constant ALICE = address(0x116);
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 1_000_000e6; }
    function _prepare() private {
        _fundTrader(ALICE, 100_000e6);
        _open(ALICE, CfdTypes.Side.LONG, 50_000e18, 1_000e6, 1e8);
    }
    function test_WithdrawGas116() public {
        _prepare();
        bytes32 curve = terminalNavBook.curveHashOf(ALICE);
        uint64 version = terminalNavBook.bookState().bookVersion;
        uint256 beforeBalance = settlementToken.balanceOf(ALICE);
        vm.prank(ALICE);
        uint256 beforeGas = gasleft();
        clearinghouse.withdraw(ALICE, 1e6);
        console2.log("withdraw gas", beforeGas - gasleft());
        assertEq(settlementToken.balanceOf(ALICE), beforeBalance + 1e6);
        assertEq(terminalNavBook.curveHashOf(ALICE), curve);
        assertEq(terminalNavBook.bookState().bookVersion, version);
        _assertTerminalCurveMatchesEngine(ALICE);
    }
    function test_WithdrawMarginGas116() public {
        _prepare();
        bytes32 curve = terminalNavBook.curveHashOf(ALICE);
        uint64 version = terminalNavBook.bookState().bookVersion;
        uint256 beforeBalance = settlementToken.balanceOf(ALICE);
        vm.prank(ALICE);
        uint256 beforeGas = gasleft();
        clearinghouse.withdrawMargin(1e6);
        console2.log("withdrawMargin gas", beforeGas - gasleft());
        assertEq(settlementToken.balanceOf(ALICE), beforeBalance + 1e6);
        assertEq(terminalNavBook.curveHashOf(ALICE), curve);
        assertEq(terminalNavBook.bookState().bookVersion, version);
        _assertTerminalCurveMatchesEngine(ALICE);
    }
}
```

Run: forge test --root packages/perps --match-path test/perps/spec/trader/WithdrawGas116.t.sol -vv

For an A/B comparison, in a separate disposable copy of the same revision keep the flat-account early return and sidecar health validation, omit only the second _beginTerminalCurveMutation/_endTerminalCurveMutation pair in CfdEngine.checkWithdraw, and omit the second settlementRealizeCarry call in CfdEngineSettlementSidecar.validateWithdraw. Run the same harness. Both tests passed before and after; each call saved 51,574 gas. The first pre-debit authenticated sync and all post-debit guards remain.
