# 58769 sc high forcerepay fails to decrement global cumulativeearmarked causing redemption accounting desynchronization and potential protocol wide redemption halt

**Submitted on Nov 4th 2025 at 12:43:00 UTC by @Pataroff for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58769
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `cumulativeEarmarked` variable, which tracks protocol-wide earmarked debt for redemptions, is properly decremented during normal `repay()` calls. However, during liquidations, which calls `_forceRepay()`, the same reduction is not applied. As a result, `cumulativeEarmarked` remains overstated even though the underlying debt and collateral have been settled.

## Vulnerability Details

When a user repays debt normally using `repay()`, the protocol decreases both the user’s earmarked amount and the global `cumulativeEarmarked`:

```solidity
function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
    ...
    // Repay debt from earmarked amount of debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
@>  account.earmarked -= earmarkToRemove;

    uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
@>  cumulativeEarmarked -= earmarkPaidGlobal;
    ...
}
```

However, during liquidations (`alchemist.liquidate()`), the protocol internally calls `_forceRepay()`. This function correctly updates the user’s local `account.earmarked` but never decreases the global `cumulativeEarmarked value`. This creates a permanent mismatch between real earmarked debt and what the protocol believes is earmarked.

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    ...
        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
@>      account.earmarked -= earmarkToRemove;
         
        //@audit-missing uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
        // @audit-missing cumulativeEarmarked -= earmarkPaidGlobal;
    ...
    }
```

## Impact Details

```
•	`cumulativeEarmarked` remains permanently overstated after liquidations.
•	The protocol believes collateral is still locked for redemptions even though it has already been repaid/liquidated.
•	Future redemptions may be blocked or reduced because the system thinks there is insufficient unearmarked collateral.
•	Users may be unable to redeem valid positions despite sufficient collateral being available.
•	Causes denial-of-service of redemption flow or inaccurate withdrawal/redemption limits.
•	No direct theft of funds, but users may face unfair delays or inability to exit positions.
•  Severity: Medium
•  Impact category: Smart contract unable to operate due to lack of token funds
```

## References

[AlchemistV3.sol#L498-L546](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L498-L546)

[AlchemistV3.sol#L738-L782](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738-L782)

## Proof of Concept

## Proof of Concept

```solidity
function test_CumulativeEarmarkedNotDecremented() external {
    // === Setup ===
    uint256 depositAmount = 500 ether;

    // Each user deposits into the vault to obtain shares
    _magicDepositToVault(address(vault), address(0xA), depositAmount);
    _magicDepositToVault(address(vault), address(0xB), depositAmount);

    // === Step 1: User A deposits into the Alchemist and creates earmarked debt ===
    vm.startPrank(address(0xA));
    SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
    alchemist.deposit(depositAmount, address(0xA), 0);

    // Mint maximum allowable debt (at min. collateralization)
    uint256 tokenA = AlchemistNFTHelper.getFirstTokenId(address(0xA), address(alchemistNFT));
    alchemist.mint(
        tokenA,
        (alchemist.totalValue(tokenA) * FIXED_POINT_SCALAR) / minimumCollateralization,
        address(0xA)
    );

    // Create redemption → increases `cumulativeEarmarked`
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), type(uint256).max);
    transmuterLogic.createRedemption(30 ether);
    vm.stopPrank();

    // Move forward to allow redemptions to mature
    vm.roll(block.number + 5_256_000);

    // === Step 2: User B opens a debt position ===
    vm.startPrank(address(0xB));
    SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
    alchemist.deposit(depositAmount, address(0xB), 0);

    uint256 tokenB = AlchemistNFTHelper.getFirstTokenId(address(0xB), address(alchemistNFT));
    alchemist.mint(
        tokenB,
        (alchemist.totalValue(tokenB) * FIXED_POINT_SCALAR) / minimumCollateralization,
        address(0xB)
    );
    vm.stopPrank();

    vm.roll(block.number + 5_256_000);

    // === Step 3: Normal repay correctly decreases cumulativeEarmarked ===
    uint256 earmarkedBeforeRepay = alchemist.cumulativeEarmarked();

    vm.startPrank(address(0xA));
    // Ensure A has enough vault shares to repay
    deal(address(vault), address(0xA), 100 ether);
    alchemist.repay(25 ether, tokenA);
    vm.stopPrank();

    uint256 earmarkedAfterRepay = alchemist.cumulativeEarmarked();

    assertLt(
        earmarkedAfterRepay,
        earmarkedBeforeRepay,
        "repay() should reduce cumulativeEarmarked"
    );

    // === Step 4: Force User B into liquidation (uses _forceRepay internally) ===
    vm.startPrank(alOwner);
    alchemist.setMinimumCollateralization(alchemist.minimumCollateralization() + 1e18);
    alchemist.setCollateralizationLowerBound(alchemist.collateralizationLowerBound() + 1e18);
    vm.stopPrank();

    // Trigger liquidation → this calls _forceRepay (buggy path)
    uint256 earmarkedBeforeLiq = alchemist.cumulativeEarmarked();

    vm.startPrank(address(0xA));
    alchemist.liquidate(tokenB);
    vm.stopPrank();

    uint256 earmarkedAfterLiq = alchemist.cumulativeEarmarked();

    // Expected failure: earmarked debt is NOT decremented
    assertEq(
        earmarkedBeforeLiq,
        earmarkedAfterLiq,
        "_forceRepay should also reduce cumulativeEarmarked but does not"
    );
}
```
