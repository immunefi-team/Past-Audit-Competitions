# 58280 sc critical repayment s fee is charged from other users causing the contract to fail when the myt total balance of a user cannot cover the fee

**Submitted on Oct 31st 2025 at 22:57:26 UTC by @Outliers for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58280
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

There’s an accounting issue in the liquidation logic that occurs after a force repay. When a user’s account doesn’t have enough collateral left to cover the liquidator fee, the protocol still transfers the full fee to the liquidator using the Alchemist contract’s own balance. In short, the protocol ends up paying liquidators out of its own funds instead of the debtor’s

## Vulnerability Details

```solidity
   /// @dev Handles repayment fee calculation and account deduction
    /// @param accountId The tokenId of the account to force a repayment on.
    /// @param repaidAmountInYield The amount of debt repaid in yield tokens.
    /// @return fee The fee in yield tokens to be sent to the liquidator.
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
       
@audit>>>           fee = repaidAmountInYield * repaymentFee / BPS;
        emit balancebefore(account.collateralBalance,fee );

@audit>>>        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;

@audit>>>          emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);                                  
@audit>>>          return fee;  // bug returned value is wrong // cap at collateral balance actually or ensure we remove from balance before 
    }



```

However, the issue arises when the user’s collateralBalance is less than the expected feeInYield. Even though the balance check is in place, the contract does not reduce or cap the fee based on what the account can actually cover. In scenarios where the user’s collateral has already been partially depleted (for example, through force repayment), the remaining balance may be insufficient to pay the full fee.

Despite that, the protocol still proceeds with the transfer to the liquidator — meaning the excess amount is effectively taken from the Alchemist contract’s token balance, not from the user’s account. This causes the system to subsidize liquidators using protocol funds.

```solidity
       // Try to repay earmarked debt if it exists
        uint256 repaidAmountInYield = 0;
        if (account.earmarked > 0) {
            repaidAmountInYield = _forceRepay(accountId, account.earmarked);
        }
        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        if (account.debt == 0) {

@audit>>            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);

@audit>>              TokenUtils.safeTransfer(myt, msg.sender, feeInYield);                    // no update on the mytdeposit // write test to force all debt to be earnmarked and force failure here no myt update also
            return (repaidAmountInYield, feeInYield, 0);
        }

```

```solidity
     if (collateralizationRatio <= collateralizationLowerBound) {
            // Do actual liquidation
            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Since only a repayment happened, send repayment fee to caller

@audit>>            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);

@audit>>              TokenUtils.safeTransfer(myt, msg.sender, feeInYield);                  // no update on the mytdeposit // we transfer above user collateral balance.

            return (repaidAmountInYield, feeInYield, 0);
        }
    }

```

In short:

1. Force repay reduces the user’s balance.
2. Liquidation logic calculates a repayment fee.
3. Even if the user has little or no collateral left, the fee is still transferred out.
4. The contract’s own MYT balance covers the shortfall.

## Impact Details

This can occur in every liquidation occurring after a partial or complete force repayment. Protocol funds (MYT tokens) will be incorrectly reduced over time as liquidators receive more than what was available in the user’s account.

## References

* `_doLiquidation()`
* Post-force-repay liquidation branch that handles `feeInYield` transfer

## Proof of Concept

## Proof of Concept

```solidity

 function testLiquidate_Full_Liquidation_Globally_Undercollateralized_trigger() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

    

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
         uint debt123 = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdFor0xBeef, debt123, address(0xbeef));
        vm.stopPrank();

         
        vm.startPrank(address(anotherExternalUser));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debt123);
        transmuterLogic.createRedemption(debt123);
        vm.stopPrank();

        vm.roll(block.number + 5_256_100);


          console.log("Balance after creation in transmuter:", alchemist.bal(tokenIdFor0xBeef));
           (uint256 depositedCollateral2, uint256 debt1,uint mark) = alchemist.getCDP(tokenIdFor0xBeef);
           console.log("collateral CDP after creation in transmuter:", depositedCollateral2);
           console.log("debt CDP after creation in transmuter:", debt1);
           console.log("mark CDP after creation in transmuter:", mark);

      
        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // modify yield token price via modifying underlying token supply
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
    

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 1590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);


        console.log("Bad debt ratio before:", transmuterLogic.baddebtratio());

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorPrevUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));

        uint256 alchemistCurrentCollateralization =
            alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();


            console.log("Current collateralization:", alchemistCurrentCollateralization);
            console.log("global collateralization:", alchemist.globalMinimumCollateralization());
        (uint256 liquidationAmount, uint256 expectedDebtToBurn,,) = alchemist.calculateLiquidation(
            alchemist.totalValue(tokenIdFor0xBeef),
            prevDebt,
            alchemist.minimumCollateralization(),
            alchemistCurrentCollateralization,
            alchemist.globalMinimumCollateralization(),
            liquidatorFeeBPS
        );
        uint256 expectedLiquidationAmountInYield = alchemist.convertDebtTokensToYield(liquidationAmount);
        uint256 expectedBaseFeeInYield = 0;

        // Account is still collateralized, but pulling from fee vault for globally bad debt scenario
        uint256 expectedFeeInUnderlying = expectedDebtToBurn * liquidatorFeeBPS / 10_000;

        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);

        (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);

        vm.stopPrank();

       (uint collaterals, uint debts, uint earmarkeds) = alchemist.getCDP(tokenIdFor0xBeef);

        vm.startPrank(address(anotherExternalUser));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();


```

```solidity
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 862812769628990509000000 [8.628e23]
    │   │   │   └─ ← [Return] 208620000000000000035256 [2.086e23]
    │   │   ├─ emit ForceRepay(accountId: 1, amount: 180000000000000000018000 [1.8e23], creditToYield: 200000000000000000000000 [2e23], protocolFeeTotal: 2000000000000000000000 [2e21])
    │   │   ├─ [26990] MockMYTVault::transfer(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 200000000000000000000000 [2e23])
    │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 200000000000000000000000 [2e23])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit balancebefore(balancebefore: 0, fee: 2000000000000000000000 [2e21])
    │   │   ├─ emit RepaymentFee(accountId: 1, amount: 200000000000000000000000 [2e23], feeReciever: 0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420, fee: 2000000000000000000000 [2e21])
    │   │   ├─ [3543] MockMYTVault::transfer(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420, 2000000000000000000000 [2e21])
    │   │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   │   └─ ← [Revert] ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)
    │   └─ ← [Revert] ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)
    └─ ← [Revert] ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 12.09ms (3.15ms CPU time)

Ran 1 test suite in 111.47ms (12.09ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/test/AlchemistV3.t.sol:AlchemistV3Test
[FAIL: ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)] testLiquidate_Full_Liquidation_Globally_Undercollateralized_trigger() (gas: 3180613)

Encountered a total of 1 failing tests, 0 tests succeeded

```
