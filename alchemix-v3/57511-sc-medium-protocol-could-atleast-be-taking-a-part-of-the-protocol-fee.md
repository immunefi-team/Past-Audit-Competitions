# 57511 sc medium protocol could atleast be taking a part of the protocol fee

**Submitted on Oct 26th 2025 at 20:43:31 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57511
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

The function `_forceRepay` is designed in a way where if the collateralBalance is smaller than the `protocolFeeTotal`, the account doesn't pay for the fee. This design makes the protocol lose on potential fees. The code could be changed in a way that if the account doesn't have enough, the code will attempt to decrease the fee to the account collateral, so at least a part of it is paid, making the most out of it.

## Vulnerability Details

If the account doesn't have enough to pay for the fee, the fee is not deducted from account's collateral.

This behavior could be changed so the fee is then decreased at least to the colalteral balance so at least a part of the fee is taken.

Code snippet:

```solidity
        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }
```

This code could be changed into clamping the fee if its larger so at least minimum is taken. This could be the change:

```solidity
        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        } else {
            protocolFeeTotal = account.collateralBalance;
            account.collateralBalance -= protocolFeeTotal;
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }
```

## Impact Details

The impact is that the protocol is losing on a potential profits that could be made if the account still has some collateral.

The protocol could take at least a part of the fee, if not the full amount.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L771>

## Proof of Concept

## Proof of Concept

Add this function in the `AlchemistV3.t.sol` test file:

This PoC demonstrates how the function doesn't deduct any fee when the account doesn't have enough to cover the fee.

Meaning it doesn't take at least a part of it.

```solidity
        function test_legendary_poc() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();
        // just ensureing global alchemist collateralization stays above the minimum required for regular
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));

        vm.stopPrank();
        // modify yield token price via modifying underlying token supply
        (, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);
        // create a redemption to start earmarking debt
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 1200 bps or 12% while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 1200 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        vm.roll(block.number + 5_256_000);
        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        alchemist.liquidate(tokenIdFor0xBeef);
    }
```

To exactly demonstrate the issue I slightly edited the implementation:

```solidity
event NoFeeTaken();

    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        if (amount == 0) {
            return 0;
        }
        _checkForValidAccountId(accountId);
        Account storage account = _accounts[accountId];

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before deciding how much is available to be repaid
        _sync(accountId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 credit = amount > debt ? debt : amount;
        uint256 creditToYield = convertDebtTokensToYield(credit);
        _subDebt(accountId, credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield;

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);
 
        protocolFeeTotal = 199943999999999999999992;
        protocolFeeTotal = 200000000000000000000000;

        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        } else {
            emit NoFeeTaken();
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
        return creditToYield;
    }
```

Result:

```solidity
 │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 892857142857142857000000 [8.928e23]
    │   │   │   └─ ← [Return] 201600000000000000052415 [2.016e23]
    │   │   ├─ emit ForceRepay(accountId: 2, amount: 50000000000000000000 [5e19], creditToYield: 56000000000000000008 [5.6e19], protocolFeeTotal: 0)
    │   │   ├─ emit NoFeeTaken()
    │   │   ├─ [26990] MockMYTVault::transfer(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 56000000000000000008 [5.6e19])
    │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 56000000000000000008 [5.6e19])
```

Result shows that no fee was taken, not even a part of it.
