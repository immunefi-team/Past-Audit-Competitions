# 56923 sc high missing cumulativeearmarked update in forcerepay causes incorrect debt accounting in alchemistv3

**Submitted on Oct 21st 2025 at 19:49:31 UTC by @godwinudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56923
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `_forceRepay` function in the `AlchemistV3.sol` contract fails to update the global `cumulativeEarmarked` variable when repaying earmarked debt during liquidations, causing an inflated value that disrupts debt accounting. This leads to incorrect `liveUnearmarked` calculations, skewing debt earmarking and collateral allocation. In normal operations with high `totalDebt`, the issue persists without triggering the clamping mechanism in `_subDebt`.

## Vulnerability Details

The protocol tracks debt in two ways: `totalDebt` (total outstanding debt across all positions) and `cumulativeEarmarked` (total debt earmarked for redemption). Each position has a local `account.earmarked` value, representing the portion of its debt earmarked for repayment. The `liveUnearmarked` value (`totalDebt - cumulativeEarmarked`) is used to calculate how much new debt can be earmarked for redemption in the `_earmark` function.

The `_forceRepay` function is designed to repay a position’s debt, prioritizing earmarked debt, using the position’s collateral.

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    if (amount == 0) {
        return 0;
    }
    _checkForValidAccountId(accountId);
    Account storage account = _accounts[accountId];

    _earmark();
    _sync(accountId);

    uint256 debt;
    _checkState((debt = account.debt) > 0);

    uint256 credit = amount > debt ? debt : amount;
    uint256 creditToYield = convertDebtTokensToYield(credit);
    _subDebt(accountId, credit);

    // Repay debt from earmarked amount of debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    // ... (rest of the function handles collateral and fees)
}
```

The function calculates `earmarkToRemove`, the amount of earmarked debt to repay (capped at `account.earmarked`). It reduces `account.earmarked` by `earmarkToRemove`, correctly updating the position’s local state. Then, it calls `_subDebt` to reduce `account.debt` and `totalDebt` by `credit`.

However, it does not reduce `cumulativeEarmarked`, leaving the global earmarked debt inflated.

For comparison, the `repay` function (used for voluntary repayments) correctly updates both local and global earmark values:

```solidity
function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
    // ... (earlier checks and calculations)
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
    cumulativeEarmarked -= earmarkPaidGlobal;
    // ... (rest of the function)
}
```

The absence of `cumulativeEarmarked -= earmarkPaidGlobal` in `_forceRepay` means that when earmarked debt is repaid during liquidation, the global `cumulativeEarmarked` remains unchanged, overstating the total earmarked debt.

The `_subDebt` function, called by `_forceRepay`, includes a clamping mechanism to prevent `cumulativeEarmarked` from exceeding `totalDebt`:

```solidity
function _subDebt(uint256 tokenId, uint256 amount) internal {
    // ... (debt and collateral updates)
    account.debt -= amount;
    totalDebt -= amount;
    _totalLocked -= toFree;
    account.rawLocked = lockedCollateral - toFree;

    // Clamp to avoid underflow due to rounding later at a later time
    if (cumulativeEarmarked > totalDebt) {
        cumulativeEarmarked = totalDebt;
    }
}
```

After reducing `totalDebt`, the clamp ensures `cumulativeEarmarked <= totalDebt`. This prevents an invalid state but only triggers if `cumulativeEarmarked` exceeds `totalDebt` after a liquidation.

However, in many cases, especially in normal operations with high `totalDebt` (e.g., many active positions), the inflated `cumulativeEarmarked` remains below `totalDebt`. Until then, the incorrect state affects `_earmark` and `_sync` calculations. For example:

* Initial state: `totalDebt = 540e18`, `cumulativeEarmarked = 360e18`.
* Liquidation repays 155e18 earmarked debt: `totalDebt = 385e18`, `cumulativeEarmarked = 360e18` (should be 205e18).
* Since `360e18 < 385e18`, the clamp doesn’t trigger, and the incorrect `cumulativeEarmarked` persists.

## Impact Details

`cumulativeEarmarked`, which tracks the total debt earmarked for redemption across all positions, becomes overstated. An inflated `cumulativeEarmarked` reduces `liveUnearmarked`, which represents the pool of debt available for new earmarking. This leads to an artificially low `liveUnearmarked`, causing the protocol to miscalculate how much debt can be earmarked for redemption.

## Proof of Concept

## Proof of Concept

Add this to the AlchemistV3.t.sol test and run

```solidity
  function testMissingGlobalEarmarkAccountingInForceRepay() public {
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();
    
    vm.startPrank(alOwner);
    alchemist.setProtocolFee(500);
    alchemist.setRepaymentFee(1000);
    vm.stopPrank();
    
    // User A: Deposit, mint, create transmuter position (EARMARKED)
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
    uint256 tokenIdA = AlchemistNFTHelper.getFirstTokenId(yetAnotherExternalUser, address(alchemistNFT));
    uint256 maxDebtA = alchemist.totalValue(tokenIdA) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdA, maxDebtA, yetAnotherExternalUser);
    IERC20(address(alToken)).approve(address(transmuterLogic), maxDebtA);
    transmuterLogic.createRedemption(maxDebtA);
    vm.stopPrank();
    
    // User B: Deposit, mint, create transmuter position (EARMARKED)
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenIdB = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 maxDebtB = alchemist.totalValue(tokenIdB) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdB, maxDebtB, address(0xbeef));
    IERC20(address(alToken)).approve(address(transmuterLogic), maxDebtB);
    transmuterLogic.createRedemption(maxDebtB);
    vm.stopPrank();
    
    // User C: Deposit, mint, NO transmuter (UNEARMARKED) - this prevents clamp
    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, anotherExternalUser, 0);
    uint256 tokenIdC = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
    uint256 maxDebtC = alchemist.totalValue(tokenIdC) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdC, maxDebtC, anotherExternalUser);
    // NO transmuter creation - debt stays unearmarked
    vm.stopPrank();
    
    // Earmark A and B's debt
    vm.roll(block.number + 5_256_000);
    alchemist.poke(tokenIdA);
    alchemist.poke(tokenIdB);
    
    // Capture state
    (, uint256 earmarkedA_before,) = alchemist.getCDP(tokenIdA);
    uint256 cumulativeEarmarked_before = alchemist.cumulativeEarmarked();
    uint256 totalDebt_before = alchemist.totalDebt();
    
    console.log("Before liquidation:");
    console.log("  totalDebt:", totalDebt_before);
    console.log("  cumulativeEarmarked:", cumulativeEarmarked_before);
    
    // Crash MYT price to make User A liquidatable
    uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply);
    uint256 crashedSupply = (initialSupply * 590 / 10_000) + initialSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(crashedSupply);
    
    // Liquidate User A (triggers _forceRepay on earmarked debt)
    vm.prank(externalUser);
    alchemist.liquidate(tokenIdA);
    
    // Check state after
    (, uint256 earmarkedA_after,) = alchemist.getCDP(tokenIdA);
    uint256 cumulativeEarmarked_after = alchemist.cumulativeEarmarked();
    uint256 totalDebt_after = alchemist.totalDebt();
    
    uint256 earmarkedRepaid = earmarkedA_before - earmarkedA_after;
    uint256 expectedCumulativeEarmarked = cumulativeEarmarked_before - earmarkedRepaid;
    
    console.log("\nAfter liquidation:");
    console.log("  totalDebt:", totalDebt_after);
    console.log("  cumulativeEarmarked (actual):", cumulativeEarmarked_after);
    console.log("  cumulativeEarmarked (expected):", expectedCumulativeEarmarked);
    console.log(" Earmarked repaid but global not reduced:", earmarkedRepaid);
    console.log("Lost earmarking capacity:", cumulativeEarmarked_after - expectedCumulativeEarmarked);
    
    assertEq(
        cumulativeEarmarked_after,
        expectedCumulativeEarmarked,
        "Force repay failed to update cumulativeEarmarked"
    );
}
```
