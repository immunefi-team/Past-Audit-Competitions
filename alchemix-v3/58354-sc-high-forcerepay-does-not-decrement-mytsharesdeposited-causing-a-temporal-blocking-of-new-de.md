# 58354 sc high forcerepay does not decrement mytsharesdeposited causing a temporal blocking of new deposits

**Submitted on Nov 1st 2025 at 14:08:17 UTC by @Pataroff for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58354
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

Unlike `repay()`, the `_forceRepay()` function does **not** decrement `_mytSharesDeposited` after transferring MYT tokens to the Transmuter and fee receiver.

As a result, `_mytSharesDeposited` is overstated relative to the actual MYT balance held by the contract, preventing new deposits from succeeding until manual intervention.

## Vulnerability Details

During liquidation, `_forceRepay()` uses a user’s collateral to repay outstanding debt and transfer funds to the Transmuter and protocol fee receiver. While the user’s per-account state is updated correctly, the global `_mytSharesDeposited` variable is not decremented, leaving it stale:

```solidity
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        ...
        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
 @> account.collateralBalance -= creditToYield;

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }

@> //@audit-missing _mytSharesDeposited -= creditToYield

        return creditToYield;
    }
```

As a consequence, subsequent deposits will fail if the deposit cap was reached since we enforce the deposit cap check against the `_mytSharesDeposited` variable:

```solidity
    function deposit(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
      ...
@> _checkState(_mytSharesDeposited + amount <= depositCap);
      ...
    }
```

## Impact Details

```
•	New deposits can be temporarily blocked, even though the strategy's real balance is below the deposit cap
•	Users cannot deposit until an admin manually adjusts the deposit cap through `setDepositCap`
•	No user funds are lost or stolen, and existing positions are unaffected.
•	Normal protocol operations are disrupted, which can affect user experience, automated strategies, and composability with other protocols.
•	Severity: Medium
•	Impact category: Temporary freezing of funds for at least 1 hour
```

## References

[AlchemistV3.sol#L738-L782](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738-L782)

[AlchemistV3.sol#L369](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L369)

## Proof of Concept

## Proof of Concept

```solidity
    function test_MytSharesDepositedNotDecremented() external {
        // --- Step 0: Setup deposit parameters ---
        uint256 depositCap = 1_000 ether;
        uint256 userADeposit = 500 ether;
        uint256 userBDeposit = 500 ether;
        uint256 userCDeposit = 250 ether;

        // --- Step 1: Mint vault shares for each user ---
        _magicDepositToVault(address(vault), address(0xA), userADeposit);
        _magicDepositToVault(address(vault), address(0xB), userBDeposit);
        _magicDepositToVault(address(vault), address(0xC), userCDeposit);

        // --- Step 2: Set the deposit cap on the strategy ---
        vm.startPrank(alOwner);
        alchemist.setDepositCap(depositCap);
        vm.stopPrank();

        // --- Step 3: User A deposits into the strategy ---
        vm.startPrank(address(0xA));
        SafeERC20.safeApprove(address(vault), address(alchemist), userADeposit);
        alchemist.deposit(userADeposit, address(0xA), 0);
        vm.stopPrank();

        // --- Step 4: User B deposits and mints a position NFT ---
        vm.startPrank(address(0xB));
        SafeERC20.safeApprove(address(vault), address(alchemist), userBDeposit);
        alchemist.deposit(userBDeposit, address(0xB), 0);

        uint256 tokenB = AlchemistNFTHelper.getFirstTokenId(address(0xB), address(alchemistNFT));

        // Mint enough debt to reach the current minimum collateralization
        alchemist.mint(
            tokenB,
            ((alchemist.totalValue(tokenB) * FIXED_POINT_SCALAR) / minimumCollateralization),
            address(0xB)
        );
        vm.stopPrank();

        // --- Step 5: Artificially increase the minimum collateralization ---
        console.log("Min collRatio before update:", alchemist.minimumCollateralization()); // 1.111e18

        vm.startPrank(alOwner);
        alchemist.setMinimumCollateralization(alchemist.minimumCollateralization() + 1e18);
        alchemist.setCollateralizationLowerBound(alchemist.collateralizationLowerBound() + 1e18);
        vm.stopPrank();

        console.log("Min collRatio post update:", alchemist.minimumCollateralization()); // 2.111e18

        // Verify total deposited before liquidation
        console.log("Total before liquidation:", alchemist.getTotalDeposited());

        // Sanity check: total deposited = deposit cap
        assert(alchemist.getTotalDeposited() == alchemist.depositCap());

        // --- Step 6: Liquidate User B ---
        vm.startPrank(address(0xC));

        // Perform liquidation
        (uint256 yieldAmount, , ) = alchemist.liquidate(tokenB);
        console.log("Liquidated assets:", yieldAmount);
        vm.stopPrank();

        // Total deposited is decremented in ERC20 accounting, but `_mytSharesDeposited` remains stale
        console.log("Total after liquidation:", alchemist.getTotalDeposited());

        // --- Step 7: User C attempts deposit ---
        vm.startPrank(address(0xC));
        SafeERC20.safeApprove(address(vault), address(alchemist), userCDeposit);

        // Sanity check: total deposited < deposit cap
        assert(alchemist.getTotalDeposited() < alchemist.depositCap());

        console.log("User C attempting deposit:", userCDeposit);
        // EXPECT: deposit still fails due to stale `_mytSharesDeposited`
        vm.expectRevert(); // Deposit cap exceeded
        alchemist.deposit(userCDeposit, address(0xC), 0);
        vm.stopPrank();
    }
```
