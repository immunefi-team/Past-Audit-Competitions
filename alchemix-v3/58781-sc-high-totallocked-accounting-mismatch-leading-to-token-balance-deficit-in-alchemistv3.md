# 58781 sc high totallocked accounting mismatch leading to token balance deficit in alchemistv3

**Submitted on Nov 4th 2025 at 13:20:13 UTC by @oct0pwn for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58781
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

In AlchemistV3.sol, the \_sync() function reduces account debt and recomputes rawLocked collateral when debt is reduced due to redemption decay, but fails to update the global \_totalLocked variable. When redeem() subsequently uses the inflated \_totalLocked value to weight collateral removals, accounts are systematically under-deducted relative to the tokens actually transferred out. This creates a growing token balance deficit that accumulates over multiple redemption cycles, eventually causing withdraw() and redeem() operations to revert due to insufficient contract balance. The issue affects all users with positions that sync between redemptions, leading to a gradual drain of the contract's yield token balance.

## Vulnerability Details

The vulnerability stems from an invariant violation where `_totalLocked` should equal the sum of all `account.rawLocked` values, but this invariant is broken when accounts sync between redemptions.

### The Accounting Flow

### 1. When `redeem()` is called (lines 589-641):

```solidity
function redeem(uint256 amount) external onlyTransmuter {
    // ... redemption logic ...

    // Line 632: Save _totalLocked before redemption
    uint256 old = _totalLocked;

    // Line 633: Update _totalLocked by tokens transferred
    _totalLocked = totalOut > old ? 0 : old - totalOut;

    // Line 634: Calculate weight increment using old _totalLocked as denominator
    _collateralWeight += PositionDecay.WeightIncrement(
        totalOut > old ? old : totalOut,
        old  // <-- Uses _totalLocked as denominator
    );

    // Line 638: Decrement tracked shares
    _mytSharesDeposited -= collRedeemed + feeCollateral;

    // Transfer tokens out
    TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
}

```

The `WeightIncrement` function calculates the weight based on the ratio of tokens removed (`totalOut`) to the total locked collateral (`old = _totalLocked`). This assumes `_totalLocked` accurately represents the sum of all `account.rawLocked` values.

### 2. When `_sync()` is called (lines 1042-1095):

```solidity
function _sync(uint256 tokenId) internal {
    Account storage account = _accounts[tokenId];

    // Line 1046: Calculate collateral removal based on account's rawLocked
    uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(
        account.rawLocked,
        _collateralWeight - account.lastCollateralWeight
    );

    // Line 1047: Deduct from account's collateral balance
    account.collateralBalance -= collateralToRemove;

    // ... redemption decay calculations ...

    // Line 1083: Reduce debt due to redemption decay
    account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;

    // Line 1086: Recompute rawLocked based on NEW reduced debt
    account.rawLocked = convertDebtTokensToYield(account.debt) *
                       minimumCollateralization / FIXED_POINT_SCALAR;

    // CRITICAL ISSUE: _totalLocked is NOT updated here
    // Even though account.rawLocked just shrunk, _totalLocked remains unchanged
}

```

### The Invariant Violation

The `AlchemistV3.sol`contract maintains `_totalLocked` correctly in other operations:

* **`_addDebt()` (line 923)**: `_totalLocked += toLock`
* **`_subDebt()` (line 946)**: `_totalLocked -= toFree`
* **`_sync()` (line 1086)**: Recomputes `rawLocked` but **does NOT update `_totalLocked`**

### Evidence of Invariant

The comment in `AlchemistV3.sol` on line 128-129 confirms the intended invariant:

```solidity
/// @dev Total locked collateral.
/// Locked collateral is the collateral that cannot be withdrawn due to LTV constraints
uint256 private _totalLocked;

```

This clearly indicates `_totalLocked` should track the sum of all locked collateral, which is the sum of all `account.rawLocked` values.

## Impact Details

This vulnerability causes a **gradual but persistent drain** of the contract's yield token balance, affecting the protocol's ability to fulfill withdrawals and redemptions.

### Direct Impact

1. **Token Balance Deficit**: The contract's actual `myt` token balance becomes less than `_mytSharesDeposited` over time. This deficit accumulates with each redemption cycle where accounts sync between redemptions.
2. **Operation Failures**:
   * `withdraw()` operations will revert when users try to withdraw, as `TokenUtils.safeTransfer(myt, recipient, amount)` will fail due to insufficient balance
   * `redeem()` operations will eventually fail for the same reason
   * The severity increases over time as more accounts sync and the deficit grows
3. **User Funds at Risk**:
   * Users cannot withdraw their deposited collateral
   * The protocol cannot fulfill redemption requests
   * All users with synced positions are affected, with the impact growing as more positions sync
4. The deficit grows proportionally to:
   * Number of redemptions (`redeem()` calls)
   * Number of accounts that sync between redemptions
   * Amount by which each synced account's `rawLocked` decreases

## References

1. `src/AlchemistV3.sol`:
   * Line 128-130: `_totalLocked` variable declaration and comment
   * Line 589-641: `redeem()` function
   * Line 632-634: Weight increment calculation using `_totalLocked`
   * Line 1042-1095: `_sync()` function
   * Line 1086: `rawLocked` recomputation without `_totalLocked` update
   * Line 913-926: `_addDebt()` function (correctly updates `_totalLocked`)
   * Line 932-953: `_subDebt()` function (correctly updates `_totalLocked`)
2. `src/libraries/PositionDecay.sol`:
   * Line 39-57: `WeightIncrement()` function
   * Line 67-84: `ScaleByWeightDelta()` function

## Proof of Concept

## Proof of Concept

Place the following test in AlchemistV3.t.sol:

```solidity
function testAudit_TotalLockedNotUpdatedOnSyncCausesDeficit() external {
        uint256 amount = 100_000e18;

        // Setup: Create two accounts with debt
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(
            address(vault),
            address(alchemist),
            type(uint256).max
        );
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(
            address(0xbeef),
            address(alchemistNFT)
        );
        uint256 maxBorrow1 = alchemist.getMaxBorrowable(tokenId1);
        alchemist.mint(tokenId1, (maxBorrow1 * 90) / 100, address(0xbeef));
        vm.stopPrank();

        vm.startPrank(externalUser);
        SafeERC20.safeApprove(
            address(vault),
            address(alchemist),
            type(uint256).max
        );
        alchemist.deposit(amount, externalUser, 0);
        uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(
            externalUser,
            address(alchemistNFT)
        );
        uint256 maxBorrow2 = alchemist.getMaxBorrowable(tokenId2);
        alchemist.mint(tokenId2, (maxBorrow2 * 90) / 100, externalUser);
        vm.stopPrank();

        (uint256 collateral1_before, , ) = alchemist.getCDP(tokenId1);
        (uint256 collateral2_before, , ) = alchemist.getCDP(tokenId2);
        uint256 initialBalance = IERC20(address(vault)).balanceOf(
            address(alchemist)
        );

        // Create redemption R1
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(
            address(alToken),
            address(transmuterLogic),
            type(uint256).max
        );
        transmuterLogic.createRedemption(50_000e18);
        vm.stopPrank();

        vm.roll(block.number + 1000);

        // Account1 syncs (reduces rawLocked but NOT _totalLocked)
        vm.startPrank(address(0xbeef));
        alchemist.poke(tokenId1);
        vm.stopPrank();

        uint256 balanceAfterSync = IERC20(address(vault)).balanceOf(
            address(alchemist)
        );

        // Complete redemption R1
        vm.roll(block.number + 5_256_000);
        vm.startPrank(address(0xdad));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

        uint256 balanceAfterR1 = IERC20(address(vault)).balanceOf(
            address(alchemist)
        );

        // Account2 syncs
        vm.startPrank(externalUser);
        alchemist.poke(tokenId2);
        vm.stopPrank();

        // Create and claim redemption R2 (uses inflated _totalLocked)
        vm.startPrank(address(0xdad));
        transmuterLogic.createRedemption(30_000e18);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);
        vm.startPrank(address(0xdad));
        transmuterLogic.claimRedemption(2);
        vm.stopPrank();

        uint256 balanceAfterR2 = IERC20(address(vault)).balanceOf(
            address(alchemist)
        );
        (uint256 collateral1_after, , ) = alchemist.getCDP(tokenId1);
        (uint256 collateral2_after, , ) = alchemist.getCDP(tokenId2);

        // Calculate deficit
        uint256 tokensTransferred = initialBalance - balanceAfterR2;
        uint256 collateralDeducted = (collateral1_before - collateral1_after) +
            (collateral2_before - collateral2_after);
        uint256 deficit = tokensTransferred - collateralDeducted;

        console.log("Tokens transferred out:", tokensTransferred);
        console.log("Collateral deducted:", collateralDeducted);
        console.log("DEFICIT:", deficit);

        // BUG: Deficit exists - tokens transferred exceed collateral deducted
        assertGt(
            deficit,
            0,
            "Deficit exists - _totalLocked not updated in _sync() causes under-deduction"
        );
    }
    
// EXPECTED OUTPUT:
//   Tokens transferred out: 80000000000000000000000
//   Collateral deducted: 79358974358974358975001
//   DEFICIT: 641025641025641024999
```
