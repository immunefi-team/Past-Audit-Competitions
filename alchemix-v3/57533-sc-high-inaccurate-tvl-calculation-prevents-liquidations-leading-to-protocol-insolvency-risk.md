# 57533 sc high inaccurate tvl calculation prevents liquidations leading to protocol insolvency risk

**Submitted on Oct 27th 2025 at 01:20:40 UTC by @Impala53732 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57533
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The \_mytSharesDeposited state variable, which is intended to track the total amount of yield-bearing tokens deposited by users, is not correctly updated during liquidations and other forced repayment scenarios. This leads to an inflated value for the protocol's Total Value Locked (TVL). This incorrect TVL is used to calculate the protocol's overall health (alchemistCurrentCollateralization). A falsely high collateralization ratio can prevent the liquidation of undercollateralized positions, creating bad debt and putting the entire protocol at risk of insolvency.

## Vulnerability Details

The core of the issue lies in the `_getTotalUnderlyingValue()` function, which relies on `_mytSharesDeposited` to determine the total value of assets held by the Alchemist.

```solidity
// filepath: /home/joe/sm-sec/v3-poc/src/AlchemistV3.sol
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

Several functions that transfer yield tokens (`myt`) out of the contract fail to decrement `_mytSharesDeposited`:

1. `_doLiquidation()`: When a position is liquidated, collateral is seized and transferred to the transmuter and the liquidator. The `account.collateralBalance` is reduced, but `_mytSharesDeposited` is not.
2. `_forceRepay()`: This internal function, called during the liquidation process, repays debt using the account's collateral. It reduces `account.collateralBalance` and transfers tokens but does not update `_mytSharesDeposited`.
3. `_resolveRepaymentFee()`: When a repayment fee is paid to a liquidator, tokens are transferred out, but again, `_mytSharesDeposited` is not decremented.

This discrepancy causes `_mytSharesDeposited` to perpetually increase or stay level, never decreasing on liquidations, even as the actual balance of `myt` held by the contract goes down.

## Impact Details

The inflated TVL directly impacts the `calculateLiquidation` logic. The `alchemistCurrentCollateralization` is calculated in `_doLiquidation` and passed to this function.

```solidity
// filepath: /home/joe/sm-sec/v3-poc/src/AlchemistV3.sol
// ...
(uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
    collateralInUnderlying,
    account.debt,
    minimumCollateralization,
    normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt, // <-- Inflated value used here
    globalMinimumCollateralization,
    liquidatorFee
);
// ...
```

If the protocol's actual health is poor but the calculated `alchemistCurrentCollateralization` remains above `alchemistMinimumCollateralization` due to the bug, the less aggressive liquidation logic is used. This can result in liquidations failing to execute (`grossCollateralToSeize` being 0) on positions that are genuinely undercollateralized and should be liquidated to protect the protocol. This allows bad debt to accumulate, threatening the solvency of the system.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L858-L865>

## Proof of Concept

## Proof of Concept

This runnable PoC test demonstrates that AlchemistV3 fails to decrement its internal MYT-share accounting when collateral is removed during forced repayments / liquidations — causing an inflated reported TVL and an overstated protocol collateralization. The logs shows the discrepency between the MYT shares deposited and the actual MYT balance in the Alchemixv3 contract.

add the following test to Alchemixv3.t.sol and run `forge test --mt test_checkPoc -vvv` to view the logs

```solidity
function test_checkPoc() external {
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
        uint256 alchemistCurrentCollateralization =
            alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();
    
        // Account is still collateralized, so not pulling from the fee vault for underlying
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);
        vm.stopPrank(); 

        //Repeat the same action to check the discrepancies in underlying value

        uint256 mytSharesAfterLiquidation = alchemist.getTotalDeposited(); // This is the correct, actual balance.
        uint256 totalUnderlyingAfterLiquidation = alchemist.getTotalUnderlyingValue(); // This uses the buggy _mytSharesDeposited.

        console.log("Actual MYT shares in contract after liquidation (correct):", mytSharesAfterLiquidation);
        console.log("Reported Total Underlying Value after liquidation (INCORRECT):", totalUnderlyingAfterLiquidation);
    }

```
