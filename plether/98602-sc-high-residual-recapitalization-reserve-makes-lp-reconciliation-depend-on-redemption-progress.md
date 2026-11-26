# \#98602 [SC-High] Residual recapitalization reserve makes LP reconciliation depend on redemption progress

**Submitted on Nov 8th 2026 at 23:30:20 UTC by @adeolu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98602
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description


## Brief/Intro

`HousePool.settleLpEpoch` can enter a circular no-progress state after a legitimate recapitalization: reconciliation provisionally allocates backed recapitalization to the tranches, but the residual pending recapitalization is then reserved against withdrawal liquidity, leaving a positive matured redemption with a zero funding budget. The no-progress guard reverts the entire transaction and thereby rolls back the reconciliation that would reduce the reserve. Ordinary permissionless settlement, elapsed time, and even full payment of the trader claims therefore cannot advance the affected redemption. The supplied PoC demonstrates the freeze for more than 24 hours. An immediate escape requires additional owner capital; a capital-free route exists through a Senior-rate configuration finalization, but it is subject to a 72-hour timelock, requires an unrelated configuration change, and interacts with the separate pending-recapitalization replay behavior. The correct impact is therefore **High — temporary freezing of funds for at least 24 hours**, not permanent or necessarily indefinite freezing.

## Vulnerability Details

### Recapitalization is intentionally queued for later reconciliation

The owner recovery entrypoint transfers real settlement assets to `HousePool`, then records the entire received amount as pending recapitalization. Allocation is deliberately deferred to normal pool reconciliation:

```solidity
function recapitalizePool(
    uint256 amountAssets
) external onlyOwner nonReentrant {
    // ...
    uint256 balanceBefore = settlementToken.balanceOf(address(pool));
    settlementToken.safeTransferFrom(msg.sender, address(pool), amountAssets);
    uint256 balanceAfter = settlementToken.balanceOf(address(pool));
    uint256 received = balanceAfter >= balanceBefore ? balanceAfter - balanceBefore : 0;
    if (received != amountAssets) {
        revert CfdEngine__UnexpectedTransferAmount(amountAssets, received);
    }
    pool.recordClaimantInflow(
        amountAssets, IHousePool.ClaimantInflowKind.Recapitalization,
        IHousePool.ClaimantInflowCashMode.CashArrived
    );
}
```

Source: [`CfdEngine.sol` lines 627-653](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L653).

`recordClaimantInflow` increases canonical accounted assets and adds the same amount to the recapitalization bucket:

```solidity
if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
    accountedAssets += amount;
}

if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
}
```

Source: [`HousePool.sol` lines 624-648](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L648).

This behavior is documented: normal reconciliation is expected to allocate the queued amount to claimant ownership. See [`ACCOUNTING_SPEC.md` lines 628-650](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L628-L650).

### Settlement provisionally reconciles before calculating withdrawal liquidity

LP epoch settlement first invokes `_reconcile`. It then builds the withdrawal snapshot using the post-reconciliation residual pending bucket:

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

// ...
uint256 freeAssets =
    _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets;

RedemptionPhase memory seniorPhase = RedemptionPhase({
    pricingPrincipal: seniorPrincipal,
    pricingSupply: _seniorShareSupply(),
    budget: _min(freeAssets, seniorPrincipal),
    // ...
});
```

Source: [`HousePool.sol` lines 823-853](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L853).

`_reconcile` caps the pending claimant amount against current terminal distributable equity and provisionally applies it:

```solidity
HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot =
    HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot);
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
    _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);

// ...
_applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
```

Source: [`HousePool.sol` lines 1555-1598](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1598).

Applying the bucket decreases pending storage and restores the waterfall state:

```solidity
HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
    state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
);
_decreasePendingClaimantBuckets(settleableClaimantBuckets);

_setWaterfallState(state.waterfall);
unassignedAssets = state.unassignedAssets;
```

Source: [`HousePool.sol` lines 1759-1776](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1776).

### The residual bucket consumes the same liquidity needed to commit reconciliation

The withdrawal snapshot reserves all residual pending claimant assets:

```solidity
snapshot = HousePoolAccountingLib.buildWithdrawalSnapshot(accountingSnapshot);
snapshot = HousePoolWithdrawalPreviewLib.reserveAssets(snapshot, reservedPendingClaimantAssets);
snapshot = HousePoolWithdrawalPreviewLib.reserveAssets(snapshot, reservedUnassignedAssets);
```

Source: [`HousePool.sol` lines 1727-1735](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1727-L1735).

Let:

- `R` be the pending recapitalization;
- `D` be terminal distributable equity; and
- all trader claims have been paid, leaving physical cash equal to `D`.

Reconciliation can apply at most `D`, so the residual bucket is `R - D`. Withdrawal free assets then become:

```text
freeAssets = max(D - (R - D), 0)
           = max(2D - R, 0)
```

Therefore `R >= 2D` is a sufficient zero-budget condition. The PoC derives `R` and `D` from executed protocol state and directly asserts this condition; it does not hardcode the balances.

When the matured redemption has positive value but `budget == 0`, `_fundRedemptionPhase` stops without processing it:

```solidity
if (phase.fundedAssets >= phase.budget) {
    break;
}
```

Source: [`HousePool.sol` lines 944-1003](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L1003).

Because no redemption or deposit epoch advanced, the final guard reverts:

```solidity
if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
    revert HousePool__NoLpEpochProgress();
}
```

Source: [`HousePool.sol` lines 1179-1185](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1179-L1185).

The revert is atomic. It restores the full pending recapitalization, tranche principals, and Senior high-water mark. Every ordinary settlement retry therefore starts from the same state and repeats the same revert.

### Paying the claims does not resolve the circularity

Claim payment removes the trader-claim liability and the corresponding cash together. In the PoC, it leaves exactly `D` of physical cash but does not consume any pending recapitalization because all prior reconciliation attempts reverted. Consequently the same `R >= 2D` reserve geometry remains and settlement still reverts.

### A timelocked capital-free route exists, but it is not normal LP recovery

This report does not claim the funds are necessarily frozen forever. A change to `seniorRateBps` causes `finalizePoolConfig` to invoke `_reconcile` without the LP settlement no-progress guard:

```solidity
if (nextConfig.seniorRateBps != currentConfig.seniorRateBps) {
    HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot = _getHousePoolStatusSnapshot();
    _requireRateChangeMarkFresh(statusSnapshot);
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory accountingSnapshot = _getHousePoolInputSnapshot();
    _requireFreshMark(accountingSnapshot, statusSnapshot);
    _reconcile(accountingSnapshot, statusSnapshot);
}
```

Source: [`HousePool.sol` lines 323-342](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L323-L342).

The proposal cannot be finalized until the 72-hour timelock expires:

```solidity
uint256 public constant TIMELOCK_DELAY = 72 hours;

// ...
poolConfigActivationTime = block.timestamp + TIMELOCK_DELAY;
```

Sources: [`HousePool.sol` lines 188-195](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L188-L195) and [`HousePool.sol` lines 300-321](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L300-L321).

The PoC snapshots the frozen state and proves that a one-basis-point Senior-rate proposal, a full 72-hour wait, an authorized mark refresh, and finalization can commit one reconciliation without new capital. The subsequent settlement advances the same queue head in this test geometry. The test then restores the snapshot and separately demonstrates the immediate additional-capital workaround.

This configuration route is not presented as a safe repair. It requires an economically unrelated parameter change, depends on owner action and a fresh mark, and the subsequent progress interacts with the separate pending-recapitalization replay behavior in which current distributable backing can be reused by another reconciliation. It only establishes that “additional capital or a code change is the sole possible escape” would be too broad.

## Impact Details

**Selected in-scope impact: High — temporary freezing of funds for at least 24 hours.**

The affected assets are a matured Senior LP redemption with positive canonical value. The redemption cannot be funded or claimed while ordinary settlement reverts. The PoC begins with no pre-existing configuration proposal, retries settlement after 25 hours, and proves the identical `HousePool__NoLpEpochProgress` revert and unchanged pending bucket. Paying every authenticated trader claim still does not advance the redemption.

The deterministic PoC run produced these live-derived values:

| Measurement | USDG |
|---|---:|
| Pre-recapitalization pool cash | 344,032.102520 |
| Authenticated trader claims | 991,182.408000 |
| Legitimate recapitalization `R` | 982,150.323744 |
| Terminal distributable equity `D` | 335,000.018264 |
| Immediate additional-capital workaround | 345,650.289042 |

The additional `345,650.289042 USDG` is not claimed to be universally required. It is the dynamically derived amount used in this test to create a progress budget equal to ten percent of `D`. Without that immediate bailout, the normal settlement path remains frozen. The demonstrated capital-free control is unavailable for 72 hours from proposal and is not a safe accounting fix.

This is not a Critical permanent freeze: existing code contains the delayed configuration-finalization route, and sufficient new liquidity can also restore progress immediately. It is nevertheless a High temporary freeze because the normal permissionless withdrawal mechanism remains unusable beyond 24 hours and recovery requires either a substantial external bailout or a delayed, unrelated privileged action.

## References

- [`CfdEngine.recapitalizePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L653)
- [`HousePool._settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L941)
- [`HousePool._reconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1598)
- [`HousePool._applyPendingClaimantBucketsLive`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1776)
- [`HousePool._buildWithdrawalSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1727-L1735)
- [`HousePool._emitLpEpochSettled`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1179-L1192)
- [`HousePool.finalizePoolConfig`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L323-L365)
- [Canonical accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L257-L272)
- [Owner-funded recapitalization security documentation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L155-L163)



## Proof of Concept


## Proof of Concept

### Prerequisites

- Git
- Foundry with support for Solidity `0.8.35`

No public network, fork, RPC endpoint, or new mock contract is required. The test uses the repository's existing `BasePerpTest` environment and production Engine, HousePool, Clearinghouse, Router, and TrancheVault contracts. The existing test settlement token is used only to fund external actors through normal entrypoints.

### Reproduction from a fresh clone

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

Create `packages/perps/test/perps/regression/ResidualRecapitalizationReserve.t.sol` with the complete contents below:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice Reproduces the circular dependency between pending recapitalization reconciliation and LP epoch progress.
/// @dev Every protocol state transition uses the production Engine, HousePool, Clearinghouse, and tranche-vault APIs.
///      The settlement token is minted only to external actors so they can fund normal public entrypoints.
contract ResidualRecapitalizationReserveRegressionTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0x51A1CE);
    address internal constant LONG_TRADER_ONE = address(0x10A601);
    address internal constant LONG_TRADER_TWO = address(0x10A602);
    address internal constant SHORT_TRADER_ONE = address(0x5A0701);
    address internal constant SHORT_TRADER_TWO = address(0x5A0702);

    function test_ResidualRecapitalizationReserveMakesReconcileDependOnRedemptionProgress() public {
        uint256 seniorDepositAssets = pool.totalAssets() / 3;
        _fundSenior(SENIOR_LP, seniorDepositAssets);

        uint256 seniorShares = seniorVault.balanceOf(SENIOR_LP);
        assertGt(seniorShares, 0, "Senior LP must enter through the real asynchronous vault flow");

        uint256 entryPrice = engine.lastMarkPrice();
        uint256 poolAssetsBeforeTrading = pool.totalAssets();
        CfdTypes.RiskParams memory riskParams = ICfdEngineRiskParamsView(address(engine)).riskParams();
        uint256 configuredSkewCapacity = poolAssetsBeforeTrading * riskParams.maxSkewRatio / 1e18;
        uint256 targetMaxProfitAssets = configuredSkewCapacity - configuredSkewCapacity / 16;
        uint256 positionLots = targetMaxProfitAssets / entryPrice;
        uint256 positionSize = positionLots * CfdTypes.SIZE_QUANTUM;
        uint256 positionMaxProfitAssets = positionLots * entryPrice;
        uint256 positionMarginAssets = positionMaxProfitAssets * riskParams.initMarginBps * 2 / 10_000;
        uint256 traderFundingAssets = positionMarginAssets + positionMaxProfitAssets / 100;

        _fundTrader(LONG_TRADER_ONE, traderFundingAssets);
        _fundTrader(LONG_TRADER_TWO, traderFundingAssets);
        _fundTrader(SHORT_TRADER_ONE, traderFundingAssets);
        _fundTrader(SHORT_TRADER_TWO, traderFundingAssets);
        _open(LONG_TRADER_ONE, CfdTypes.Side.LONG, positionSize, positionMarginAssets, entryPrice);
        _open(SHORT_TRADER_ONE, CfdTypes.Side.SHORT, positionSize, positionMarginAssets, entryPrice);
        _open(LONG_TRADER_TWO, CfdTypes.Side.LONG, positionSize, positionMarginAssets, entryPrice);
        _open(SHORT_TRADER_TWO, CfdTypes.Side.SHORT, positionSize, positionMarginAssets, entryPrice);

        vm.warp(seniorVault.lastDepositTime(SENIOR_LP) + seniorVault.DEPOSIT_COOLDOWN());
        vm.prank(SENIOR_LP);
        uint256 redeemId = seniorVault.requestRedeem(seniorShares / 2, SENIOR_LP, SENIOR_LP);
        uint256 redeemShares = seniorVault.pendingRedeemRequest(redeemId, SENIOR_LP);
        assertEq(redeemShares, seniorShares / 2, "half of the real Senior position must be queued");

        uint256 lowPrice = entryPrice / 100;
        _closeLiquidLong(LONG_TRADER_ONE, positionSize, lowPrice);
        _closeLiquidLong(LONG_TRADER_TWO, positionSize, lowPrice);
        assertTrue(engine.degradedMode(), "real payouts must expose insolvency against the remaining SHORT liability");

        uint256 highPrice = engine.CAP_PRICE() - lowPrice;
        _closeDeferredShort(SHORT_TRADER_ONE, positionSize, highPrice);
        _closeDeferredShort(SHORT_TRADER_TWO, positionSize, highPrice);

        uint256 firstTraderClaimAssets = engine.traderClaimBalanceAssets(SHORT_TRADER_ONE);
        uint256 secondTraderClaimAssets = engine.traderClaimBalanceAssets(SHORT_TRADER_TWO);
        uint256 traderClaimAssets = firstTraderClaimAssets + secondTraderClaimAssets;
        uint256 preRecapitalizationCash = pool.totalAssets();
        assertEq(traderClaimAssets, engine.totalTraderClaimBalanceAssets(), "production closes must create the claims");
        assertGt(traderClaimAssets, preRecapitalizationCash, "the executed state must have a real cash shortfall");

        _assertAllPositionsClosed();

        uint256 targetDistributableEquity = pool.seniorPrincipal();
        uint256 cashShortfall = traderClaimAssets - preRecapitalizationCash;
        assertGe(
            cashShortfall,
            targetDistributableEquity,
            "organic trading shortfall must be large enough for the residual reserve to exhaust free cash"
        );

        uint256 recapitalizationAssets = cashShortfall + targetDistributableEquity;
        assertGe(
            recapitalizationAssets,
            2 * targetDistributableEquity,
            "recapitalization must satisfy the R >= 2D zero-budget failure condition"
        );
        _recapitalizeThroughEngine(recapitalizationAssets);
        engine.clearDegradedMode();

        assertFalse(engine.degradedMode(), "owner must clear degraded mode through the production recovery path");
        assertEq(pool.pendingRecapitalizationAssets(), recapitalizationAssets);
        assertEq(
            pool.totalAssets() - engine.totalTraderClaimBalanceAssets(),
            targetDistributableEquity,
            "recapitalization must leave positive terminal distributable equity"
        );
        emit log_named_uint("pre-recapitalization cash (USDG atoms)", preRecapitalizationCash);
        emit log_named_uint("trader claims (USDG atoms)", traderClaimAssets);
        emit log_named_uint("recapitalization (USDG atoms)", recapitalizationAssets);
        emit log_named_uint("terminal distributable equity D (USDG atoms)", targetDistributableEquity);

        vm.warp(pool.lpEpochStart(redeemId));
        assertGt(seniorVault.estimateRedeemAssets(redeemShares), 0, "the matured redemption head must have value");
        assertEq(pool.poolConfigActivationTime(), 0, "the freeze must start without a pre-existing config proposal");

        uint256 seniorPrincipalBefore = pool.seniorPrincipal();
        uint256 juniorPrincipalBefore = pool.juniorPrincipal();
        uint256 seniorHighWaterMarkBefore = pool.seniorHighWaterMark();

        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);

        assertEq(pool.pendingRecapitalizationAssets(), recapitalizationAssets, "revert restores the pending bucket");
        assertEq(pool.seniorPrincipal(), seniorPrincipalBefore, "provisional Senior reconciliation must roll back");
        assertEq(pool.juniorPrincipal(), juniorPrincipalBefore, "provisional Junior reconciliation must roll back");
        assertEq(pool.seniorHighWaterMark(), seniorHighWaterMarkBefore, "provisional HWM update must roll back");
        assertEq(seniorVault.pendingRedeemRequest(redeemId, SENIOR_LP), redeemShares);

        vm.warp(block.timestamp + 25 hours);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);
        assertEq(pool.pendingRecapitalizationAssets(), recapitalizationAssets, "time alone cannot break the cycle");

        vm.prank(SHORT_TRADER_ONE);
        engine.settleTraderClaim(SHORT_TRADER_ONE);
        vm.prank(SHORT_TRADER_TWO);
        engine.settleTraderClaim(SHORT_TRADER_TWO);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
        assertEq(
            pool.totalAssets(),
            targetDistributableEquity,
            "production claim settlement must leave exactly the terminal distributable cash"
        );

        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);
        assertEq(pool.pendingRecapitalizationAssets(), recapitalizationAssets);
        assertEq(seniorVault.pendingRedeemRequest(redeemId, SENIOR_LP), redeemShares);

        // Capital-free liveness control. A Senior-rate finalization invokes _reconcile outside LP settlement's
        // no-progress guard. It is delayed by 72 hours and is not a safe accounting repair: the following settlement
        // reuses current distributable backing through the separate pending-recapitalization replay behavior.
        uint256 timelockedRecoveryState = vm.snapshotState();
        uint256 proposalTime = block.timestamp;
        uint256 recoveryCashBefore = pool.totalAssets();
        IHousePool.PoolConfig memory recoveryConfig = _currentPoolConfig();
        recoveryConfig.seniorRateBps += 1;
        pool.proposePoolConfig(recoveryConfig);
        uint256 activationTime = pool.poolConfigActivationTime();
        assertEq(
            activationTime - proposalTime, pool.TIMELOCK_DELAY(), "capital-free control must wait the full timelock"
        );

        vm.warp(activationTime);
        uint256 recoveryMarkPrice = engine.lastMarkPrice();
        uint64 recoveryMarkTime = uint64(vm.getBlockTimestamp());
        vm.prank(address(router));
        engine.updateMarkPrice(recoveryMarkPrice, recoveryMarkTime);
        pool.finalizePoolConfig();

        assertEq(pool.totalAssets(), recoveryCashBefore, "config finalization must introduce no additional capital");
        assertEq(
            pool.pendingRecapitalizationAssets(),
            recapitalizationAssets - targetDistributableEquity,
            "timelocked finalization must commit one reconciliation outside the no-progress guard"
        );
        IHousePool.LpEpochSettlementResult memory timelockedRecovery = pool.settleLpEpoch(0, 0);
        assertGt(timelockedRecovery.seniorFundedShares, 0, "timelocked control must advance the same queue head");
        assertGt(timelockedRecovery.seniorFundedAssets, 0, "timelocked control must fund the same queue head");
        assertTrue(vm.revertToState(timelockedRecoveryState));
        vm.deleteStateSnapshot(timelockedRecoveryState);

        uint256 stalledResidualAssets = recapitalizationAssets - targetDistributableEquity;
        uint256 desiredProgressBudget = targetDistributableEquity / 10;
        uint256 additionalRecapitalizationAssets =
            stalledResidualAssets - targetDistributableEquity + desiredProgressBudget;
        emit log_named_uint("immediate additional-capital workaround (USDG atoms)", additionalRecapitalizationAssets);
        _recapitalizeThroughEngine(additionalRecapitalizationAssets);

        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);

        assertGt(result.seniorFundedShares, 0, "additional production recapitalization must advance the queue");
        assertGt(result.seniorFundedAssets, 0, "additional production recapitalization must fund the queue");
        assertLe(result.seniorFundedAssets, desiredProgressBudget, "funding cannot exceed the derived free budget");
        assertEq(
            pool.pendingRecapitalizationAssets(),
            stalledResidualAssets,
            "successful settlement must persist the partial reconciliation"
        );
        assertLt(seniorVault.pendingRedeemRequest(redeemId, SENIOR_LP), redeemShares);
    }

    function _closeLiquidLong(address trader, uint256 positionSize, uint256 closePrice) internal {
        ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(trader, positionSize, closePrice);
        assertTrue(preview.valid, "LONG close must be executable");
        assertGt(preview.freshTraderPayoutAssets, 0, "LONG close must earn a real pool-funded payout");
        assertLt(preview.freshTraderPayoutAssets, pool.totalAssets(), "LONG payout must be liquid");

        uint256 cashBefore = pool.totalAssets();
        _close(trader, CfdTypes.Side.LONG, positionSize, closePrice);

        assertEq(engine.traderClaimBalanceAssets(trader), 0, "liquid LONG profit must be paid immediately");
        assertLt(pool.totalAssets(), cashBefore, "production LONG close must reduce pool cash");
    }

    function _closeDeferredShort(address trader, uint256 positionSize, uint256 closePrice) internal {
        ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(trader, positionSize, closePrice);
        assertTrue(preview.valid, "SHORT close must be executable in degraded mode");

        uint256 physicalAssets = pool.totalAssets();
        uint256 existingClaims = engine.totalTraderClaimBalanceAssets();
        uint256 freshPayoutCash = physicalAssets > existingClaims ? physicalAssets - existingClaims : 0;
        assertGt(
            preview.freshTraderPayoutAssets,
            freshPayoutCash,
            "SHORT profit must exceed cash available after existing claim priority"
        );

        uint256 claimBefore = engine.traderClaimBalanceAssets(trader);
        _close(trader, CfdTypes.Side.SHORT, positionSize, closePrice);
        assertEq(
            engine.traderClaimBalanceAssets(trader) - claimBefore,
            preview.freshTraderPayoutAssets,
            "production SHORT close must record its full fresh payout as a claim"
        );
    }

    function _assertAllPositionsClosed() internal view {
        (uint256 longOneSize,,,,,,) = engine.positions(LONG_TRADER_ONE);
        (uint256 longTwoSize,,,,,,) = engine.positions(LONG_TRADER_TWO);
        (uint256 shortOneSize,,,,,,) = engine.positions(SHORT_TRADER_ONE);
        (uint256 shortTwoSize,,,,,,) = engine.positions(SHORT_TRADER_TWO);
        assertEq(
            longOneSize + longTwoSize + shortOneSize + shortTwoSize,
            0,
            "all production positions must close before LP settlement"
        );
    }

    function _recapitalizeThroughEngine(
        uint256 amountAssets
    ) internal {
        settlementToken.mint(address(this), amountAssets);
        settlementToken.approve(address(engine), amountAssets);
        engine.recapitalizePool(amountAssets);
    }

}
```

Run the test from the perps package:

```bash
cd packages/perps
forge test \
  --match-path test/perps/regression/ResidualRecapitalizationReserve.t.sol \
  --match-test test_ResidualRecapitalizationReserveMakesReconcileDependOnRedemptionProgress \
  -vv
```

Expected result:

```text
Ran 1 test for test/perps/regression/ResidualRecapitalizationReserve.t.sol:ResidualRecapitalizationReserveRegressionTest
[PASS] test_ResidualRecapitalizationReserveMakesReconcileDependOnRedemptionProgress()
Logs:
  pre-recapitalization cash (USDG atoms): 344032102520
  trader claims (USDG atoms): 991182408000
  recapitalization (USDG atoms): 982150323744
  terminal distributable equity D (USDG atoms): 335000018264
  immediate additional-capital workaround (USDG atoms): 345650289042

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The PoC establishes all of the following without direct storage writes or hand-built protocol state:

1. A real Senior LP deposit is activated through the asynchronous vault flow.
2. A real redemption for half the LP's shares is requested and matures.
3. Production Engine opens and closes create authenticated deferred trader claims and degraded mode.
4. The owner uses `CfdEngine.recapitalizePool` and `clearDegradedMode` normally.
5. The derived relationship `R >= 2D` holds.
6. The matured redemption has positive value but settlement reverts with `HousePool__NoLpEpochProgress`.
7. The revert restores the pending bucket, principals, HWM, and queue head.
8. A retry 25 hours later still reverts.
9. Full payment of all trader claims still does not unblock settlement.
10. A separate snapshot branch proves that capital-free progress requires a full 72-hour configuration timelock and an unrelated Senior-rate finalization in this state.
11. Restoring the frozen snapshot and providing the dynamically derived additional capital immediately advances the same queue head.

