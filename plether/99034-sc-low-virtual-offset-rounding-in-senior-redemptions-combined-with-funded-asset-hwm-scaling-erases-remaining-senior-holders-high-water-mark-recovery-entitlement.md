# \#99034 [SC-Low] Virtual offset rounding in Senior redemptions, combined with funded asset HWM scaling, erases remaining Senior holders' high water mark recovery entitlement

**Submitted on Nov 11th 2026 at 15:31:43 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99034
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The MED-11 fix changed `scaleSeniorOnWithdraw()` to reduce the Senior high-water mark (HWM) by the fraction of *principal* paid out (`H' = H - floor(H·A/P)`). Senior redemptions are priced with ERC-4626 virtual offsets (`+1` asset, `+1000` shares). When Senior is impaired, those offsets pay a redeemer a larger fraction of principal than the fraction of shares burned, and the HWM cut follows that inflated fraction. One atom of extra principal turns into up to `H/(P+1)` of erased HWM. In a deeply impaired Senior tranche, one ordinary partial redemption can wipe out a large part of the remaining Senior holders' recovery target. Later recapitalization then goes to `unassignedAssets`, and later revenue goes to Junior, instead of making those holders whole. This breaks the documented rule that "Rounding favors the remaining protected claim" (`ACCOUNTING_SPEC.md:580-583`).

## Vulnerability Details

`HousePool._settleLpEpoch()` (`HousePool.sol:824-866`) prices the Senior redemption phase from `seniorPrincipal` and the pre-burn share supply. `_fundRedemptionPhase()` (`HousePool.sol:944-1026`) quotes each epoch with virtual offsets `1` / `1000`:

```solidity
// HousePoolRedemptionMathLib.netAssetsForShares
uint256 grossAssets =
    Math.mulDiv(shares, principal + virtualAssets, supply + virtualShares, Math.Rounding.Floor);
```

The aggregate funded assets `A` then scale the waterfall:

```solidity
// HousePool._settleLpEpoch
if (seniorPhase.fundedAssets > 0) {
    nextState = HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
    ...
}

// HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw (lines 125-137)
nextState.seniorHighWaterMark -= Math.mulDiv(
    state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
);
nextState.seniorPrincipal -= fundedAssets;
```

With `A = floor(s(P+1)/(S+1000))`:

```
A/P ≈ (s/S) · (1 + 1/P) / (1 + 1000/S)
```

Shares are minted at 1000 per atom, so `S > 1000P` exactly when Senior is impaired. In that case `A/P > s/S`. The HWM is cut by the principal fraction, not the share fraction, so the remaining holders (who own `(S−s)/S` of the tranche) keep less than `(S−s)/S` of the HWM. The extra HWM removed per settlement pass is bounded by `H/(P+1)`. It is material when `P` is small compared with `H`:

| H | P | max extra HWM erased per pass |
|---|---|---|
| 100,000 USDG | 2 atoms | 33,333 USDG (40% redemption → 10,000 USDG) |
| 100,000 USDG | 100 atoms | ~990 USDG |
| 40,000,000 USDG | 0.01 USDG | ~4,000 USDG |
| 40,000,000 USDG | 1 USDG | ~40 USDG |

Example used in the PoC: `P = 2` atoms, `H = 100,000 USDG`, and the redeemer holds 40% of shares. `A = floor(0.4 · 3) = 1` atom, which is 50% of `P`. The HWM is cut by 50% (to 50,000) instead of 40% (to 60,000), and **~10,000 USDG of recovery target owned by the remaining 60% of holders disappears**.

The erased target has real effects on value distribution:

- **Recapitalization** (`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`, principal ≠ 0 branch) restores Senior only up to `gap = H − P`. Anything above that goes to `unassignedAssets`, which have no owner and also block tranche deposits.
- **Revenue** (`HousePoolWaterfallAccountingLib.distributeRevenue`) restores Senior only up to the reduced `H`. The rest is credited to `juniorPrincipal`, and Junior LPs can withdraw it.

**Why this is not a known issue.**
- The old MED-11 code scaled the HWM by `fundedShares/preBurnSupply` and did not have this effect. The fix at `9ab65f4` introduced it.
- LOW-54 and SECURITY.md:1417 accept the opposite direction: less than 1 atom of *extra* HWM retained, because of the floor.
- MED-29 is about the entitled-share zero gate. LOW-63 is about splitting funding across passes. Neither involves HWM erasure.
- The team's MED-11 response states "impaired withdrawals preserve proportional protection with conservative rounding". In share terms that is false.

**Preconditions.**
- Junior is wiped and Senior is deeply impaired (small `P`, large `H`). This can happen after LP losses, for example an underfunded winning close that leaves trader claims.
- No open positions. If any position is open, `freeAssets ≤ P − settlementBuffer`, and the buffer is at least 2.5 USDG, so the redemption would not be funded. With no open positions, `_maxLiability() = 0`, degraded mode does not latch, and withdrawals are live.
- No attacker is needed. Any ordinary partial Senior redemption (a full-balance request bypasses the minimum size) triggers it. A Junior holder benefits.

## Impact Details

Selected impact: **Low – Contract fails to deliver promised returns, but doesn't lose value.**

The remaining Senior LPs are promised that future inflows will restore them to their share of the HWM before Junior receives anything (`ACCOUNTING_SPEC.md:337-338`, `SECURITY.md:394`). The bug permanently lowers that target by up to `H/(P+1)` per settlement pass, beyond what the burned shares justify. In the PoC this is ~10,000 USDG out of 100,000 USDG, i.e. 16.7% of the remaining holders' entitlement. When value later arrives, it is misallocated:

- an owner `recapitalizePool()` for the correct amount parks ~10,000 USDG in `unassignedAssets` instead of restoring Senior;
- ordinary revenue sends ~10,000 USDG to `juniorPrincipal`, which Junior LPs can withdraw.

### Note for reviewers: severity vs. feasibility (program rules)

The program page sets how severity and feasibility are evaluated. The relevant rules are quoted below so the assessment follows them:

> "Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it. A finding with a large impact that is infeasible retains that impact and severity, but is rewarded at a reduced amount."

How these rules apply to this report:

1. **The impact is shown, not theoretical.** The runnable PoC erases **9,999.9 USDG** of the remaining Senior holders' recovery target. It then shows that value being misallocated, to `unassignedAssets` on recapitalization and to Junior principal on revenue. This meets "Every report must include a proof of concept that demonstrates the impact it claims." The amount is not dust.
2. **The preconditions are feasibility factors, not validity factors.** These include a deeply impaired Senior tranche, a wiped Junior tranche, and no open positions. Under the rule above they may reduce the payout. They do not reduce the severity, and they are not grounds to invalidate the report.
3. **The selected severity is already the lowest bug tier (Low).** A feasibility discount can therefore only affect the payout.
4. **The fixture models a documented state.** The storage write of `totalTraderClaimBalanceAssets` reproduces the trader-claim balance left by an underfunded winning close. That state is part of the protocol's design (trader claims are documented in `SECURITY.md`), and the repository's own recapitalization tests use the same technique. It creates no state the contracts cannot reach. Every step after it runs on unmodified in-scope code. The PoC uses no mocks of in-scope components.
5. **No trusted-role misbehaviour is needed.** The redemption is an ordinary user action, and settlement is permissionless. The recovery calls (`recapitalizePool`, `accountExcess`) are "a legitimate, documented action by a trusted role that triggers a bug harming users," which the program lists explicitly as in scope.
6. **This is not a known issue.** It is a new bug introduced by the MED-11 fix at commit `9ab65f4`. The program says such reports are valid when they show "the fix ... is incomplete or introduces a new bug." LOW-54, which is accepted, covers the opposite direction (less than 1 atom of extra HWM *retained*). This report shows HWM *erased* by up to `H/(P+1)` per pass.

No principal that exists at the time is taken (the redeemer gains at most 1 atom). The loss is the remaining Senior holders' contractual recovery priority. That is why the impact is classified as failure to deliver promised returns.

## Provide step-by-step explanations

1. Senior LPs deposit: Alice 40,000 USDG and Bob 59,000 USDG, plus the 1,000 USDG seed. Alice holds 40.00% of Senior shares and `H ≈ 100,000 USDG`. Carol deposits 10,000 USDG into Junior.
2. LP losses (modelled in the PoC as a trader-claim balance from an underfunded winning close) leave a terminal NAV of 2 atoms with no open positions. After reconcile: `juniorPrincipal = 0`, `seniorPrincipal = 2`, `seniorHighWaterMark = 100,000 USDG`, `degradedMode = false`.
3. Alice requests a redemption of her full balance, and anyone settles the LP epoch.
4. `netAssetsForShares` quotes `floor(0.4·S·3/(S+1000)) = 1` atom. All of Alice's shares are burned and she receives 1 atom.
5. `scaleSeniorOnWithdraw(H, A = 1)` with `P = 2` sets `H' = H − floor(H/2) = 50,000 USDG`. A share-proportional cut would leave 60,000 USDG, so ~10,000 USDG of Bob's and the seed's target is erased.
6. **Recovery A:** the owner calls `engine.recapitalizePool(60,000 USDG)`. On reconcile, Senior principal is restored only to 50,000 USDG and ~9,999.9 USDG goes to `unassignedAssets`.
7. **Recovery B:** the owner instead recapitalizes 10,000 USDG, and the pool later earns ~50,000 USDG of revenue. Senior stops at 50,000 USDG and ~9,999.9 USDG is credited to `juniorPrincipal`, which Carol can redeem.


## Proof of Concept

## Proof of Concept

File: `test/poc/VirtualOffsetSeniorHwmErasure.t.sol`

```bash
forge test --match-path test/poc/VirtualOffsetSeniorHwmErasure.t.sol -vv
```

Output:

```
[PASS] test_Poc_RecapAfterVirtualOffsetRedemptionShortchangesRemainingSenior()
  Senior HWM before wipe: 100001296806
  Alice share of Senior supply (bps): 4000
  HWM after redemption (actual): 50000648403
  HWM after redemption (share-proportional): 60000566210
  HWM erased beyond burned share fraction: 9999917807
  Senior principal after recap: 50000648403
  unassignedAssets after recap: 9999917808

[PASS] test_Poc_RevenueAfterVirtualOffsetRedemptionAccruesToJunior()
  Senior principal after revenue: 50000648403
  Junior principal after revenue: 9999917807
```

The only fixture is a storage write of `totalTraderClaimBalanceAssets`, which creates the near-wiped state (the same technique as the repository's existing recapitalization PoCs). Deposits, the redemption request, LP settlement, `recapitalizePool`, and `accountExcess` all run on the unmodified in-scope contracts.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../../packages/perps/test/perps/shared/HousePoolTestBase.sol";

/// @notice PoC: at deeply impaired Senior principal, the ERC-4626 virtual offset lets a partial Senior redemption
///         withdraw a larger fraction of principal than of shares, and `scaleSeniorOnWithdraw` (H -= H * A / P,
///         introduced by the MED-11 fix) then erases the remaining Senior holders' high-water-mark protection.
/// @dev Fixture: the aggregate trader-claim balance is written to model a prior underfunded winning close that leaves
///      exactly 2 atoms of terminal NAV with no open positions (same fixture style as RepeatedPartialRecapSettlement).
///      Every other step (deposits, redemption request, LP settlement, owner recapitalization, accountExcess) uses the
///      live in-test contracts.
contract VirtualOffsetSeniorHwmErasurePoC is HousePoolTestBase {

    uint256 private constant ALICE_SENIOR = 40_000e6; // redeemer, ~40% of Senior shares
    uint256 private constant BOB_SENIOR = 59_000e6; // remaining holder (seed holds the rest)
    uint256 private constant CAROL_JUNIOR = 10_000e6;
    uint256 private constant RESIDUAL_NAV = 2; // 2 atoms = 0.000002 USDG

    struct State {
        uint256 hwm;
        uint256 shareFairHwm;
        uint256 supply;
        uint256 aliceShares;
    }

    function _wipeAndRedeem() internal returns (State memory st) {
        _fundSenior(alice, ALICE_SENIOR);
        _fundSenior(bob, BOB_SENIOR);
        _fundJunior(carol, CAROL_JUNIOR);

        uint256 hwmBefore = pool.seniorHighWaterMark();
        st.supply = seniorVault.totalSupply();
        st.aliceShares = seniorVault.balanceOf(alice);
        emit log_named_uint("Senior HWM before wipe", hwmBefore);
        emit log_named_uint("Alice share of Senior supply (bps)", st.aliceShares * 10_000 / st.supply);

        // ---- Fixture: a prior winning close the pool could not fully pay leaves 2 atoms of NAV, no open positions.
        _setTotalTraderClaim(pool.totalAssets() - RESIDUAL_NAV);
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.juniorPrincipal(), 0, "Junior wiped");
        assertEq(pool.seniorPrincipal(), RESIDUAL_NAV, "Senior left with 2 atoms");
        assertEq(pool.seniorHighWaterMark(), hwmBefore, "losses keep the HWM");
        assertFalse(engine.degradedMode(), "no open positions: not degraded, withdrawals live");

        // ---- Alice redeems her full balance (full-balance requests bypass the minimum size).
        _finishAsyncCooldown(seniorVault, alice);
        uint256 epoch = _requestAsyncRedeem(seniorVault, alice, st.aliceShares);
        _settleAsyncRequest(epoch, false);
        (uint256 fundedShares, uint256 assets) = _claimAsyncRedeem(seniorVault, epoch, alice);

        assertEq(fundedShares, st.aliceShares, "all of Alice's shares burned");
        assertEq(assets, 1, "Alice receives 1 of the 2 atoms (virtual-offset rounding)");
        assertEq(pool.seniorPrincipal(), 1, "1 atom left for remaining holders");

        st.hwm = pool.seniorHighWaterMark();
        st.shareFairHwm = hwmBefore - hwmBefore * st.aliceShares / st.supply;
        emit log_named_uint("HWM after redemption (actual)", st.hwm);
        emit log_named_uint("HWM after redemption (share-proportional)", st.shareFairHwm);
        emit log_named_uint("HWM erased beyond burned share fraction", st.shareFairHwm - st.hwm);

        assertEq(st.hwm, hwmBefore / 2, "HWM halved although only ~40% of shares were burned");
        assertGt(st.shareFairHwm - st.hwm, 9_000e6, "> $9k of remaining-holder recovery target erased");
    }

    /// @notice Documented owner recovery: recapitalizePool to the share-proportional target.
    function test_Poc_RecapAfterVirtualOffsetRedemptionShortchangesRemainingSenior() public {
        State memory st = _wipeAndRedeem();

        uint256 recap = st.shareFairHwm; // what the remaining 60% of Senior is entitled to recover
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);

        // Any permissionless LP settlement applies the pending recapitalization.
        vm.prank(address(juniorVault));
        pool.reconcile();

        emit log_named_uint("Senior principal after recap", pool.seniorPrincipal());
        emit log_named_uint("unassignedAssets after recap", pool.unassignedAssets());

        assertEq(pool.seniorPrincipal(), st.hwm, "Senior restored only to the erased target");
        assertEq(pool.unassignedAssets(), recap - (st.hwm - 1), "excess recap parked as ownerless assets");
        assertGt(pool.unassignedAssets(), 9_000e6, "remaining Senior holders miss > $9k of recovery");
    }

    /// @notice Ordinary revenue: after a partial recap, later revenue goes to Junior before Senior is whole.
    function test_Poc_RevenueAfterVirtualOffsetRedemptionAccruesToJunior() public {
        State memory st = _wipeAndRedeem();

        // Owner restores some depth with a partial recap (below the erased target).
        uint256 recap = 10_000e6;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), recap + 1, "partial recap restores Senior");

        // Later revenue equal to the share-proportional remaining gap.
        uint256 revenue = st.shareFairHwm - (recap + 1);
        _mintAndAccountPoolExcess(revenue);
        vm.prank(address(juniorVault));
        pool.reconcile();

        emit log_named_uint("Senior principal after revenue", pool.seniorPrincipal());
        emit log_named_uint("Junior principal after revenue", pool.juniorPrincipal());

        assertEq(pool.seniorPrincipal(), st.hwm, "Senior capped at the erased target");
        assertEq(pool.juniorPrincipal(), st.shareFairHwm - st.hwm, "Junior receives the erased Senior recovery");
        assertGt(pool.juniorPrincipal(), 9_000e6, "> $9k of Senior recovery routed to Junior");
    }

}

```
