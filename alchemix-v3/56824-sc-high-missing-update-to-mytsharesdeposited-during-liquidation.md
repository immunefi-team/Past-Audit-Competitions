# 56824 sc high missing update to mytsharesdeposited during liquidation

**Submitted on Oct 21st 2025 at 00:42:56 UTC by @Josh4324 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56824
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The \_mytSharesDeposited variable serves as an aggregate counter for all MYT shares held by the contract on behalf of users. It is incremented during deposits and decremented in scenarios where MYT is transferred out, such as withdrawals, repayments, burns, and redemptions. This ensures that \_mytSharesDeposited accurately reflects the contract's MYT balance attributable to user positions.

However, in liquidation related functions specifically \_forceRepay and \_doLiquidation MYT tokens are transferred to external addresses (e.g., the transmuter, protocol fee receiver, or liquidator) without corresponding decrements to \_mytSharesDeposited. This omission causes the variable to become progressively overstated as liquidations occur, leading to discrepancies between the tracked value and the actual MYT balance.

## Vulnerability Details

The contract assumes that all MYT outflows are accounted for in primary user-facing functions, but overlooks internal helper functions used in liquidations.

Liquidations involve repaying debt using a position's collateral, which reduces the position's collateralBalance but does not adjust the global \_mytSharesDeposited.

In \_forceRepay:

```sol
// Repaid amount is deducted from account.collateralBalance
account.collateralBalance -= creditToYield;

// Protocol fee is also deducted if applicable
if (account.collateralBalance > protocolFeeTotal) {
    account.collateralBalance -= protocolFeeTotal;
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
}

// Transfer to transmuter
if (creditToYield > 0) {
    TokenUtils.safeTransfer(myt, transmuter, creditToYield);
}

// Missing: _mytSharesDeposited -= (creditToYield + protocolFeeTotal);
```

In \_doLiquidation:

```sol
// Deduct from position's collateral
account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;

// Transfers out
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
}

// Missing: _mytSharesDeposited -= amountLiquidated;
```

The contract relies on accurate global collateralization ratios for critical decisions, such as in calculateLiquidation. This function computes alchemistCurrentCollateralization as:

```sol
normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt
```

Here, an inflated \_getTotalUnderlyingValue() (due to overstated \_mytSharesDeposited) makes the protocol appear more collateralized than it truly is (e.g., reporting a ratio above globalMinimumCollateralization when it's actually below).

## Impact Details

1. The function \_getTotalUnderlyingValue() reports a higher total value than actual, as it relies on the overstated \_mytSharesDeposited
2. Inflated global collateralization ratios mask undercollateralization, delaying or preventing full liquidations. .
3. Users redeeming via the transmuter may receive less value than expected due to insufficient MYT backing, as the system appears healthier than it is
4. In stressed scenarios (e.g., market downturns), unchecked bad debt can cause liabilities to exceed assets, making the protocol unable to honor redemptions, repayments, or synthetic burns.
5. New deposits fail prematurely against the depositCap due to the inflated tracker, reducing liquidity inflows and hindering protocol growth.
6. Critical checks in calculateLiquidation (e.g., global collateralization) become inaccurate, leading to incomplete liquidations and prolonged exposure to risky positions.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L549>

## Proof of Concept

## Proof of Concept

Copy test and paste into src/test/AlchemistV3.t.sol Run forge test --mt testLiquidate\_Undercollateralized\_PositionJosh -vv

```sol
function testLiquidate_Undercollateralized_PositionJosh() external {
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

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Total underlying value after liquidation
        uint256 total1 = alchemist.getTotalUnderlyingValue();
        console.log("before", total1);

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorPrevUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));

        alchemist.liquidate(tokenIdFor0xBeef);

        vm.stopPrank();

        // Total underlying value after liquidation
        uint256 total2 = alchemist.getTotalUnderlyingValue();
        console.log("after", total2);
    }
```

Result

Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test

\[PASS] testLiquidate\_Undercollateralized\_PositionJosh() (gas: 1411926)

Logs:

before 377714825306893295200000

after 377714825306893295200000
