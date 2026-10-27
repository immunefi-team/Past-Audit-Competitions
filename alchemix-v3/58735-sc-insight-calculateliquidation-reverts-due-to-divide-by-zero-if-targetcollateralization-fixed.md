# 58735 sc insight calculateliquidation reverts due to divide by zero if targetcollateralization fixed point scalar&#x20;

**Submitted on Nov 4th 2025 at 10:32:59 UTC by @randomnpc for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58735
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`calculateLiquidation` reverts due to divide by `ZERO` if `targetCollateralization = FIXED_POINT_SCALAR` .

## Vulnerability Details

In the `calculateLiquidation()` function, `denom` is calculated as:

```solidity
    function calculateLiquidation(
        uint256 collateral,
        uint256 debt,
        uint256 targetCollateralization,
        uint256 alchemistCurrentCollateralization,
        uint256 alchemistMinimumCollateralization,
        uint256 feeBps
    ) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee, uint256 outsourcedFee) {

...

        // denom = m - 1  =>  (targetCollateralization - FIXED_POINT_SCALAR)/FIXED_POINT_SCALAR
        uint256 denom = targetCollateralization - FIXED_POINT_SCALAR;

        // debtToBurn = (num * FIXED_POINT_SCALAR) / denom
        debtToBurn = (num * FIXED_POINT_SCALAR) / denom;
...

```

If `targetCollateralization = FIXED_POINT_SCALAR`, `denom == 0`. And `targetCollateralization` can be set to `FIXED_POINT_SCALAR`.

```solidity
    /// @inheritdoc IAlchemistV3AdminActions
    function setMinimumCollateralization(uint256 value) external onlyAdmin {
        _checkArgument(value >= FIXED_POINT_SCALAR);
        minimumCollateralization = value;

        emit MinimumCollateralizationUpdated(value);
    }
```

Thus, if `denom == 0`, while calculating `debtToBurn`, divide by `ZERO` occurs:

```solidity
        debtToBurn = (num * FIXED_POINT_SCALAR) / denom; @audit can be divide by zero
```

## Impact Details

When `targetCollateralization == minimumCollateralization == FIXED_POINT_SCALAR`, all of the liquidations will `revert`. `undercollateralized` positions can't be liquidated. Bad debt will start accumulating and the protocol `solvency` will be at risk.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1284-L1287>

## Proof of Concept

## Proof of Concept

```solidity
function test_PocDivideByZeroInCalculateLiquidation() external {
 vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    // Set minimumCollateralization to exactly 1e18 (100%)
    vm.prank(alOwner);
    alchemist.setMinimumCollateralization(1e18);

    uint256 depositAmount = 1000e18;

    // Create large healthy position to ensure global collateralization is good
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 100);
    alchemist.deposit(depositAmount * 50, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // Create position where collateral is slightly above debt but not enough after fee
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    
    // Mint debt that is close to collateral value
    uint256 debtAmount = 980e18;
    alchemist.mint(tokenId, debtAmount, address(0xbeef));
    vm.stopPrank();

    // Small price drop to make collateral slightly less but still > debt
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    uint256 modifiedVaultSupply = (initialVaultSupply * 50 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    // Trigger actual liquidation which internally calls calculateLiquidation
    vm.startPrank(externalUser);
    vm.expectRevert();
    alchemist.liquidate(tokenId);
    vm.stopPrank();
}
```

Paste the above code in `AlchemistV3.t.sol` file, setup `MAINNET_RPC_URL` and run it using:

```solidity
FOUNDRY_PROFILE=default forge test --fork-url $MAINNET_RPC_URL --match-path src/test/AlchemistV3.t.sol --match-test test_PocDivideByZeroInCalculateLiquidation -vv --evm-version cancun
```
