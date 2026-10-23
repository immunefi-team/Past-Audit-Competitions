# 56628 sc high liquidate does not update mytsharesdeposited that is reduced by fees

**Submitted on Oct 18th 2025 at 15:52:45 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56628
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency

## Description

## Brief/Intro

at `_liquidate` the MYT would get sent into `transmuter`, `protocolFeeReceiver` and the `caller` depending on the case. however the transfer into `protocolFeeReceiver` and `caller` does not update `_mytSharesDeposited` making each time earmark used for repayment inside `_liquidate` it would reduce the actual MYT holding of Alchemist contract but not reflected in `_mytSharesDeposited`

## Vulnerability Details

in the function `_liquidate` we can see that MYT that act as fee get sent outside the contract:

```solidity
        uint256 repaidAmountInYield = 0;
        if (account.earmarked > 0) {
@>          repaidAmountInYield = _forceRepay(accountId, account.earmarked);
        }
        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        if (account.debt == 0) {
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
@>          TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }

        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            // Do actual liquidation
            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Since only a repayment happened, send repayment fee to caller
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
@>          TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
```

there are no issue if the receiver is `Transmuter` contract as it would correctly handle the `_mytSharesDeposited` at later via redeem.

the issue is that sending MYT as `feeInYield` and as part of protocol fee inside `_forceRepay` . as we can see there are no `_mytSharesDeposited` update after the transfer is done.

## Impact Details

given how `_mytSharesDeposited` is used in `_getTotalUnderlyingValue` which is also used in `_doLiquidation` function, over course of many liquidation happening:

1. `calculateLiquidation` would overestimate the `alchemistCurrentCollateralization` potentially making the check if `alchemistCurrentCollateralization < alchemistMinimumCollateralization` returning false while it should be true. preventing the detection if the whole contract is undercollateralized and thus preventing full liquidation of any position.
2. failing to detect global undercollateralization would makes protocol prone to insolvency
3. inflated `_mytSharesDeposited` would cause deposit cap check an issue
4. an issue with `withdraw` function because of the unsynced shares deposited vs collateralBalance

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L819-L842>

## Proof of Concept

## Proof of Concept

add this test to `src/test/AlchemistV3.t.sol`. this test is duplicate of `testLiquidate_Undercollateralized_Position_With_Earmarked_Debt_Sufficient_Repayment_With_Protocol_Fee` , adding a few assert at the bottom of the test:

```solidity
    function testModify_Liquidate_Undercollateralized_Position_With_Earmarked_Debt_Sufficient_Repayment_With_Protocol_Fee() external {
        uint256 amount = 200_000e18; // 200,000 yvdai
        // uint256 protocolFee = 100; // 10%
        vm.prank(alOwner);
        alchemist.setProtocolFee(protocolFee);
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount * 2);
        alchemist.deposit(amount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;

        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Need to start a transmutator deposit, to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // skip to a future block. Lets say 60% of the way through the transmutation period (5_256_000 blocks)
        vm.roll(block.number + (5_256_000 * 60 / 100));

        // Earmarked debt should be 60% of the total debt
        (uint256 prevCollateral, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        require(earmarked == prevDebt * 60 / 100, "Earmarked debt should be 60% of the total debt");

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

        // snapshot alchemist MYT balance before liquidation
        uint256 alchemistMYTBalBefore = vault.balanceOf(address(alchemist));

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);

        uint256 credit = earmarked > prevDebt ? prevDebt : earmarked;
        uint256 creditToYield = alchemist.convertDebtTokensToYield(credit);
        uint256 protocolFeeInYield = (creditToYield * protocolFee / BPS);

        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorPrevUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);

        (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);

        uint256 repaymentFee = alchemist.convertDebtTokensToYield(earmarked) * 100 / BPS;

        vm.stopPrank();

        // ensure debt is reduced only by the repayment of max earmarked amount
        vm.assertApproxEqAbs(debt, prevDebt - earmarked, minimumDepositOrWithdrawalLoss);

        // ensure depositedCollateral is reduced only by the repayment of max earmarked amount
        vm.assertApproxEqAbs(
            depositedCollateral,
            prevCollateral - alchemist.convertDebtTokensToYield(earmarked) - protocolFeeInYield - repaymentFee,
            minimumDepositOrWithdrawalLoss
        );

        // ensure assets is equal to repayment of max earmarked amount
        // vm.assertApproxEqAbs(assets, alchemist.convertDebtTokensToYield(earmarked), minimumDepositOrWithdrawalLoss);

        // ensure liquidator fee is correct (i.e.0, since only a repayment is done)
        vm.assertApproxEqAbs(feeInYield, repaymentFee, 1e18);
        vm.assertEq(feeInUnderlying, 0);

        // liquidator gets correct amount of fee, i.e. 0
        _validateLiquidiatorState(
            externalUser,
            liquidatorPrevTokenBalance,
            liquidatorPrevUnderlyingBalance,
            feeInYield,
            feeInUnderlying,
            assets,
            alchemist.convertDebtTokensToYield(earmarked)
        );
        vm.assertEq(alchemistFeeVault.totalDeposits(), 10_000 ether);

        // transmuter recieves the liquidation amount in yield token minus the fee
        vm.assertApproxEqAbs(
            IERC20(address(vault)).balanceOf(address(transmuterLogic)), transmuterPreviousBalance + alchemist.convertDebtTokensToYield(earmarked), 1e18
        );

        // check protocolfeereciever received the protocl fee transfer from _forceRepay
        vm.assertApproxEqAbs(IERC20(address(vault)).balanceOf(address(protocolFeeReceiver)), protocolFeeInYield, 1e18);

        // check the total MYT alchemist contract have after the liquidation
        uint256 alchemistMYTBalAfter = vault.balanceOf(address(alchemist));
        uint256 amountMYTOut = alchemistMYTBalBefore - alchemistMYTBalAfter;
        // assert that amountMYTOut is the same as the amount sent to transmuter + fee
        vm.assertApproxEqAbs(amountMYTOut, alchemist.convertDebtTokensToYield(earmarked) + protocolFeeInYield + repaymentFee, 1e18);

        // check the state of totalUnderlyingValue and compared to actual MYT holding after liquidation repayment
        uint256 alchemistMYTToUnderlying = alchemist.convertYieldTokensToUnderlying(alchemistMYTBalAfter);
        uint256 alchemistTotalUnderlyingValue = alchemist.getTotalUnderlyingValue();
        vm.assertApproxEqAbs(alchemistMYTToUnderlying, alchemistTotalUnderlyingValue, 1e18, "after liquidation repayment, alchemist MYT holding does not match the _getTotalUnderlyingValue");
    }

```

run with `forge test --mt testModify_Liquidate_Undercollateralized_Position_With_Earmarked_Debt_Sufficient_Repayment_With_Protocol_Fee`

```bash
Failing tests:
Encountered 1 failing test in src/test/AlchemistV3.t.sol:AlchemistV3Test
[FAIL: after liquidation repayment, alchemist MYT holding does not match the _getTotalUnderlyingValue: 267554825306893295188984 !~= 377714825306893295200000 (max delta: 1000000000000000000, real delta: 110160000000000000011016)] testModify_Liquidate_Undercollateralized_Position_With_Earmarked_Debt_Sufficient_Repayment_With_Protocol_Fee() (gas: 3901034)

Encountered a total of 1 failing tests, 0 tests succeeded
```

this prove that the MYT to underlying value that is stored in contract state is inflated if compared to actual MYT to underlying value from contract balance
