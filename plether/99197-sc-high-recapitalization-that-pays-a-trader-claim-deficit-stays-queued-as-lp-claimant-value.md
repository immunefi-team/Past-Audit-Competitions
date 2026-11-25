# \#99197 [SC-High] Recapitalization that pays a trader-claim deficit stays queued as LP claimant value

**Submitted on Nov 12th 2026 at 11:07:58 UTC by @Nyksx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99197
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Summary

When trader claims exceed pool cash and the owner funds the shortfall through `CfdEngine.recapitalizePool()`, the amount is queued in `HousePool.pendingRecapitalizationAssets`, but the claimant's later `settleTraderClaim()` pays that cash out through `HousePool.payOut()` without reducing the queue. The unbacked residual is then reserved from LP withdrawals, so a later partial restoration of the senior tranche cannot be redeemed and `settleLpEpoch()` reverts with `HousePool__NoLpEpochProgress`, while a full restoration sets the senior high-water mark above the restored principal by about the deficit, so later trading revenue that would reach junior holders is credited to senior. Revenue larger than that gap which arrives while the residual is still queued is reduced by the residual before the waterfall runs, and the residual is then applied as recapitalization, so up to the deficit of the junior holders' revenue is stored in `unassignedAssets`, where it blocks deposit requests on both vaults until the owner assigns it to a receiver of its choice.

## Finding Description

A closing trader whose payout exceeds the cash the pool can pay keeps the remainder as a trader claim. SECURITY.md:1501-1502 describes the recovery runbook for "trader payout claims that physical cash cannot currently service", whose steps are "4. Recapitalize or clear positions." and "5. Clear degraded mode only after raw solvency `E >= L` is genuinely restored" (SECURITY.md:1509-1510). Following that runbook with `recapitalizePool()` leaves a recapitalization intent in the pool that no cash backs.

**1. Each recapitalization is queued as claimant intent.** `recapitalizePool()` records the inflow as `Recapitalization`, and `recordClaimantInflow()` adds the full amount to `pendingRecapitalizationAssets`, whether or not the cash is needed to service outstanding trader claims:

```solidity
// packages/perps/src/CfdEngine.sol:649-651 (a82d4fef), recapitalizePool()
pool.recordClaimantInflow(
    amountAssets, IHousePool.ClaimantInflowKind.Recapitalization, IHousePool.ClaimantInflowCashMode.CashArrived
);

// packages/perps/src/HousePool.sol:640-648 (a82d4fef), recordClaimantInflow()
if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
    accountedAssets += amount;
}

if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount); // pendingRecapitalizationAssets += amount, also when claims > cash
} else if (seniorPrincipal + juniorPrincipal == 0) {
    _recordPendingClaimantInflow(kind, amount);
}
```

**2. While claims exceed cash nothing of the queue is settled, and the claim payout does not reduce it.** Reconciliation settles the queue only up to distributable terminal equity, which is zero while trader claims exceed pool assets. Once the recapitalization makes `pool.totalAssets() >= totalTraderClaimBalanceAssets`, the claimant can call `settleTraderClaim()`, which pays the claim through `payOut()`. `payOut()` lowers `accountedAssets` and transfers the cash but leaves both claimant buckets unchanged:

```solidity
// packages/perps/src/HousePool.sol:1829-1834 (a82d4fef)
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable); // 0 while C > P
}

// packages/perps/src/CfdEngine.sol (a82d4fef):
// settleTraderClaim() (:577) -> _settleTraderClaimBalance() (:224-229) -> _payTraderClaim() (:238)
if (amount == 0 || pool.totalAssets() < totalTraderClaimBalanceAssets) { // passes once the recapitalization arrived
    revert CfdEngine__InsufficientPoolLiquidity();
}
claimAmountAssets = amount;
_payTraderClaim(account, claimAmountAssets, positionRemainsOpen);
// ...
pool.payOut(address(clearinghouse), amountAssets);

// packages/perps/src/HousePool.sol:604-613 (a82d4fef)
function payOut(
    address recipient,
    uint256 amount
) external {
    if (msg.sender != address(ENGINE) && msg.sender != ENGINE.settlementSidecar()) {
        revert HousePool__Unauthorized();
    }
    accountedAssets -= amount; // the recapitalized cash leaves the pool ...
    settlementToken.safeTransfer(recipient, amount); // ... but pendingRecapitalizationAssets keeps the full amount
}
```

In the proof of concept, after the claimant settles, the pool holds 0 USDG while `pendingRecapitalizationAssets` is 16,582.14 USDG, the amount that was paid to the claimant.

**3. The unbacked residual is reserved from LP withdrawals.** Later reconciles settle the queue only up to distributable equity, and the unsettled remainder of the queue is reserved out of free cash before senior redemptions are budgeted. When the owner later restores less than the residual, the restored cash is assigned to senior principal but the senior redemption budget is zero, and the settlement reverts, which also rolls back the reconcile:

```solidity
// packages/perps/src/HousePool.sol:842-848 (a82d4fef), _settleLpEpoch() after _reconcile()
uint256 freeAssets =
    _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets; // cash - residual
// ...
    budget: _min(freeAssets, seniorPrincipal), // 0 while the residual is at least the restored cash

// packages/perps/src/HousePool.sol:1727-1735 (a82d4fef)
function _buildWithdrawalSnapshot(
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory accountingSnapshot,
    uint256 reservedUnassignedAssets,
    uint256 reservedPendingClaimantAssets
) internal pure returns (HousePoolAccountingLib.WithdrawalSnapshot memory snapshot) {
    snapshot = HousePoolAccountingLib.buildWithdrawalSnapshot(accountingSnapshot);
    snapshot = HousePoolWithdrawalPreviewLib.reserveAssets(snapshot, reservedPendingClaimantAssets); // unbacked residual reserved
    snapshot = HousePoolWithdrawalPreviewLib.reserveAssets(snapshot, reservedUnassignedAssets);
}

// packages/perps/src/HousePool.sol:1183-1185 (a82d4fef), _emitLpEpochSettled()
if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
    revert HousePool__NoLpEpochProgress();
}
```

**4. A full restoration takes the senior high-water mark from the whole queue.** When both tranche principals are zero, the settled part of the queue bootstraps senior principal and the high-water mark is set to the full outstanding intent, which still contains the already-paid deficit:

```solidity
// packages/perps/src/HousePool.sol:1769-1772 (a82d4fef), _applyPendingClaimantBucketsLive()
HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
    state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation // claimantBuckets = full queue incl. paid deficit
);
_decreasePendingClaimantBuckets(settleableClaimantBuckets);

// packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:111-122 (a82d4fef)
function applyClaimantRecapitalizationIntent(
    PendingAccountingState memory state,
    uint256 amount,
    uint256 recapitalizationTargetAssets
) internal pure {
    uint256 remaining = amount;
    if (state.seniorSupply > 0) {
        if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
            state.waterfall.seniorPrincipal += remaining;
            state.waterfall.seniorHighWaterMark =
                recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining; // = restoration + deficit
            remaining = 0;
```

Revenue restores senior principal up to its high-water mark before any of it reaches junior principal (`applyRevenueIntent()`), so after a full restoration the next revenue, up to about the deficit, goes to senior while junior holders still hold their shares. In the proof of concept the senior high-water mark after a full restoration is 56,017.33 USDG, above the 40,001.00 USDG of senior principal held before the loss, so the mark exceeds any principal the senior tranche held at any point in the test.

**5. Revenue that arrives while the residual is queued is applied as recapitalization, and the part beyond the senior gap is stored in `unassignedAssets`.** Each reconcile takes the settleable part of the queue out of distributable equity, plans the waterfall on what is left, and applies the queue afterwards as recapitalization. Recapitalization restores senior principal up to its high-water mark, and the remainder becomes unassigned:

```solidity
// packages/perps/src/HousePool.sol:1564-1575, 1598 (a82d4fef), _reconcile()
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
    _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets); // min(queue, distributable)
bool allowRevenueContinuation = seniorPrincipal + juniorPrincipal != 0;
HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
    _currentPendingAccountingState(),
    reconcileSnapshot,
    _pendingClaimantBucketAssets(settleableClaimantBuckets), // pendingBucketAssets
    // ...
);
// ...
_applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation); // queue applied last

// packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-71 (a82d4fef), planReconcile()
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0; // waterfall sees X - d
}

// packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:123-136 (a82d4fef), applyClaimantRecapitalizationIntent()
} else {
    uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
        ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
        : 0;
    if (gap > 0) {
        uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
        state.waterfall.seniorPrincipal += seniorAssignedAssets;
        remaining -= seniorAssignedAssets;
    }
}
// ...
if (remaining > 0) {
    state.unassignedAssets += remaining; // "recapitalization never accrues to junior principal" (:107)
}
```

After the full restoration of item 4 and the LP epoch that settles it, pool cash equals senior principal (39,601.00 USDG in the proof of concept), the senior gap `g` to the high-water mark is 16,416.32 USDG, and the residual `d` of 16,582.14 USDG is still queued. If the next LP settlement runs before any revenue, the residual is settled against the existing cash, `pendingRecapitalizationAssets` returns to 0, and later revenue follows the ordinary waterfall: senior up to its mark, then junior. If trading revenue `X` reaches the pool first, the two steps above split it differently. The waterfall sees a gain of only `X - d`, which restores senior towards its mark and then credits junior. The remaining `d` of the same cash is applied as recapitalization, which fills what is left of the senior gap and sends the rest to `unassignedAssets`. Junior holders therefore receive `min(d, X - g)` less than in the other order when `X > g`, and that amount is stored in `unassignedAssets`. When `X <= g`, all of `X` restores senior in both orders and the two orders end the same.

The revenue in this path is not queued revenue. `recordClaimantInflow()` queues trading revenue only while both stored principals are zero (HousePool.sol:644-648), and here senior principal is nonzero, so `X` is ordinary pool cash and the only queued amount is the residual, which the owner's payment to the trader claimant created and which no cash backs.

A nonzero `unassignedAssets` closes both deposit gates, and only the owner can clear it, to one receiver:

```solidity
// packages/perps/src/libraries/HousePoolTrancheGateLib.sol:28-30 (a82d4fef), trancheDepositsAllowed()
if (!ordinaryDepositsAllowed || paused || unassignedAssets > 0 || !markFreshForReconcile) {
    return false;
}

// packages/perps/src/HousePool.sol:660-663, 699-702 (a82d4fef)
function assignUnassignedAssets(
    bool toSenior,
    address receiver
) external onlyOwner {
// ...
    juniorPrincipal += amount;
}
unassignedAssets = 0;
ITrancheVaultBootstrap(targetVault).bootstrapMint(shares, receiver); // new shares for the whole amount go to `receiver`
```

The assignment mints new shares for the whole amount to the receiver the owner names, so the existing junior holders' value does not rise unless the owner names them, and with more than one junior holder a single receiver cannot reproduce their pro-rata shares.

The behaviour occurs when trader claims exceed pool cash after both tranches have absorbed losses, the owner recapitalizes the shortfall with `recapitalizePool()` and the claimant settles. It applies to any amount that is later restored while the residual is still queued: a restoration smaller than the residual cannot be redeemed, a restoration while both principals are zero inherits the inflated high-water mark, and revenue above the senior gap that arrives before the residual is settled is partly stored in `unassignedAssets`.

The behaviour depends on the owner using `recapitalizePool()` to fund the claim deficit. A direct USDG transfer to the pool followed by `HousePool.accountExcess()` also funds the claim, and on that path nothing is queued: in the same scenario `pendingRecapitalizationAssets` stays 0 after the claim is paid, and a later 5,000 USDG restoration is redeemed by the senior LP at the next epoch. The documentation describes `recapitalizePool()` as the owner's recovery action for this step. SECURITY.md:155-163 ("Owner-funded recapitalization") describes it as "an immediate, owner-only, non-reentrant recovery action" (SECURITY.md:157) and states that "Existing excess cannot substitute for the incoming transfer" (SECURITY.md:160), and the terminal-deficit runbook above applies "Recapitalize" to "trader payout claims that physical cash cannot currently service". `accountExcess()` is described as "owner-governed admission of unsolicited raw pool cash into canonical assets" (ACCOUNTING_SPEC.md:637; similarly SECURITY.md:299), and an owner transfer made to service a claim is not unsolicited cash.


## Impact Explanation

After a trader-claim deficit has been recapitalized and paid, a later 5,000 USDG owner restoration is assigned to senior principal and is therefore senior LP funds, yet none of it is withdrawable: the senior LP's redemption is not funded through 26 hourly settlement attempts spanning 25 hours and one more eight days after the request, with no positions, claims or unassigned assets left, whereas in the control the senior LP receives 8,281.75 USDG at the first epoch. The freeze has no time-based end, because `_canAcceptTrancheDeposits()` (HousePool.sol:480-503) refuses junior deposits while junior principal is 0 with outstanding junior supply and refuses tranche deposits while projected residual claimant assets or a senior impairment remain (`_projectedEntriesMaySettle()`, HousePool.sol:1120-1137), which the test confirms for both tranches. The only inflows that can then raise pool cash above the 16,582.14 USDG residual are later trading revenue, with new risk admitted against the roughly 5,000 USDG the pool holds, or a further owner payment, and in the test the restoration is released only after the owner pays the deficit a second time. When the senior tranche is instead restored in full, its high-water mark exceeds the restored principal by 16,416.32 USDG, so up to about that amount of later LP revenue may be credited to senior instead of junior holders, who still hold their shares; in the test the 1,505.40 USDG of revenue from one losing trade goes to senior and junior receives 0, against 1,504.68 USDG to junior in the control. When revenue above the senior gap arrives before the next LP settlement, part of it leaves the claimant ledger: with the same trades and the same 49,384.78 USDG of revenue in both orders, the junior LP ends with 16,385.53 USDG instead of 32,968.56 USDG and 16,582.14 USDG, the deficit, is stored in `unassignedAssets`, and with 19,964.06 USDG of revenue the junior LP ends with 0 instead of 3,548.33 USDG. The senior LP ends within 1.17 USDG of its control value, so the amount does not move to another LP. While it is stored, both vaults reject deposit requests, the junior LP cannot recover it, and the owner's `assignUnassignedAssets()` credits all of it to the one receiver the owner names, after which the junior LP's value is unchanged.


## Recommendation

Consider keeping the claimant queue bounded by the cash that backs it, so that a recapitalization that has already serviced a trader claim is no longer treated as value owed to LPs. One approach could be to cap both buckets after each pool payout:

```diff
  // HousePool.payOut()
  accountedAssets -= amount;
+ _capPendingClaimantBucketsToAccountedAssets();
  settlementToken.safeTransfer(recipient, amount);
```

```solidity
/// @dev Sketch: queued claimant intent cannot exceed the canonical cash left to back it (recapitalization keeps
///      priority, as in capClaimantBuckets()).
function _capPendingClaimantBucketsToAccountedAssets() internal {
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory capped =
        HousePoolPendingPreviewLib.capClaimantBuckets(_getPendingClaimantBuckets(), accountedAssets);
    pendingRecapitalizationAssets = capped.recapitalizationAssets;
    pendingTradingRevenueAssets = capped.revenueAssets;
}
```



## Proof of Concept

## Proof of Concept

The test deploys the production `HousePool`, tranche vaults, `CfdEngine`, `PletherOracle` and Router from `BasePerpTest` with the `DEPLOYMENT.md` release risk parameters, the release Router configuration (minimum open notional 1,000 USDG, adverse confidence multiplier 2,500 bps, protection trigger bounty 0.20 USDG) and the release senior capacity (`maxSeniorExposureAssets = 40,000,000e6`, `maxSeniorShareBps = 8,000`), both applied through their timelocks, and 1 USDG seeds. Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked. A junior LP deposits 60,000 USDG and a senior LP 40,000 USDG through the ERC-7540 vault functions. Each trade is committed by the trader through the `BasePerpTest` Router, the test-only `LegacyOrderRouterHarness` (`packages/perps/test/utils/LegacyOrderRouterHarness.sol`), whose scalar `commitOrder()` delegatecalls the production `commitOrder()` with unbounded `ExecutionBounds`, and executed by a third-party keeper with `router.executeOrder()`; marks are published with `router.updateMarkPrice()` and LP epochs are settled with the permissionless `router.settleLpEpoch()`. The claimant calls `settleTraderClaim()` for its own account, and the owner calls `recapitalizePool()` and `clearDegradedMode()`. No call pranks a protocol contract. The lens function `previewOpen()` is used only to size the long admitted at the cap.

1. `test_Control_PartialRestore_NoClaim_SeniorRedeemsAtFirstEpoch()`: the long closes at 1.20, its payout fits in pool cash, and after a 5,000 USDG restoration the senior LP's full redemption is funded at the first epoch.
2. `test_PartialRestoreAfterClaimDeficit_LocksSeniorCashPast24h()`: the long closes at 0.80 and leaves a 16,582.14 USDG deficit; after the runbook and the same 5,000 USDG restoration, both tranches refuse deposits, and 26 hourly `settleLpEpoch()` calls spanning 25 hours and one more eight days after the request revert with `HousePool__NoLpEpochProgress`.
3. `test_LockedState_OwnerMustPayDeficitAgainToRelease()`: from the locked state, a second recapitalization of the whole deficit is queued as well, and only then the earlier 5,000 USDG becomes withdrawable.
4. `test_Control_FullRestore_NoClaim_RevenueReachesJunior()` and `test_FullRestoreAfterClaimDeficit_RevenueRoutedToSenior()`: the owner restores senior to its original principal; the same losing trade then credits junior in the control and senior after the deficit, where the senior high-water mark also exceeds the senior principal held before the loss.

Command (from the repository root at commit a82d4fef, forge 1.9.0, commit 6056c584):

```
forge test --root packages/perps --match-path test/Submission_H04.t.sol -vv
```

Key output (excerpt: test results only, compiler warnings omitted; USDG atoms with 6 decimals):

```
Ran 5 tests for test/Submission_H04.t.sol:RecapitalizationQueueAfterClaimTest
[PASS] test_Control_FullRestore_NoClaim_RevenueReachesJunior() (gas: 23450419)
Logs:
  control: senior principal after recovery: 39601001772
  control: senior high-water mark after recovery: 39601725078
  LP revenue from the losing trade: 1505400006
  control: junior principal after the revenue epoch: 1504676700

[PASS] test_Control_PartialRestore_NoClaim_SeniorRedeemsAtFirstEpoch() (gas: 16497678)
Logs:
  control: pool cash: 8281954853
  control: senior principal (pending view): 8281954853
  control: max senior withdraw: 8281954853
  control: senior LP paid at the first epoch: 8281747443

[PASS] test_FullRestoreAfterClaimDeficit_RevenueRoutedToSenior() (gas: 23940686)
Logs:
  pool cash before recapitalization: 42582081099
  trader claims: 59164222246
  deficit (claims - cash): 16582141147
  pendingRecapitalizationAssets after the claim is paid: 16582141147
  senior principal before the loss: 40001001771
  senior principal after recovery: 39601001772
  senior high-water mark after recovery: 56017325661
  senior impairment (HWM - principal): 16416323889
  LP revenue from the losing trade: 1505400006
  projected senior gain from the revenue: 1505400006
  projected junior gain from the revenue: 0
  junior principal after the revenue epoch: 0
  junior LP shares outstanding: 60105906501348

[PASS] test_LockedState_OwnerMustPayDeficitAgainToRelease() (gas: 18139050)
Logs:
  pool cash before recapitalization: 42582081099
  trader claims: 59164222246
  deficit (claims - cash): 16582141147
  pendingRecapitalizationAssets after the claim is paid: 16582141147
  second recapitalization (= deficit): 16582141147
  pendingRecapitalizationAssets after it: 38164282294
  max senior withdraw after it: 5000000000
  senior LP paid: 5000000000
  pool cash left after the senior exit: 16582141147
  senior LP paid at the next epoch: 16581600651
  senior LP total paid: 21581600651
  owner recapitalized in total: 38164282294

[PASS] test_PartialRestoreAfterClaimDeficit_LocksSeniorCashPast24h() (gas: 28690798)
Logs:
  pool cash before recapitalization: 42582081099
  trader claims: 59164222246
  deficit (claims - cash): 16582141147
  pendingRecapitalizationAssets after the claim is paid: 16582141147
  pool cash: 5000000000
  pendingRecapitalizationAssets: 21582141147
  senior principal (pending view): 5000000000
  max senior withdraw: 0
  hourly settlement attempts: 26
  hours from the request to the last hourly attempt: 25
  days from the request to the final attempt: 8
  restored senior cash locked: 5000000000

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 31.12ms (89.35ms CPU time)

Ran 1 test suite in 31.54ms (31.12ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

After the runbook the pool holds 0 USDG while 16,582.14 USDG of recapitalization is still queued. A 5,000 USDG restoration is then owned by senior but has a maximum senior withdrawal of 0, and the senior LP's redemption stays unfunded through 26 hourly settlement attempts spanning 25 hours and one more eight days after the request; in the control the senior LP receives 8,281.75 USDG at the first epoch. Releasing it took a second owner payment of 16,582.14 USDG (38,164.28 USDG recapitalized in total), which then also reached the senior LP. After a full restoration the senior high-water mark is 56,017.33 USDG on 39,601.00 USDG of principal (control: 39,601.73 USDG), above the 40,001.00 USDG of senior principal before the loss, and the 1,505.40 USDG revenue goes to senior (junior 0, control junior 1,504.68 USDG).

`packages/perps/test/Submission_H04.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice When trader claims exceed pool cash and the owner follows the documented recovery runbook (recapitalize the
///         shortfall, the claimant settles its claim, the owner clears degraded mode), the recapitalization that paid
///         the claimant stays queued in `pendingRecapitalizationAssets` with no cash behind it.
///         - A later partial restoration of the senior tranche is reserved behind that residual: the senior LP's
///           redemption is not funded and `settleLpEpoch()` reverts with `HousePool__NoLpEpochProgress`.
///         - A full restoration bootstraps the senior high-water mark from the whole queued intent, so the senior
///           tranche is treated as impaired by the deficit and later trading revenue goes to senior instead of junior.
/// @dev    Release risk parameters, release Router configuration and release senior capacity (DEPLOYMENT.md), 1 USDG
///         seeds. Only Pyth (MockPyth) and USDG (MockSettlementToken) are mocked. Every trade is committed by the
///         trader through the BasePerpTest router (`LegacyOrderRouterHarness`, whose scalar `commitOrder()`
///         delegatecalls the production `commitOrder()` with unbounded `ExecutionBounds`) and executed by a
///         third-party keeper with `router.executeOrder()`. LP actions use the public ERC-7540 vault functions and LP
///         settlement is the permissionless `router.settleLpEpoch()`.
contract RecapitalizationQueueAfterClaimTest is BasePerpTest {

    address internal constant SHORT_TRADER = address(0x5A01);
    address internal constant LONG_TRADER = address(0x5A02);
    address internal constant LOSING_TRADER = address(0x5A03);
    address internal constant JUNIOR_LP = address(0x1A17);
    address internal constant SENIOR_LP = address(0x1B17);
    address internal constant KEEPER = address(0x4EE717);

    uint256 internal constant LOT = 1e20; // positionSizeQuantum
    uint256 internal constant BASKET = 0.8e8; // basket != $1.00
    uint256 internal constant CONTROL_LONG_CLOSE = 1.2e8; // control: the long's payout fits in pool cash
    uint256 internal constant SHORT_LOTS = 480; // 48,000 units, 38,400 USDG notional at 0.80
    uint256 internal constant TRADE_MARGIN = 8000e6;
    uint256 internal constant TRADER_DEPOSIT = 20_000e6;
    uint256 internal constant JUNIOR_DEPOSIT = 60_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 40_000e6;
    uint256 internal constant PARTIAL_RESTORE = 5000e6; // owner restores part of the senior tranche
    uint256 internal constant SCENARIO_START = SETUP_TIMESTAMP + 8 days; // Tuesday 2024-03-12 10:00 UTC

    // ------------------------------------------------------------------ release configuration

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        // Release senior capacity and Router configuration, applied through their timelocks.
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        config.adverseConfidenceMultiplierBps = 2500;
        config.positionProtectionTriggerBountyAssets = 200_000;
        _setRouterConfig(config);
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release min open notional");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");

        vm.warp(SCENARIO_START);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(BASKET);

        // A junior LP and a senior LP fund the tranches through the vaults; a keeper settles each epoch.
        _deposit(juniorVault, JUNIOR_LP, JUNIOR_DEPOSIT);
        _nextEpoch();
        _keeperSettle();
        _claimDeposit(juniorVault, JUNIOR_LP);
        _deposit(seniorVault, SENIOR_LP, SENIOR_DEPOSIT); // senior capacity follows settled junior (80% covenant)
        _nextEpoch();
        _keeperSettle();
        _claimDeposit(seniorVault, SENIOR_LP);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // DEPOSIT_COOLDOWN elapses before the trading path starts
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(BASKET);
        assertApproxEqAbs(pool.juniorPrincipal(), JUNIOR_DEPOSIT + 1e6, 1e6, "junior funded (less senior coupon)");
        assertApproxEqAbs(pool.seniorPrincipal(), SENIOR_DEPOSIT + 1e6, 1e6, "senior funded");

        _fundTrader(SHORT_TRADER, TRADER_DEPOSIT);
        _fundTrader(LONG_TRADER, TRADER_DEPOSIT);
        _fundTrader(LOSING_TRADER, TRADER_DEPOSIT);
    }

    // ------------------------------------------------------------------ oracle, trade and LP helpers

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _nextOracleSecond() internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev Any keeper publishes a fresh mark through the Router.
    function _refreshMark(
        uint256 price
    ) internal {
        _nextOracleSecond();
        bytes[] memory data = _livePriceData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    /// @dev Real order path: the trader commits, a third-party keeper executes with the first post-commit Pyth tick.
    function _routerTrade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "router order executed");
    }

    /// @dev Moves to the next hourly LP epoch boundary (plus one second).
    function _nextEpoch() internal {
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _tryKeeperSettle() internal returns (bool ok, bytes4 err) {
        _nextOracleSecond();
        bytes[] memory data = _livePriceData(BASKET);
        vm.prank(KEEPER);
        try router.settleLpEpoch(data) {
            ok = true;
        } catch (bytes memory reason) {
            if (reason.length >= 4) {
                err = bytes4(reason);
            }
        }
    }

    function _keeperSettle() internal {
        (bool ok, bytes4 err) = _tryKeeperSettle();
        if (!ok) {
            emit log_named_bytes32("unexpected settle revert", bytes32(err));
        }
        assertTrue(ok, "keeper LP settlement succeeds");
    }

    function _deposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp
    ) internal {
        uint256 id = vault.controllerDepositHead(lp);
        uint256 claimable = vault.claimableDepositRequest(id, lp);
        vm.prank(lp);
        vault.claimDeposit(id, claimable, lp, lp);
    }

    function _seniorRequestRedeem(
        uint256 shares
    ) internal returns (uint256 requestId) {
        vm.prank(SENIOR_LP);
        requestId = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
    }

    function _seniorClaimRedeem(
        uint256 requestId
    ) internal returns (uint256 assets) {
        uint256 shares = seniorVault.claimableRedeemRequest(requestId, SENIOR_LP);
        if (shares == 0) {
            return 0;
        }
        vm.prank(SENIOR_LP);
        assets = seniorVault.claimRedeem(requestId, shares, SENIOR_LP, SENIOR_LP);
    }

    /// @dev Owner recovery action: USDG from the owner's wallet into the pool through `CfdEngine.recapitalizePool()`.
    function _recapitalize(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    /// @dev Largest LONG the engine admits at the cap with `TRADE_MARGIN` (E >= L + B headroom, VPI rebate included).
    function _admissibleLongLotsAtCap() internal view returns (uint256 lots) {
        uint256 hi = pool.totalAssets() / CAP_PRICE; // one lot is 200 USDG of notional at the 2.00 cap
        while (lots < hi) {
            uint256 mid = (lots + hi + 1) / 2;
            if (engineLens.previewOpen(LONG_TRADER, CfdTypes.Side.LONG, mid * LOT, TRADE_MARGIN, CAP_PRICE).valid) {
                lots = mid;
            } else {
                hi = mid - 1;
            }
        }
    }

    // ------------------------------------------------------------------ scenario

    /// @dev Short at 0.80; the basket reaches the 2.00 cap; a long is admitted into the remaining `E >= L + B`
    ///      headroom; the short takes its maximum profit (degraded mode latches); the basket returns to `longClose`
    ///      and the long closes. At 0.80 the long's payout exceeds pool cash and the remainder is a trader claim.
    function _runTradingPath(
        uint256 longClose
    ) internal returns (uint256 longLots) {
        _routerTrade(SHORT_TRADER, CfdTypes.Side.SHORT, SHORT_LOTS * LOT, TRADE_MARGIN, false, BASKET);

        _refreshMark(CAP_PRICE);
        longLots = _admissibleLongLotsAtCap();
        _routerTrade(LONG_TRADER, CfdTypes.Side.LONG, longLots * LOT, TRADE_MARGIN, false, CAP_PRICE);
        assertFalse(engine.degradedMode(), "long admitted under E >= L + B");

        _routerTrade(SHORT_TRADER, CfdTypes.Side.SHORT, SHORT_LOTS * LOT, 0, true, CAP_PRICE);
        assertTrue(engine.degradedMode(), "short profit-taking latches degraded mode");

        _refreshMark(longClose);
        _routerTrade(LONG_TRADER, CfdTypes.Side.LONG, longLots * LOT, 0, true, longClose);
        assertEq(_maxLiability(), 0, "book is flat");
    }

    /// @dev Trading path ending in a trader-claim deficit, then the documented runbook: the owner recapitalizes the
    ///      shortfall, the claimant settles its claim, and the owner clears degraded mode.
    function _claimDeficitAndRunbook() internal returns (uint256 deficit) {
        _runTradingPath(BASKET);
        uint256 cash = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        assertGt(claims, cash, "trader claim exceeds pool cash");
        deficit = claims - cash;
        emit log_named_uint("pool cash before recapitalization", cash);
        emit log_named_uint("trader claims", claims);
        emit log_named_uint("deficit (claims - cash)", deficit);

        _recapitalize(deficit);
        vm.prank(LONG_TRADER);
        engine.settleTraderClaim(LONG_TRADER);
        engine.clearDegradedMode();

        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "claim fully paid");
        assertEq(pool.rawAssets(), 0, "every recapitalized atom left the pool to the claimant");
        assertEq(pool.pendingRecapitalizationAssets(), deficit, "the paid recapitalization is still queued");
        emit log_named_uint(
            "pendingRecapitalizationAssets after the claim is paid", pool.pendingRecapitalizationAssets()
        );
    }

    /// @dev Control trading path: the long closes at 1.20, its payout fits in pool cash, so there is no trader claim.
    function _noClaimPathAndRunbook() internal {
        _runTradingPath(CONTROL_LONG_CLOSE);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "control: payout fit in pool cash");
        engine.clearDegradedMode();
        assertEq(pool.pendingRecapitalizationAssets(), 0, "control: nothing queued");
    }

    /// @dev A losing LONG (opens at 0.80, closes at 0.90) produces LP trading revenue.
    function _losingTrade() internal returns (uint256 revenue) {
        _refreshMark(BASKET);
        uint256 before = pool.totalAssets();
        _routerTrade(LOSING_TRADER, CfdTypes.Side.LONG, 150 * LOT, 3000e6, false, BASKET);
        _refreshMark(0.9e8);
        _routerTrade(LOSING_TRADER, CfdTypes.Side.LONG, 150 * LOT, 0, true, 0.9e8);
        revenue = pool.totalAssets() - before;
        emit log_named_uint("LP revenue from the losing trade", revenue);
    }

    // ------------------------------------------------------------------ partial restoration: senior cash locked

    /// @dev Control: no trader claim. After the same loss, the owner restores 5,000 USDG to senior and the senior LP's
    ///      full redemption is funded at the first matured epoch.
    function test_Control_PartialRestore_NoClaim_SeniorRedeemsAtFirstEpoch() public {
        _noClaimPathAndRunbook();
        _recapitalize(PARTIAL_RESTORE);
        _refreshMark(BASKET);
        (uint256 seniorPending,, uint256 maxSeniorWithdraw,) = pool.getPendingTrancheState();
        emit log_named_uint("control: pool cash", pool.rawAssets());
        emit log_named_uint("control: senior principal (pending view)", seniorPending);
        emit log_named_uint("control: max senior withdraw", maxSeniorWithdraw);
        assertEq(maxSeniorWithdraw, seniorPending, "control: all senior principal is withdrawable");

        uint256 requestId = _seniorRequestRedeem(seniorVault.maxRequestRedeem(SENIOR_LP));
        _nextEpoch();
        _keeperSettle();
        uint256 paid = _seniorClaimRedeem(requestId);
        emit log_named_uint("control: senior LP paid at the first epoch", paid);
        assertGt(paid, PARTIAL_RESTORE, "control: the restored cash is redeemed");
    }

    /// @dev After a trader-claim deficit and the runbook, the same 5,000 USDG restoration belongs to senior but none
    ///      of it is withdrawable. The senior LP's redemption stays unfunded through 26 hourly settlement attempts
    ///      spanning 25 hours and one more 8 days after the request, with no claims, positions or unassigned assets.
    function test_PartialRestoreAfterClaimDeficit_LocksSeniorCashPast24h() public {
        uint256 deficit = _claimDeficitAndRunbook();
        assertGt(deficit, PARTIAL_RESTORE, "the restoration is smaller than the queued residual");

        _recapitalize(PARTIAL_RESTORE);
        _refreshMark(BASKET);
        (uint256 seniorPending,, uint256 maxSeniorWithdraw,) = pool.getPendingTrancheState();
        emit log_named_uint("pool cash", pool.rawAssets());
        emit log_named_uint("pendingRecapitalizationAssets", pool.pendingRecapitalizationAssets());
        emit log_named_uint("senior principal (pending view)", seniorPending);
        emit log_named_uint("max senior withdraw", maxSeniorWithdraw);
        assertEq(pool.rawAssets(), PARTIAL_RESTORE, "restored cash sits in the pool");
        assertEq(pool.pendingRecapitalizationAssets(), deficit + PARTIAL_RESTORE, "both recapitalizations queued");
        assertEq(
            engine.totalTraderClaimBalanceAssets() + _maxLiability() + pool.unassignedAssets(), 0, "no claims or risk"
        );
        assertTrue(pool.isWithdrawalLive(), "withdrawal gate is open");
        assertEq(seniorPending, PARTIAL_RESTORE, "senior owns the restored cash");
        assertEq(maxSeniorWithdraw, 0, "none of it is withdrawable");
        assertFalse(pool.canAcceptTrancheDeposits(true), "senior deposits are refused");
        assertFalse(pool.canAcceptTrancheDeposits(false), "junior deposits are refused");

        uint256 requestId = _seniorRequestRedeem(seniorVault.maxRequestRedeem(SENIOR_LP));
        uint256 lockStart = vm.getBlockTimestamp();
        uint256 attempts;
        for (uint256 h; h < 26; ++h) {
            _nextEpoch();
            (bool ok, bytes4 err) = _tryKeeperSettle();
            assertFalse(ok, "settlement makes no progress");
            assertEq(err, IHousePool.HousePool__NoLpEpochProgress.selector, "reverts NoLpEpochProgress");
            ++attempts;
        }
        uint256 hoursLocked = (vm.getBlockTimestamp() - lockStart) / 1 hours;
        assertGe(hoursLocked, 24, "locked for at least 24 hours");

        vm.warp(lockStart + 8 days); // the following Wednesday, live FX session
        vm.roll(vm.getBlockNumber() + 1);
        (bool okLater, bytes4 errLater) = _tryKeeperSettle();
        assertFalse(okLater, "still no progress 8 days later");
        assertEq(errLater, IHousePool.HousePool__NoLpEpochProgress.selector, "reverts NoLpEpochProgress");

        assertEq(seniorVault.claimableRedeemRequest(requestId, SENIOR_LP), 0, "nothing claimable");
        assertEq(pool.rawAssets(), PARTIAL_RESTORE, "the restored cash never left the pool");
        (,, maxSeniorWithdraw,) = pool.getPendingTrancheState();
        assertEq(maxSeniorWithdraw, 0, "still not withdrawable");
        emit log_named_uint("hourly settlement attempts", attempts);
        emit log_named_uint("hours from the request to the last hourly attempt", hoursLocked);
        emit log_named_uint("days from the request to the final attempt", (vm.getBlockTimestamp() - lockStart) / 1 days);
        emit log_named_uint("restored senior cash locked", PARTIAL_RESTORE);
    }

    /// @dev From the locked state, the owner can release the restored cash only by recapitalizing the whole deficit a
    ///      second time; that payment is queued as well and exactly the earlier 5,000 USDG becomes withdrawable.
    function test_LockedState_OwnerMustPayDeficitAgainToRelease() public {
        uint256 deficit = _claimDeficitAndRunbook();
        _recapitalize(PARTIAL_RESTORE);
        uint256 requestId = _seniorRequestRedeem(seniorVault.maxRequestRedeem(SENIOR_LP));
        _nextEpoch();
        (bool ok,) = _tryKeeperSettle();
        assertFalse(ok, "locked");

        _recapitalize(deficit);
        _refreshMark(BASKET);
        (,, uint256 maxSeniorWithdraw,) = pool.getPendingTrancheState();
        emit log_named_uint("second recapitalization (= deficit)", deficit);
        emit log_named_uint("pendingRecapitalizationAssets after it", pool.pendingRecapitalizationAssets());
        emit log_named_uint("max senior withdraw after it", maxSeniorWithdraw);
        assertEq(pool.pendingRecapitalizationAssets(), 2 * deficit + PARTIAL_RESTORE, "every recapitalization queued");
        assertEq(maxSeniorWithdraw, PARTIAL_RESTORE, "only the earlier restoration becomes withdrawable");

        _keeperSettle();
        uint256 paid = _seniorClaimRedeem(requestId);
        emit log_named_uint("senior LP paid", paid);
        emit log_named_uint("pool cash left after the senior exit", pool.rawAssets());
        assertApproxEqAbs(paid, PARTIAL_RESTORE, 1e6, "senior LP receives the earlier restoration");
        assertEq(pool.rawAssets(), deficit + PARTIAL_RESTORE - paid, "the second payment stays in the pool");

        // The second payment is attributed to senior principal and is redeemed by the senior LP at the next epoch.
        _nextEpoch();
        _keeperSettle();
        uint256 paidNext = _seniorClaimRedeem(requestId);
        emit log_named_uint("senior LP paid at the next epoch", paidNext);
        emit log_named_uint("senior LP total paid", paid + paidNext);
        emit log_named_uint("owner recapitalized in total", 2 * deficit + PARTIAL_RESTORE);
        assertGt(paidNext, (deficit * 99) / 100, "the owner's second payment of the deficit reaches the senior LP");
    }

    // ------------------------------------------------------------------ full restoration: revenue routed to senior

    /// @dev Control: no trader claim. The owner restores senior to its original principal; after one epoch only a
    ///      coupon-sized gap remains and the next LP revenue reaches junior.
    function test_Control_FullRestore_NoClaim_RevenueReachesJunior() public {
        uint256 seniorBefore = pool.seniorPrincipal();
        _noClaimPathAndRunbook();
        (uint256 seniorAfterLoss,,,) = pool.getPendingTrancheState();
        assertLt(seniorAfterLoss, seniorBefore, "control: senior absorbed part of the loss");
        _recapitalize(seniorBefore - seniorAfterLoss);
        _seniorRequestRedeem(seniorVault.maxRequestRedeem(SENIOR_LP) / 100);
        _nextEpoch();
        _keeperSettle();
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        emit log_named_uint("control: senior principal after recovery", pool.seniorPrincipal());
        emit log_named_uint("control: senior high-water mark after recovery", pool.seniorHighWaterMark());
        assertLt(gap, seniorBefore / 200, "control: only a coupon-sized gap");

        uint256 revenue = _losingTrade();
        _seniorRequestRedeem(seniorVault.maxRequestRedeem(SENIOR_LP) / 100);
        _nextEpoch();
        _keeperSettle();
        emit log_named_uint("control: junior principal after the revenue epoch", pool.juniorPrincipal());
        assertGt(pool.juniorPrincipal(), (revenue * 80) / 100, "control: junior receives the revenue");
    }

    /// @dev After a trader-claim deficit and the runbook, the same restoration leaves senior shown as impaired by the
    ///      deficit, and the same revenue goes to senior; junior receives none of it.
    function test_FullRestoreAfterClaimDeficit_RevenueRoutedToSenior() public {
        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 deficit = _claimDeficitAndRunbook();
        _recapitalize(seniorBefore);
        _seniorRequestRedeem(seniorVault.maxRequestRedeem(SENIOR_LP) / 100);
        _nextEpoch();
        _keeperSettle();
        uint256 s = pool.seniorPrincipal();
        uint256 hwm = pool.seniorHighWaterMark();
        emit log_named_uint("senior principal before the loss", seniorBefore);
        emit log_named_uint("senior principal after recovery", s);
        emit log_named_uint("senior high-water mark after recovery", hwm);
        emit log_named_uint("senior impairment (HWM - principal)", hwm - s);
        assertGt(hwm, seniorBefore, "the high-water mark exceeds any senior principal held before");
        assertApproxEqRel(hwm - s, deficit, 0.02e18, "restored senior is shown impaired by the deficit");
        assertTrue(pool.isSeniorImpairedAfterPendingDepositReconcile(), "pool treats senior as impaired");

        uint256 seniorBeforeRevenue = pool.seniorPrincipal();
        uint256 revenue = _losingTrade();
        (uint256 seniorProjected, uint256 juniorProjected,,) = pool.getPendingTrancheState();
        emit log_named_uint("projected senior gain from the revenue", seniorProjected - seniorBeforeRevenue);
        emit log_named_uint("projected junior gain from the revenue", juniorProjected);
        assertGt(seniorProjected - seniorBeforeRevenue, (revenue * 95) / 100, "senior is projected to receive it");
        assertEq(juniorProjected, 0, "junior is projected to receive none of it");

        _seniorRequestRedeem(seniorVault.maxRequestRedeem(SENIOR_LP) / 100);
        _nextEpoch();
        _keeperSettle();
        emit log_named_uint("junior principal after the revenue epoch", pool.juniorPrincipal());
        emit log_named_uint("junior LP shares outstanding", juniorVault.balanceOf(JUNIOR_LP));
        assertGt(juniorVault.balanceOf(JUNIOR_LP), 0, "junior LP still holds its shares");
        assertEq(pool.juniorPrincipal(), 0, "junior receives none of the revenue");
        assertGt(pool.seniorHighWaterMark(), pool.seniorPrincipal(), "senior still treated as impaired");
    }

}
```
