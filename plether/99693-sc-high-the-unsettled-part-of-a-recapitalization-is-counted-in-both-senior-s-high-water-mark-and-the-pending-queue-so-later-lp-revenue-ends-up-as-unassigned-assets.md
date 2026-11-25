# \#99693 [SC-High] The unsettled part of a recapitalization is counted in both Senior's high-water mark and the pending queue, so later LP revenue ends up as unassigned assets

**Submitted on Nov 14th 2026 at 20:25:55 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99693
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description


# Summary

When the owner recapitalizes a wiped pool and only part of the amount can be settled, the part that could not be settled is recorded in two places. HousePoolPendingPreviewLib sets Senior's high-water mark to the full recapitalization, so the unsettled part becomes a Senior shortfall that ordinary revenue has to fill. HousePool also keeps the same unsettled part in pendingRecapitalizationAssets.

On the next reconcile the ordinary waterfall runs first, so revenue the pool has earned in the meantime restores Senior to the mark. The queued remainder is applied afterwards, finds no shortfall left, and is added to unassignedAssets. The pool's revenue has paid for the same shortfall twice, and the second half now belongs to no LP.

Continuing the repo's own test, if the pool earns 15,000 before the second reconcile, all 15,000 ends up in unassignedAssets. Junior receives none of it.

# Description

A recapitalization is always queued in pendingRecapitalizationAssets (HousePool.sol:644-645). At reconcile, only as much as the pool's distributable equity allows is settled, and the rest stays queued (HousePool.sol:1787-1792).

When both tranche principals are zero, the settled part goes through the start-from-zero branch of applyClaimantRecapitalizationIntent. That branch credits Senior with the settled amount, but sets the high-water mark to max(recapitalizationTargetAssets, amount), where the target is the whole queued recapitalization (HousePoolPendingPreviewLib.sol:117-122). So after a partial settlement Senior's mark already includes the part that was not settled. From the waterfall's point of view Senior is now impaired by exactly that amount, and distributeRevenue will fill it from ordinary revenue before Junior gets anything (HousePoolWaterfallAccountingLib.sol:140-162).

The same unsettled amount is still sitting in the queue. On the next reconcile, planReconcile first sets the settleable queue aside, then runs the ordinary waterfall on what is left (HousePoolReconcilePlanLib.sol:68-71 and :84-99). Only after that does HousePool apply the queue (HousePool.sol:1598, :1759-1776). If the pool has earned revenue since the first reconcile, the waterfall has already used it to bring Senior up to the mark. The queued recapitalization then takes the other branch, sees no gap between Senior and its mark, and falls through to unassignedAssets (HousePoolPendingPreviewLib.sol:123-136).

The result is that one shortfall is charged against later equity twice: once through the mark, and once through the queue.

The repo's own test shows the first half of this. In test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater, LP principal is wiped, the owner recapitalizes 40,000, 15,000 of revenue was queued during the wipe, and traders are owed 30,000. The first reconcile leaves Senior at 25,000 with a mark of 40,000, and 15,000 of recapitalization plus 15,000 of revenue still queued. The test then clears the trader claims and reconciles again, and expects Senior 40,000, Junior 15,000 and nothing unassigned. That is the intended outcome: the queued recapitalization restores Senior and the queued revenue goes to Junior.

Now add one thing to that test. Before the second reconcile, the pool earns 15,000 of ordinary revenue. Because principal is no longer zero, this revenue is not queued. At the second reconcile the 30,000 in the queue is set aside, which leaves 40,000 for the waterfall. Senior holds 25,000, so the waterfall sees 15,000 of surplus and uses all of it to restore Senior to its 40,000 mark. Junior gets nothing from it. Then the queue is applied. The 15,000 of queued recapitalization finds Senior already at its mark and goes to unassignedAssets. The 15,000 of queued revenue goes to Junior. The pool ends with Senior 40,000, Junior 15,000 and 15,000 unassigned. Compared with the test's own endpoint, the extra 15,000 the pool earned went entirely to unassigned assets.

The same thing happens with a recapitalization on its own. The owner puts 40,000 into a pool that owes traders 30,000. The first reconcile leaves Senior at 10,000 with a mark of 40,000, and 30,000 still queued. The pool then earns 70,000. At the next reconcile, 30,000 of that revenue restores Senior to the mark, another 30,000 backs the queue and becomes unassigned, and Junior is left with 10,000. If the shortfall were counted once, Junior would have 40,000. In general Junior receives nothing until later revenue exceeds twice the unsettled amount.

What triggers this is the timing of the next stored reconcile. If one is stored while equity is unchanged, the queued remainder is settled against the same equity again, which is a different failure. This report is about the case where equity has grown past the recapitalization amount by the time the next reconcile is stored. It is also not the mark being overwritten lower on a first recapitalization; here the mark is set too high for what Senior was actually credited.

Once the value is in unassignedAssets, no LP can reach it. It is reserved out of LP free cash (HousePool.sol:1727-1735), and any nonzero amount blocks deposit requests and activations for both tranches (HousePoolTrancheGateLib.sol:28-34). The only function that moves it is assignUnassignedAssets, which is owner-only and mints new shares for the whole amount to a single receiver (HousePool.sol:660-704). The contract has no path that returns it to the Junior holders who earned it.

A fix would be to record the shortfall in one place only. Either the mark should reflect what was actually credited and the queue alone should carry the remainder, or the queued recapitalization should be applied before the ordinary waterfall so that it fills the shortfall it was reserved for and later revenue flows on to Junior.

# Impact

Permanent freezing of funds.

Revenue the pool earns after a partly settled recapitalization is taken out of LP ownership. In the example built on the repo's test, the pool earns 15,000 and Junior receives none of it; the full 15,000 is recorded as unassigned assets. In the recapitalization-only example, Junior receives 10,000 out of 70,000 earned where it should receive 40,000, and 30,000 is left unassigned.

Junior LPs cannot claim that value through any redemption path. It stays in the pool, excluded from both tranches, until the owner assigns it, and the owner can only assign it by minting new shares to one receiver of its choosing. Existing Junior holders are not made whole by that.

While the unassigned balance is nonzero, deposit requests and deposit activations are refused for both tranches, so the pool cannot take in new LP capital.

Senior is not harmed, since it reaches its mark either way, and traders are not affected. The loss is Junior's revenue, in an amount equal to the unsettled part of the recapitalization.

## Proof of Concept

## Proof of Concept



Run:

```bash
forge test --match-contract PoC_RecapShortfallCountedTwice -vv
```

## What the tests show

| Test | Result | Claim it demonstrates |
| --- | --- | --- |
| `test_A_RepoTestPlus15kEarnedBeforeSecondReconcile_15kBecomesUnassigned` | Senior 40,000 / Junior 15,000 / **unassigned 15,000** | Continuing the repo test, 15,000 earned before the second reconcile goes entirely to `unassignedAssets`. Junior receives none of it. |
| `test_A_Control_15kEarnedAfterResidualSettles_GoesToJunior` | Senior 40,000 / Junior **30,000** / unassigned 0 | Same cash, same owners. The only difference is that the 15,000 is earned after the residual settled. |
| `test_B_RecapOnly_Then70kEarned_30kBecomesUnassigned_JuniorGets10kInsteadOf40k` | Senior 40,000 / Junior **10,000** / **unassigned 30,000** | 40,000 recapitalization into a pool owing 30,000, then 70,000 earned. 30,000 of revenue restores Senior to the mark, another 30,000 backs the queue and becomes unassigned. |
| `test_B_Control_RecapBackedAtFirstReconcile_JuniorGets40k` | Senior 40,000 / Junior **40,000** / unassigned 0 | Same recap, claims, revenue and final equity (80,000); the recap was backed when first reconciled, so the shortfall was never recorded twice. |
| `testFuzz_B_JuniorGetsNothingUntilRevenueExceedsTwiceTheShortfall` (256 runs) | `unassigned == min(R − U, U)`, `Junior == max(R − 2U, 0)` | Junior receives nothing until later revenue exceeds twice the unsettled amount. |
| `test_C_UnassignedIsUnreachable_BlocksDeposits_OnlyOwnerMovesItToOneReceiver` | free cash 55,000 of 70,000; both deposit gates closed; persists across reconciles; non-owner reverts; owner assignment mints ~15,000 of value to a fresh receiver while the existing Junior LP stays at exactly 15,000 | The value is reserved out of LP free cash, blocks deposits for both tranches, and the only path that moves it cannot make the Junior holders who earned it whole. |

## Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAccountingParityBase} from "./HousePoolSnapshotParity.t.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";

/// @title PoC: the unsettled part of a recapitalization is counted in both Senior's high-water mark and the pending
///        queue, so later LP revenue ends up as unassigned assets
///
/// @dev Root cause
///   - Partial settlement goes through the start-from-zero branch of `applyClaimantRecapitalizationIntent`, which credits
///     Senior with the settled amount but sets the mark to max(fullTarget, amount) (HousePoolPendingPreviewLib.sol:117-122).
///   - The unsettled part also stays in `pendingRecapitalizationAssets` (HousePool.sol:1787-1792).
///   - On the next reconcile `planReconcile` reserves the settleable queue, then runs the ordinary waterfall on the rest
///     (HousePoolReconcilePlanLib.sol:68-71, :84-99). Revenue earned since restores Senior to the mark.
///   - The queue is applied afterwards (HousePool.sol:1759-1776); the recap remainder finds no gap and falls through to
///     `unassignedAssets` (HousePoolPendingPreviewLib.sol:123-136).
///
/// @dev Tests
///   A. Repo test `test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater` + 15k earned before the 2nd reconcile
///      -> Senior 40k, Junior 15k, unassigned 15k   (repo endpoint is 40k / 15k / 0; the extra 15k went to nobody)
///   A'. Control: same 15k earned AFTER the residual settles -> Junior 30k, unassigned 0
///   B. Recap-only: 40k recap into a pool owing 30k, then 70k earned -> Senior 40k, Junior 10k, unassigned 30k
///   B'. Control: same flows with the recap backed when first reconciled -> Junior 40k, unassigned 0
///   Fuzz: Junior receives nothing until later revenue exceeds twice the unsettled amount; unassigned = min(R-U, U)
///   C. The unassigned value is unreachable by LPs, blocks deposits for both tranches, and only the owner can move it,
///      by minting shares to a single receiver - which does not make the Junior holders who earned it whole.
contract PoC_RecapShortfallCountedTwice is HousePoolAccountingParityBase {

    using stdStorage for StdStorage;

    function _setClaims(uint256 amount) internal {
        stdstore.target(address(engine)).sig("totalTraderClaimBalanceAssets()").checked_write(amount);
    }

    function _reconcile() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _wipePrincipal() internal {
        settlementToken.burn(address(pool), pool.totalAssets());
        _reconcile();
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(seniorVault.totalSupply(), 0, "senior ownership survives the wipe");
        assertGt(juniorVault.totalSupply(), 0, "junior ownership survives the wipe");
    }

    function _dump(string memory label) internal {
        emit log_string(label);
        emit log_named_uint("  seniorPrincipal     ", pool.seniorPrincipal());
        emit log_named_uint("  juniorPrincipal     ", pool.juniorPrincipal());
        emit log_named_uint("  seniorHighWaterMark ", pool.seniorHighWaterMark());
        emit log_named_uint("  unassignedAssets    ", pool.unassignedAssets());
        emit log_named_uint("  pendingRecap        ", pool.pendingRecapitalizationAssets());
        emit log_named_uint("  pendingRevenue      ", pool.pendingTradingRevenueAssets());
        emit log_named_uint("  pool token balance  ", settlementToken.balanceOf(address(pool)));
        emit log_named_uint("  traderClaims        ", engine.totalTraderClaimBalanceAssets());
    }

    // =====================================================================================================
    // A. Continuing the repo's own test
    // =====================================================================================================

    /// @dev Exactly the setup and first reconcile of test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater.
    function _repoTestFirstPass() internal {
        _wipePrincipal();
        _recordInflow(IHousePool.ClaimantInflowKind.Recapitalization, 40_000e6);
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 15_000e6);
        _setClaims(30_000e6);
        _reconcile();
        _dump("=== A: after reconcile #1 (as asserted by the repo test) ===");
        assertEq(pool.seniorPrincipal(), 25_000e6);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), 40_000e6, "mark set to the FULL 40k while only 25k was credited");
        assertEq(pool.pendingRecapitalizationAssets(), 15_000e6, "the same 15k shortfall is also still queued");
        assertEq(pool.pendingTradingRevenueAssets(), 15_000e6);
        assertEq(pool.unassignedAssets(), 0);
    }

    function test_A_RepoTestPlus15kEarnedBeforeSecondReconcile_15kBecomesUnassigned() public {
        _repoTestFirstPass();

        // Same as the repo test: trader claims clear ...
        _setClaims(0);
        // ... plus one thing: the pool earns 15k of ordinary revenue before the second reconcile.
        // Principal is nonzero now, so this is NOT queued (HousePool.sol:646-648).
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 15_000e6);
        assertEq(pool.pendingTradingRevenueAssets(), 15_000e6, "new revenue was not queued");
        _dump("=== A: before reconcile #2 (pool 70k, owes 0) ===");

        _reconcile();
        _dump("=== A: after reconcile #2 ===");

        assertEq(pool.seniorPrincipal(), 40_000e6, "waterfall restored Senior to the mark using the new revenue");
        assertEq(pool.juniorPrincipal(), 15_000e6, "Junior got only the queued 15k revenue");
        assertEq(pool.unassignedAssets(), 15_000e6, "queued recap remainder found no gap -> ownerless");
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.pendingTradingRevenueAssets(), 0);
        // Repo endpoint is 40k / 15k / 0 on 55k of equity. With 70k of equity the extra 15k belongs to nobody.
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal() + pool.unassignedAssets(), 70_000e6, "conservation");
    }

    /// @dev Control: identical cash and owners; the 15k is earned AFTER the residual has settled.
    function test_A_Control_15kEarnedAfterResidualSettles_GoesToJunior() public {
        _repoTestFirstPass();
        _setClaims(0);
        _reconcile(); // the repo test's own second reconcile
        assertEq(pool.seniorPrincipal(), 40_000e6);
        assertEq(pool.juniorPrincipal(), 15_000e6);
        assertEq(pool.unassignedAssets(), 0);

        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 15_000e6);
        _reconcile();
        _dump("=== A control: residual settled first, then 15k earned ===");
        assertEq(pool.seniorPrincipal(), 40_000e6);
        assertEq(pool.juniorPrincipal(), 30_000e6, "control: the 15k goes to Junior");
        assertEq(pool.unassignedAssets(), 0, "control: nothing is ownerless");
    }

    // =====================================================================================================
    // B. Recapitalization on its own
    // =====================================================================================================

    uint256 constant RECAP = 40_000e6;
    uint256 constant CLAIMS = 30_000e6;
    uint256 constant UNSETTLED = 30_000e6; // RECAP - (RECAP - CLAIMS) = 30k of the recap is not backed at first

    function _recapOnlyFirstPass() internal {
        _wipePrincipal();
        _recordInflow(IHousePool.ClaimantInflowKind.Recapitalization, RECAP);
        _setClaims(CLAIMS);
        _reconcile();
        _dump("=== B: after reconcile #1 ===");
        assertEq(pool.seniorPrincipal(), 10_000e6, "only the backed 10k credited");
        assertEq(pool.seniorHighWaterMark(), RECAP, "mark already 40k");
        assertEq(pool.pendingRecapitalizationAssets(), UNSETTLED, "30k still queued");
        assertEq(pool.unassignedAssets(), 0);
    }

    function test_B_RecapOnly_Then70kEarned_30kBecomesUnassigned_JuniorGets10kInsteadOf40k() public {
        _recapOnlyFirstPass();

        // Trader claims stay at 30k. The pool earns 70k. Equity: 40k + 70k - 30k = 80k.
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 70_000e6);
        _reconcile();
        _dump("=== B: after reconcile #2 ===");

        assertEq(pool.seniorPrincipal(), RECAP, "30k of revenue restored Senior to the mark");
        assertEq(pool.unassignedAssets(), UNSETTLED, "another 30k backed the queue and became ownerless");
        assertEq(pool.juniorPrincipal(), 10_000e6, "Junior is left with 10k of the 70k earned");
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal() + pool.unassignedAssets(), 80_000e6, "conservation");
        // Counted once, Junior would hold equity - Senior = 80k - 40k = 40k.
    }

    /// @dev Control: same recap, same claims, same revenue, same final equity (80k). The recap was backed when first
    ///      reconciled, so the shortfall never existed in two places.
    function test_B_Control_RecapBackedAtFirstReconcile_JuniorGets40k() public {
        _wipePrincipal();
        _recordInflow(IHousePool.ClaimantInflowKind.Recapitalization, RECAP);
        _setClaims(0);
        _reconcile();
        assertEq(pool.seniorPrincipal(), RECAP);
        assertEq(pool.seniorHighWaterMark(), RECAP);
        assertEq(pool.pendingRecapitalizationAssets(), 0);

        _setClaims(CLAIMS); // traders are now owed 30k ...
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 70_000e6); // ... and the pool earns 70k
        _reconcile();
        _dump("=== B control ===");
        assertEq(pool.seniorPrincipal(), RECAP);
        assertEq(pool.juniorPrincipal(), 40_000e6, "control: Junior receives the full 40k");
        assertEq(pool.unassignedAssets(), 0, "control: nothing is ownerless");
    }

    /// @dev "Junior receives nothing until later revenue exceeds twice the unsettled amount."
    ///      Revenue R >= UNSETTLED keeps equity >= the recapitalization, which is the regime this report is about.
    function testFuzz_B_JuniorGetsNothingUntilRevenueExceedsTwiceTheShortfall(uint256 revenue) public {
        revenue = bound(revenue, UNSETTLED, 300_000e6);
        _recapOnlyFirstPass();

        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, revenue);
        _reconcile();

        uint256 availableBeyondMark = revenue - UNSETTLED; // equity - 40k
        uint256 expectedUnassigned = availableBeyondMark < UNSETTLED ? availableBeyondMark : UNSETTLED;
        uint256 expectedJunior = availableBeyondMark - expectedUnassigned; // = max(R - 2U, 0)

        assertEq(pool.seniorPrincipal(), RECAP, "Senior always ends at its mark");
        assertEq(pool.unassignedAssets(), expectedUnassigned, "ownerless = min(R - U, U)");
        assertEq(pool.juniorPrincipal(), expectedJunior, "Junior = max(R - 2U, 0)");
        if (revenue <= 2 * UNSETTLED) {
            assertEq(pool.juniorPrincipal(), 0, "nothing for Junior until R > 2U");
        }
        assertEq(pool.pendingRecapitalizationAssets(), 0);
    }

    // =====================================================================================================
    // C. The value is unreachable, blocks deposits, and only the owner can move it (to one receiver)
    // =====================================================================================================

    function test_C_UnassignedIsUnreachable_BlocksDeposits_OnlyOwnerMovesItToOneReceiver() public {
        _repoTestFirstPass();
        _setClaims(0);
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 15_000e6);
        _reconcile();
        assertEq(pool.unassignedAssets(), 15_000e6);
        assertEq(pool.juniorPrincipal(), 15_000e6);

        // Reserved out of LP free cash (HousePool.sol:1727-1735): 70k in the pool, no liabilities, 55k free.
        assertEq(settlementToken.balanceOf(address(pool)), 70_000e6);
        assertEq(pool.getFreeAssets(), 55_000e6, "the 15k is carved out of LP free cash");
        assertLe(pool.getMaxJuniorWithdraw(), 15_000e6, "Junior can reach at most its 15k, not the 15k it earned");
        assertGt(pool.getMaxJuniorWithdraw(), 14_900e6);

        // Blocks deposit requests and activation for BOTH tranches (HousePoolTrancheGateLib.sol:28-34).
        assertFalse(pool.canAcceptTrancheDeposits(true), "senior deposits refused");
        assertFalse(pool.canAcceptTrancheDeposits(false), "junior deposits refused");
        assertFalse(pool.canSettleDepositEntries(), "queued deposits cannot activate");

        // Does not self-heal.
        _reconcile();
        _reconcile();
        assertEq(pool.unassignedAssets(), 15_000e6, "stays ownerless on later reconciles");

        // Only the owner can move it (HousePool.sol:660-704).
        vm.prank(address(0xBAD));
        vm.expectRevert();
        pool.assignUnassignedAssets(false, address(0xBAD));

        // Even "returning" it to Junior mints the whole amount as NEW shares to a single receiver of the owner's
        // choosing, so the Junior holders who actually earned the 15k are not made whole.
        address receiver = address(0xCAFE);
        address existingJuniorLp = address(this); // the fixture's junior seed holder
        uint256 existingShares = juniorVault.balanceOf(existingJuniorLp);
        uint256 supplyBefore = juniorVault.totalSupply();
        uint256 existingValueBefore = 15_000e6 * existingShares / supplyBefore;

        pool.assignUnassignedAssets(false, receiver);
        _dump("=== C: after owner assigns the 15k to Junior via a fresh receiver ===");
        assertEq(pool.unassignedAssets(), 0);
        assertEq(pool.juniorPrincipal(), 30_000e6);

        uint256 receiverShares = juniorVault.balanceOf(receiver);
        uint256 supplyAfter = juniorVault.totalSupply();
        assertGt(receiverShares, 0, "new shares minted to the owner's chosen receiver");
        uint256 existingValueAfter = 30_000e6 * existingShares / supplyAfter;
        uint256 receiverValue = 30_000e6 * receiverShares / supplyAfter;
        emit log_named_uint("existing Junior LP value before owner action", existingValueBefore);
        emit log_named_uint("existing Junior LP value after  owner action", existingValueAfter);
        emit log_named_uint("receiver value (never deposited)            ", receiverValue);
        assertApproxEqAbs(existingValueAfter, existingValueBefore, 2, "existing Junior LPs gain nothing");
        assertApproxEqAbs(receiverValue, 15_000e6, 2, "the receiver captures the 15k Junior earned");
    }

}
```

## Output

```text
Ran 6 tests for packages/perps/test/perps/differential/PoC_RecapShortfallCountedTwice.t.sol:PoC_RecapShortfallCountedTwice
[PASS] testFuzz_B_JuniorGetsNothingUntilRevenueExceedsTwiceTheShortfall(uint256) (runs: 256)
[PASS] test_A_Control_15kEarnedAfterResidualSettles_GoesToJunior()
[PASS] test_A_RepoTestPlus15kEarnedBeforeSecondReconcile_15kBecomesUnassigned()
[PASS] test_B_Control_RecapBackedAtFirstReconcile_JuniorGets40k()
[PASS] test_B_RecapOnly_Then70kEarned_30kBecomesUnassigned_JuniorGets10kInsteadOf40k()
[PASS] test_C_UnassignedIsUnreachable_BlocksDeposits_OnlyOwnerMovesItToOneReceiver()
Suite result: ok. 6 passed; 0 failed; 0 skipped
```

Key logs from `test_A_RepoTestPlus15kEarnedBeforeSecondReconcile_15kBecomesUnassigned`:

```text
=== A: after reconcile #1 (as asserted by the repo test) ===
  seniorPrincipal     : 25000000000
  juniorPrincipal     : 0
  seniorHighWaterMark : 40000000000
  unassignedAssets    : 0
  pendingRecap        : 15000000000
  pendingRevenue      : 15000000000
=== A: after reconcile #2 ===
  seniorPrincipal     : 40000000000
  juniorPrincipal     : 15000000000
  seniorHighWaterMark : 40000000000
  unassignedAssets    : 15000000000
  pendingRecap        : 0
  pendingRevenue      : 0
```

Key logs from `test_B_RecapOnly_Then70kEarned_30kBecomesUnassigned_JuniorGets10kInsteadOf40k`:

```text
=== B: after reconcile #1 ===
  seniorPrincipal     : 10000000000
  seniorHighWaterMark : 40000000000
  pendingRecap        : 30000000000
=== B: after reconcile #2 ===
  seniorPrincipal     : 40000000000
  juniorPrincipal     : 10000000000
  unassignedAssets    : 30000000000
```

Key logs from `test_C_UnassignedIsUnreachable_BlocksDeposits_OnlyOwnerMovesItToOneReceiver`:

```text
existing Junior LP value before owner action: 15000000000
existing Junior LP value after  owner action: 15000000000
receiver value (never deposited)            : 14999999999
```
