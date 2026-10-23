# 58450 sc high missing transmuter balance update after redemption blocks future earmarking and underfunds redemptions

**Submitted on Nov 2nd 2025 at 13:24:20 UTC by @godwinudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58450
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `AlchemistV3` contract fails to update the `lastTransmuterTokenBalance` state variable after redemptions occur in the `redeem()` function. This causes the earmarking mechanism to incorrectly detect phantom cover in subsequent earmarking operations, which prevents legitimate debt from being earmarked for redemption. As a result, transmuter positions become systematically underfunded, meaning users who deposit synthetic tokens into the transmuter expecting to receive yield-bearing collateral will be unable to claim their full entitled amounts because the system failed to reserve sufficient collateral for their redemptions.

## Vulnerability Details

The `AlchemistV3` protocol implements an earmarking system to ensure that when users create transmuter positions, sufficient collateral is gradually reserved from borrowers to fulfill those redemption obligations. This earmarking process happens continuously through the `_earmark()` function, which gets called whenever users interact with their positions through functions like `poke()`, `mint()`, `withdraw()`, and others.

The `_earmark()` function includes logic to detect and account for cover, which represents yield tokens that borrowers have repaid directly to the transmuter. This cover should reduce the amount of new debt that needs to be earmarked because those tokens are already available for redemption. The function tracks the transmuter's token balance using a state variable called `lastTransmuterTokenBalance` to calculate how much new cover has accumulated since the last earmark operation.

Here is the relevant section of the `_earmark()` function that calculates cover:

```solidity
function _earmark() internal {
    if (totalDebt == 0) return;
    if (block.number <= lastEarmarkBlock) return;

    // Calculate yield accumulated in transmuter since last earmark
    uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
    uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance 
        ? transmuterCurrentBalance - lastTransmuterTokenBalance 
        : 0;

    uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

    // Subtract cover from amount to be earmarked
    uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
    amount = amount > coverInDebt ? amount - coverInDebt : 0;

    lastTransmuterTokenBalance = transmuterCurrentBalance; // ← Updated here
    
    // ... rest of earmarking logic
}
```

The issue is that while `_earmark()` correctly updates `lastTransmuterTokenBalance` at the end of its execution, the `redeem()` function does not update this variable after it transfers collateral to the transmuter. Let me show you the relevant portion of the `redeem()` function:

```solidity
function redeem(uint256 amount) external onlyTransmuter {
    _earmark();

    uint256 liveEarmarked = cumulativeEarmarked;
    if (amount > liveEarmarked) amount = liveEarmarked;

    // ... redemption weight calculations and state updates ...
    
    // Transfer collateral to transmuter
    uint256 collRedeemed = convertDebtTokensToYield(amount);
    uint256 feeCollateral = collRedeemed * protocolFee / BPS;
    uint256 totalOut = collRedeemed + feeCollateral;

    TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
    _mytSharesDeposited -= collRedeemed + feeCollateral;

    emit Redemption(redeemedDebtTotal);
    // ← lastTransmuterTokenBalance is NOT updated here
}
```

Notice that `redeem()` transfers `collRedeemed` amount of MYT tokens to the transmuter contract, which increases the transmuter's token balance. However, the function never updates `lastTransmuterTokenBalance` to reflect this transfer. This creates a dangerous inconsistency between the actual transmuter balance and what the system thinks the last recorded balance was.

The consequence of this missing update becomes apparent in the next earmarking operation. When `_earmark()` is called again, it reads the transmuter's current balance and compares it against the stale `lastTransmuterTokenBalance`. Since `redeem()` increased the transmuter's balance without updating the tracking variable, the calculation produces a false positive for cover:

```solidity
// In next _earmark() call after redeem():
uint256 transmuterCurrentBalance = 25e18; // Actual balance (from redeem transfer)
uint256 transmuterDifference = 25e18 - 0; // lastTransmuterTokenBalance was never updated
// Result: 25e18 phantom cover detected
```

This phantom cover then gets subtracted from the amount that should be earmarked, effectively canceling out legitimate earmarking that needs to happen to fund future transmuter redemptions. The tokens sitting in the transmuter from the previous redemption are mistakenly counted as new cover, when in reality they are already committed to fulfill an existing transmuter position claim.

To illustrate the full attack path, consider this sequence of events. First, **2 borrowers** each deposit **100 yield tokens** and mint **50 debt tokens**. A transmuter user then deposits **50 debt tokens** to create a redemption position that will mature over time. After the position is halfway mature, meaning **25 debt tokens** worth of earmarking should have occurred, the first borrower calls `poke()` which triggers earmarking of those **25 debt tokens**. The transmuter then calls `redeem()` to claim the earmarked collateral, which successfully transfers **25 yield tokens** to the transmuter, but crucially fails to update `lastTransmuterTokenBalance`.

Now when time passes and the transmuter position becomes fully mature, meaning another **25 debt tokens** should be earmarked from the second borrower's position, the second borrower calls `poke()`. The `_earmark()` function executes and queries the transmuter graph, which correctly reports that **25 debt tokens** should be earmarked. However, the function also checks the transmuter balance and sees **25 yield tokens** sitting there from the previous redemption. Because `lastTransmuterTokenBalance` was never updated to account for that previous transfer, the system incorrectly identifies these tokens as new cover and subtracts them from the earmarking amount. The result is that **0 debt** gets earmarked instead of the required **25 debt tokens**.

When the transmuter user eventually tries to claim their fully mature position, they will find that only half the expected collateral has been reserved for them, because the second half of the earmarking never occurred due to the phantom cover detection bug.

## Impact Details

Users who deposit synthetic tokens into the transmuter expecting to receive their equivalent value in yield-bearing collateral will receive less than their entitled amount because the earmarking mechanism fails to reserve sufficient collateral from borrowers.

## Proof of Concept

## Proof of Concept

Add this to AlchemistV3.t.sol, and run

```solidity
 function testIncorrectLastTransmuterTokenBalanceTracking() external {
    uint256 depositAmt = 100e18;
    uint256 debtAmt = 50e18;
    
    // Two positions
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmt);
    alchemist.deposit(depositAmt, address(0xbeef), 0);
    uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId1, debtAmt, address(0xbeef));
    vm.stopPrank();
    
    vm.startPrank(externalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmt);
    alchemist.deposit(depositAmt, externalUser, 0);
    uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
    alchemist.mint(tokenId2, debtAmt, externalUser);
    vm.stopPrank();
    
    // Transmuter + earmark
    vm.startPrank(address(0xdad));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debtAmt);
    transmuterLogic.createRedemption(debtAmt);
    vm.stopPrank();
    
    vm.roll(block.number + 2_628_000); // Half maturation
    vm.prank(address(0xbeef));
    alchemist.poke(tokenId1);
    
    // Redeem - consumes 25e18 earmarked
    vm.prank(address(transmuterLogic));
    alchemist.redeem(debtAmt);
    
    // Roll to mature the second half
    vm.roll(block.number + 2_628_000);
    
    uint256 cumulativeEarmarkedBefore = alchemist.cumulativeEarmarked();
    
    // This should earmark another 25e18
    vm.prank(externalUser);
    alchemist.poke(tokenId2);
    
    uint256 cumulativeEarmarkedAfter = alchemist.cumulativeEarmarked();
    
    // The 25e18 phantom cover consumed the earmarkable amount
    // Expected: cumulativeEarmarkedAfter = cumulativeEarmarkedBefore + 25e18
    // Actual: cumulativeEarmarkedAfter = cumulativeEarmarkedBefore (no change!)
    assertEq(cumulativeEarmarkedAfter, cumulativeEarmarkedBefore + 25e18); // This will fail
}
```
