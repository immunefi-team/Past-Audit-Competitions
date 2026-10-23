# 56363 sc high mytsharesdeposited not correctly updated in all cases leading to incorrect protocol collateralization and reduced liquidation incentives

**Submitted on Oct 15th 2025 at 04:19:12 UTC by @silencedogood for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56363
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`_mytSharesDeposited` is not updated in all cases correctly in `AlchemistV3`, when MYT funds are transferred out. This leads to an overestimation of the current protocol collateralization, as well as reduced liquidation fees.

## Vulnerability Details

`_mytSharesDeposited` is intended to differentiate between tokens deposited into a CDP and balance of the contract. However during functions `_forceRepay()`, `_liquidate()` and `_doLiquidation()`, MYT tokens may be transferred out of the contract but not accounted for by reducing `_mytSharesDeposited` accordingly. This leads to `_mytSharesDeposited` being artificially high.

An artificially high `_mytSharesDeposited` value leads to an overestimation of the protocol's current collateralization and an artificially high value for `getTotalUnderlyingValue()`. In partial liquidations and in cases where `alchemistCurrentCollateralization` is slightly higher than `alchemistMinimumCollateralizatio`, this overestimation can lead to liquidations incorrectly skipping setting `outsourcedFee` to non-zero. Thus leading to a reduce fee and incentive for partial liquidations:

```solidity
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


        if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
            outsourcedFee = (debt * feeBps) / BPS;
            // fully liquidate debt in high ltv global environment
            return (debt, debt, 0, outsourcedFee);
        }


        // fee is taken from surplus = collateral - debt
        uint256 surplus = collateral > debt ? collateral - debt : 0;


        fee = (surplus * feeBps) / BPS;


        // collateral remaining for margin‐restore calc
        uint256 adjCollat = collateral - fee;


        // compute m*d  (both plain units)
        uint256 md = (targetCollateralization * debt) / FIXED_POINT_SCALAR;


        // if md <= adjCollat, nothing to liquidate
        if (md <= adjCollat) {
            return (0, 0, fee, 0);
        }


        // numerator = md - adjCollat
        uint256 num = md - adjCollat;


        // denom = m - 1  =>  (targetCollateralization - FIXED_POINT_SCALAR)/FIXED_POINT_SCALAR
        uint256 denom = targetCollateralization - FIXED_POINT_SCALAR;


        // debtToBurn = (num * FIXED_POINT_SCALAR) / denom
        debtToBurn = (num * FIXED_POINT_SCALAR) / denom;


        // gross collateral seize = net + fee
        grossCollateralToSeize = debtToBurn + fee;
    }
```

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L1244-L1291>

## Impact Details

Overestimating protocol MYT shares leads to an overestimation of protocol health, which is systemically dangerous. Furthermore, during partial liquidations, this overestimation can lead to the liquidator's `outsourcedFee` being skipped entirely, leading to loss of incentive for liquidators to act during high global LTV situations.

In my following PoC, two users deposit the same amount of tokens, but user 2 is fully liquidated, leading to:

1. No `outsourcedFee` for partial liquidation of user 1, when there should have been
2. Reported protocol current collateralization appearing 2x bigger than it should, because the full liquidation transferred collateral tokens out but didn't reduce tracked balance. Thus protocol collateral massively overstated

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L1244-L1291>

## Proof of Concept

## Proof of Concept

Paste the following unit test into `src/test/AlchemistV3.t.sol` and run:

```solidity
    function testLiquidate_mytSharesDeposited() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensuring global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        // Open debt position for yetAnotherExternalUser to ensure totalDebt after full liquidation > 0
        uint256 tokenIdForYAEU = AlchemistNFTHelper.getFirstTokenId(yetAnotherExternalUser, address(alchemistNFT));
        alchemist.mint(tokenIdForYAEU, alchemist.totalValue(tokenIdForYAEU)/10 * FIXED_POINT_SCALAR / minimumCollateralization, yetAnotherExternalUser);    // Don't want user to be fully liquidatable
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
        (, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 1200 bps or 12%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 1200 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

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
        uint256 expectedFeeInUnderlying = expectedDebtToBurn * liquidatorFeeBPS / 10_000;

        uint256 mytBefore = IERC20(address(vault)).balanceOf(address(alchemist)); 
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);
        vm.stopPrank();

        // ensure debt is reduced by the result of (collateral - y)/(debt - y) = minimum collateral ratio
        vm.assertApproxEqAbs(debt, 0, minimumDepositOrWithdrawalLoss);

        // ensure depositedCollateral is reduced by the result of (collateral - y)/(debt - y) = minimum collateral ratio
        vm.assertApproxEqAbs(depositedCollateral, 0, minimumDepositOrWithdrawalLoss);

        // ensure assets liquidated is equal (collateral - (90% of collateral))
        // vm.assertApproxEqAbs(assets, expectedLiquidationAmountInYield, minimumDepositOrWithdrawalLoss);

        // ensure liquidator fee is correct (3% of 0 if collateral fully liquidated as a result of bad debt)
        vm.assertApproxEqAbs(feeInYield, 0, 1e18);
        vm.assertEq(feeInUnderlying, expectedFeeInUnderlying);

        // liquidator gets correct amount of fee
        _validateLiquidiatorState(
            externalUser, liquidatorPrevTokenBalance, liquidatorPrevUnderlyingBalance, feeInYield, feeInUnderlying, assets, expectedLiquidationAmountInYield
        );

        vm.assertEq(alchemistFeeVault.totalDeposits(), 10_000 ether - feeInUnderlying);

        // transmuter recieves the liquidation amount in yield token minus the fee
        vm.assertApproxEqAbs(
            IERC20(address(vault)).balanceOf(address(transmuterLogic)),
            transmuterPreviousBalance + expectedLiquidationAmountInYield - expectedBaseFeeInYield,
            1e18
        );

        uint256 mytAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        console.log("mytBefore", mytBefore);
        console.log("mytAfter", mytAfter);
        assertLt(mytAfter, mytBefore, "MYT should decrease after liquidation");

        // compute reported vs true global CR AFTER liquidation
        // reportedCR uses _mytSharesDeposited internally via getTotalUnderlyingValue()
        uint256 reportedTVLU = alchemist.getTotalUnderlyingValue(); // uses _mytSharesDeposited  
        uint256 reportedCR = alchemist.normalizeUnderlyingTokensToDebt(reportedTVLU) * FIXED_POINT_SCALAR / alchemist.totalDebt(); 

        // trueCR recomputes using the ACTUAL MYT balance held by the contract
        uint256 trueTVLU = vault.convertToAssets(mytAfter); // 4626 assets for actual shares in contract  
        uint256 trueCR = alchemist.normalizeUnderlyingTokensToDebt(trueTVLU) * FIXED_POINT_SCALAR / alchemist.totalDebt(); 

        console.log("reportedCR", reportedCR);
        console.log("trueCR", trueCR);
        assertGt(reportedCR, trueCR, "reported global CR should be inflated if _mytSharesDeposited not decremented"); 

        // Show how B's liquidation math changes (outsourced fee suppressed under inflated CR) 
        (, uint256 debtYAEU,) = alchemist.getCDP(tokenIdForYAEU); 
        uint256 collateralBInDebt = alchemist.totalValue(tokenIdForYAEU);
        require(collateralBInDebt > debtYAEU, "Must not be under-water");

        // Using reportedCR (what protocol will use)
        (,, uint256 baseFeeRep, uint256 outsourcedRep) = alchemist.calculateLiquidation( 
            alchemist.totalValue(tokenIdForYAEU),
            debtYAEU,
            alchemist.minimumCollateralization(),
            reportedCR,
            alchemist.globalMinimumCollateralization(),
            liquidatorFeeBPS
        );

        // Using trueCR (what it actually is)
        (,, uint256 baseFeeTrue, uint256 outsourcedTrue) = alchemist.calculateLiquidation( 
            alchemist.totalValue(tokenIdForYAEU),
            debtYAEU,
            alchemist.minimumCollateralization(),
            trueCR,
            alchemist.globalMinimumCollateralization(),
            liquidatorFeeBPS
        );

        // Expect outsourcedRep <= outsourcedTrue; often 0 vs >0 depending on params
        assertLe(outsourcedRep, outsourcedTrue, "outsourced fee suppressed under inflated (reported) CR"); 
    }
```
