# 57725 sc high alchemistv liquidate is not updating the mytsharesdeposited which makes it inflated and can cause deposits dos and liquidations malfunction that may cause protocol insolvency&#x20;

**Submitted on Oct 28th 2025 at 13:43:58 UTC by @luc1jan for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57725
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Users can't deposit funds (DoS)
  * Potential insolvency

## Description

## Brief/Intro

Liquidations freeze and transfer user's collateral from contract, but lack `_mytSharesDeposited` update which makes the state variable inflated and will cause deposit DoS along with potential malfunctions in liquidations that can lead to insolvency.

## Vulnerability Details

Liquidations reduce liquidated collateral (yield tokens) and transfer it from the contract to `Transmuter` and fees recipients. However, the state variable `_mytSharesDeposited` which represents global deposited amount of yield tokens is not updated. With every liquidation, it becomes more and more inflated. This will effectively cause deposit DoS because inflated `_mytSharesDeposited` will reach `depositCap` and line [369](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L369) will revert. Additionally the protocol underlying value will be inflated because it uses `_mytSharesDeposited` as a source of truth. This will cause protocol to appear healthy even if it's globally undercollateralized, leading to eventual insolvency.

## Impact Details

During the lifecycle of the protocol, it is almost certain we will reach `depositCap` because of inflated `_mytSharesDeposited`. The check on line [369](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L369) will revert causing deposit DoS:

```js
    _checkState(_mytSharesDeposited + amount <= depositCap);
```

Additionally, the `_getTotalUnderlyingValue` will return inflated amount because it's using `_mytSharesDeposited`. This will make `alchemistCurrentCollateralization` argument inside `calculateLiquidation` inflated as well, causing this check to always return false even in cases when protocol is unhealthy:

```js
if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
    outsourcedFee = (debt * feeBps) / BPS;
    // fully liquidate debt in high ltv global environment
    return (debt, debt, 0, outsourcedFee);
}
```

Global undercollateralization will go unnoticed leading to even deeper undercollateralization and eventually to insolvency.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L369>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L862>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1244-L1262>

## Proof of Concept

## Proof of Concept

```js
    function test_mytSharesInflationPOC() external {
        // 1. create a position and then adjust the price so it becomes liquidatable
        address user = address(0xbeef);
        uint256 _depositAmount = 100_000e18;
        
        // Setup whale with yield tokens
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();
        
        // Create a position that's at the edge of liquidation
        vm.startPrank(user);
        SafeERC20.safeApprove(address(vault), address(alchemist), _depositAmount + 100e18);
        alchemist.deposit(_depositAmount, user, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        // Mint at exactly the minimum collateralization ratio
        alchemist.mint(tokenId, alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization, user);
        vm.stopPrank();
        
        // Make position liquidatable by decreasing yield token price (increasing supply)
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // Increase yield token supply by 6% to make position undercollateralized
        uint256 modifiedVaultSupply = (initialVaultSupply * 600 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        
        // 2. log current mytSharesDeposited and myt contract balance
        uint256 mytContractBalanceBefore = alchemist.getTotalDeposited();
        uint256 mytAccountingValueBefore = alchemist.getTotalUnderlyingValue();
        uint256 mytAccountingValueBeforeInYield = alchemist.convertUnderlyingTokensToYield(mytAccountingValueBefore);

        emit log_string("Before Liquidation:");
        emit log_named_uint("MYT contract balance", mytContractBalanceBefore / 1e16);
        emit log_named_uint("_mytSharesDeposited", mytAccountingValueBeforeInYield / 1e16);

        // 3. liquidate the position
        vm.startPrank(externalUser);
        (uint256 liquidationAmount, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        vm.stopPrank();

        // 4. log balances
        uint256 mytContractBalanceAfter = alchemist.getTotalDeposited();
        uint256 mytAccountingValueAfter = alchemist.getTotalUnderlyingValue();
        uint256 mytAccountingValueAfterInYield = alchemist.convertUnderlyingTokensToYield(mytAccountingValueAfter);

        emit log_string("After Liquidation:");
        emit log_named_uint("MYT contract balance", mytContractBalanceAfter / 1e16);
        emit log_named_uint("_mytSharesDeposited", mytAccountingValueAfterInYield / 1e16);
    }
```

```
Logs:
  Before Liquidation:
  MYT contract balance: 10000000
  _mytSharesDeposited: 9999999
  After Liquidation:
  MYT contract balance: 459999
  _mytSharesDeposited: 9999999
```

The PoC demonstrated that \_mytSharesDeposited is not reduced in liquidations which means inflation will accrue indefinitely with each liquidation.
