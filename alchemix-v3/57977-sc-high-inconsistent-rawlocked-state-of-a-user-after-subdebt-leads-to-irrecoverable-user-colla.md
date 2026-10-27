# 57977 sc high inconsistent rawlocked state of a user after subdebt leads to irrecoverable user collateral loss

**Submitted on Oct 29th 2025 at 18:43:24 UTC by @damdam0249 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57977
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A logic flaw exists in the collateral management system where a user's locked collateral (`rawLocked`) is not properly recalculated after their debt is fully extinguished via liquidation/subdebt. This state inconsistency leads to the user being incorrectly charged collateral during subsequent synchronization events, resulting in a direct, permanent loss of user funds.

## Vulnerability Details

The root cause is a missing state synchronization between a user's debt and their locked collateral. The liquidation logic correctly caps the amount of collateral to free (`toFree`) to prevent underflows but fails to ensure that the `rawLocked` value is updated to reflect the new debt state. This can leave the accounting in an inconsistent state where `account.debt == 0` but `account.rawLocked > 0`.

The problematic flow occurs in two stages:

**Stage 1: Incomplete State Update during Liquidation** In the `liquidate()` function, the following code handles the debt and collateral update in the subdebt function:

```solidity
// For cases when someone above minimum LTV gets liquidated.
if (toFree > _totalLocked) {
    toFree = _totalLocked;
}

account.debt -= amount;
totalDebt -= amount;
_totalLocked -= toFree;
account.rawLocked = lockedCollateral - toFree; // @audit `rawLocked` not recalculated based on new debt
```

While this prevents underflows, it uses a delta-based update for `rawLocked` instead of recalculating it from the remaining debt. If the liquidation clears the user's debt (`account.debt` becomes 0), the `rawLocked` value should also be zero. However, the current logic leaves a non-zero `rawLocked` and a 0 total debt value.

**Stage 2: Incorrect Collateral Deduction during Synchronization** Later, when `_sync()` is called (e.g., after a redemption), it calculates collateral to remove based on the outdated `rawLocked` value:

```solidity
function _sync(uint256 tokenId) internal {
    Account storage account = _accounts[tokenId];

    uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(
        account.rawLocked, // @audit Uses outdated, non-zero value
        _collateralWeight - account.lastCollateralWeight
    );

    account.collateralBalance -= collateralToRemove; // @audit User unfairly loses collateral
}
```

Because `account.rawLocked` is non-zero despite the user having no debt, the `ScaleByWeightDelta` function returns a positive `collateralToRemove` value. This unfairly reduces the user's free `collateralBalance`.

## Impact Details

**Primary Impact:** Direct loss of user collateral. Users who have had their debt fully liquidated will have further collateral incorrectly deducted from their balance during the next sync operation.

**Secondary Impacts:** \* Protocol accounting inconsistency between total locked collateral and actual user debt. \* In severe cases, this could contribute to protocol insolvency by incorrectly removing collateral from the system that is not backed by any debt.

## References

## Proof of Concept

## Proof of Concept

1. User A has a position with `debt = 100` and `rawLocked = 120`.
2. A liquidation event clears User A's entire debt (`debt` becomes 0). The capped logic sets `rawLocked = 20` (for example), instead of recalculating it to 0.
3. Later, a transmuter redemption triggers a `_sync()` for User A's position.
4. `_sync()` calculates `collateralToRemove` based on `rawLocked = 20`.
5. User A's `collateralBalance` is decremented by `collateralToRemove`, even though they have no outstanding debt, resulting in a permanent loss of funds.

```solidity
   
function testLiquidate_cause_Rawlockedbug() external {
    // Seed the system with whale liquidity
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    // User 0xbeef opens a position
    vm.startPrank(address(0xbeef));
    uint256 depositAmount = 5e18;
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);

    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
    vm.stopPrank();

    // Snapshot before manipulation
    (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("prevCollateral",prevCollateral);

    // Manipulate yield token supply to simulate undercollateralization
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    uint256 modifiedVaultSupply = initialVaultSupply + (initialVaultSupply * 590 / 10_000); // +5.9%
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    // Check new global collateralization
    uint256 currentCollat = alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();
    console.log("Global collateralization:", currentCollat);

    // Liquidate 0xbeef position
    vm.startPrank(externalUser);
    (uint256 liquidationAmount,,,) = alchemist.calculateLiquidation(
        alchemist.totalValue(tokenIdFor0xBeef),
        prevDebt,
        alchemist.minimumCollateralization(),
        currentCollat,
        alchemist.globalMinimumCollateralization(),
        liquidatorFeeBPS
    );

    alchemist.liquidate(tokenIdFor0xBeef);
    vm.stopPrank();

    // New user deposits to rebalance system collateralization
    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), 80e18);
    alchemist.deposit(40e18, anotherExternalUser, 0);

    uint256 tokenIdForAnother = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
    uint256 mintAmount2 = alchemist.totalValue(tokenIdForAnother) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdForAnother, mintAmount2, anotherExternalUser);
    vm.stopPrank();

    // Redemption process to reduce global bad debt
   vm.startPrank(address(anotherExternalUser));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount2);
    transmuterLogic.createRedemption(mintAmount2);
    vm.stopPrank();

    vm.roll(block.number + 1_000_000);

    vm.startPrank(address(anotherExternalUser));
    transmuterLogic.claimRedemption(1);
    vm.stopPrank();

    // Final poke to sync weights and verify post-liquidation state
    vm.startPrank(address(0xbeef));
    console.log("Collateral balance before poke:", alchemist.collateralbalancechecker(tokenIdFor0xBeef));
    alchemist.poke(tokenIdFor0xBeef);
    console.log("Collateral balance after:", alchemist.collateralbalancechecker(tokenIdFor0xBeef));
    vm.stopPrank();
}
```

Result of test

```solidity
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testLiquidate_cause_Rawlockedbug() (gas: 4195979)
Logs:
  prevCollateral 5000000000000000000
  Global collateralization: 1049207848074703597
  Collateral balance before poke: 234499999999999996
  Collateral balance after: 218978182508561639



```
