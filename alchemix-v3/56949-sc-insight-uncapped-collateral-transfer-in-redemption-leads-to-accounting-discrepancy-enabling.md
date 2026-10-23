# 56949 sc insight uncapped collateral transfer in redemption leads to accounting discrepancy enabling theft of user funds

**Submitted on Oct 22nd 2025 at 04:12:22 UTC by @enoch for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56949
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Permanent freezing of funds
  * Smart contract unable to operate due to lack of token funds

## Description

## Relevant Context

The AlchemistV3 contract manages user collateral deposits through the `_mytSharesDeposited` variable, which tracks the total yield tokens deposited by all users. When users create positions and mint debt, a portion of their collateral becomes "locked" based on the `minimumCollateralization` ratio. This locked amount is tracked globally in the `_totalLocked` variable and represents collateral that cannot be withdrawn due to loan-to-value (LTV) constraints.

During redemptions, the `AlchemistV3#redeem` function processes debt repayments by releasing locked collateral back to the transmuter. The function calculates the amount of collateral to release (`collRedeemed`) plus a protocol fee (`feeCollateral`), with their sum stored as `totalOut`. The function then updates `_totalLocked` by subtracting `totalOut`, and this update is properly capped to prevent underflow when `totalOut` exceeds the old locked amount.

## Finding Description

The `AlchemistV3#redeem` function contains an accounting inconsistency in how it handles redemption amounts that exceed the available locked collateral. While the function correctly caps the decrease to `_totalLocked` at zero when `totalOut > old`, it proceeds to transfer the full uncapped `collRedeemed` and `feeCollateral` amounts and decrements `_mytSharesDeposited` by the full uncapped sum.

Specifically, when a redemption occurs where `totalOut > _totalLocked`:

1. The function correctly performs: `_totalLocked = totalOut > old ? 0 : old - totalOut` (capping at zero)
2. The function correctly caps the collateral weight: `_collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old)`
3. However, the function then executes uncapped transfers:
   * `TokenUtils.safeTransfer(myt, transmuter, collRedeemed)`
   * `TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral)`
4. And performs an uncapped state update: `_mytSharesDeposited -= collRedeemed + feeCollateral`

This creates a dangerous accounting discrepancy. When yield token prices decrease significantly, the debt-to-collateral conversion can result in redemption amounts that exceed the actual locked collateral.

### Detailed Accounting Mismatch

The correct behavior when `totalOut > _totalLocked` should be to transfer out collateral and decrease `_mytSharesDeposited` by at most the available `_totalLocked` amount. However, the bug causes excessive collateral transfer and state reduction beyond what's available.

**Expected behavior:**

* Only transfer out collateral up to the available `_totalLocked` amount
* Decrease `_mytSharesDeposited` by at most the old `_totalLocked` value
* Example: If `_totalLocked = 5,500e18` and `totalOut = 6,000e18`, only transfer 5,500e18 and decrease `_mytSharesDeposited` by 5,500e18

**Actual buggy behavior:**

* Transfer out the full uncapped `totalOut` amount
* Decrease `_mytSharesDeposited` by the full uncapped amount
* Example: Transfer all 6,000e18 and decrease `_mytSharesDeposited` by 6,000e18

Using concrete numbers, if before redemption:

* `_totalLocked` = 5,500e18
* `_mytSharesDeposited` = 10,000e18

And redemption calculates `totalOut` = 6,000e18 due to price movements:

* `_totalLocked` is correctly capped to 0
* But 6,000e18 tokens are transferred out (500e18 excess)
* `_mytSharesDeposited` becomes 4,000e18 (should be 4,500e18)
* Contract balance becomes 4,000e18 (should be 4,500e18)

This **500e18 excess transfer** creates an immediate shortfall. The contract's physical token balance becomes insufficient to honor all user claims. The function correctly recognized that only 5,500e18 of locked collateral should be released (by capping `_totalLocked` at zero), but then proceeded to transfer 6,000e18 anyway. This excess comes directly from the general collateral pool, reducing all users' effective claim on the contract's assets.

The root cause is the assumption that `totalOut` will always be less than or equal to `_totalLocked`. However, adverse price movements in the underlying yield token violate this assumption, causing the accounting mismatch between global collateral tracking and actual available funds.

## Impact

Users in the AlchemistV3 contract suffer direct loss equal to the excess collateral transferred during the buggy redemption. When later users attempt to withdraw their deposits, they cannot recover their full amounts as the contract's actual balance is lower than the sum of all user claims. The shortfall is socialized across depositors, with users who withdraw later bearing the loss as earlier users drain the insufficient funds.

## Recommendation

Cap the actual token transfers and state updates to match the capped locked collateral amount. When `totalOut` exceeds the old `_totalLocked` value, proportionally reduce both `collRedeemed` and `feeCollateral` so their sum equals the old locked amount:

```diff
     // update locked collateral + collateral weight
     uint256 old = _totalLocked;
     _totalLocked = totalOut > old ? 0 : old - totalOut;
     _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);

-    TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
-    TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
-    _mytSharesDeposited -= collRedeemed + feeCollateral;
+    uint256 actualTotalOut = totalOut > old ? old : totalOut;
+    uint256 actualCollRedeemed = collRedeemed * actualTotalOut / totalOut;
+    uint256 actualFeeCollateral = actualTotalOut - actualCollRedeemed;
+
+    TokenUtils.safeTransfer(myt, transmuter, actualCollRedeemed);
+    TokenUtils.safeTransfer(myt, protocolFeeReceiver, actualFeeCollateral);
+    _mytSharesDeposited -= actualTotalOut;

     emit Redemption(redeemedDebtTotal);
```

This ensures that when redemptions exceed available locked collateral, the function only transfers and accounts for the actual available amount, maintaining consistency between `_mytSharesDeposited`, `_totalLocked`, individual user balances, and the contract's physical token balance.

## Proof of Concept

## Proof of Concept

This proof of concept demonstrates how the accounting discrepancy causes loss of user funds through the following sequence:

1. **Initial Setup**: A user (whale) deposits 10,000e18 collateral and mints 5,000e18 debt with a 1.1x minimum collateralization ratio. This locks 5,500e18 of collateral (`5,000e18 * 1.1 = 5,500e18`).
2. **Redemption Creation**: The whale creates a redemption for the full 5,000e18 debt in the transmuter.
3. **Price Shock**: The yield token price decreases by 20% (simulated by increasing the total supply from `initialVaultSupply` to `initialVaultSupply * 1.2`). This means each yield token is now worth less underlying value.
4. **Claim Redemption**: After the maturity period, the whale claims the redemption. Due to the price decrease, the conversion from debt tokens to yield tokens results in `totalOut > _totalLocked`. The contract correctly caps `_totalLocked` to zero but incorrectly transfers out excess collateral and decrements `_mytSharesDeposited` by more than the old locked amount.
5. **Accounting State After Redemption**:
   * `_mytSharesDeposited`: \~4,000e18 (should be 4,500e18)
   * Contract actual balance: \~4,000e18 (should be 4,500e18)
   * Whale's `collateralBalance`: 4,500e18
   * **Shortfall created: 500e18**
6. **New Deposit**: An external user deposits 1,000e18 of fresh collateral, bringing `_mytSharesDeposited` to \~5,000e18 and contract balance to 5,000e18.
7. **First Withdrawal**: The whale withdraws their full 4,500e18 balance, leaving only 500e18 in the contract.
8. **Loss Materialized**: The new user who deposited 1,000e18 can only withdraw 500e18 as that's all that remains. They lose 500e18, exactly equal to the excess amount transferred during the buggy redemption.

To reproduce this issue, add the following test to `src/test/AlchemistV3.t.sol`:

```solidity
function test_poc_uncapped_total_collateral_decrease_in_redemption() external {
    // Modify minimumCollateralization to 1.1e18
    vm.prank(alOwner);
    alchemist.setMinimumCollateralization(1.1e18);

    vm.startPrank(someWhale);
    SafeERC20.safeApprove(address(vault), address(alchemist), 10_000e18);
    alchemist.deposit(10_000e18, someWhale, 0);
    uint256 tokenIdWhale = AlchemistNFTHelper.getFirstTokenId(someWhale, address(alchemistNFT));
    alchemist.mint(tokenIdWhale, 5_000e18, someWhale);
    vm.stopPrank();

    // Total locked should be 5_000e18 * minimumCollateralization = 5_500e18
    assertEq(alchemist.getTotalLocked(), 5_500e18);      

    vm.startPrank(someWhale);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 5_000e18);
    transmuterLogic.createRedemption(5_000e18);
    uint256 tokenIdRedemptionWhale = AlchemistNFTHelper.getFirstTokenId(someWhale, address(transmuterLogic));
    vm.stopPrank();  

    // Simulate yield token price decrease by 20%
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply * 12 / 10);

    vm.roll(block.number + 5_256_000 + 10);
    vm.prank(someWhale);
    transmuterLogic.claimRedemption(tokenIdRedemptionWhale);

    alchemist.poke(tokenIdWhale);

    // Demonstrates the accounting discrepancy
    console.log("alchemist.getMytSharesDeposited()", alchemist.getMytSharesDeposited());
    console.log("alchemist.getAccountCollateralBalance(tokenIdWhale)", alchemist.getAccountCollateralBalance(tokenIdWhale));

    // Another user deposits to the contract
    vm.startPrank(externalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), 1_000e18);
    alchemist.deposit(1_000e18, externalUser, 0);
    uint256 tokenIdExternalUser = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
    vm.stopPrank();

    // The whale can withdraw using the shortfall covered by new user's deposit
    vm.prank(someWhale);
    alchemist.withdraw(4_500e18, someWhale, tokenIdWhale);

    // The new user cannot withdraw their full deposit
    vm.startPrank(externalUser);
    vm.expectRevert();
    alchemist.withdraw(1_000e18, externalUser, tokenIdExternalUser);
    vm.stopPrank();
}
```

Run the test with: `forge test --match-test test_poc_uncapped_total_collateral_decrease_in_redemption -vv`

The test requires the following helper functions to be added to `AlchemistV3.sol` for querying internal state:

```solidity
function getTotalLocked() external view returns (uint256) {
    return _totalLocked;
}

function getMytSharesDeposited() external view returns (uint256) {
    return _mytSharesDeposited;
}

function getAccountCollateralBalance(uint256 tokenId) external view returns (uint256) {
    return _accounts[tokenId].collateralBalance;
}
```
