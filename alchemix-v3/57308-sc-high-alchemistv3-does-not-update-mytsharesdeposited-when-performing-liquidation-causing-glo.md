# 57308 sc high alchemistv3 does not update mytsharesdeposited when performing liquidation causing global accounting and liquidation logic mismatch

**Submitted on Oct 25th 2025 at 05:41:22 UTC by @joicygiore for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57308
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

During liquidation and forced-repay flows, AlchemistV3 deducts user `account.collateralBalance` (user collateral recorded in yield tokens) but fails to update the corresponding global tracking variable(s) (e.g. `_mytSharesDeposited` or other cached total underlying). Subsequent calculations of the protocol’s collateralization (`alchemistCurrentCollateralization`) use the stale global value, overestimating available collateral and preventing expected liquidation behavior (including full liquidation when global LTV is too high). This results in incorrect accounting, skipped liquidations and incorrect user balances.

## Vulnerability Details

Several code paths deduct from `account.collateralBalance` (fee deduction in `_resolveRepaymentFee()`, forced repay `_forceRepay()`, liquidation `_doLiquidation()` — marked `@>1` in supplied code) but do not synchronously update the global deposited shares metric used by `_getTotalUnderlyingValue()` / `_mytSharesDeposited`.

`_liquidate()` later recomputes the alchemist’s current collateralization using `normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt` (see `@>2`), and because `_mytSharesDeposited` was not updated, the returned total underlying is too large. This causes `calculateLiquidation` to receive inflated global collateralization and to skip liquidation branches that should run (particularly the high-LTV full liquidation branch).

```js
    // AlchemistV3::_liquidate()
    function _liquidate(uint256 accountId) internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) {
        // Query transmuter and earmark global debt
        _earmark();
        // Sync current user debt before deciding how much needs to be liquidated
        _sync(accountId);

        Account storage account = _accounts[accountId];

        // Early return if no debt exists
        if (account.debt == 0) {
            return (0, 0, 0);
        }

        // In the rare scenario where 1 share is worth 0 underlying asset
        if (IVaultV2(myt).convertToAssets(1e18) == 0) {
            return (0, 0, 0);
        }

        // Calculate initial collateralization ratio
        uint256 collateralInUnderlying = totalValue(accountId);
        uint256 collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        // If account is healthy, nothing to liquidate
        if (collateralizationRatio > collateralizationLowerBound) {
            return (0, 0, 0);
        }

        // Try to repay earmarked debt if it exists
        uint256 repaidAmountInYield = 0;
        if (account.earmarked > 0) {
@>            repaidAmountInYield = _forceRepay(accountId, account.earmarked);
        }
        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        if (account.debt == 0) {
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }

        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            // Do actual liquidation
@>            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Since only a repayment happened, send repayment fee to caller
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
    }

    // AlchemistV3::_resolveRepaymentFee()
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;
@>1        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;
    }

    // AlchemistV3::_doLiquidation()
    function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        Account storage account = _accounts[accountId];

        (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
            collateralInUnderlying,
            account.debt,
            minimumCollateralization,
@>2            normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,
            globalMinimumCollateralization,
            liquidatorFee
        );

        amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
@>1        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }

        // Handle outsourced fee from vault
        if (outsourcedFee > 0) {
            uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
            if (vaultBalance > 0) {
                uint256 feeBonus = normalizeDebtTokensToUnderlying(outsourcedFee);
                feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
                IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
            }
        }

        emit Liquidated(accountId, msg.sender, amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
        return (amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
    }
    // AlchemistV3::_forceRepay(
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
@>1        account.collateralBalance -= creditToYield;

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

        if (account.collateralBalance > protocolFeeTotal) {
@>1            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
        return creditToYield;
    }
    // AlchemistV3::calculateLiquidation()
    function calculateLiquidation(
        uint256 collateral,
        uint256 debt,
        uint256 targetCollateralization,
        uint256 alchemistCurrentCollateralization,
        uint256 alchemistMinimumCollateralization,
        uint256 feeBps
    ) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee, uint256 outsourcedFee) {
        if (debt >= collateral) {
            outsourcedFee = (debt * feeBps) / BPS;
            // fully liquidate debt if debt is greater than collateral
            return (collateral, debt, 0, outsourcedFee);
        }

@>2        if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
            outsourcedFee = (debt * feeBps) / BPS;
            // fully liquidate debt in high ltv global environment
            return (debt, debt, 0, outsourcedFee);
        }

        // SNIP...
    }
```

## Impact Details

* Accounting Inconsistency & Reporting Errors: Global underlying / share totals reported by the contract diverge from actual vault state. Any logic depending on these values (liquidation sizing, minting limits, withdrawals, fees) is affected.
* Liquidation Failure: Overstated global collateralization may prevent protocol-required liquidations, leaving risky positions uncleared and increasing systemic risk.
* Incorrect Yield/Share Allocation: Mis-tracked global totals lead to incorrect conversion between yield tokens and underlying assets, producing wrong user balances on withdraw/mint.
* Availability / Operational Risk (DoS-like): Operations that rely on correct global totals (rebalances, automatic liquidations, withdrawals) can fail or require manual remediations.
* Severity: Medium–High depending on protocol exposure; the issue does not directly enable theft but undermines core safety mechanisms (liquidation & accounting).

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1244-L1291>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852-L894>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L900-L907>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738-L782>

## Proof of Concept

## Proof of Concept

Add the following test to `src/test/AlchemistV3.t.sol` and run it:

```js
    function testLiquidate_mytSharesDeposited() external {

        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

 
        ////////////////////////////////
        //  0xbeef deposit() + mint() //
        ////////////////////////////////
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

    
        //////////////////////////////
        //  user deposit() + mint() //
        //////////////////////////////
        address user = makeAddr("user");
        vm.prank(address(0xdead));
        whitelist.add(user);

        deal(address(vault), user, accountFunds);
        vm.startPrank(user);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        
        alchemist.deposit(depositAmount, user, 0);
        
        uint256 tokenIdForUser = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        alchemist.mint(tokenIdForUser, mintAmount, user);
        vm.stopPrank();

        // Need to start a transmutator deposit, to start earmarking debt
        deal(address(alToken), address(anotherExternalUser), accountFunds * 2);
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount * 2);
        transmuterLogic.createRedemption(mintAmount * 2);
        vm.stopPrank();

        uint256 mytSharesDeposited = alchemist.convertUnderlyingTokensToYield(alchemist.getTotalUnderlyingValue());

        vm.assertApproxEqAbs(depositAmount * 2, mytSharesDeposited, 1);


        assertEq(depositAmount * 2, vault.balanceOf(address(alchemist)));

        vm.roll(block.number + (5_256_000 * 5 / 100));

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // decreasing yeild token suppy by 50%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 5000 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        

        // Add the collateral amount to ensure debt < collateral
        deal(address(vault), address(0xbeef), accountFunds);
        deal(address(vault), user, accountFunds);

        vm.startPrank(user);
        SafeERC20.safeApprove(address(vault), address(alchemist), 15555555555555555637111 * 5 + 100e18);
        alchemist.deposit(15555555555555555637111 * 5, user, tokenIdForUser);
        vm.stopPrank();
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), 15555555555555555637111 * 5 + 100e18);
        alchemist.deposit(15555555555555555637111 * 5, address(0xbeef), tokenIdFor0xBeef);
        vm.stopPrank();

        // Account status is exactly the same
        (uint256 collateral, uint256 debt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        (uint256 userCollateral, uint256 userDebt, uint256 userEarmarked) = alchemist.getCDP(tokenIdForUser);

        assertEq(collateral, userCollateral);
        assertEq(debt, userDebt);
        assertEq(userEarmarked, earmarked);
        ///////////////// 
        //  liquidate  //
        ///////////////// 
        // alchemistCurrentCollateralization < globalMinimumCollateralization
        uint256 alchemistCurrentCollateralization = alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * 1e18 / alchemist.totalDebt();
        assertLt(alchemistCurrentCollateralization, alchemist.globalMinimumCollateralization());

        // liquidate tokenIdFor0xBeef
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();
        // Liquidation results in normal state
        assertEq(270000000000000000296998, assets);
        assertEq(0, feeInYield);
        assertEq(5130000000000000000513, feeInUnderlying);
        // The first liquidation ended, but because _mytSharesDeposited failed to be updated correctly, the current mortgage rate is much greater than the actual mortgage rate.
        // alchemistCurrentCollateralization > globalMinimumCollateralization ❌
        // The liquidation results are completely different
        alchemistCurrentCollateralization = alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * 1e18 / alchemist.totalDebt();
        assertGt(alchemistCurrentCollateralization, alchemist.globalMinimumCollateralization());

        
        // liquidate tokenIdForUser
        (uint256 userAssets, uint256 userFeeInYield, uint256 userFeeInUnderlying) = alchemist.liquidate(tokenIdForUser);
        // Liquidation not performed as expected
        assertEq(202333333333333332598641, userAssets);  
        // Liquidator fees were much lower than expected
        assertEq(233333333333333336655, userFeeInYield);
        assertEq(0, userFeeInUnderlying);
        assertLt(userFeeInYield + userFeeInUnderlying, feeInYield + feeInUnderlying);
        vm.stopPrank();
 
        // The current contract deposit amount is only 83222222222222223475471, but the return value of `getTotalUnderlyingValue()` is wrong ❌
        mytSharesDeposited = alchemist.convertUnderlyingTokensToYield(alchemist.getTotalUnderlyingValue());

        vm.assertApproxEqAbs(555555555555555556371110, mytSharesDeposited, 1);

        assertEq(83222222222222223475471, vault.balanceOf(address(alchemist)));
    }

```
