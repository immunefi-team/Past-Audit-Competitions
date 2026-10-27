# 57918 sc high incorrect totallocked collateral accounting in alchemistv3

**Submitted on Oct 29th 2025 at 13:03:39 UTC by @Razkky for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57918
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

A critical accounting mismatch exists in the AlchemistV3 protocol’s handling of the `totalLocked` collateral variable. When the system’s `minimumCollateralization` ratio is updated after users have already minted debt, the protocol’s global `totalLocked` value becomes inconsistent with the sum of all users’ actual locked collateral (`rawLocked`). This discrepancy can lead to incorrect calculations of collateral weight during redemptions, resulting in unfair or inaccurate outcomes for users.

***

## Technical Details

### Debt Minting and Collateral Locking

When a user mints debt, the `mint` is called which calls `_mint`, which in turn calls `_addDebt`. This function:

* Updates the user’s debt.
* Locks the required amount of collateral for the new debt, based on the **current** `minimumCollateralization` value.
* Recalculates the user’s previous locked collateral using the **current** `minimumCollateralization` value, ensuring the user’s `rawLocked` is always correct and up-to-date.

```solidity

    function _addDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];

        // Update collateral variables
        uint256 toLock = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
        uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        if (account.collateralBalance - lockedCollateral < toLock) revert Undercollateralized();

        account.rawLocked = lockedCollateral + toLock;
        console.log("User rawLocked", account.rawLocked);
        _totalLocked += toLock;
        console.log("Total locked after addDebt:", _totalLocked);
        account.debt += amount;
        totalDebt += amount;
    }

```

### The Mismatch

However, the protocol’s global `totalLocked` variable is only incremented by the new collateral locked for each mint, calculated using the current `minimumCollateralization`. It does **not** retroactively update the previously calculated locked collateral for the users with the current `minimumCollateralization`.

#### Example Scenario

1. **Initial State:**
   * `minimumCollateralization = 1.5e18`
   * User mints 1000 debt tokens.
   * User’s `rawLocked = 1000 * 1.5e18 / 1e18 = 1500e18`
   * `totalLocked = 1500e18`
2. **Collateralization Ratio Update:**
   * `minimumCollateralization` is increased to `1.6e18`.
3. **Second Mint:**
   * User mints another 1000 debt tokens.
   * The user’s `rawLocked` is recalculated to include both the old and new debt at the new ratio:
     * Old locked collateral is updated: `1000 * 1.6e18 / 1.0e18 = 1600e18`
     * New locked collateral: `1000 * 1.6e18 / 1e18 = 1600e18`
     * Total `rawLocked = 3200e18`
   * **But** `totalLocked` is only incremented by the new collateral locked for the second mint:
     * `totalLocked = 1500e18 (old) + 1600e18 (new) = 3100e18`
   * The correct total locked collateral should be the sum of all users’ `rawLocked`, which is now `3200e18`, not `3100e18` since we only have one user with debt and locked collateral.

### Impact on Collateral Weight and Redemptions

The protocol uses `totalLocked` to calculate the collateral weight during redemptions:

```solidity
uint256 old = _totalLocked;
_totalLocked = totalOut > old ? 0 : old - totalOut;
_collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);
```

If `totalLocked` is incorrect, the `old` value used in this calculation is wrong, leading to an incorrect update of `_collateralWeight`. This, in turn, causes the protocol to redeem the wrong amount of collateral for users, resulting in unfair or inaccurate redemptions.

## Security and Fairness Implications

* **Unfair Redemptions:** Users may receive less or more collateral than they are entitled to during redemptions, depending on the direction of the mismatch.
* **Protocol Risk:** If the mismatch is exploited or grows over time, it could lead to systemic insolvency or unfair liquidations.
* **User Trust:** Inaccurate accounting undermines user trust in the protocol’s safety and correctness.

## Recommendations

* **Synchronize `totalLocked`:** Ensure that `totalLocked` is always updated to reflect the sum of all users’ `rawLocked` whenever `minimumCollateralization` changes.
  * **Concrete approach:** When a user mints new debt, first subtract their previous `rawLocked` value from `totalLocked`, then recalculate their new `rawLocked` using the current `minimumCollateralization`, and finally add this new value back to `totalLocked`. This ensures that `totalLocked` always matches the sum of all users’ locked collateral, even as the collateralization ratio changes.

````solidity
    ```solidity
        function _addDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];

        // Update collateral variables
        uint256 oldLock = account.rawLocked;
        uint256 toLock = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
        uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        if (account.collateralBalance - lockedCollateral < toLock) revert Undercollateralized();

        account.rawLocked = lockedCollateral + toLock;
        // Remove the previous calculated locked collateral for user.
        _totalLocked -= oldLocked > _totalLocked: _totalLocked: oldLocked;
        // updated with the total rawLocked for user using the current minimumCollateralization
        _totalLocked += account.rawLocked;
        account.debt += amount;
        totalDebt += amount;
    }

````

## Proof of Concept

## Proof of Concept

Add the below test case to `AlchemistV3.t.sol` and run the following command to run the test `forge test --mt test_AccountingMissmatchTotalRawlockedAndtotalLocked -vvv`

```solidity

    function test_AccountingMissmatchTotalRawlockedAndtotalLocked() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        minimumCollateralization = 1.5e18;
        vm.startPrank(alOwner);
        alchemist.setGlobalMinimumCollateralization(1.5e18);
        alchemist.setMinimumCollateralization(1.5e18);
        vm.stopPrank();

        uint256 debtToMint = 100e18;

        vm.startPrank(yetAnotherExternalUser);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(yetAnotherExternalUser), address(alchemistNFT));
        alchemist.mint(tokenId, debtToMint, address(yetAnotherExternalUser));
        vm.stopPrank();

        // Update minimum collateralization
        vm.startPrank(alOwner);
        alchemist.setGlobalMinimumCollateralization(1.5e18);
        alchemist.setMinimumCollateralization(1.6e18);
        vm.stopPrank();
        // Mint again to have debt with new minimum collateralization
        vm.startPrank(yetAnotherExternalUser);
        alchemist.mint(tokenId, debtToMint, address(yetAnotherExternalUser));
        vm.stopPrank();
        (uint256 userCollateral, uint256 userDebt,) = alchemist.getCDP(tokenId);

        // the totalLocked should equal the actual totalLocked collateral which is the sum of all the rawLocked of all account since we have one account with debt
        // totalLocked = the rawLocked for that account
        // totalLocked = Total Debt * minimumCollateralization / FIXED_POINT_SCALAR
        // since user is the only one with debt Total Debt = userDebt of user
        assertEq(userDebt, debtToMint * 2);
        assertEq(alchemist.totalDebt(), debtToMint * 2);
        uint256 actualTotalLocked = userDebt * alchemist.minimumCollateralization() / FIXED_POINT_SCALAR;
        console.log("Actual total Locked", actualTotalLocked);
        // Total Locked in the protocol accounting system
        uint256 totalLockedByProtocol = alchemist.totalLocked();
        console.log("Total Locked recorded by the protocol", totalLockedByProtocol);
        assertGt(actualTotalLocked, totalLockedByProtocol);
        // Total locked by the protocol is incorrect which would lead to incorrect calculation of collateralWeight which would lead to incorrect redemption amount
    }
```

Add console.log to the `_addDebt` function to log the raw locked and totalLocked to match the logs outpout

Test Logs

```solidity
Logs:
  User rawLocked 150000000000000000000
  Total locked after addDebt: 150000000000000000000
  User rawLocked 320000000000000000000
  Total locked after addDebt: 310000000000000000000
  Actual total Locked 320000000000000000000
  Total Locked recorded by the protocol 310000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 31.03ms (3.40ms CPU time)

Ran 1 test suite in 1.35s (31.03ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)


```
