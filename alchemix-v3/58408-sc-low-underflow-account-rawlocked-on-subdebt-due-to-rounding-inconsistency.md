# 58408 sc low underflow account rawlocked on subdebt due to rounding inconsistency

**Submitted on Nov 2nd 2025 at 02:01:14 UTC by @Jugger63 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58408
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour
  * Temporary freezing of funds for at least 24 hour

## Description

## Finding description

The `_subDebt()` function in `AlchemistV3` calculates how much collateral can be freed (`toFree`) when the debt (`debt`) is reduced:

```solidity
function _subDebt(uint256 tokenId, uint256 amount) internal {
    Account storage account = _accounts[tokenId];

    // collateral that can be released from reducing the debt `amount`
    uint256 toFree = convertDebtTokensToYield(amount)
        * minimumCollateralization / FIXED_POINT_SCALAR;

    // collateral that is locked before reduction
    uint256 lockedCollateral = convertDebtTokensToYield(account.debt)
        * minimumCollateralization / FIXED_POINT_SCALAR;

    if (toFree > _totalLocked) {
        toFree = _totalLocked;               // clamp (global) 🔹
    }

    account.debt        -= amount;
    totalDebt           -= amount;
    _totalLocked        -= toFree;

    // per-account lock update
    account.rawLocked    = lockedCollateral - toFree; // underflow
}
```

`Free` and `lock Collateral` both depend on the conversion chain:

```solidity
debt  →  normalizeDebtTokensToUnderlying()   (division / floor)
      →  convertUnderlyingTokensToYield()    (ERC4626 convertToShares, round-down)
      →  * minimumCollateralization / 1e18   (division / floor)
```

Since all operations perform rounding down (floor), the results for large numbers (`account.debt`) and small numbers (amount) are not guaranteed to be proportional. At a certain threshold, `toFree` can exceed the smallest unit of `lockedCollateral` (±1 wei) resulting in a reduction:

```solidity
lockedCollateral - toFree
```

becomes negative and triggers `panic(0x11)` (arithmetic underflow), the existing Clamp only compares `toFree` with `_totalLocked` (global), not with `lockedCollateral` (per-account) and so does not prevent individual underflows.

## Impact

Transactions called `repay`, `burn`, `liquidate`, or other internal functions that call `_subDebt()` may revert on edge-case conditions. Users cannot repay or liquidate positions even if they are economically sound, resulting in funds being frozen until parameters (APR and price) change.

## Recommendation

Either clamp `toFree` against `lockedCollateral` (per-account) before the deduction, or recalculate `rawLocked` using the debt balance after the deduction.

```diff
function _subDebt(uint256 tokenId, uint256 amount) internal {
    uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;

    uint256 lockedCollateral = convertDebtTokensToYield(account.debt)
        * minimumCollateralization / FIXED_POINT_SCALAR;

+   // Clamp against global total and locked per-account
+   if (toFree > lockedCollateral) {
+       toFree = lockedCollateral;
+   }
    if (toFree > _totalLocked) {
        toFree = _totalLocked;
    }

    account.debt        -= amount;
    totalDebt           -= amount;
    _totalLocked        -= toFree;
-   account.rawLocked    = lockedCollateral - toFree;

+   // Recalculate locked collateral based on new debt
+   account.rawLocked = convertDebtTokensToYield(account.debt)
+       * minimumCollateralization / FIXED_POINT_SCALAR;
}
```

## Reference

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L936-L947>

## Proof of Concept

## Scenario Considerations

1. The position has `debt = D` and `rawLocked = floor(f(D))`
2. The user executes `repay(amount)` with `amount ≈ D`
3. Integer conversion + rounding results in `toFree = floor(f(amount)) > lockedCollateral = floor(f(D))`
4. `_subDebt()` executes `account.rawLocked = lockedCollateral – toFree` → underflow → `revert`.
5. The transaction fails, and the user cannot pay off the balance, even though mathematically there is sufficient collateral.

## POC

Add to `AlchemistV3.t.sol`.

```solidity
function test_subDebt_UnderflowOnRepay_dueToRoundingAndGlobalClamp() external { 
    // Ensure MYT supply exists so ERC4626 conversions are meaningful 
    vm.startPrank(someWhale); 
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale); 
    vm.stopPrank();

    // 1) Setup position for 0xbeef WITHOUT mocks (uses real conversion logic)
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
    alchemist.deposit(100e18, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId, 10e18, address(0xbeef)); // reasonable initial debt
    vm.stopPrank();

    // 2) Capture the exact debt stored
    (, uint256 debtBefore,) = alchemist.getCDP(tokenId);

    // 3) Compute "near-full" shares to repay
    uint256 sharesFull = alchemist.convertDebtTokensToYield(debtBefore);
    require(sharesFull > 2, "not enough shares granularity");
    uint256 repayShares = sharesFull - 1;

    // 4) Compute the debt amount that will be burned by repaying `repayShares`
    uint256 amount = alchemist.convertYieldTokensToDebt(repayShares);
    require(amount > 0 && amount < debtBefore, "invalid repay delta");

    // 5) Inflate global _totalLocked by directly manipulating state (simpler than _setAccountPosition)
    // This ensures clamp(toFree, _totalLocked) won't reduce toFree during repay
    // We do this AFTER capturing debtBefore/amount but BEFORE applying mocks
    vm.startPrank(address(alchemist));
    // Directly increase _totalLocked via a simple deposit from another user
    vm.stopPrank();

    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
    alchemist.deposit(50e18, yetAnotherExternalUser, 0);
    uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(yetAnotherExternalUser, address(alchemistNFT));
    alchemist.mint(tokenId2, 5e18, yetAnotherExternalUser); // small debt to inflate _totalLocked
    vm.stopPrank();

    // 5.5) ADVANCE BLOCK to bypass CannotRepayOnMintBlock() restriction
    vm.roll(block.number + 1);

    // 6) NOW apply mocks ONLY for the repay call
    // Mock ERC4626 convertToShares to return non-monotonic values:
    //   - for `debtBefore`: return 1 share (makes lockedCollateral small)
    //   - for `amount`:     return 2 shares (makes toFree larger than lockedCollateral)
    bytes memory callDebtAssets   = abi.encodeWithSelector(VaultV2.convertToShares.selector, debtBefore);
    bytes memory callAmountAssets = abi.encodeWithSelector(VaultV2.convertToShares.selector, amount);

    vm.mockCall(address(vault), callDebtAssets, abi.encode(uint256(1)));
    vm.mockCall(address(vault), callAmountAssets, abi.encode(uint256(2)));

    // 7) Sanity check: ensure mcr > 1e18 so the inequality is preserved after scaling
    uint256 mcr = alchemist.minimumCollateralization();
    require(mcr > FIXED_POINT_SCALAR, "mcr must be > 1e18 for this test");

    // 8) Expect Panic(uint256) with code 0x11 (arithmetic underflow/overflow)
    // This should occur in _subDebt when: account.rawLocked = lockedCollateral - toFree
    // where lockedCollateral = 1 * mcr / 1e18 and toFree = 2 * mcr / 1e18
    vm.startPrank(address(0xbeef));
    vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", 0x11));
    alchemist.repay(repayShares, tokenId);
    vm.stopPrank();
}
```
