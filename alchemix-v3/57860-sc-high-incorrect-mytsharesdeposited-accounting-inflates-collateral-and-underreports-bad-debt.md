# 57860 sc high incorrect mytsharesdeposited accounting inflates collateral and underreports bad debt enabling insolvency

**Submitted on Oct 29th 2025 at 09:15:51 UTC by @algiz for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57860
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`AlchemistV3` does not reduce `_mytSharesDeposited` when MYT collateral is seized in liquidation, paid as fees, or force-repaid and sent to the transmuter. This means the protocol continues to report that it holds collateral (in `getTotalUnderlyingValue()`) that it has already transferred out.

That corrupts two critical safety mechanisms:

1. Liquidation math, which decides whether to fully liquidate underwater debt
2. The transmuter’s `badDebtRatio`, which is used to reduce redemptions when the system has bad debt.

As a result, the system can under-liquidate underwater positions (allowing bad debt to persist) and at the same time overpay claimants from the transmuter relative to true reserves. This creates a path to protocol-wide insolvency.

## Vulnerability Details

The contract tracks the total MYT shares deposited across all CDPs in a global variable:

```solidity
    uint256 private _mytSharesDeposited;
```

This variable is increased or decreased during `deposit()` or when collateral is pulled out from any CDP (`withdraw()`, `burn()`, `repay()`, `redeem()`).

However, in several flows, MYT leaves the contract but `_mytSharesDeposited` is not decremented. This desynchronizes accounting and causes the system to believe it has more collateral than it really does.

Those flows are: `_forceRepay()`, `_resolveRepaymentFee()`, `_doLiquidation()`

After any liquidation or forced repayment, the actual MYT balance of `AlchemistV3` went down, but `_mytSharesDeposited` did not. Since `_getTotalUnderlyingValue()` is calculated from `_mytSharesDeposited`, the protocol’s own view of its TVL gets gradually inflated.

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
        uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
        totalUnderlyingValue = yieldTokenTVLInUnderlying;
    }
```

That incorrect value is used in:

1. Alchemist's full liquidation path, when debt should be fully liquidated in high LTV global environment

```solidity
        if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
            outsourcedFee = (debt * feeBps) / BPS;                     
            // fully liquidate debt in high ltv global environment
            return (debt, debt, 0, outsourcedFee);
        }
```

2. Calculating `badDebtRatio` during Transmuter's `claimRedemption()`, when scaling redemption amounts

```solidity
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;


        uint256 scaledTransmuted = amountTransmuted;


        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
```

## Impact Details

This incorrect value has impact in multiple areas:

1. Under-liquidation when in high TVL environment In case when the `alchemistCurrentCollateralization` is lower than `alchemistMinimumCollateralization`, the debt should be fully liquidated. However, due to inflated `_mytSharesDeposited`, position is only partially liquidated.
2. Transmuter's `badDebtRatio` is understated and users' redemptions are not scaled down as they should Economically, this means the Transmuter may continue honouring 1:1 exits from alAssets into MYT, when it should already be scaling down. That drains what little collateral remains and accelerates insolvency. Redemptions are supposed to be throttled when the system is undercollateralized, but instead they're paid out at close to full value because the undercollateralization is being hidden. This could further prevent future users to withdraw or redeem their collateral.

## References

* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1238-L1241>
* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L219-L226>

## Proof of Concept

## Proof of Concept

The existing `testLiquidate_Undercollateralized_Position` test has been extended to reflect the actual/reported `totalUnderlyingValue` before/after liquidations.

1. Add the following test inside `AlchemistV3.t.sol`:

```solidity
function testPoCLiquidate_Undercollateralized_Position() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // keep global collateralization above the global minimum so we use partial liquidation path
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        uint256 sharesBalance = IERC20(address(vault)).balanceOf(address(yetAnotherExternalUser));
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // victim position
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);

        uint256 tokenIdFor0xBeef =
            AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        // max borrow up to min collateralization
        alchemist.mint(
            tokenIdFor0xBeef,
            alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization,
            address(0xbeef)
        );
        vm.stopPrank();

        uint256 transmuterPreviousBalance =
            IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // capture pre-liquidation CDP + create undercollateralization by moving share price
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);

        // increase vault share supply ~5.9% without adding underlying -> PPS down
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // sanity: initial debt is as expected
        vm.assertApproxEqAbs(
            prevDebt,
            180_000_000_000_000_000_018_000,
            minimumDepositOrWithdrawalLoss
        );

        // --- SNAPSHOT GLOBAL STATE BEFORE LIQUIDATION ---

        // what protocol *reports* as total underlying (uses _mytSharesDeposited internally)
        uint256 reportedUnderlyingBefore = alchemist.getTotalUnderlyingValue();

        // what the protocol ACTUALLY has in shares sitting on the alchemist
        uint256 actualSharesBefore =
            IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 actualUnderlyingBefore =
            alchemist.convertYieldTokensToUnderlying(actualSharesBefore);

        // liquidator calls liquidate
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance =
            IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorPrevUnderlyingBalance =
            IERC20(vault.asset()).balanceOf(address(externalUser));

        uint256 alchemistCurrentCollateralization =
            alchemist.normalizeUnderlyingTokensToDebt(
                alchemist.getTotalUnderlyingValue()
            )
            * FIXED_POINT_SCALAR
            / alchemist.totalDebt();

        (uint256 liquidationAmount,
        uint256 expectedDebtToBurn,
        uint256 expectedBaseFee,) =
            alchemist.calculateLiquidation(
                alchemist.totalValue(tokenIdFor0xBeef),
                prevDebt,
                alchemist.minimumCollateralization(),
                alchemistCurrentCollateralization,
                alchemist.globalMinimumCollateralization(),
                liquidatorFeeBPS
            );

        uint256 expectedLiquidationAmountInYield =
            alchemist.convertDebtTokensToYield(liquidationAmount);
        uint256 expectedBaseFeeInYield =
            alchemist.convertDebtTokensToYield(expectedBaseFee);

        uint256 expectedFeeInUnderlying = 0;

        (uint256 assets,
        uint256 feeInYield,
        uint256 feeInUnderlying) =
            alchemist.liquidate(tokenIdFor0xBeef);

        (uint256 depositedCollateral,
        uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);

        vm.stopPrank();

        // --- SNAPSHOT GLOBAL STATE AFTER LIQUIDATION ---

        uint256 reportedUnderlyingAfter = alchemist.getTotalUnderlyingValue();

        uint256 actualSharesAfter =
            IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 actualUnderlyingAfter =
            alchemist.convertYieldTokensToUnderlying(actualSharesAfter);

        // --- EXISTING ASSERTIONS ---

        // borrower debt reduced by what liquidation said it would burn
        vm.assertApproxEqAbs(
            debt,
            prevDebt - expectedDebtToBurn,
            minimumDepositOrWithdrawalLoss
        );

        // borrower collateral shares reduced according to liquidation
        vm.assertApproxEqAbs(
            depositedCollateral,
            prevCollateral - expectedLiquidationAmountInYield,
            minimumDepositOrWithdrawalLoss
        );

        // liquidator fee matches expectation
        vm.assertApproxEqAbs(feeInYield, expectedBaseFeeInYield, 1e18);
        vm.assertEq(feeInUnderlying, expectedFeeInUnderlying);

        // liquidator paid correctly
        _validateLiquidiatorState(
            externalUser,
            liquidatorPrevTokenBalance,
            liquidatorPrevUnderlyingBalance,
            feeInYield,
            feeInUnderlying,
            assets,
            expectedLiquidationAmountInYield
        );

        vm.assertEq(
            alchemistFeeVault.totalDeposits(),
            10_000 ether - feeInUnderlying
        );

        // transmuter received the post-fee liquidation flow
        vm.assertApproxEqAbs(
            IERC20(address(vault)).balanceOf(address(transmuterLogic)),
            transmuterPreviousBalance
                + expectedLiquidationAmountInYield
                - expectedBaseFeeInYield,
            1e18
        );

        console.log("Actual TVL before liquidation:    ", actualUnderlyingBefore);
        console.log("Reported TVL before liquidation:  ", reportedUnderlyingBefore);
        
        console.log("============================================");
        console.log("Actual TVL after liquidation:     ", actualUnderlyingAfter);
        console.log("Reported TVL after liquidation:   ", reportedUnderlyingAfter);
    }
```

2. Execute the test with

```solidity
forge test --mt testPoCLiquidate_Undercollateralized_Position -vv
```

3. Output:

```solidity
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testPoCLiquidate_Undercollateralized_Position() (gas: 1656781)
Logs:
  Actual TVL before liquidation:      377714825306893295200000
  Reported TVL before liquidation:    377714825306893295200000
  ============================================
  Actual TVL after liquidation:      274774315391879129222735
  Reported TVL after liquidation:     377714825306893295200000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 22.22ms (4.51ms CPU time)

Ran 1 test suite in 23.51ms (22.22ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
