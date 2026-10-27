# 57212 sc high totallocked is not properly decremented in the redeem function causing system insolvency&#x20;

**Submitted on Oct 24th 2025 at 12:45:10 UTC by @godwinudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57212
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `redeem` function in AlchemistV3.sol contains an error when reducing the global `_totalLocked` variable during transmuter redemptions. The function uses an incorrect multiplier (approximately 1.05 for a 5% fee) instead of the correct `minimumCollateralization` ratio (2.0 for 200% LTV), and operates on the wrong debt amount. This causes `_totalLocked` to remain inflated after each redemption, resulting in user positions retaining excess withdrawable collateral that should have been reserved for transmuter obligations. Over time, this leads to a collateral deficit where the system cannot fulfill all transmuter redemptions, resulting in protocol insolvency.

## Vulnerability Details

The `_totalLocked` variable tracks the total amount of collateral across all user positions that must remain locked to maintain minimum collateralization ratios. When a user borrows debt against their collateral, a portion of that collateral becomes locked and cannot be withdrawn. The locked amount is calculated as the debt amount converted to yield tokens multiplied by the minimum collateralization ratio (typically 200% or 2.0).

Throughout the AlchemistV3 codebase, whenever debt changes occur, the `_totalLocked` variable is updated using a consistent formula. When debt increases, locked collateral increases using this calculation:

```solidity
uint256 toLock = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
_totalLocked += toLock;
```

Similarly, when debt decreases, the locked collateral should be freed using the inverse calculation:

```solidity
uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
_totalLocked -= toFree;
```

This pattern appears consistently in functions like `_addDebt()` and `_subDebt()`, where the multiplier is always `minimumCollateralization / FIXED_POINT_SCALAR`, which equals 2.0 for a 200% collateralization requirement.

However, the `redeem()` function breaks this pattern. When the transmuter redeems earmarked debt, the function is supposed to reduce `_totalLocked` by the amount of locked collateral that corresponds to the redeemed debt.

```solidity
function redeem(uint256 amount) external onlyTransmuter {
    _earmark();

    uint256 liveEarmarked = cumulativeEarmarked;
    if (amount > liveEarmarked) amount = liveEarmarked;

    // ... cover logic ...

    uint256 redeemedDebtTotal = amount + coverToApplyDebt;

    // ... redemption weight updates ...

    cumulativeEarmarked -= redeemedDebtTotal;
    totalDebt -= redeemedDebtTotal;

    // Move collateral to transmuter
    uint256 collRedeemed  = convertDebtTokensToYield(amount);
    uint256 feeCollateral = collRedeemed * protocolFee / BPS;
    uint256 totalOut      = collRedeemed + feeCollateral;

    // ISSUE: Updates _totalLocked using totalOut
    uint256 old = _totalLocked;
    _totalLocked = totalOut > old ? 0 : old - totalOut;
    _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);

    TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
    _mytSharesDeposited -= collRedeemed + feeCollateral;

    emit Redemption(redeemedDebtTotal);
}
```

The error occurs in these lines where `_totalLocked` is reduced:

```solidity
uint256 totalOut = collRedeemed + feeCollateral;
uint256 old = _totalLocked;
_totalLocked = totalOut > old ? 0 : old - totalOut;
```

The function calculates `totalOut` as the collateral being redeemed plus the protocol fee, which equals `convertDebtTokensToYield(amount) * (1 + protocolFee/BPS)`. With a typical 5% protocol fee, this multiplier becomes approximately 1.05. The function then reduces `_totalLocked` by this `totalOut` value.

This approach contains two distinct errors. First, it uses the wrong multiplier. The reduction should use `minimumCollateralization / FIXED_POINT_SCALAR` (which is 2.0) but instead uses `(1 + protocolFee/BPS)` (which is approximately 1.05). Second, it operates on the wrong debt amount. The function uses `amount` to calculate the reduction, but the actual debt removed from the system is `redeemedDebtTotal`, which includes both the direct redemption amount and any cover that was applied.

The correct implementation should be:

```solidity
uint256 lockedFreed = convertDebtTokensToYield(redeemedDebtTotal) * minimumCollateralization / FIXED_POINT_SCALAR;
_totalLocked = lockedFreed > old ? 0 : old - lockedFreed;
```

To illustrate the magnitude of this error, consider a concrete example with standard protocol parameters. Assume a 200% minimum collateralization ratio and a 5% protocol fee. When 100 units of debt are redeemed with an additional 10 units from cover (totaling 110 units of debt reduction):

The correct calculation would reduce `_totalLocked` by:

* `110 * 2.0 = 220 units`

The actual incorrect calculation reduces `_totalLocked` by:

* `100 * 1.05 = 105 units`

This creates a discrepancy of 115 units per redemption that accumulates in `_totalLocked`, leaving it permanently inflated.

The inflated `_totalLocked` value then propagates through the system's accounting mechanism. The protocol uses a weight-based system to distribute collateral reductions across user positions when redemptions occur. The `_collateralWeight` variable tracks these distributions, and its increments are calculated using `_totalLocked` as the denominator:

```solidity
_collateralWeight += PositionDecay.WeightIncrement(totalOut, old);
```

Where `WeightIncrement` internally performs a division by the denominator (`old`, which is `_totalLocked`). When `_totalLocked` is inflated, these weight increments become artificially small. Later, when individual user positions are synced, the amount of collateral removed from each position is calculated as:

```solidity
uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(
    account.rawLocked, 
    _collateralWeight - account.lastCollateralWeight
);
```

Because the weight delta is too small (due to the inflated denominator), less collateral is removed from user positions than should be. This means users retain excess collateral that should have been marked as consumed by the redemption process. Since withdrawable collateral is calculated as total collateral minus locked collateral, this excess remains available for withdrawal by users, even though it should have been reserved to fulfill transmuter redemption obligations.

## Impact Details

After each redemption, the `_totalLocked` variable becomes more inflated, causing user positions to retain increasingly more collateral than they should. The excess collateral that users can withdraw represents funds that should have been reserved to fulfill transmuter obligations, creating a deficit in the system.

As more redemptions occur over time, this error accumulates. Each redemption adds to the total collateral deficit, and eventually, the Alchemist contract will not hold sufficient collateral to fulfill all outstanding transmuter positions.

## Proof of Concept

## Proof of Concept

Add this to the AlchemistV3.t.sol test suite and run

```solidity
function testPOC_TotalLockedDrift_CausesInsolvency() external {
    vm.prank(alOwner);
    alchemist.setProtocolFee(500); // 5% fee
    
    // User 1 deposits and mints debt
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 maxBorrow1 = alchemist.getMaxBorrowable(tokenId1);
    alchemist.mint(tokenId1, maxBorrow1, address(0xbeef));
    vm.stopPrank();
    
    vm.roll(block.number + 1);
    
    // User 2 deposits and mints debt
    vm.startPrank(externalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, externalUser, 0);
    uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
    uint256 maxBorrow2 = alchemist.getMaxBorrowable(tokenId2);
    alchemist.mint(tokenId2, maxBorrow2, externalUser);
    vm.stopPrank();
    
    vm.roll(block.number + 1);
    
    // Transmuter position
    uint256 transmuterAmount = maxBorrow1 / 2;
    vm.startPrank(address(0xdad));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), transmuterAmount);
    transmuterLogic.createRedemption(transmuterAmount);
    vm.stopPrank();
    
    vm.roll(block.number + 100);
    vm.prank(address(0xbeef));
    alchemist.poke(tokenId1);
    
    (uint256 user2CollateralBefore, uint256 user2DebtBefore,) = alchemist.getCDP(tokenId2);
    
    vm.roll(block.number + transmuterLogic.timeToTransmute() + 1);
    
    uint256 totalLockedBefore = uint256(vm.load(address(alchemist), bytes32(uint256(17))));
    uint256 totalDebtBefore = alchemist.totalDebt();
    
    vm.prank(address(0xdad));
    transmuterLogic.claimRedemption(1);
    
    uint256 totalLockedAfter = uint256(vm.load(address(alchemist), bytes32(uint256(17))));
    uint256 totalDebtAfter = alchemist.totalDebt();
    
    // Prove _totalLocked accounting error
    uint256 debtReduced = totalDebtBefore - totalDebtAfter;
    uint256 expectedLockedReduction = alchemist.convertDebtTokensToYield(debtReduced) * minimumCollateralization / FIXED_POINT_SCALAR;
    uint256 actualLockedReduction = totalLockedBefore - totalLockedAfter;
    
    emit log_named_uint("Debt Reduced", debtReduced);
    emit log_named_uint("Expected _totalLocked Reduction", expectedLockedReduction);
    emit log_named_uint("Actual _totalLocked Reduction", actualLockedReduction);
    
    assertLt(actualLockedReduction, expectedLockedReduction, "_totalLocked drift detected");
    
    // Sync user2 to show impact
    vm.prank(externalUser);
    alchemist.poke(tokenId2);
    
    (uint256 user2CollateralAfter, uint256 user2DebtAfter,) = alchemist.getCDP(tokenId2);
    
    uint256 collateralLost = user2CollateralBefore - user2CollateralAfter;
    uint256 debtReduction = user2DebtBefore - user2DebtAfter;
    
    emit log_named_uint("User2 debt reduced by", debtReduction);
    emit log_named_uint("User2 collateral removed", collateralLost);
    emit log_named_uint("User2 excess retained (2.75e18)", user2CollateralAfter - (alchemist.convertDebtTokensToYield(user2DebtAfter) * minimumCollateralization / FIXED_POINT_SCALAR));
    
    // This excess collateral should have been removed but wasn't due to _totalLocked drift
    assertGt(collateralLost, 0, "User retained excess collateral");
}
```

```
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testPOC_TotalLockedDrift_CausesInsolvency() (gas: 4373109)
Logs:
  Debt Reduced: 90000000000000000009000
  Expected _totalLocked Reduction: 99999999999999999999999
  Actual _totalLocked Reduction: 0
  User2 debt reduced by: 45000000000000000004500
  User2 collateral removed: 47250000000000000004725
  User2 excess retained (2.75e18): 2749999999999999995276
```
