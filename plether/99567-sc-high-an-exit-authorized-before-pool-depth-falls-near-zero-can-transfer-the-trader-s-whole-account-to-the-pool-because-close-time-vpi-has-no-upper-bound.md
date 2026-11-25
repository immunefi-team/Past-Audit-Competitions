# \#99567 [SC-High] An exit authorized before pool depth falls near zero can transfer the trader's whole account to the pool because close-time VPI has no upper bound

**Submitted on Nov 14th 2026 at 09:06:57 UTC by @Nyksx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99567
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

Close-time virtual price impact (VPI) divides by net pool depth, is skipped only when that depth is exactly zero, and has no upper bound. After a terminal trader-claim deficit, net depth returns to at or just above zero through the owner's documented `recapitalizePool()` or, with no owner action, through a trader's loss payment that exceeds the shortfall by a small amount, and a minority-side exit that the trader authorized earlier is then assessed a charge many times the account's value: in the proof of concept a stop-loss armed in a normal market with the unlimited limits, and an ordinary close committed while every preview showed a 128.13 USDG charge, each take the account's whole custody of 50,063.79 USDG on an 80,000 USDG close, and the pool gains about 50,032 USDG. 

## Finding Description

Close VPI is `0.5 * k * (S_post^2 - S_pre^2) / D`, where `S` is the absolute directional skew and `D` is net economic depth (pool assets minus outstanding trader claims, floored at zero). When a close increases skew (the closing position is on the minority side), the charge is positive and scales with `1 / D`. The close path skips the fresh charge only when `D == 0` and applies a lifetime clamp that limits rebates only. No guard relates the charge to the notional being closed, and closes are not subject to the skew-to-depth cap that applies to opens.

With the release parameters (`vpiFactor = 0.01e18`) and an 80,000 USDG minority SHORT closing against a 160,000 USDG LONG, `k * (S_post^2 - S_pre^2) / 2` is 96,000,000 USDG², so the charge is about `96,000,000 / D` USDG. It exceeds a 1,600 USDG allowance (2% of the closed notional) below about 60,000 USDG of net depth, and it exceeds the account's whole custody (about 50,064 USDG) below about 1,900 USDG of net depth.

The zero-depth skip does not protect an exit after a recapitalization of exactly the shortfall. The views then show `D = 0`, but the close first realizes the closing account's own unpaid carry into the pool and prices VPI on depth that includes it. An order cannot execute in the second of its commit, so at least one second of carry is always realized. In the proof of concept that carry is 0.000174 USDG to 0.243642 USDG, and the charge assessed on it is between 384 million and 551 billion USDG.

**1. The fresh charge is skipped only at exactly zero depth, and the clamp only limits rebates.**

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:105-118 (a82d4fef)
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
// The lifetime clamp below still recovers any previously received negative VPI.
if (inputs.poolDepthAssets != 0) { // any D > 0, however small, prices the full curve
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets; // lower bound only; positive charges are not capped
}
```

```solidity
// packages/perps/src/CfdMath.sol:123-139 (a82d4fef), tryGetSkewCost()
if (depthAssets == 0 || skewAssets == 0) {
    return (true, 0);
}
// ...
uint256 sqSkewOverDepthWad = (skewWad * skewWad) / depthWad; // S^2 / D: grows without limit as D -> 0+
// ...
uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2;
return (true, costWad / 1e12);
```

**2. The depth used is net of trader claims, and the close adds the account's own carry to it before pricing.**

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:814-815 (a82d4fef), planClose()
delta.pendingCarryAssets = _pendingCarryAssets(snap);
delta.realizedCarryAssets = _applyPendingCarryRealizationToSnapshot(snap, delta.pendingCarryAssets);

// packages/perps/src/libraries/CfdEnginePlanLib.sol:638 (a82d4fef), _applyPendingCarryRealizationToSnapshot()
snap.poolAssets += realizedCarryAssets; // the closing account's carry enters the depth used below

// packages/perps/src/libraries/CfdEnginePlanLib.sol:1339-1341 (a82d4fef), _buildCloseState()
poolDepthAssets: PositionRiskAccountingLib.computeNetPoolDepthAssets(
    snap.poolAssets, snap.totalTraderClaimBalanceAssets // D = poolAssets - claims, floored at 0
),
```

**3. An order executes at least one second after its commit.** Some carry is therefore realized at every live execution:

```solidity
// packages/perps/src/PletherOracle.sol:610 (a82d4fef)
uint64 minPublishTime = request.commitTime + 1; // the execution tick is later than the commit

// packages/perps/src/OrderRouterExecutionSidecar.sol:447-452 (a82d4fef)
if (!request.oracleFrozen && ExecutionBlockNumberLib.current() == order.commitBlock) {
    return _pendingResult(order.orderId, OrderV3Types.PendingReason.SameBlock);
}
if (!request.oracleFrozen && request.oraclePublishTime <= order.commitTime) {
    return _pendingResult(order.orderId, OrderV3Types.PendingReason.MevBoundary);
}
```

**4. Opens are guarded against low depth; closes are not.** Opens reject at zero depth and must keep post-trade skew within `maxSkewRatio` of depth:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:448-452 and :473 (a82d4fef), _planOpenState()
uint256 netDepthAssets =
    PositionRiskAccountingLib.computeNetPoolDepthAssets(snap.poolAssets, snap.totalTraderClaimBalanceAssets);
if (netDepthAssets == 0) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.SOLVENCY_EXCEEDED;
    return;
// ...
uint256 maxSkewAssets = Math.mulDiv(netDepthAssets, snap.riskParams.maxSkewRatio, CfdMath.WAD); // opens only
```

**5. A full close collects the charge from the account's reachable buckets and waives the rest; the trader's limits are the only bound.** The charge is taken from the action reserve, free settlement, released margin and, for a full close, committed order margin. The account's free settlement balance is therefore taken as well as the position margin:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:1130-1133 (a82d4fef), _planCloseCashSources()
delta.actionChargeCollectedAssets = delta.vpiRebateReserveConsumedAssets + delta.actionReserveConsumedAssets
    + freeConsumedAssets + delta.actionChargeFromReleasedMarginAssets
    + delta.actionCommittedMarginConsumedAssets;
delta.actionChargeWaivedAssets = delta.actionChargeToCollectAssets - delta.actionChargeCollectedAssets;
```

```solidity
// packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:509-513 (a82d4fef), _enforceBounds()
_requireMaximum(
    OrderV3Types.ConstraintKind.ActionCharge,
    assessment.actionChargeAssessedAssets, // includes the fresh close VPI
    bounds.maxActionChargeAssets
);
```

The limits are set when the exit is authorized, not when it executes. For a protection they are stored at creation and reused by every later trigger and retry, and the documents name the unlimited value as a supported choice:

```solidity
// packages/perps/src/interfaces/PositionProtectionTypes.sol:44-46 (a82d4fef)
/// @notice Trader-authorized financial ceilings for every close generated by this protection.
/// @dev Values use 6-decimal settlement token. Zero is a real zero allowance; `type(uint256).max` is an explicit
///      unlimited allowance. The Book retains these limits across trigger and permissionless retries.
```

**6. In degraded mode an account with an open position cannot withdraw, and an account with an active protection cannot commit an ordinary close.**

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:274-276 (a82d4fef), validateWithdraw()
if (engine.degradedMode()) { // reached only when the account still has an open position
    revert ICfdEngineTypes.CfdEngine__DegradedMode();
}

// packages/perps/src/router/OrderLiquidationBatchLogic.sol:313-315 (a82d4fef)
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

**Sequence measured in the proof of concept.**

1. In a normal market (net depth 1,000,034.01 USDG) a trader holds an 80,000 USDG SHORT on the minority side with 50,063.79 USDG of custody. The trader arms a stop-loss with all four close limits set to `type(uint256).max`. The lens quotes 95.99 USDG of VPI for the close.
2. A two-sided basket move leaves a terminal trader-claim deficit: trader claims exceed pool assets, and the engine latches degraded mode. Net depth is 0. A close in this state still costs the normal amount, because the fresh charge is skipped.
3. The owner follows the documented recovery and calls `recapitalizePool()` with exactly the shortfall read from the views. The views still show a net depth of 0.
4. The mark crosses the stop-loss price. A keeper triggers the protection and executes its close one second later. The execution realizes 0.243642 USDG of the account's carry, prices VPI on that depth, and assesses an action charge of 384,231,814.54 USDG. It collects 49,063.59 USDG, which is everything the account has left after the 1,000.00 USDG price loss and the 0.20 USDG trigger bounty, and waives the rest. The account's custody goes from 50,063.79 USDG to 0, and pool assets rise by 50,031.99 USDG. The same stop-loss at normal depth, and the same stop-loss after the deficit without a recapitalization, leave the account 48,935.99 USDG and 48,935.76 USDG.

The same outcome occurs in the other measured arrangements. A recapitalization of the shortfall plus 1 USDG gives a charge of 75,274,909.98 USDG and the same loss. A stop-loss armed during degraded mode at `D = 0` also ends with a custody of 0. An ordinary close with an unlimited action-charge bound, committed at `D = 0` while `previewClose()` showed a 128.13 USDG action charge, is executed one second later, after the owner's recapitalization landed inside its 60-second execution window: it is assessed 551,724,137,963.03 USDG, the whole custody of 50,063.79 USDG is collected, and the pool gains 50,031.65 USDG. In this arrangement the lens still quotes 95.99 USDG after the recapitalization, because the views show `D = 0`; the depth that the execution prices on is 0.000174 USDG of carry realized in the same call. Without the recapitalization the same order leaves the account 49,935.66 USDG.

When the trader's limits are finite, the close fails instead. With an action-charge ceiling of 1,600 USDG the stop-loss attempt ends with `ConstraintViolation(ActionCharge)`, and the protection becomes `Latched`. While it is latched, an ordinary close reverts with `OrderRouter__ProtectionActive`, `cancelPositionProtection()` reverts with `OrderRouter__ProtectionNotArmed`, and a withdrawal of the account's 39,967.60 USDG of free settlement reverts with `CfdEngine__DegradedMode`. 

**Sequence without an owner action.** The same state is reached when the owner does nothing. In a second arrangement of the deficit, a third-party SHORT of 150,000 units that opened at the 2.00 cap is still open when the basket is back at 0.80, with a mark loss of 180,000 USDG, and trader claims exceed pool assets by 96,419.86 USDG. When that third party closes part of its position, its realized loss is paid into the pool, and net depth becomes the amount by which the payment exceeds the shortfall. The third party's remaining SHORT reduces the skew term, so the charge at a given depth is lower than in the first arrangement.

**Conditions.** Net depth uses the same formula as the solvency cash `E` (pool assets net of trader claims, packages/perps/src/CfdEngine.sol:865-871), so `D < L` is raw insolvency. Near-zero depth with open positions follows trader payouts and claims that reach pool assets, which on the real paths are recorded by `_payOrRecordTraderClaim()` as a terminal trader-claim deficit once cash cannot cover them (packages/perps/src/CfdEngine.sol:849-855); the close or liquidation that leaves `E < L` latches degraded mode through `_enterDegradedModeIfInsolvent()` (packages/perps/src/CfdEngine.sol:1351-1368). `recapitalizePool()` does not clear degraded mode (packages/perps/SECURITY.md:161), and `clearDegradedMode()` requires `E >= L` (packages/perps/src/CfdEngine.sol:658-667). While claims exceed pool assets by more than the carry a close realizes, `D` stays zero at pricing, and exits cost the normal amount. The behaviour therefore starts when pool assets rise to just above trader claims. Two measured ways lead there: the owner's recapitalization of about the shortfall, and a payment of a trader's realized loss that exceeds the shortfall, after which `D` equals the excess. An account that closes part of a losing position chooses the size of that payment in steps of one position quantum (100 units, which is 120 USDG of realized loss per step in the proof of concept). Other inflows to pool assets of a similar size were not measured. The behaviour applies to minority-side closes until depth recovers or the majority side reduces skew. New LP deposits do not restore depth in this state, because `_entryStatusBlocked()` treats degraded mode as a blocked-entry state 


## Impact Explanation

An account whose exit was authorized with the documented unlimited limits loses its whole custody to the pool when that exit executes while net depth is just above zero, after a recapitalization or after a third party's loss payment: 50,063.79 USDG on an 80,000 USDG close in the proof of concept, of which 48,935.99 USDG is in excess of what the same stop-loss costs at normal depth, and the loss includes the account's free settlement balance as well as the position margin. At a net depth of 4,740.14 USDG the same exit costs 11,002.66 USDG more than its control. The amount is booked as pool revenue, so it accrues to LPs and backs outstanding trader claims. The trader takes no action in the low-depth state, and in the committed-order case the lens preview does not show the charge even after the recapitalization. An account with finite limits does not lose funds but cannot exit.

## Recommendation

Consider bounding the depth used to price a skew-increasing close from below, so that the fresh charge stays proportional to the skew it creates.


## Proof of Concept

## Proof of Concept

The test deploys the production contracts from `BasePerpTest` with the release risk parameters, the release Router configuration, release senior capacity and 1 USDG seeds. Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked. The junior LP deposits through the public ERC-7540 vault functions. Every trade is a production V3 `router.commitOrder(OrderRequest)` by the trader, executed by a third-party keeper with `router.executeOrder()`; protections use `createPositionProtection()`, the keeper's `triggerPositionProtection()` and `retryPositionProtectionClose()`. The owner calls only `recapitalizePool()`, and in the tests of item 5 the owner calls nothing. The test writes no storage, mocks no call and impersonates no protocol contract. The basket price is 0.80 (0.79 at the stop-loss). The deficit is reached on real paths: a third-party SHORT opens at 0.80, the basket reaches the 2.00 cap, a third-party LONG opens there, the SHORT closes and is paid in cash, the basket returns to 0.80, and the LONG closes with a profit above the remaining pool cash, which the engine records as a trader claim. The tests of item 5 use a second arrangement of the same basket move, with a smaller winning SHORT and one more third-party SHORT that opens at the cap and is still open at 0.80.

1. `test_Control_ArmedBeforeDeficit_NoRecapitalization()` is the control: the stop-loss is armed in the normal market and executes after the deficit with no recapitalization. The account keeps 48,935.76 USDG.
2. `test_ArmedBeforeDeficit_ExactShortfallRecapitalization()` is the same sequence with the owner's recapitalization of exactly the shortfall: custody goes to 0.
3. `test_CommittedAtZeroDepth_ExactShortfallLandsBeforeExecution()` commits an ordinary close at `D = 0`, where the preview shows a 128.13 USDG charge, and executes it one second later, after the recapitalization: custody goes to 0.
4. `test_CarefulLimits_ArmedBeforeDeficit_ExactShortfallRecapitalization()` uses a 1,600 USDG action-charge ceiling: the close fails, the protection latches, the account cannot close, cancel or withdraw, and a retry 25 hours later fails again.
5. `test_Control_NoOwnerAction_ThirdPartyDoesNotClose()`, `test_NoOwnerAction_ThirdPartyPartialCloseLeavesSixtyUsdgDepth()` and `test_NoOwnerAction_ThirdPartyPartialCloseLeavesFiveThousandUsdgDepth()` involve no owner action: a third party closes part of a losing SHORT after the deficit and its realized loss lifts pool assets above trader claims. The account keeps 48,935.78 USDG in the control, 0 at a net depth of 60.14 USDG and 37,933.12 USDG at a net depth of 4,740.14 USDG.

Command (from the repository root, forge 1.9.0, run against unmodified a82d4fef sources):

```
forge test --root packages/perps --match-path test/Submission_C08.t.sol -vv
```

Output (test results and logs; compiler warnings omitted; amounts in USDG):

```
Ran 7 tests for test/Submission_C08.t.sol:ArmedExitNearZeroDepthTest
[PASS] test_ArmedBeforeDeficit_ExactShortfallRecapitalization() (gas: 23294170)
Logs:
  case: armed before the deficit, recapitalization = exact shortfall
    net depth when the exit was authorized: 1000034.012222
    lens VPI quote when the exit was authorized: 95.987522
    net depth just before the exit: 0.000000
    lens VPI quote just before the exit: 384507962.064674
    status (2 = Executed, 3 = Failed): 2
    receipt: VPI charged: 384231782.697564
    receipt: action charge collected: 49063.587503
    receipt: action charge waived: 384182750.953703
    receipt: failed action-charge check, assessed: 0.000000
    minority custody before: 50063.787503
    minority custody after: 0.000000
    pool assets gained at execution: 50031.987503

[PASS] test_CarefulLimits_ArmedBeforeDeficit_ExactShortfallRecapitalization() (gas: 26460957)
Logs:
  case: careful limits, armed before the deficit, recapitalization = exact shortfall
    net depth when the exit was authorized: 1000034.012222
    lens VPI quote when the exit was authorized: 95.987522
    net depth just before the exit: 0.000000
    lens VPI quote just before the exit: 384507962.064674
    status (2 = Executed, 3 = Failed): 3
    receipt: VPI charged: 0.000000
    receipt: action charge collected: 0.000000
    receipt: action charge waived: 0.000000
    receipt: failed action-charge check, assessed: 384231814.541206
    minority custody before: 50063.787503
    minority custody after: 50063.587503
    pool assets gained at execution: 0.000000
    retry 25 hours later: status (3 = Failed): 3
    retry 25 hours later: assessed action charge: 5868745.565738
    minority custody after the retry: 50063.587503
    minority free settlement (not withdrawable): 39967.600000

[PASS] test_CommittedAtZeroDepth_ExactShortfallLandsBeforeExecution() (gas: 22599500)
Logs:
  case: ordinary close committed at D == 0, recapitalization = exact shortfall
    net depth when the exit was authorized: 0.000000
    lens VPI quote when the exit was authorized: 95.987522
    net depth just before the exit: 0.000000
    lens VPI quote just before the exit: 95.987522
    status (2 = Executed, 3 = Failed): 2
    receipt: VPI charged: 551724137931.034483
    receipt: action charge collected: 50063.787522
    receipt: action charge waived: 551724087899.385884
    receipt: failed action-charge check, assessed: 0.000000
    minority custody before: 50063.787522
    minority custody after: 0.000000
    pool assets gained at execution: 50031.648773
    CfdClosePreview at commit: action charge: 128.126271

[PASS] test_Control_ArmedBeforeDeficit_NoRecapitalization() (gas: 22994632)
Logs:
  case: control: armed before the deficit, no recapitalization
    net depth when the exit was authorized: 1000034.012222
    lens VPI quote when the exit was authorized: 95.987522
    net depth just before the exit: 0.000000
    lens VPI quote just before the exit: 95.987522
    status (2 = Executed, 3 = Failed): 2
    receipt: VPI charged: 95.987522
    receipt: action charge collected: 127.831164
    receipt: action charge waived: 0.000000
    receipt: failed action-charge check, assessed: 0.000000
    minority custody before: 50063.787503
    minority custody after: 48935.756339
    pool assets gained at execution: 1096.231164

[PASS] test_Control_NoOwnerAction_ThirdPartyDoesNotClose() (gas: 26841881)
Logs:
  case: control: no owner action, the third party does not close
    net depth when the exit was authorized: 1000034.012222
    lens VPI quote when the exit was authorized: 95.987522
    net depth just before the exit: 0.000000
    lens VPI quote just before the exit: 95.987522
    status (2 = Executed, 3 = Failed): 2
    receipt: VPI charged: 95.987522
    receipt: action charge collected: 127.806532
    receipt: action charge waived: 0.000000
    receipt: failed action-charge check, assessed: 0.000000
    minority custody before: 50063.787503
    minority custody after: 48935.780971
    pool assets gained at execution: 1096.206532
    shortfall after the deficit (claims - pool assets): 96419.855144
    pool assets gained from the third party's close: 0.000000
    third party's SHORT still open (units): 150000.000000000000000000

[PASS] test_NoOwnerAction_ThirdPartyPartialCloseLeavesFiveThousandUsdgDepth() (gas: 30545258)
Logs:
  case: no owner action, the third party closes 84,300 units
    net depth when the exit was authorized: 1000034.012222
    lens VPI quote when the exit was authorized: 95.987522
    net depth just before the exit: 4740.144856
    lens VPI quote just before the exit: 11098.648013
    status (2 = Executed, 3 = Failed): 2
    receipt: VPI charged: 11098.647605
    receipt: action charge collected: 11130.466789
    receipt: action charge waived: 0.000000
    receipt: failed action-charge check, assessed: 0.000000
    minority custody before: 50063.787503
    minority custody after: 37933.120714
    pool assets gained at execution: 12098.866789
    shortfall after the deficit (claims - pool assets): 96419.855144
    pool assets gained from the third party's close: 101160.000000
    third party's SHORT still open (units): 65700.000000000000000000

[PASS] test_NoOwnerAction_ThirdPartyPartialCloseLeavesSixtyUsdgDepth() (gas: 30571530)
Logs:
  case: no owner action, the third party closes 80,400 units
    net depth when the exit was authorized: 1000034.012222
    lens VPI quote when the exit was authorized: 95.987522
    net depth just before the exit: 60.144856
    lens VPI quote just before the exit: 831252.922071
    status (2 = Executed, 3 = Failed): 2
    receipt: VPI charged: 831250.525975
    receipt: action charge collected: 49063.587503
    receipt: action charge waived: 782218.757656
    receipt: failed action-charge check, assessed: 0.000000
    minority custody before: 50063.787503
    minority custody after: 0.000000
    pool assets gained at execution: 50031.987503
    shortfall after the deficit (claims - pool assets): 96419.855144
    pool assets gained from the third party's close: 96480.000000
    third party's SHORT still open (units): 69600.000000000000000000

Suite result: ok. 7 passed; 0 failed; 0 skipped
```

Test file `packages/perps/test/Submission_C08.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdClosePreview} from "@plether/perps/CfdClosePreview.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Exits authorized before net depth became small. Close-time VPI is `0.5 * k * (S_post^2 - S_pre^2) / D`
///         with D = pool assets minus trader claims; `CloseAccountingLib.buildCloseState()` skips the fresh charge
///         only when D == 0. The tests measure the minority SHORT's exit (a stop-loss armed at normal depth, or an
///         ordinary close committed at D == 0) when D is just above zero at execution. Two ways lead there after a
///         trader-claim deficit: the owner's documented `engine.recapitalizePool()` of exactly the shortfall (tests
///         2 to 4), and, with no owner action, a third party that closes a losing position and pays its realized
///         loss into the pool (tests 6 and 7). Tests 1 and 5 are the controls. Keepers act at once and honestly.
/// @dev    Release risk parameters, release Router configuration, release senior capacity and 1 USDG seeds
///         (DEPLOYMENT.md). Only Pyth (MockPyth) and USDG (MockSettlementToken) are mocked. The junior LP deposits
///         through the public ERC-7540 vault functions and a third-party keeper settles the epoch with
///         `router.settleLpEpoch()`. Every trade is a production V3 `router.commitOrder(OrderRequest)` by the trader,
///         executed by a third-party keeper with `router.executeOrder()`; between orders a keeper publishes marks
///         with `router.updateMarkPrice()`. Protections use the public `PositionProtectionBook` actions
///         (`createPositionProtection`, keeper `triggerPositionProtection`, permissionless
///         `retryPositionProtectionClose`). `CfdClosePreview` is the stateless pre-commit preview from src. Router:
///         `LegacyOrderRouterHarness`, a subclass of `OrderRouter` whose only addition (a scalar commit helper) is not
///         used here. No storage write, call mock or impersonated protocol contract.
///         Real-path deficit (`_arrangeDeficit()`, Tuesday 2024-03-12 08:10-08:30 UTC): the majority LONG and the
///         minority SHORT are open at 0.80; a third-party SHORT opens at 0.80; the basket reaches the 2.00 cap and
///         a second third party opens a LONG; the SHORT closes at 2.00 and is paid in cash, which latches degraded
///         mode; the basket returns to 0.80 and the LONG's profit is recorded as a trader claim above pool assets
///         (D == 0).
contract ArmedExitNearZeroDepthTest is BasePerpTest {

    /// @dev Measurements for one exit, from the moment it was authorized to its execution.
    struct Exit {
        uint64 protectionId;
        uint64 orderId;
        uint256 depthAtAuthorization;
        int256 quoteAtAuthorization;
        int256 vpiAccruedAtAuthorization;
        uint256 depthBeforeExit;
        int256 quoteBeforeExit;
        uint256 custodyBefore;
        uint256 custodyAfter;
        uint256 poolBefore;
        uint256 poolAfter;
        uint64 executionDeadline;
        uint64 executedAt;
        OrderV3Types.ExecutionResult result;
        OrderV3Types.OrderReceipt receipt;
    }

    address internal constant MAJORITY = address(0xA001);
    address internal constant MINORITY = address(0xA002);
    address internal constant CLAIMANT = address(0xA003); // LONG whose profit at 0.80 becomes the trader claim
    address internal constant WINNER = address(0xA004); // SHORT paid in cash at the cap
    address internal constant LOSER = address(0xA005); // no-owner tests: SHORT opened at the cap, losing at 0.80
    address internal constant JUNIOR_LP = address(0x1A18);
    address internal constant KEEPER = address(0x4EE718);

    uint256 internal constant LOT = 1e20; // positionSizeQuantum
    uint256 internal constant PRICE = 0.8e8; // basket != $1.00
    uint256 internal constant HIGH = CAP_PRICE; // 2.00
    uint256 internal constant STOP_LOSS = 0.79e8; // the minority SHORT's stop-loss (a SHORT loses as the basket falls)
    uint256 internal constant MAJ_SIZE = 200_000e18; // 160,000 USDG LONG notional
    uint256 internal constant MIN_SIZE = 100_000e18; // 80,000 USDG SHORT notional (minority side)
    uint256 internal constant WINNER_SIZE = 600_000e18; // largest SHORT the skew cap admits at 0.80
    uint256 internal constant CLAIMANT_SIZE = 416_800e18; // largest LONG the solvency buffer admits at 2.00
    uint256 internal constant SMALLER_WINNER_SIZE = 500_000e18; // no-owner tests: 400,000 USDG SHORT notional at 0.80
    uint256 internal constant LOSER_SIZE = 150_000e18; // no-owner tests: 300,000 USDG SHORT notional at 2.00
    uint256 internal constant MAJ_FUNDING = 300_000e6;
    uint256 internal constant MIN_FUNDING = 50_000e6;
    uint256 internal constant SWING_FUNDING = 20_000e6;
    uint256 internal constant MAJ_MARGIN = 260_000e6; // covers the 240,000 USDG mark loss at the cap
    uint256 internal constant MIN_MARGIN = 10_000e6;
    uint256 internal constant SWING_MARGIN = 10_000e6;
    uint256 internal constant LOSER_FUNDING = 210_000e6;
    uint256 internal constant LOSER_MARGIN = 200_000e6; // covers the 180,000 USDG mark loss at 0.80
    uint256 internal constant JUNIOR_DEPOSIT = 1_000_000e6;
    uint256 internal constant UNLIMITED = type(uint256).max; // "an explicit unlimited allowance" (PositionProtectionTypes)
    uint256 internal constant CAREFUL_ACTION_CHARGE = 1600e6; // 2% of the 80,000 USDG closed notional
    uint256 internal constant SCENARIO_START = SETUP_TIMESTAMP + 8 days; // Tuesday 2024-03-12 06:00 UTC

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    CfdClosePreview internal closePreview;
    uint256 internal winnerCashGain;
    uint256 internal shortfallAfterDeficit;
    uint256 internal poolGainFromLoserClose;

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
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        config.adverseConfidenceMultiplierBps = 2500;
        config.positionProtectionTriggerBountyAssets = 200_000;
        _setRouterConfig(config);
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release min open notional");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen close spread");

        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        closePreview = new CfdClosePreview();

        vm.warp(SCENARIO_START);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(PRICE);

        // A junior LP funds the pool through the vault; a third-party keeper settles the epoch.
        settlementToken.mint(JUNIOR_LP, JUNIOR_DEPOSIT);
        vm.startPrank(JUNIOR_LP);
        settlementToken.approve(address(juniorVault), JUNIOR_DEPOSIT);
        uint256 requestId = juniorVault.requestDeposit(JUNIOR_DEPOSIT, JUNIOR_LP, JUNIOR_LP);
        vm.stopPrank();
        _nextEpoch();
        _keeperSettle();
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorVault.claimDeposit(requestId, claimable, JUNIOR_LP, JUNIOR_LP);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(PRICE);

        // A majority LONG and a minority SHORT through the real commit -> keeper execute path (08:00 UTC).
        _fundTrader(MAJORITY, MAJ_FUNDING);
        _fundTrader(MINORITY, MIN_FUNDING);
        _executeOk(_commit(MAJORITY, CfdTypes.Side.LONG, MAJ_SIZE, MAJ_MARGIN, false, UNLIMITED), PRICE);
        _executeOk(_commit(MINORITY, CfdTypes.Side.SHORT, MIN_SIZE, MIN_MARGIN, false, UNLIMITED), PRICE);
        assertEq(_majoritySize(), MAJ_SIZE, "majority LONG open");
        assertEq(_minoritySize(), MIN_SIZE, "minority SHORT open");
    }

    // ------------------------------------------------------------------ oracle, order and LP helpers

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

    /// @dev Any keeper publishes a fresh mark through the Router.
    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _livePriceData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _warpBy(
        uint256 secondsForward
    ) internal {
        vm.warp(vm.getBlockTimestamp() + secondsForward);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _nextEpoch() internal {
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _keeperSettle() internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _livePriceData(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    /// @dev Production V3 request by the account itself. Opens use an unrestrictive slippage target; closes of a SHORT
    ///      accept any price at or above 1. `maxActionChargeAssets` is the trader's VPI/carry allowance.
    function _request(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 maxActionChargeAssets
    ) internal view returns (OrderV3Types.OrderRequest memory req) {
        uint256 target =
            isClose ? (side == CfdTypes.Side.LONG ? CAP_PRICE : 1) : (side == CfdTypes.Side.LONG ? 1 : CAP_PRICE);
        req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, size, isClose, vm.getBlockTimestamp(), router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: UNLIMITED,
                maxExecutionNotionalAssets: UNLIMITED,
                maxGrossAccountDebitAssets: UNLIMITED,
                maxActionChargeAssets: maxActionChargeAssets,
                maxExplicitFeesAssets: UNLIMITED,
                maxPostPositionSize: UNLIMITED,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 maxActionChargeAssets
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory req = _request(account, side, size, margin, isClose, maxActionChargeAssets);
        vm.prank(account);
        orderId = router.commitOrder(req);
    }

    /// @dev A third-party keeper executes with the first post-commit Pyth tick at `price`.
    function _execute(
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        result = router.executeOrder(orderId, data);
    }

    function _executeOk(
        uint64 orderId,
        uint256 price
    ) internal {
        OrderV3Types.ExecutionResult memory result = _execute(orderId, price);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    // ------------------------------------------------------------------ protection helpers

    /// @dev The minority creates a stop-loss protection for its whole SHORT through the public Book action. Only the
    ///      action-charge ceiling varies; the other three close limits are unlimited. Creation reads the latest posted
    ///      Pyth basket, which must not be older than the engine mark, so a keeper first publishes the current 0.80
    ///      mark through the Router (MockPyth keeps the last posted tick, not the execution tick).
    function _protect(
        uint256 maxActionChargeAssets
    ) internal returns (uint64 protectionId) {
        _refreshMark(PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: UNLIMITED,
            maxGrossAccountDebitAssets: UNLIMITED,
            maxActionChargeAssets: maxActionChargeAssets,
            maxExplicitFeesAssets: UNLIMITED
        });
        vm.prank(MINORITY);
        protectionId = protectionActions.createPositionProtection(params);
        assertEq(
            uint8(_protectionStatus(protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "protection armed"
        );
    }

    /// @dev A keeper publishes the tick that crosses the stop-loss and triggers the protection in the same call.
    function _trigger(
        uint64 protectionId,
        uint256 price
    ) internal returns (uint64 linkedOrderId) {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _livePriceData(price);
        vm.prank(KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, data);
    }

    function _protectionStatus(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _receipt(
        uint64 orderId
    ) internal returns (OrderV3Types.OrderReceipt memory) {
        return _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), orderId);
    }

    // ------------------------------------------------------------------ real-path deficit and owner recovery

    /// @dev Two third parties trade a 0.80 -> 2.00 -> 0.80 basket move (Tuesday 08:10-08:30 UTC) while the majority
    ///      LONG and the minority SHORT stay open. The open guard caps each side's maximum profit at pool cash
    ///      (`E >= L + B` with L the larger side), so a single winner cannot exceed pool cash; the second winner opens
    ///      while the first is still open, and the first winner's close at the cap leaves E < L.
    function _arrangeDeficit() internal {
        _fundTrader(WINNER, SWING_FUNDING);
        _fundTrader(CLAIMANT, SWING_FUNDING);

        // 08:10: the winner opens the largest SHORT the skew cap admits at 0.80.
        _warpBy(10 minutes);
        _refreshMark(PRICE);
        ICfdEngineTypes.OpenPreview memory next =
            engineLens.previewOpen(WINNER, CfdTypes.Side.SHORT, WINNER_SIZE + LOT, SWING_MARGIN, PRICE);
        assertEq(uint8(next.invalidReason), uint8(CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH), "short at cap");
        _executeOk(_commit(WINNER, CfdTypes.Side.SHORT, WINNER_SIZE, SWING_MARGIN, false, UNLIMITED), PRICE);

        // 08:20: the basket reaches the 2.00 cap; the claimant opens the largest LONG the solvency buffer admits.
        _warpBy(10 minutes);
        _refreshMark(HIGH);
        next = engineLens.previewOpen(CLAIMANT, CfdTypes.Side.LONG, CLAIMANT_SIZE + LOT, SWING_MARGIN, HIGH);
        assertEq(uint8(next.invalidReason), uint8(CfdEnginePlanTypes.OpenRevertCode.SOLVENCY_EXCEEDED), "long at cap");
        _executeOk(_commit(CLAIMANT, CfdTypes.Side.LONG, CLAIMANT_SIZE, SWING_MARGIN, false, UNLIMITED), HIGH);
        assertFalse(engine.degradedMode(), "long admitted under E >= L + B");

        // The winner takes its profit at the cap; it is paid in cash and the close leaves E < L.
        uint256 winnerBefore = clearinghouse.balanceAssets(WINNER);
        _executeOk(_commit(WINNER, CfdTypes.Side.SHORT, WINNER_SIZE, 0, true, UNLIMITED), HIGH);
        winnerCashGain = clearinghouse.balanceAssets(WINNER) - winnerBefore;
        assertEq(winnerCashGain, 714_295_789_724, "winner paid in cash");
        assertTrue(engine.degradedMode(), "the engine latches degraded mode at the winner's close");

        // 08:30: the basket returns to 0.80; the claimant's profit exceeds pool cash and is recorded as a claim.
        _warpBy(10 minutes);
        _refreshMark(PRICE);
        _executeOk(_commit(CLAIMANT, CfdTypes.Side.LONG, CLAIMANT_SIZE, 0, true, UNLIMITED), PRICE);
        assertEq(engine.traderClaimBalanceAssets(CLAIMANT), 495_168_964_576, "claimant's profit is a trader claim");
        assertEq(pool.totalAssets(), 281_788_996_943, "pool cash left");
        assertEq(_shortfall(), 213_379_967_633, "claims exceed pool assets");
        assertEq(_netDepth(), 0, "no net depth");
        assertEq(_majoritySize(), MAJ_SIZE, "majority LONG still open");
        assertEq(_minoritySize(), MIN_SIZE, "minority SHORT still open");
        assertEq(_maxLiability(), 160_000e6, "L is the majority LONG's maximum profit");
        assertEq(engine.lastMarkPrice(), PRICE, "mark back at 0.80");
    }

    /// @dev Variant for the tests without an owner action. Same basket move and same order of trades as
    ///      `_arrangeDeficit()`, with a smaller winning SHORT and one more third party: a SHORT that opens at the 2.00
    ///      cap next to the claimant's LONG and is still open, with a mark loss of 180,000 USDG, when the basket is
    ///      back at 0.80. The deficit (claims above pool assets) is smaller than that loss.
    function _arrangeDeficitWithOpenLoser() internal {
        _fundTrader(WINNER, SWING_FUNDING);
        _fundTrader(CLAIMANT, SWING_FUNDING);
        _fundTrader(LOSER, LOSER_FUNDING);

        // 08:10: the winner opens a SHORT at 0.80.
        _warpBy(10 minutes);
        _refreshMark(PRICE);
        _executeOk(_commit(WINNER, CfdTypes.Side.SHORT, SMALLER_WINNER_SIZE, SWING_MARGIN, false, UNLIMITED), PRICE);

        // 08:20: the basket reaches the 2.00 cap; the claimant opens a LONG and the later loser opens a SHORT.
        _warpBy(10 minutes);
        _refreshMark(HIGH);
        _executeOk(_commit(CLAIMANT, CfdTypes.Side.LONG, CLAIMANT_SIZE, SWING_MARGIN, false, UNLIMITED), HIGH);
        _executeOk(_commit(LOSER, CfdTypes.Side.SHORT, LOSER_SIZE, LOSER_MARGIN, false, UNLIMITED), HIGH);
        assertFalse(engine.degradedMode(), "both opens admitted in a solvent market");

        // The winner takes its profit at the cap; it is paid in cash and the close leaves E < L.
        _executeOk(_commit(WINNER, CfdTypes.Side.SHORT, SMALLER_WINNER_SIZE, 0, true, UNLIMITED), HIGH);
        assertTrue(engine.degradedMode(), "the engine latches degraded mode at the winner's close");

        // 08:30: the basket returns to 0.80; the claimant's profit exceeds pool cash and is recorded as a claim.
        _warpBy(10 minutes);
        _refreshMark(PRICE);
        _executeOk(_commit(CLAIMANT, CfdTypes.Side.LONG, CLAIMANT_SIZE, 0, true, UNLIMITED), PRICE);
        assertGt(_shortfall(), 0, "claims exceed pool assets");
        assertEq(_netDepth(), 0, "no net depth");
        assertEq(_majoritySize(), MAJ_SIZE, "majority LONG still open");
        assertEq(_minoritySize(), MIN_SIZE, "minority SHORT still open");
        assertEq(_size(LOSER), LOSER_SIZE, "losing SHORT still open");
        assertEq(engine.lastMarkPrice(), PRICE, "mark back at 0.80");
    }

    /// @dev Documented owner action: recapitalize exactly the shortfall read from the views.
    function _ownerRecapitalizeShortfall() internal {
        uint256 amount = _shortfall();
        settlementToken.mint(engine.owner(), amount);
        vm.startPrank(engine.owner());
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
        vm.stopPrank();
        assertEq(_netDepth(), 0, "the views show no net depth after the recapitalization");
    }

    function _shortfall() internal view returns (uint256) {
        uint256 assets = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return claims > assets ? claims - assets : 0;
    }

    function _netDepth() internal view returns (uint256) {
        uint256 assets = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return assets > claims ? assets - claims : 0;
    }

    /// @dev Planner diagnostic quote (`CfdEngineLens.previewClose`) for the minority's full close at `price`.
    function _quote(
        uint256 price
    ) internal view returns (int256) {
        return engineLens.previewClose(MINORITY, MIN_SIZE, price).vpiDeltaAssets;
    }

    function _minorityVpiAccrued() internal view returns (int256 vpiAccrued) {
        (,,,,,, vpiAccrued) = engine.positions(MINORITY);
    }

    function _majoritySize() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(MAJORITY);
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _minoritySize() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(MINORITY);
    }

    // ------------------------------------------------------------------ scenario runners

    /// @dev The minority arms a stop-loss for its whole SHORT at normal depth, and the deficit then forms on real
    ///      paths. After it, the owner recapitalizes exactly the shortfall (`recapitalize`), or a third party closes
    ///      `thirdPartyCloseSize` of its losing SHORT at 0.80 and its realized loss is paid into the pool, or nothing
    ///      is paid in (control). Ten minutes later the basket falls to the stop-loss; a keeper triggers the
    ///      protection with that tick and a keeper executes the linked close with the next tick.
    function _armedStopLossExit(
        bool recapitalize,
        bool thirdPartyShort,
        uint256 thirdPartyCloseSize,
        uint256 maxActionChargeAssets
    ) internal returns (Exit memory e) {
        _startRecordingLogs();
        e.depthAtAuthorization = _netDepth();
        e.quoteAtAuthorization = _quote(STOP_LOSS);
        e.vpiAccruedAtAuthorization = _minorityVpiAccrued();
        e.protectionId = _protect(maxActionChargeAssets);
        if (thirdPartyShort) {
            _arrangeDeficitWithOpenLoser();
        } else {
            _arrangeDeficit();
        }
        assertEq(
            uint8(_protectionStatus(e.protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "still armed after the deficit"
        );
        shortfallAfterDeficit = _shortfall();
        if (recapitalize) {
            _ownerRecapitalizeShortfall();
        }
        if (thirdPartyCloseSize != 0) {
            uint256 poolBeforeLoserClose = pool.totalAssets();
            _executeOk(_commit(LOSER, CfdTypes.Side.SHORT, thirdPartyCloseSize, 0, true, UNLIMITED), PRICE);
            poolGainFromLoserClose = pool.totalAssets() - poolBeforeLoserClose;
        }
        assertTrue(engine.degradedMode(), "degraded mode still latched");
        _warpBy(10 minutes);
        e.depthBeforeExit = _netDepth();
        e.quoteBeforeExit = _quote(STOP_LOSS);
        e.custodyBefore = clearinghouse.balanceAssets(MINORITY);
        e.poolBefore = pool.totalAssets();
        e.orderId = _trigger(e.protectionId, STOP_LOSS);
        e.executionDeadline = router.lifecycleBook().orderTiming(e.orderId).executionDeadline;
        e.result = _execute(e.orderId, STOP_LOSS);
        e.executedAt = uint64(vm.getBlockTimestamp());
        e.custodyAfter = clearinghouse.balanceAssets(MINORITY);
        e.poolAfter = pool.totalAssets();
        e.receipt = _receipt(e.orderId);
        assertLe(e.executedAt, e.executionDeadline, "executed inside the execution window");
    }

    /// @dev Ordinary V3 full close with unlimited limits, committed by the minority at D == 0 (degraded mode, claims
    ///      above pool assets). The owner's recapitalization of exactly the shortfall lands in the same block, after
    ///      the commit; a keeper executes in the next block with the first post-commit tick at a flat 0.80.
    function _committedCloseThenRecapitalization()
        internal
        returns (Exit memory e, CfdClosePreview.ClosePreview memory preCommit)
    {
        _startRecordingLogs();
        _arrangeDeficit();
        OrderV3Types.OrderRequest memory request = _request(MINORITY, CfdTypes.Side.SHORT, MIN_SIZE, 0, true, UNLIMITED);
        e.depthAtAuthorization = _netDepth();
        e.quoteAtAuthorization = _quote(PRICE);
        e.vpiAccruedAtAuthorization = _minorityVpiAccrued();
        preCommit = closePreview.previewClose(address(engine), MINORITY, request, KEEPER, PRICE);
        e.custodyBefore = clearinghouse.balanceAssets(MINORITY);
        vm.prank(MINORITY);
        e.orderId = router.commitOrder(request);
        e.executionDeadline = router.lifecycleBook().orderTiming(e.orderId).executionDeadline;
        _ownerRecapitalizeShortfall();
        e.depthBeforeExit = _netDepth();
        e.quoteBeforeExit = _quote(PRICE);
        e.poolBefore = pool.totalAssets();
        e.result = _execute(e.orderId, PRICE);
        e.executedAt = uint64(vm.getBlockTimestamp());
        e.custodyAfter = clearinghouse.balanceAssets(MINORITY);
        e.poolAfter = pool.totalAssets();
        e.receipt = _receipt(e.orderId);
        assertLe(e.executedAt, e.executionDeadline, "executed inside the execution window");
    }

    function _poolGain(
        Exit memory e
    ) internal pure returns (int256) {
        return int256(e.poolAfter) - int256(e.poolBefore);
    }

    /// @dev Amounts are printed in USDG.
    function _logExit(
        string memory label,
        Exit memory e
    ) internal {
        emit log_named_string("case", label);
        emit log_named_decimal_uint("  net depth when the exit was authorized", e.depthAtAuthorization, 6);
        emit log_named_decimal_int("  lens VPI quote when the exit was authorized", e.quoteAtAuthorization, 6);
        emit log_named_decimal_uint("  net depth just before the exit", e.depthBeforeExit, 6);
        emit log_named_decimal_int("  lens VPI quote just before the exit", e.quoteBeforeExit, 6);
        emit log_named_uint("  status (2 = Executed, 3 = Failed)", uint8(e.result.status));
        emit log_named_decimal_int("  receipt: VPI charged", e.receipt.economics.vpiAssets, 6);
        emit log_named_decimal_uint(
            "  receipt: action charge collected", e.receipt.economics.actionChargeCollectedAssets, 6
        );
        emit log_named_decimal_uint(
            "  receipt: action charge waived", e.receipt.economics.close.actionChargeWaivedAssets, 6
        );
        emit log_named_decimal_uint("  receipt: failed action-charge check, assessed", e.receipt.failure.actual, 6);
        emit log_named_decimal_uint("  minority custody before", e.custodyBefore, 6);
        emit log_named_decimal_uint("  minority custody after", e.custodyAfter, 6);
        emit log_named_decimal_int("  pool assets gained at execution", _poolGain(e), 6);
    }

    function _logThirdPartyClose() internal {
        emit log_named_decimal_uint("  shortfall after the deficit (claims - pool assets)", shortfallAfterDeficit, 6);
        emit log_named_decimal_uint("  pool assets gained from the third party's close", poolGainFromLoserClose, 6);
        emit log_named_decimal_uint("  third party's SHORT still open (units)", _size(LOSER), 18);
    }

    function _assertExecutedWipe(
        Exit memory e
    ) internal view {
        assertEq(uint8(e.result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "the close executes");
        assertEq(_minoritySize(), 0, "position closed");
        assertEq(e.custodyAfter, 0, "nothing of the account is left");
        assertGt(_poolGain(e), 50_000e6, "the pool gains more than 50,000 USDG");
        assertGt(e.receipt.economics.close.actionChargeWaivedAssets, 0, "the uncollectible rest is waived");
        assertTrue(engine.degradedMode(), "degraded mode still latched");
    }

    // ------------------------------------------------------------------ tests

    /// @dev Test 1, control for tests 2 and 4. The stop-loss is armed at normal depth with all four close limits
    ///      unlimited and triggers after the deficit, with no recapitalization. D stays 0 (claims exceed pool assets
    ///      by far more than the account's carry), the fresh charge is skipped, and the close costs the normal
    ///      amount: the 1,000 USDG price move, 95.99 USDG of VPI (the open rebate is recovered), the execution fee,
    ///      carry and the 0.20 USDG trigger bounty.
    function test_Control_ArmedBeforeDeficit_NoRecapitalization() public {
        Exit memory e = _armedStopLossExit(false, false, 0, UNLIMITED);
        _logExit("control: armed before the deficit, no recapitalization", e);
        assertEq(uint8(e.result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "executed");
        assertEq(_minoritySize(), 0, "closed");
        assertEq(e.depthAtAuthorization, 1_000_034_012_222, "armed at normal depth");
        assertEq(e.quoteAtAuthorization, 95_987_522, "quote when armed");
        assertEq(shortfallAfterDeficit, 213_379_967_633, "claims exceed pool assets");
        assertEq(e.depthBeforeExit, 0, "D == 0 before the exit");
        assertEq(e.quoteBeforeExit, 95_987_522, "no fresh VPI quoted");
        assertEq(e.receipt.economics.vpiAssets, 95_987_522, "no fresh VPI charged");
        assertEq(e.receipt.economics.close.actionChargeWaivedAssets, 0, "nothing waived");
        assertEq(e.custodyBefore, 50_063_787_503, "custody before");
        assertEq(e.custodyAfter, 48_935_756_339, "custody after");
        assertEq(_poolGain(e), 1_096_231_164, "pool gain");
    }

    /// @dev Test 2. As test 1, but the owner recapitalizes exactly the shortfall read from the views (D == 0)
    ///      before the basket falls to the stop-loss. At execution the account's own carry since arming (0.24 USDG)
    ///      lifts D above zero and the close is charged 384,231,782.70 USDG of VPI: the account's whole custody is
    ///      collected.
    function test_ArmedBeforeDeficit_ExactShortfallRecapitalization() public {
        Exit memory e = _armedStopLossExit(true, false, 0, UNLIMITED);
        _logExit("armed before the deficit, recapitalization = exact shortfall", e);
        _assertExecutedWipe(e);
        assertEq(
            uint8(_protectionStatus(e.protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Executed),
            "protection executed"
        );
        assertEq(e.depthAtAuthorization, 1_000_034_012_222, "armed at normal depth");
        assertEq(e.quoteAtAuthorization, 95_987_522, "quote when armed");
        assertEq(e.depthBeforeExit, 0, "views show D == 0 before the trigger");
        assertEq(e.quoteBeforeExit, 384_507_962_064_674, "quote after the recapitalization");
        assertEq(e.receipt.economics.carryAssets, 243_642, "carry realized at execution");
        assertEq(e.receipt.economics.vpiAssets, 384_231_782_697_564, "VPI charged");
        assertEq(e.receipt.economics.actionChargeCollectedAssets, 49_063_587_503, "action charge collected");
        assertEq(e.custodyBefore, 50_063_787_503, "custody before");
        assertEq(_poolGain(e), 50_031_987_503, "pool gain");
    }

    /// @dev Test 3. Ordinary close with an unlimited action-charge bound committed at D == 0, where the pre-commit
    ///      preview (`CfdClosePreview`) assesses a 128.13 USDG action charge. The owner's recapitalization of
    ///      exactly the shortfall lands in the same block; the planner lens still quotes no fresh VPI. One second
    ///      later a keeper executes inside the 60-second window: one second of the account's carry (174 atoms)
    ///      lifts D above zero and the close is charged 551,724,137,931.03 USDG of VPI.
    function test_CommittedAtZeroDepth_ExactShortfallLandsBeforeExecution() public {
        (Exit memory e, CfdClosePreview.ClosePreview memory pre) = _committedCloseThenRecapitalization();
        _logExit("ordinary close committed at D == 0, recapitalization = exact shortfall", e);
        emit log_named_decimal_uint(
            "  CfdClosePreview at commit: action charge", pre.assessment.actionChargeAssessedAssets, 6
        );
        _assertExecutedWipe(e);
        assertEq(e.depthAtAuthorization, 0, "committed at D == 0");
        assertEq(pre.assessment.vpiAssets, 95_987_522, "pre-commit preview VPI");
        assertEq(pre.assessment.vpiAssets, -e.vpiAccruedAtAuthorization, "pre-commit preview: no fresh VPI");
        assertEq(pre.assessment.actionChargeAssessedAssets, 128_126_271, "pre-commit preview action charge");
        assertEq(e.depthBeforeExit, 0, "views show D == 0 after the recapitalization");
        assertEq(e.quoteBeforeExit, 95_987_522, "the lens still quotes no fresh VPI after the recapitalization");
        assertEq(e.executedAt, e.receipt.timing.commitTimestamp + 1, "executed one second after the commit");
        assertEq(e.executionDeadline, e.receipt.timing.commitTimestamp + 60, "60-second execution window");
        assertEq(e.receipt.economics.carryAssets, 138_923, "carry at commit plus one second at execution");
        assertEq(e.receipt.economics.vpiAssets, 551_724_137_931_034_483, "VPI charged");
        assertEq(e.receipt.economics.actionChargeCollectedAssets, 50_063_787_522, "action charge collected");
        assertEq(e.custodyBefore, 50_063_787_522, "custody before the commit");
        assertEq(_poolGain(e), 50_031_648_773, "pool gain at execution");
    }

    /// @dev Test 4. As test 2, with a 2% action-charge ceiling (1,600 USDG on the 80,000 USDG notional). The linked
    ///      close fails and the protection latches. While latched, the trader cannot commit an ordinary close
    ///      (`OrderRouter__ProtectionActive`), cannot cancel (`OrderRouter__ProtectionNotArmed`) and cannot withdraw
    ///      (`CfdEngine__DegradedMode`). A third-party retry 25 hours later fails again.
    function test_CarefulLimits_ArmedBeforeDeficit_ExactShortfallRecapitalization() public {
        Exit memory e = _armedStopLossExit(true, false, 0, CAREFUL_ACTION_CHARGE);
        _logExit("careful limits, armed before the deficit, recapitalization = exact shortfall", e);
        assertEq(uint8(e.result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "the close fails");
        assertEq(uint8(e.result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "reason");
        assertEq(
            e.receipt.failure.selector,
            ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
            "failure selector"
        );
        assertEq(uint8(e.receipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.ActionCharge), "constraint");
        assertEq(e.receipt.failure.actual, 384_231_814_541_206, "assessed action charge");
        assertEq(e.receipt.failure.limit, CAREFUL_ACTION_CHARGE, "the 2% ceiling");
        assertEq(_poolGain(e), 0, "nothing moves to the pool");
        assertEq(
            uint8(_protectionStatus(e.protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "protection latched"
        );
        assertEq(e.custodyBefore - e.custodyAfter, 200_000, "only the 0.20 USDG trigger bounty left the account");

        OrderV3Types.OrderRequest memory req =
            _request(MINORITY, CfdTypes.Side.SHORT, MIN_SIZE, 0, true, CAREFUL_ACTION_CHARGE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        vm.prank(MINORITY);
        router.commitOrder(req);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        vm.prank(MINORITY);
        protectionActions.cancelPositionProtection(e.protectionId);
        IMarginClearinghouse.AccountAssetBuckets memory buckets = clearinghouse.getAccountAssetBuckets(MINORITY);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        vm.prank(MINORITY);
        clearinghouse.withdraw(MINORITY, buckets.freeSettlementAssets);

        _warpBy(25 hours);
        _refreshMark(STOP_LOSS);
        vm.prank(KEEPER);
        uint64 retryId = protectionActions.retryPositionProtectionClose(e.protectionId);
        OrderV3Types.ExecutionResult memory r2 = _execute(retryId, STOP_LOSS);
        OrderV3Types.OrderReceipt memory rc2 = _receipt(retryId);
        uint256 custodyLater = clearinghouse.balanceAssets(MINORITY);
        emit log_named_uint("  retry 25 hours later: status (3 = Failed)", uint8(r2.status));
        emit log_named_decimal_uint("  retry 25 hours later: assessed action charge", rc2.failure.actual, 6);
        emit log_named_decimal_uint("  minority custody after the retry", custodyLater, 6);
        emit log_named_decimal_uint("  minority free settlement (not withdrawable)", buckets.freeSettlementAssets, 6);

        assertEq(uint8(r2.status), uint8(OrderV3Types.LifecycleStatus.Failed), "retry fails");
        assertEq(uint8(rc2.failure.constraint), uint8(OrderV3Types.ConstraintKind.ActionCharge), "retry constraint");
        assertEq(rc2.failure.actual, 5_868_745_565_738, "retry assessed action charge");
        assertEq(
            uint8(_protectionStatus(e.protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "still latched"
        );
        assertEq(_minoritySize(), MIN_SIZE, "position still open after 25 hours");
        assertEq(custodyLater, 50_063_587_503, "custody unchanged");
        assertEq(buckets.freeSettlementAssets, 39_967_600_000, "free settlement, not withdrawable");
        assertTrue(engine.degradedMode(), "degraded mode still latched");
    }

    /// @dev Test 5, control for tests 6 and 7 (no owner action). The stop-loss is armed at normal depth with
    ///      unlimited limits, the deficit forms while a third party's losing SHORT is still open, and nothing is paid
    ///      into the pool: D stays 0, the fresh charge is skipped and the close costs the normal amount.
    function test_Control_NoOwnerAction_ThirdPartyDoesNotClose() public {
        Exit memory e = _armedStopLossExit(false, true, 0, UNLIMITED);
        _logExit("control: no owner action, the third party does not close", e);
        _logThirdPartyClose();
        assertEq(uint8(e.result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "executed");
        assertEq(_minoritySize(), 0, "closed");
        assertEq(shortfallAfterDeficit, 96_419_855_144, "claims exceed pool assets");
        assertEq(e.depthBeforeExit, 0, "D == 0 before the exit");
        assertEq(e.quoteBeforeExit, 95_987_522, "no fresh VPI quoted");
        assertEq(e.receipt.economics.vpiAssets, 95_987_522, "no fresh VPI charged");
        assertEq(e.receipt.economics.close.actionChargeWaivedAssets, 0, "nothing waived");
        assertEq(e.custodyBefore, 50_063_787_503, "custody before");
        assertEq(e.custodyAfter, 48_935_780_971, "custody after");
        assertEq(_poolGain(e), 1_096_206_532, "pool gain");
    }

    /// @dev Test 6. No owner action. After the deficit the third party closes 80,400 of its 150,000 units at 0.80.
    ///      Its realized loss of 96,480 USDG is paid into the pool and exceeds the 96,419.86 USDG shortfall by
    ///      60.14 USDG, so D is 60.14 USDG. Ten minutes later the minority's stop-loss, armed at normal depth with
    ///      unlimited limits, executes: the close is charged 831,250.53 USDG of VPI and the account's whole custody is
    ///      collected.
    function test_NoOwnerAction_ThirdPartyPartialCloseLeavesSixtyUsdgDepth() public {
        Exit memory e = _armedStopLossExit(false, true, 80_400e18, UNLIMITED);
        _logExit("no owner action, the third party closes 80,400 units", e);
        _logThirdPartyClose();
        _assertExecutedWipe(e);
        assertEq(
            uint8(_protectionStatus(e.protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Executed),
            "protection executed"
        );
        assertEq(e.depthAtAuthorization, 1_000_034_012_222, "armed at normal depth");
        assertEq(e.quoteAtAuthorization, 95_987_522, "quote when armed");
        assertEq(shortfallAfterDeficit, 96_419_855_144, "claims exceed pool assets");
        assertEq(poolGainFromLoserClose, 96_480e6, "the third party's realized loss reaches the pool");
        assertEq(e.depthBeforeExit, 60_144_856, "D == loss paid in minus the shortfall");
        assertEq(e.receipt.economics.vpiAssets, 831_250_525_975, "VPI charged");
        assertEq(e.receipt.economics.actionChargeCollectedAssets, 49_063_587_503, "action charge collected");
        assertEq(e.custodyBefore, 50_063_787_503, "custody before");
        assertEq(_poolGain(e), 50_031_987_503, "pool gain");
    }

    /// @dev Test 7. As test 6, with a close of 84,300 units: the loss paid in exceeds the shortfall by 4,740.14 USDG.
    ///      The stop-loss is charged 11,098.65 USDG of VPI (about 14% of the 79,000 USDG closed notional) and the
    ///      account ends 11,002.66 USDG below the control of test 5.
    function test_NoOwnerAction_ThirdPartyPartialCloseLeavesFiveThousandUsdgDepth() public {
        Exit memory e = _armedStopLossExit(false, true, 84_300e18, UNLIMITED);
        _logExit("no owner action, the third party closes 84,300 units", e);
        _logThirdPartyClose();
        assertEq(uint8(e.result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "the close executes");
        assertEq(_minoritySize(), 0, "position closed");
        assertEq(poolGainFromLoserClose, 101_160e6, "the third party's realized loss reaches the pool");
        assertEq(e.depthBeforeExit, 4_740_144_856, "D == loss paid in minus the shortfall");
        assertEq(e.receipt.economics.vpiAssets, 11_098_647_605, "VPI charged");
        assertEq(e.receipt.economics.close.actionChargeWaivedAssets, 0, "nothing waived");
        assertEq(e.custodyBefore, 50_063_787_503, "custody before");
        assertEq(e.custodyAfter, 37_933_120_714, "custody after");
        assertEq(48_935_780_971 - e.custodyAfter, 11_002_660_257, "extra cost against the control of test 5");
        assertEq(_poolGain(e), 12_098_866_789, "pool gain");
    }

}
```
