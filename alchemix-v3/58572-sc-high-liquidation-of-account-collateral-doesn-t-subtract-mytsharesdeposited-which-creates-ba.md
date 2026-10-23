# 58572 sc high liquidation of account collateral doesn t subtract mytsharesdeposited which creates bad debt in the system and causes insolvency&#x20;

**Submitted on Nov 3rd 2025 at 09:42:10 UTC by @Tarnishedx0 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58572
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Liquidation of account `collateral` doesn't subtract `_mytSharesDeposited` which creates bad debt in the system and causes insolvency.

## Vulnerability Details

`_mytSharesDeposited` is defined as:

```soliditiy
    /// @dev Total yield tokens deposited
    /// This is used to differentiate between tokens deposited into a CDP and balance of the contract
    uint256 private _mytSharesDeposited;
```

When `liquidation` happens, `collateral` is liquidated and transferred to the `transmuter`.

```solidity
    function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        Account storage account = _accounts[accountId];

        (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
            collateralInUnderlying,
            account.debt,
            minimumCollateralization,
            normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,
            globalMinimumCollateralization,
            liquidatorFee
        );

        amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

....

```

But, `_mytSharesDeposited` is not updated accordingly. Also, during liquidation if only `_forceRepay()` happens:

```solidity
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

        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
        return creditToYield;
    }
```

Some `portion + fee` of `account.CollateralBalance` is repaid and subtracted but not accounted in the `_mytSharesDeposited`. Thus, `_mytSharesDeposited` inaccurately represents the total amount of collateral in the contract and is `inflated`.

## Impact Details

`_mytSharesDeposited` is used to calculate the `totalUnderlyingValue` of the `AlchemistV3Contract`.

```solidity
    function getTotalUnderlyingValue() external view returns (uint256) {
        return _getTotalUnderlyingValue();
    }
```

```solidity
    function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
        uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
        totalUnderlyingValue = yieldTokenTVLInUnderlying;
    }
```

This is directly used in the `Transmuter.sol` contract to calculate `badDebtRatio`.

```solidity
    function claimRedemption(uint256 id) external {

... 

        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

...
```

Now the overinflated `_mytSharesDeposited` will also `inflate` the `denominator` which in turn will `deflate` the `badDebtRatio`. This `badDebtRatio` is directly used to calculate the amount that `users` can redeem if a system is in bad debt.

```solidity
        uint256 scaledTransmuted = amountTransmuted;

        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }

        // If the contract has a balance of yield tokens from alchemist repayments then we only need to redeem partial or none from Alchemist earmarked
        uint256 debtValue = alchemist.convertYieldTokensToDebt(yieldTokenBalance);
        uint256 amountToRedeem = scaledTransmuted > debtValue ? scaledTransmuted - debtValue : 0;
```

Thus, even when system is in a `bad Debt` state, Users will be able to `claim more` than they should due to `bad debt ratio` being calculated `incorrectly` and `less` than it should. Thus, again increasing `bad debt` and causing `protocol insolvency`.

## References

All the code snippets used above can be verified here: claimRedemption()#Transmuter.sol: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L191-L266>

\_doLiquidation()#AlchemistV3.sol: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852-L894>

\_forceRepay()#AlchemistV3.sol:<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738-L782>

## Proof of Concept

## Proof of Concept

Here is a `PoC` to prove that after liquidation `_mytSharesDeposited` doesn't change.

```solidity
    function testLiquidatePocMyt() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        uint256 sharesBalance = IERC20(address(vault)).balanceOf(address(yetAnotherExternalUser));
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // modify yield token price via modifying underlying token supply
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorPrevUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));

        uint256 alchemistCurrentCollateralization =
            alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();
        (uint256 liquidationAmount, uint256 expectedDebtToBurn, uint256 expectedBaseFee,) = alchemist.calculateLiquidation(
            alchemist.totalValue(tokenIdFor0xBeef),
            prevDebt,
            alchemist.minimumCollateralization(),
            alchemistCurrentCollateralization,
            alchemist.globalMinimumCollateralization(),
            liquidatorFeeBPS
        );
        uint256 expectedLiquidationAmountInYield = alchemist.convertDebtTokensToYield(liquidationAmount);
        uint256 expectedBaseFeeInYield = alchemist.convertDebtTokensToYield(expectedBaseFee);

        // Account is still collateralized, so not pulling from the fee vault for underlying
        uint256 expectedFeeInUnderlying = 0;

        uint256 totalUnderlyingBefore = alchemist.getTotalUnderlyingValue();
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 totalUnderlyingAfter = alchemist.getTotalUnderlyingValue();

        vm.stopPrank();

        // verify both value before and after are equal and no change happened
        assertEq(totalUnderlyingBefore, totalUnderlyingAfter);
    }
```

Paste this in the `AlchemistV3.t.sol` file, set up `$MAINNET_RPC_URL` and run it using:

```solidity
FOUNDRY_PROFILE=default forge test --fork-url $MAINNET_RPC_URL --match-path src/test/AlchemistV3.t.sol --match-test testLiquidatePocMyt -vv --evm-version cancun
```

To prove how not updating `_mytSharesDeposited` results in users `redeeming` more than they should:

```solidity
function testPocMytAndClaim1() external {
    // First Case, Setting Underlying Value to Accurate Value
    uint256 transmuterYieldBalance = 100e18;
    deal(address(collateralToken), address(transmuter), transmuterYieldBalance);
    
    alchemist.setSyntheticsIssued(1000e18);
    alchemist.setUnderlyingValue(400e18);
    
    // User deposits 100e18 synthetic tokens
    uint256 depositAmount = 100e18;
    vm.prank(address(0xbeef));
    transmuter.createRedemption(depositAmount);
    
    // Fast forward to full maturation
    vm.roll(block.number + 5_256_000);
    
    uint256 balanceBefore = collateralToken.balanceOf(address(0xbeef));
    
    // Claim the redemption
    vm.prank(address(0xbeef));
    transmuter.claimRedemption(1);
    
    uint256 balanceAfter = collateralToken.balanceOf(address(0xbeef));
    // Amount that the user receives after bad debt calculation
    uint256 actualReceivedFirst = balanceAfter - balanceBefore;
    vm.stopPrank();

    // Amount received when UnderlyingValue is lower
    console.log("Amount Received First: ", actualReceivedFirst);
}
```

```solidity

function testPocMytAndClaim2() external {
    //Second Case: Setting Underlying Value to inflate value due to ``_mytSharesDeposited`` not being subtracted.
    uint256 transmuterYieldBalance = 100e18;
    deal(address(collateralToken), address(transmuter), transmuterYieldBalance);
    
    alchemist.setSyntheticsIssued(1000e18);
    alchemist.setUnderlyingValue(500e18);  // Inflating by 100e18
    
    // User again deposits 100e18 synthetic tokens
    uint256 depositAmount = 100e18;
    vm.prank(address(0xbeef));
    transmuter.createRedemption(depositAmount);
    
    // Fast forward to full maturation
    vm.roll(block.number + 5_256_000);
    
    uint256 balanceBeforeSecond = collateralToken.balanceOf(address(0xbeef));
    
    // Claim the redemption
    vm.prank(address(0xbeef));
    transmuter.claimRedemption(1);
    
    uint256 balanceAfterSecond = collateralToken.balanceOf(address(0xbeef));
    // Amount that the user receives after bad debt calculation
    uint256 actualReceivedSecond = balanceAfterSecond - balanceBeforeSecond;
    vm.stopPrank();

    //Amount received when UnderLyingValue is higher
    console.log("Amount Received Second: ", actualReceivedSecond);
}
```

Paste both of these tests in `Transmuter.t.sol` and run it using the following commands:

```solidity
FOUNDRY_PROFILE=default forge test --fork-url $MAINNET_RPC_URL --match-path src/test/Transmuter.t.sol --match-test testPocMytAndClaim1 -vv --evm-version cancun

FOUNDRY_PROFILE=default forge test --fork-url $MAINNET_RPC_URL --match-path src/test/Transmuter.t.sol --match-test testPocMytAndClaim2 -vv --evm-version cancun
```

If we look at the `output` of first test where `totalUnderlyingValue` is accurate:

> Amount Received First: 30000000000000000012

And the `output` of `second test` where `totalUnderlyingValue` is not accurate and inflated:

> Amount Received Second: 35000000000000000010

Thus, it is proven that `users` will receive more than they should due to inaccurate `_mytSharesDeposited` accounting and cause bad debt to the protocol which will result in protocol insolvency.
