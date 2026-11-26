# \#98258 [SC-Critical] Bootstrap mint is quoted before the principal credit: `assignUnassignedAssets` on a wiped Junior tranche mints `amount × (totalSupply + 1000)` shares and permanently transfers the surviving LPs' entire forward revenue claim to the bootstrap receiver

**Submitted on Nov 6th 2026 at 12:09:08 UTC by @Naveen0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98258
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield
  - Permanent freezing of unclaimed yield

## Description

## Summary

A trading loss larger than the Junior principal drives the pool into a state that is both reachable and, after a bad day, expected. `absorbLoss` is Junior-first, so any loss at or above the Junior principal saturates `juniorPrincipal` to exactly 0 while the Senior tranche absorbs the overshoot. Junior share supply does not go to zero with it — the seed share floor pins it above zero forever. The tranche is now `{principal == 0, supply > 0}`: `_isTerminallyWiped()` is true, ordinary deposits are refused, and the survivors' shares are worth nothing today.

Their shares are not worthless going forward, though. `juniorRevenueWithoutOwners` only routes Junior revenue away from the tranche when Junior supply is zero. Because supply survived the wipe, the whole future Junior revenue stream is contractually theirs — that is the asset at risk here.

The only way back out of this state is `assignUnassignedAssets(false, receiver)`. Permissionless deposits are blocked by the terminal-wipe gate, so no LP can restore pricing assets; the owner's assign is the designed recovery mechanism, and the PoC proves end to end that it is the sole Junior-regrowth path. But that function computes the bootstrap share quote **before** it credits the principal. At quote time `juniorPrincipal` is still 0, so `_convertToSharesUsingAssets` divides by `0 + 1` and returns `amount × (accruedTotalSupply() + 1000)` shares — in the PoC, 11,006,908,650,165,000,000,000,000 shares against a parity rate of roughly 500,328,983,317,381, a factor of about 22 billion. The receiver is minted that number and ends up owning effectively the entire tranche: not just the assigned bucket, but every Junior revenue accrual from that point on. The survivors keep under 0.1%, and the receiver converts the whole thing to settlement token through a completely normal async redeem a cooldown later.

The same vault already refuses to price this exact state on the deposit lane. `estimateDepositShares` returns 0 when pricing assets are zero and shares survive. The bootstrap lane runs the same division on the same vault, and it is the one that actually mints.

## Severity

**High** — Theft of unclaimed yield / Permanent freezing of unclaimed yield.

The value that is confiscated is the surviving Junior LPs' claim on all future Junior revenue. In the wiped state their principal backing is zero, but their forward claim is not: `juniorRevenueWithoutOwners` diverts Junior revenue to ownerless assets **only when Junior supply is zero**, so with supply pinned above zero every future revenue delta accrues 100% to the survivors. The corner mint converts that claim to ~0 and hands it to the bootstrap receiver, who realises it through an ordinary async redeem. The confiscation is permanent and repeats on every subsequent revenue accrual.

This is not an attack that requires a privileged role to act maliciously or outside its documented procedures. `assignUnassignedAssets` is listed among the owner's immediate, documented powers ("assign ownerless pool assets"), and the PoC shows that in the post-wipe state it is the **only** Junior-regrowth path that exists. An entirely honest owner, performing the documented recovery action, assigning to a neutral third-party receiver, still destroys the surviving LPs' forward claim. That places it squarely in "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol."

## Affected code

- `HousePool.sol:660-703` — `assignUnassignedAssets`. The share quote is taken at `:685`, **before** `juniorPrincipal += amount` at `:699`; the mint executes at `:702`.
- `HousePool.sol:653-657` — the function's own natspec contract: it bootstraps quarantined LP assets into a tranche "by minting matching shares".
- `TrancheVault.sol:1392-1395` — `quoteBootstrapDeposit` delegates straight to `_convertToSharesUsingAssets(assets, _depositPricingAssets())` with no state guard.
- `TrancheVault.sol:1848-1852` — `_depositPricingAssets()` resolves to the tranche principal from `POOL.getPendingDepositTrancheState()`, which is exactly 0 at the call site.
- `TrancheVault.sol:1853-1859` — `_convertToSharesUsingAssets`: `Math.mulDiv(assets, accruedTotalSupply() + VIRTUAL_SHARES, totalAssets_ + 1)`. The denominator is `0 + 1`.
- `TrancheVault.sol:36` — `VIRTUAL_SHARES = 1000`.
- `TrancheVault.sol:1361-1369` — `estimateDepositShares` guards the identical condition at `:1366` (`if (pricingAssets == 0 && pricingSupply != 0) return 0;`). The bootstrap lane has no equivalent.
- `TrancheVault.sol:1317-1323` and `:1582-1586` — in this state `maxRequestDeposit` returns 0 and `requestDeposit` reverts `TrancheVault__TerminallyWiped`, so no permissionless deposit can restore pricing assets.
- `TrancheVault.sol:1639-1641` — `_isTerminallyWiped()` is `totalSupply() > 0 && totalAssets() == 0`.
- `TrancheVault.sol:322-323` — the seed share floor keeps Junior supply above zero permanently, which is what makes the corner persistent rather than transient.
- `HousePoolReconcilePlanLib.sol:113-132` — `juniorRevenueWithoutOwners`: the supply-above-zero carve-out that gives the wiped survivors their forward claim, applied at `HousePool.sol:1591-1595`.
- `SECURITY.md:244-245` — the owner capability assertion that this breaks (quoted below).

## Root cause

1. **The corner state is produced by the code, not assumed.** `absorbLoss` is Junior-first; a loss at or above the Junior principal saturates it to exactly 0. The seed receiver's shares cannot drop below `seedShareFloor` (`TrancheVault.sol:322-323`), so Junior supply survives indefinitely. Result: `juniorPrincipal == 0`, `totalSupply() > 0`, `_isTerminallyWiped() == true`.

2. **A non-zero unassigned bucket coexists with it.** Recapitalisation that arrives while the Senior gap is closed lands in `unassignedAssets`, and the bucket survives the wipe reconcile. The PoC asserts both.

3. **The state has exactly one exit, and it is this function.** `maxRequestDeposit` returns 0 through `_isTerminallyWiped()` (`TrancheVault.sol:1319`) and `requestDeposit` reverts `TrancheVault__TerminallyWiped` (`:1585`), so nobody can deposit their way out. The survivors' own redeems return 0 because principal backing is 0. The PoC (test A2) proves the Junior assign is the only regrowth path and that it consumes the whole bucket. The Senior branch is not an alternative: under the release parameters it is capacity-dead in the wipe state, which test A1 proves by expected revert.

4. **The quote precedes the credit.** At `HousePool.sol:685` the function runs `uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);` while `juniorPrincipal` is still 0. Only afterwards does `:699` run `juniorPrincipal += amount;` and `:702` `bootstrapMint(shares, receiver)`. Swapping those two statements would remove the bug entirely.

5. **The vault divides by one.** `quoteBootstrapDeposit` → `_convertToSharesUsingAssets(amount, _depositPricingAssets())`, where `_depositPricingAssets()` is the tranche principal = 0. The formula at `:1858` is `Math.mulDiv(assets, accruedTotalSupply() + VIRTUAL_SHARES, totalAssets_ + 1)`, so the result is `amount × (supply + 1000) / 1`. The virtual-share offset that normally keeps inflation attacks harmless becomes irrelevant, because the real supply term dominates.

6. **The mint executes at that quote.** After the assign, the Junior principal equals the assigned bucket and backs a supply that is more than 99.999999999% owned by the receiver. The survivors' residual claim on the bucket falls below 0.1%, and — the part that actually matters — every future Junior revenue delta now backs the receiver's shares instead of theirs, because the diversion carve-out in `juniorRevenueWithoutOwners` only fires when supply is zero and their supply is not zero.

7. **No guard on this path covers it.** The guards on `assignUnassignedAssets` are oracle liveness (`:669`), mark freshness (`:670`) and Senior deposit capacity (`:677-679`, Senior branch only). None of them looks at the zero-principal quote, in direct contrast with the deposit lane's explicit check for the identical condition.

8. **Realisation needs no privilege.** After the one-hour deposit cooldown the receiver redeems through the standard async epoch and walks away with settlement token. Nothing exotic happens at the exit.

## Impact

Measured in the PoC, with a real balance movement: the receiver's settlement-token balance increases by **489,749,835,616 base units (489,749.84 USDG)** through an ordinary async redeem — 96.03% of the 510,000.00 USDG then backing the tranche — while the wiped Junior LP's final share value is **0**. The remaining ~4% is epoch funding budget and virtual-share dust and stays claimable; it is not value that returns to the victims.

Breaking that figure into what is actually owner-provided and what is victim-owned, because the distinction matters for severity:

The **500,000.00 USDG bucket** is owner-recycled recapitalisation value. Assigning it to a chosen receiver is a documented owner power, and on that component the strict reading of "existing claimant backing" is arguable, since the survivors' *current* principal backing at assign time is zero. I am not claiming that component as theft.

The victim-owned deltas are the ones that make this a vulnerability:

- **The revenue accrual the receiver captures.** 10,000.00 USDG in the PoC. Without the corner mint, the supply-above-zero carve-out in `juniorRevenueWithoutOwners` routes it 100% to the surviving Junior supply. With the corner mint, more than 99.9% of it backs the receiver's shares and leaves through the receiver's redeem. In production this amount is unbounded.
- **The permanent forward claim.** This is not a one-off. The survivors' entire future revenue stream is structurally confiscated by a single mint. Every subsequent accrual repeats the capture automatically, with no further owner action, for as long as those shares exist.
- **The counterfactual share of the bucket itself.** Priced correctly — against post-credit principal — the receiver would be minted roughly the existing supply, leaving the survivors about half the post-assign tranche instead of a rounding error.

**Preconditions, stated plainly.** The corner needs a terminal Junior wipe: a trading loss at least equal to the full Junior principal. That is a severe event, but it is exactly the event the Junior tranche exists to absorb, and it is fully reachable through ordinary trading — the PoC produces it with a normal open and close, no special configuration. It also needs a non-zero unassigned bucket and one owner transaction. What makes this worse than a rare corner is that test A2 proves this call **is** the recovery mechanism: there is no alternative path, so *every* post-wipe recovery mints at the broken rate until the quote is fixed.

**The victims cannot protect themselves.** Junior deposits are blocked in this state, their redeems are worth zero, and the assign is a single atomic owner transaction. There is nothing to front-run, nothing to dilute, no exit.

**SECURITY.md** states, in the "Instant owner controls" list (`SECURITY.md:244-245`), that the owner may admit unaccounted pool excess, sweep that excess, or assign unowned accounted assets through the guarded bootstrap path, and that these powers do not permit withdrawal of existing claimant backing. The corner withdraws more than the bucket: existing-claimant revenue leaves through the owner power. The "guarded" qualifier does not hold for the guard that matters. This sentence is a capability assertion about what owner powers do *not* permit, not a Known Limitations entry, so it is not covered by the known-issues exclusion.

## Failure path

1. The pool runs under the release parameters (Senior cap 40,000,000 USDG, Senior share cap 80%), installed through the pool's own timelocked config path. A Junior LP and a Senior LP are funded.
2. The owner recapitalises 500,000.00 USDG while the Senior gap is closed, so the inflow lands in `unassignedAssets`. A reconcile confirms the bucket sits beside untouched tranches.
3. A trader opens and closes a position at a loss to the pool larger than the full Junior principal. Junior-first `absorbLoss` saturates `juniorPrincipal` to exactly 0, the Senior tranche shaves only the overshoot, the seed floor holds Junior supply at 22,013,817,299,330 shares, and the bucket survives the wipe reconcile.
4. The Junior deposit lane is now dead: `maxRequestDeposit` is 0 and `requestDeposit` reverts `TrancheVault__TerminallyWiped`. No LP can restore pricing assets, and the wiped LPs' shares redeem for 0.
5. The owner calls `assignUnassignedAssets(false, receiver)` — the documented recovery action, and the only one available. The quote is computed while `juniorPrincipal == 0`: `quoteBootstrapDeposit(500_000e6)` returns `500_000e6 × (22,013,817,299,330 + 1000)` = 11,006,908,650,165,000,000,000,000 shares. Parity would have been about 500,328,983,317,381.
6. `juniorPrincipal += bucket` runs and the receiver is minted the quoted shares, taking effectively the whole tranche. The survivors' residual claim on the bucket drops below 0.1%.
7. 10,000.00 USDG of Junior-bound revenue accrues. The Senior gap covenant still holds and closes first, then the revenue lands in `juniorPrincipal` behind the receiver-dominated supply. The survivors retain under 0.1% of bucket plus revenue.
8. After the one-hour cooldown the receiver redeems asynchronously and receives 489,749,835,616 base units. The survivors' final value is 0, and every later revenue delta repeats step 7 in the receiver's favour.

## Proof of Concept

## Proof of Concept

Four Foundry tests, each independently falsifiable. The suite drives the real in-scope `HousePool`, both `TrancheVault` instances, `CfdEngine` and the waterfall/reconcile libraries through the repository's own `HousePoolTestBase` / `HousePoolAsyncTestBase` harnesses. No contract in scope is modified. The only substitutions are `MockSettlementToken` for USDG (out of scope, explicitly permitted) and the suite-standard Router-pranked `engine.updateMarkPrice` for mark freshness — no Pyth path is involved in this flow. No fork and no external state; reruns are deterministic.

- **A1** — after a full wipe of both tranches, the Senior assign branch reverts `HousePool__ExceedsSeniorDepositCapacity` under the release parameters (with Junior principal 0 the ratio exposure limit is 0), and the blocked assign leaves all state untouched. The Senior bootstrap lane is capacity-dead exactly in the wipe states, so the surviving lane is the Junior branch.
- **A2** — the Junior assign is the only Junior-regrowth path and consumes the whole bucket; later inflows gap-fill the open Senior gap first; only overflow beyond the restored high-water mark re-funds `unassignedAssets`, and the then-runnable Senior assign takes the preserving `+=` branch. A1 and A2 together establish that the corner configuration is the canonical post-wipe state and that its only exit prices at the corner.
- **B** — the corner itself, end to end with realised value: the quote, the mint, receiver ownership above 99.9%, the survivors' claim below 0.1% of bucket and below 0.1% of bucket plus revenue (with the Senior-gap covenant verified intact), then a normal async redeem measuring the receiver's balance delta at +489,749,835,616 and the survivors' final value at 0.
- **C** — negative control: the identical owner assign with pricing assets above zero leaves every pre-existing shareholder's claim exactly unchanged and mints the receiver a parity-rate slice. This isolates the zero-principal state as the sole cause and shows the mint mechanism is claim-neutral whenever it is priced against non-zero principal.

The test imports `HousePoolTestBase` from `../perps/shared/HousePoolTestBase.sol`, so it must sit in a directory that is a sibling of the perps test directory — `test/audit/SeniorWipeBootstrapCornerPoC.t.sol` under the perps package. Run:

```
forge test --root packages/perps --match-path "test/audit/SeniorWipeBootstrapCornerPoC.t.sol" -vvv
```

### SeniorWipeBootstrapCornerPoC.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../perps/shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Zero-principal bootstrap-quote corner on the Junior tranche.
///
/// After a terminal Junior wipe (juniorPrincipal == 0 while Junior share supply stays
/// pinned above zero by the seed floor), the owner's documented recovery call
/// assignUnassignedAssets(false, receiver) quotes the bootstrap mint BEFORE crediting
/// Junior principal (HousePool.sol:685 vs :699). TrancheVault._convertToSharesUsingAssets
/// therefore divides by (pricingAssets + 1) == 1 and mints amount * (supply + 1000) shares
/// (TrancheVault.sol:1392-1395 -> :1853-1859), handing the receiver effectively the whole
/// tranche and all future Junior revenue, which the supply-above-zero covenant
/// (HousePoolReconcilePlanLib.juniorRevenueWithoutOwners, :113-132) routes to the wiped
/// survivors. The receiver realises the value through a normal async redeem.
///
/// A1/A2 prove the corner is the canonical post-wipe state under the release parameters
/// (Senior cap 40M USDG, Senior share cap 80%): the Senior assign branch is capacity-dead
/// after a wipe, and the Junior branch is the only regrowth path.
/// B proves the corner mint and the realised extraction.
/// C is the negative control: with pricing assets above zero the same assign is exactly
/// claim-neutral, which is also the condition the ordinary deposit path already guards
/// (estimateDepositShares returns 0 when pricingAssets == 0 && supply != 0,
/// TrancheVault.sol:1366) while the bootstrap path does not.
contract SeniorWipeBootstrapCornerPoC is HousePoolTestBase {
    uint256 internal constant SENIOR_FUND = 80_000e6; // alice (funded after junior: ratio gate)
    uint256 internal constant JUNIOR_FUND = 21_000e6; // bob
    uint256 internal constant RECAP_U = 500_000e6; // ownerless bucket accrued while senior gap == 0
    uint256 internal constant SENIOR_SHAVE = 2_000e6; // loss overshoot beyond junior
    uint256 internal constant REVENUE = 10_000e6; // post-bootstrap junior-bound revenue
    uint256 internal constant PROD_MAX_SENIOR_EXPOSURE = 40_000_000e6; // release parameter
    uint256 internal constant PROD_MAX_SENIOR_SHARE_BPS = 8000; // release parameter
    uint256 internal constant TRADE_MARGIN = 30_000e6;

    address internal constant RECEIVER_X = address(0xBAD);
    address internal constant RECEIVER_Y = address(0xBEE);

    uint256 internal seniorP; // funded senior principal (== HWM, unimpaired)
    uint256 internal juniorJ; // funded junior principal
    uint256 internal preservedHwm;

    function _installProductionParamsAndZeroCoupon() internal {
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.seniorRateBps = 0;
        config.maxSeniorExposureAssets = PROD_MAX_SENIOR_EXPOSURE;
        config.maxSeniorShareBps = PROD_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        // finalizePoolConfig hard-requires a fresh raw mark for rate changes.
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));
        pool.finalizePoolConfig();
        assertEq(pool.seniorRateBps(), 0, "coupon must be zero for exact accounting");
        assertEq(pool.maxSeniorShareBps(), PROD_MAX_SENIOR_SHARE_BPS, "release share cap");
        assertEq(pool.maxSeniorExposureAssets(), PROD_MAX_SENIOR_EXPOSURE, "release exposure cap");
    }

    function _reconcile() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _recapitalize(uint256 amount) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    /// @dev Quantum-compliant LONG notional (18-dec, multiple of 100e18) whose profit between
    ///      entry 1e8 and exit 0.5e8 is at least `minProfitAssets` (6-dec).
    function _profitNotional(uint256 minProfitAssets) internal pure returns (uint256) {
        uint256 units = (minProfitAssets + 49_999_999) / 50_000_000; // ceil to 50e6-profit units
        return units * 50_000_000 * 2 * 1e12; // each unit: 50e6 profit = 100e18 notional
    }

    /// @dev Healthy funded pool under release parameters; senior unimpaired; unassigned bucket
    ///      loaded via recapitalization while the senior gap is closed (gap-fill cannot fire).
    function _fundPool() internal {
        _installProductionParamsAndZeroCoupon();
        _fundJunior(bob, JUNIOR_FUND);
        _fundSenior(alice, SENIOR_FUND);
        seniorP = pool.seniorPrincipal();
        juniorJ = pool.juniorPrincipal();
        preservedHwm = pool.seniorHighWaterMark();
        assertEq(seniorP, preservedHwm, "senior unimpaired before the wipe");
        assertGt(juniorJ, 0, "junior funded");

        _recapitalize(RECAP_U);
        _reconcile();
        assertEq(pool.unassignedAssets(), RECAP_U, "recap with zero senior gap must land unassigned");
        assertEq(pool.seniorPrincipal(), seniorP, "recap left senior untouched");
    }

    /// @dev Trading loss greater than junior principal (junior-first absorbLoss saturates the
    ///      JUNIOR tranche to exactly 0; senior absorbs the overshoot).
    function _wipeJuniorOnly() internal {
        _refreshMarkForAsyncSettlement();
        _fundTrader(carol, TRADE_MARGIN);
        uint256 size = _profitNotional(juniorJ + SENIOR_SHAVE);
        _open(carol, CfdTypes.Side.LONG, size, TRADE_MARGIN, 1e8);
        _close(carol, CfdTypes.Side.LONG, size, 0.5e8);
        _reconcile();

        assertEq(pool.juniorPrincipal(), 0, "junior tranche wiped to exactly 0 (saturation)");
        assertApproxEqAbs(pool.seniorPrincipal(), seniorP - SENIOR_SHAVE, 50_000e6, "senior shaves the overshoot");
        assertEq(pool.seniorHighWaterMark(), preservedHwm, "HWM pinned");
        assertEq(pool.unassignedAssets(), RECAP_U, "unassigned bucket survives the wipe reconcile");
        assertGt(juniorVault.totalSupply(), 0, "wiped juniors still hold shares (seed floor pins supply)");
    }

    /// @dev Trading loss at least full claimed equity: both tranches saturate to 0, senior HWM
    ///      preserved by absorbLoss.
    function _wipeEverything() internal {
        _refreshMarkForAsyncSettlement();
        _fundTrader(carol, TRADE_MARGIN);
        uint256 size = _profitNotional(seniorP + juniorJ + SENIOR_SHAVE);
        _open(carol, CfdTypes.Side.LONG, size, TRADE_MARGIN, 1e8);
        _close(carol, CfdTypes.Side.LONG, size, 0.5e8);
        _reconcile();

        assertEq(pool.seniorPrincipal(), 0, "senior fully wiped");
        assertEq(pool.juniorPrincipal(), 0, "junior fully wiped");
        assertEq(pool.seniorHighWaterMark(), preservedHwm, "impairment HWM preserved by absorbLoss");
        // The write-down clamp trims the bucket to distributable; what matters for the horns is
        // that a large assignable bucket survives the wipe.
        assertGt(pool.unassignedAssets(), RECAP_U / 2, "unassigned bucket survives the wipe reconcile");
        assertGt(seniorVault.totalSupply(), 0, "wiped seniors still hold shares");
    }

    // =====================================================================
    // A1 — release params: senior assign after full wipe is capacity-dead
    // =====================================================================
    function test_A1_ProdParams_FullWipe_SeniorAssign_CapacityRevert() public {
        _fundPool();
        _wipeEverything();
        uint256 bucketAfterWipe = pool.unassignedAssets();

        vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);
        pool.assignUnassignedAssets(true, alice);

        assertEq(pool.seniorPrincipal(), 0, "blocked assign must not touch senior principal");
        assertEq(pool.seniorHighWaterMark(), preservedHwm, "blocked assign must not touch HWM");
        assertEq(pool.unassignedAssets(), bucketAfterWipe, "unassigned survives the blocked assign");
    }

    // =====================================================================
    // A2 — release params: junior regrow starves unassigned; the gap eats
    //      every new inflow; the HWM is never erased end-to-end.
    // =====================================================================
    function test_A2_ProdParams_JuniorRegrowStarvesUnassigned_HwmNeverErased() public {
        _fundPool();
        _wipeEverything();

        // Only junior-regrowth path: consumes the whole bucket (note: this very call also
        // exercises the B corner mint, since juniorPrincipal == 0 here).
        uint256 bucketAfterWipe = pool.unassignedAssets();
        pool.assignUnassignedAssets(false, bob);
        assertEq(pool.juniorPrincipal(), bucketAfterWipe, "junior assign regrows junior principal");
        assertEq(pool.unassignedAssets(), 0, "junior assign consumes the whole bucket");
        assertEq(pool.seniorHighWaterMark(), preservedHwm, "impairment still preserved");

        vm.expectRevert(IHousePool.HousePool__NoUnassignedAssets.selector);
        pool.assignUnassignedAssets(true, alice);

        // New inflow restores the senior gap first — never re-funds unassigned while open.
        _recapitalize(60_000e6);
        _reconcile();
        assertEq(pool.seniorPrincipal(), 60_000e6, "inflow gap-fills the wiped senior first");
        assertEq(pool.unassignedAssets(), 0, "nothing routes to unassigned while the gap is open");

        // Close the gap, then overflow re-funds unassigned and the runnable senior assign
        // takes the += (preserving) branch.
        _recapitalize(preservedHwm - pool.seniorPrincipal() + 5_000e6);
        _reconcile();
        assertEq(pool.unassignedAssets(), 5_000e6, "overflow beyond the gap lands unassigned");
        assertEq(pool.seniorPrincipal(), preservedHwm, "senior restored to its HWM");

        _refreshMarkForAsyncSettlement();
        pool.assignUnassignedAssets(true, RECEIVER_Y);
        assertEq(pool.seniorPrincipal(), preservedHwm + 5_000e6, "runnable assign adds principal");
        assertEq(
            pool.seniorHighWaterMark(),
            preservedHwm + 5_000e6,
            "runnable assign PRESERVES via += (HousePool.sol:694-696), never hard-sets"
        );
        assertGe(pool.seniorHighWaterMark(), preservedHwm, "HWM never fell below the preserved value");
    }

    // =====================================================================
    // B — the uncovered corner: zero-principal BOOTSTRAP QUOTE on the junior
    //     vault mints amount*(supply+1000)/1 shares; receiver captures the
    //     whole tranche plus ~all future junior revenue.
    // =====================================================================
    function test_B_ZeroPrincipalBootstrapCorner_ReceiverCapturesTrancheAndRevenue() public {
        _fundPool();
        _wipeJuniorOnly();

        uint256 supplyBefore = juniorVault.totalSupply();
        assertGt(supplyBefore, 0, "corner precondition: supply pinned > 0");
        assertEq(pool.juniorPrincipal(), 0, "corner precondition: principal exactly 0");

        // The pathological quote: parity rate would be ~1000 shares/asset; the corner
        // yields supply+1000 shares PER ASSET.
        uint256 quotedShares = juniorVault.quoteBootstrapDeposit(RECAP_U);
        assertEq(
            quotedShares,
            RECAP_U * (supplyBefore + 1000),
            "corner quote = amount*(supply+1000)/1 (TrancheVault.sol:1858, denominator 0+1)"
        );

        uint256 bobShares = juniorVault.balanceOf(bob);
        assertEq(juniorVault.convertToAssets(bobShares), 0, "wiped juniors start at zero value");

        _refreshMarkForAsyncSettlement();
        pool.assignUnassignedAssets(false, RECEIVER_X);

        assertEq(pool.juniorPrincipal(), RECAP_U, "assign credits principal with the bucket");
        assertEq(pool.unassignedAssets(), 0, "bucket consumed");
        uint256 xShares = juniorVault.balanceOf(RECEIVER_X);
        assertEq(xShares, quotedShares, "receiver got the corner-priced mint");
        assertGt(xShares * 1e18 / (xShares + supplyBefore), 0.999e18, "receiver owns >99.9% of tranche");

        // Wiped juniors' residual claim on the 500k bucket is ~0:
        assertLt(
            juniorVault.convertToAssets(bobShares), RECAP_U / 1000, "wiped juniors retain <0.1% of the bucket"
        );

        // Future junior-bound revenue accrues ~entirely to the receiver: first close the small
        // senior shave (covenant: senior gap restores first), then revenue flows to junior
        // principal, shared by the (now receiver-dominated) share count.
        _mintAndAccountPoolExcess((preservedHwm - pool.seniorPrincipal()) + REVENUE);
        _reconcile();
        assertEq(pool.seniorPrincipal(), preservedHwm, "senior gap closed first (covenant intact)");
        assertEq(pool.juniorPrincipal(), RECAP_U + REVENUE, "revenue accrues to junior principal");
        assertLt(
            juniorVault.convertToAssets(bobShares),
            (RECAP_U + REVENUE) / 1000,
            "wiped juniors retain <0.1% of bucket + revenue"
        );

        // Receiver drains ~the whole bucket + revenue through a normal async redeem
        // (epoch funding budget and virtual-share dust take a few percent).
        vm.warp(block.timestamp + 1 hours); // bootstrapMint set lastDepositTime; redeem cooldown
        uint256 xBalanceBefore = settlementToken.balanceOf(RECEIVER_X);
        _redeemAsync(juniorVault, RECEIVER_X, xShares, true);
        uint256 xDrained = settlementToken.balanceOf(RECEIVER_X) - xBalanceBefore;
        assertGt(xDrained, (RECAP_U + REVENUE) * 9 / 10, "receiver drains >=90% of bucket + revenue");

        console2.log("corner mint shares (receiver):", xShares);
        console2.log("pre-existing junior supply:   ", supplyBefore);
        console2.log("receiver drained assets:      ", xDrained);
        console2.log("wiped juniors' final value:   ", juniorVault.convertToAssets(bobShares));
        console2.log("counterfactual revenue route: wiped juniors would own 100% of", RECAP_U + REVENUE);
    }

    // =====================================================================
    // C — negative control: with pricing assets > 0 the same assign keeps
    //     every pre-existing shareholder's pro-rata ownership of the tranche
    //     (the dilution-neutrality that only holds off the corner), proving
    //     the divergence is specific to the zero-principal state.
    // =====================================================================
    function test_C_NegativeControl_NonzeroPrincipalAssignIsRatioNeutral() public {
        _fundPool(); // junior principal = juniorJ > 0, no wipe
        assertEq(pool.unassignedAssets(), RECAP_U, "bucket loaded");

        uint256 bobShares = juniorVault.balanceOf(bob);
        uint256 bobFracBefore = bobShares * 1e18 / juniorVault.accruedTotalSupply();
        uint256 bobClaimBefore = bobFracBefore * pool.juniorPrincipal() / 1e18;

        _refreshMarkForAsyncSettlement();
        pool.assignUnassignedAssets(false, RECEIVER_Y);

        uint256 bobFracAfter = juniorVault.balanceOf(bob) * 1e18 / juniorVault.accruedTotalSupply();
        uint256 bobClaimAfter = bobFracAfter * pool.juniorPrincipal() / 1e18;

        // A fair bootstrap mint leaves every pre-existing shareholder's claim on principal
        // UNCHANGED (the receiver's shares are worth exactly the bucket it brought): the mint
        // is neutral when pricing assets > 0.
        assertApproxEqRel(bobClaimAfter, bobClaimBefore, 1e15);

        // And the receiver's share count is the parity scale (~1000 shares per asset), not the
        // supply-scale corner rate.
        uint256 yShares = juniorVault.balanceOf(RECEIVER_Y);
        assertApproxEqRel(yShares, RECAP_U * 1000, 1e15, "receiver's shares are a parity-rate slice");

        console2.log("C: bob claim before:", bobClaimBefore);
        console2.log("C: bob claim after (neutral expectation scaled):", bobClaimAfter);
        console2.log("C: receiver shares:", yShares);
    }
}
```

## Why this is not covered by the already-fixed wiped-tranche quote issue

The previously reported item "Pending depositor can receive impossible share quotes for a wiped tranche" is a view-level defect on the pending-deposit path: an estimation function returns an impossible number, no mint executes, no owner action is involved, and no value moves. The published disposition on that item is that it was addressed by guarding the zero-principal quote — `estimateDepositShares` now returns zero when projected deposit-pricing assets are zero and shares survive.

That fix establishes the project's own standard: this state is unpriceable and must be refused. The identical condition on the bootstrap path, which *executes* a mint rather than merely estimating one, was left unguarded. Under the competition's own rule that a report on the same root cause is valid where it shows the fix at the commit in scope is incomplete, this qualifies: same root cause, sibling call site, no guard, and real value movement instead of a bad view return.

It is also distinct from the two nearest acknowledged items. "Owner can misroute ownerless pool assets to the junior seed receiver" concerns routing discretion — which address receives fairly priced shares — not the mint mathematics, and nothing there confiscates existing claimants' accrual share. "Mixed recovery inflows can strand LP revenue in ownerless pool assets" is the inverse failure: revenue is stranded outside the tranche and never reaches shareholders at all. Here the value reaches the tranche and the corner mint assigns almost all of it to one party. Negative control C confirms the dilution-neutrality property of the bootstrap mint holds whenever pricing assets are above zero, so no unconditional reading of that property covers this case.

## Recommended fix

The minimal change is to reorder the two statements in `assignUnassignedAssets`: credit the tranche principal at `HousePool.sol:699` *before* taking the quote at `:685`. The amount is already known at the quote site, so nothing else has to move. That makes the division `mulDiv(amount, supply + 1000, amount + 1)`, which mints the receiver roughly the existing supply — a fair rate — and keeps the recovery path executable.

If reordering is undesirable for other reasons, the equivalent is to pass the post-credit pricing assets explicitly:

```solidity
// HousePool.assignUnassignedAssets — junior branch
uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount, juniorPrincipal + amount);
```

As a defence-in-depth guard independent of the call-site fix, mirror the deposit lane's own check inside `TrancheVault.quoteBootstrapDeposit`:

```solidity
// TrancheVault.sol:1392-1395
function quoteBootstrapDeposit(uint256 assets) external view returns (uint256) {
    uint256 pricingAssets = _depositPricingAssets();
    if (pricingAssets == 0 && accruedTotalSupply() != 0) revert TrancheVault__TerminallyWiped();
    return _convertToSharesUsingAssets(assets, pricingAssets);
}
```

Reverting alone is not sufficient on its own, though, because test A2 shows this call is the only Junior-regrowth path out of a wipe: a bare revert would leave the pool with no recovery mechanism at all. The call-site fix (or the explicit post-credit pricing argument) is the one to ship; the vault-level guard should sit behind it so the pathological quote can never be reached from any future call site.

Either way the function's documented contract is restored — "mint matching shares" should mean the receiver's claim equals the assigned amount and nothing more. Control C already demonstrates the mechanism is exactly claim-neutral when priced against non-zero principal, so the fix changes nothing in normal operation.
