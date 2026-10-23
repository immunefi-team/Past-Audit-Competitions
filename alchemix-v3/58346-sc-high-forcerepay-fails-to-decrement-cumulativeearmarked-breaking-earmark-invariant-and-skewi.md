# 58346 sc high forcerepay fails to decrement cumulativeearmarked breaking earmark invariant and skewing redemptions

**Submitted on Nov 1st 2025 at 12:37:12 UTC by @mzfr for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58346
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `_forceRepay()` function in `AlchemistV3.sol` correctly reduces an individual account's `earmarked` debt but fails to decrement the global `cumulativeEarmarked` counter, breaking the core accounting invariant that `cumulativeEarmarked == Σ(account.earmarked)` across all positions. This causes persistent accounting drift that accumulates with every liquidation, skewing the redemption earmarking distribution mechanism. When exploited through repeated liquidations, this bug causes the protocol to incorrectly calculate the proportion of unearmarked debt, leading to unfair distribution of redemption proceeds among borrowers and degraded protocol functionality.

## Vulnerability Details

The AlchemistV3 protocol maintains a critical accounting invariant:

```
cumulativeEarmarked (global) == Σ(account.earmarked) for all accounts
```

This invariant feeds directly into the `_earmark()` logic. In the actual code (`src/AlchemistV3.sol`, around the earmark body(<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_latest/src/AlchemistV3.sol#L1098-L1128>)), the protocol computes the live unearmarked debt and the fraction to earmark, then updates survival and weights:

```solidity
// Yield accumulated since last earmark, minus cover observed at the transmuter
uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance
    ? transmuterCurrentBalance - lastTransmuterTokenBalance
    : 0;

uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);
uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
amount = amount > coverInDebt ? amount - coverInDebt : 0;

lastTransmuterTokenBalance = transmuterCurrentBalance;

uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
if (amount > liveUnearmarked) amount = liveUnearmarked;

if (amount > 0 && liveUnearmarked != 0) {
    uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
    if (previousSurvival == 0) previousSurvival = ONE_Q128;

    // Fraction of unearmarked being earmarked now (UQ128.128)
    uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

    _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
    _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

    cumulativeEarmarked += amount;
}
```

The `repay()` function correctly maintains this invariant by updating both counters [`src/AlchemistV3.sol:521-526`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_latest/src/AlchemistV3.sol#L521-L526):

```solidity
uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
account.earmarked -= earmarkToRemove;  // Update individual account

uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
cumulativeEarmarked -= earmarkPaidGlobal;  // Update global counter
```

However, `_forceRepay()` only updates the individual account ([`src/AlchemistV3.sol:760-767`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_latest/src/AlchemistV3.sol#L760-L767)):

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    Account storage account = _accounts[accountId];
    uint256 startingDebt = account.debt;
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;  // ✓ Updates individual account

    // MISSING: No reduction of cumulativeEarmarked
    // Should be:
    // uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
    // cumulativeEarmarked -= earmarkPaidGlobal;

    // ... rest of function
}
```

When `_forceRepay()` is Called

`_forceRepay()` is invoked from `_liquidate()` around line <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_latest/src/AlchemistV3.sol#L821> before deciding whether to proceed with a full liquidation. If the account has earmarked debt, the function repays from collateral first, then reassesses the health ratio and may call `_doLiquidation()`:

```solidity
// Inside _liquidate(accountId)
if (account.earmarked > 0) {
    // BUG: _forceRepay reduces account.earmarked but not cumulativeEarmarked
    repaidAmountInYield = _forceRepay(accountId, account.earmarked);
}
// If healthy after forced repay, only a repayment fee is handled; otherwise do liquidation
collateralInUnderlying = totalValue(accountId);
collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;
if (collateralizationRatio <= collateralizationLowerBound) {
    return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
}
```

## Proof of Concept

## Proof of Concept

```sol
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {Test} from "../../lib/forge-std/src/Test.sol";
import {AlchemistV3Test} from "./AlchemistV3.t.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";

/**
 * @title PoC: _forceRepay doesn't reduce cumulativeEarmarked
 * @notice This test extends AlchemistV3Test to demonstrate that _forceRepay() reduces
 *         account.earmarked but NOT cumulativeEarmarked, breaking the accounting invariant.
 */
contract PoC_ForceRepayCumulativeEarmarkedBug_Simple is AlchemistV3Test {

    function test_ForceRepayCumulativeEarmarkedBug() public {

        // Setup: Create two positions with debt using existing test infrastructure
        uint256 depositAmt = 2000e18;

        // User1: Deposit and mint near maximum debt to be close to liquidation
        uint256 shares1 = _magicDepositToVault(address(vault), externalUser, depositAmt);
        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), shares1);
        alchemist.deposit(shares1, externalUser, 0);
        uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
        // Mint close to max to be near liquidation threshold
        uint256 maxDebt1 = alchemist.getMaxBorrowable(tokenId1);
        uint256 mintAmount1 = (maxDebt1 * 99) / 100; // 99% of max to be very close to threshold
        alchemist.mint(tokenId1, mintAmount1, externalUser);
        vm.stopPrank();

        // User2: Deposit and mint debt
        uint256 shares2 = _magicDepositToVault(address(vault), anotherExternalUser, depositAmt);
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), shares2);
        alchemist.deposit(shares2, anotherExternalUser, 0);
        uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
        uint256 maxDebt2 = alchemist.getMaxBorrowable(tokenId2);
        uint256 mintAmount2 = (maxDebt2 * 70) / 100; // 70% of max
        alchemist.mint(tokenId2, mintAmount2, anotherExternalUser);
        vm.stopPrank();

        // Sanity: ensure some debt exists
        assertGt(alchemist.totalDebt(), 0);

        // Set short transmutation time BEFORE creating redemption
        vm.prank(alOwner);
        transmuterLogic.setTransmutationTime(100);
        vm.stopPrank();

        // Create redemption to earmark debt
        uint256 redemptionAmount = mintAmount1; // Redeem amount equal to user1's debt
        vm.startPrank(externalUser);
        alToken.approve(address(transmuterLogic), redemptionAmount);
        transmuterLogic.createRedemption(redemptionAmount);
        vm.stopPrank();

        // Process earmarking - advance enough blocks for full maturation
        vm.roll(block.number + 100);
        alchemist.poke(tokenId1);

        uint256 cumulativeEarmarkedBefore = alchemist.cumulativeEarmarked();
        (,, uint256 earmarked1) = alchemist.getCDP(tokenId1);
        (,, uint256 earmarked2) = alchemist.getCDP(tokenId2);

        // Verify invariant holds BEFORE liquidation
        uint256 sumOfEarmarksBefore = earmarked1 + earmarked2;
        assertApproxEqAbs(cumulativeEarmarkedBefore, sumOfEarmarksBefore, 1, "Invariant should hold before liquidation");
        assertGt(cumulativeEarmarkedBefore, 0);

        // Option A: Ensure liquidation path unconditionally by tightening bounds via admin
        vm.startPrank(alOwner);
        // Raise both minimumCollateralization and the liquidation lower bound above current ratio
        // so the account is considered undercollateralized for the purpose of exercising _forceRepay
        alchemist.setMinimumCollateralization(2e18);           // 2.0x
        alchemist.setCollateralizationLowerBound(2e18);        // 2.0x
        vm.stopPrank();

        // Optionally also crash price (not required with tightened bounds)
        uint256 currentUnderlying = IERC20(mockVaultCollateral).balanceOf(mockStrategyYieldToken);
        MockYieldToken(mockStrategyYieldToken).siphon((currentUnderlying * 7) / 10); // ~70% crash

        // Liquidate user1
        vm.prank(yetAnotherExternalUser);
        alchemist.liquidate(tokenId1);

        // Check state after liquidation
        uint256 cumulativeEarmarkedAfter = alchemist.cumulativeEarmarked();
        (,, uint256 earmarked1After) = alchemist.getCDP(tokenId1);
        (,, uint256 earmarked2After) = alchemist.getCDP(tokenId2);

        uint256 sumOfEarmarksAfter = earmarked1After + earmarked2After;

        // The bug: cumulativeEarmarked is higher than sum of individual earmarks
        assertGt(cumulativeEarmarkedAfter, sumOfEarmarksAfter, "BUG: cumulativeEarmarked not reduced by _forceRepay");
    }
}

```

* I placed my POC file here(<https://github.com/alchemix-finance/v3-poc/tree/immunefi\\_latest/src/test>) and then just ran `forge test --match-path src/test/PoC_ForceRepayCumulativeEarmarkedBug.t.sol -vv`
