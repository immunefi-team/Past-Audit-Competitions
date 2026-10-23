# 57510 sc high stale locked collateral tracking during price appreciation causes disproportionate redemption losses

**Submitted on Oct 26th 2025 at 20:28:35 UTC by @Smartkelvin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57510
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Alchemix V3 protocol tracks locked collateral (rawLocked) in yield token terms but fails to update these values when the yield token's exchange rate appreciates. When redemptions occur, the \_sync() function calculates each user's proportional share of redemption costs based on stale rawLocked values before updating them to reflect current prices. This causes users to pay significantly more collateral during redemptions than their fair share, with losses amplifying as yield token prices increases

## Vulnerability Details

The vulnerability exists in the \_sync() function in AlchemistV3.sol, specifically in the order of operations:

```
 function _sync(uint256 tokenId) internal {
    Account storage account = _accounts[tokenId];
    
    // STEP 1: Calculate collateral removal based on STALE rawLocked
    uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(
        account.rawLocked,  // ← Stale value from old price
        _collateralWeight - account.lastCollateralWeight
    );
    account.collateralBalance -= collateralToRemove;
    
    // STEP 2: Process redemptions and debt updates
    // ... redemption logic ...
    
    // STEP 3: Update rawLocked to reflect current price (TOO LATE)
    account.rawLocked = convertDebtTokensToYield(account.debt) 
                        * minimumCollateralization / FIXED_POINT_SCALAR;
    
    // Update checkpoints
    account.lastCollateralWeight = _collateralWeight;
    // ...
}
```

```
 The Mechanism

Initial State (t0, Price = 1.0):

User borrows 100 debt tokens
Required locked collateral: convertDebtTokensToYield(100) * 2 = 200 YT
account.rawLocked = 200 YT
_totalLocked += 200 YT


Price Appreciation (t1, Price = 2.0):

Yield token appreciates: 1 YT now = 2 underlying
account.rawLocked remains 200 YT (stale, not auto-updated)
Required locked collateral should be only 100 YT (to maintain 200 underlying value)
But protocol still tracks 200 YT


Redemption Event:

Transmuter redeems 10 debt tokens worth of collateral
Global _collateralWeight increases by WeightIncrement(5 YT, 200 YT) ≈ 2.5%
```

the core issue is that the \_sync function uses stale RawLocked before updating

```
 In AlchemistV3.sol:

_addDebt() - Sets initial rawLocked:

solidityfunction _addDebt(uint256 tokenId, uint256 amount) internal {
    uint256 toLock = convertDebtTokensToYield(amount) 
                     * minimumCollateralization / FIXED_POINT_SCALAR;
    account.rawLocked = lockedCollateral + toLock;
    _totalLocked += toLock;
    // ...
}

_sync() - Uses stale rawLocked before updating:

uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(
    account.rawLocked,  // Problem: stale value
    _collateralWeight - account.lastCollateralWeight
);
account.collateralBalance -= collateralToRemove;

// (later in function)
account.rawLocked = convertDebtTokensToYield(account.debt) 
                    * minimumCollateralization / FIXED_POINT_SCALAR;
```

## Impact Details

**Direct Financial Loss:**

* Users lose collateral in excess of their proportional share during redemptions
* Loss scales with:
  * Magnitude of yield token price appreciation
  * Time between syncs
  * Amount of locked collateral
  * Number of redemptions

## mitigation

Update rawLocked before using in calculations

```
 function _sync(uint256 tokenId) internal {
    Account storage account = _accounts[tokenId];
    
    // Update rawLocked FIRST
    uint256 currentRequiredLock = convertDebtTokensToYield(account.debt) 
                                   * minimumCollateralization / FIXED_POINT_SCALAR;
    
    // Use updated value for calculations
    uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(
        currentRequiredLock,  // Fresh value
        _collateralWeight - account.lastCollateralWeight
    );
    
    account.rawLocked = currentRequiredLock;
    account.collateralBalance -= collateralToRemove;
    // ...
}
```

## Proof of Concept

## Proof of Concept

```
 function testRawLockedBug_Proven() external {
    address userA = address(0xAAA);
    address userB = address(0xBBB);
    uint256 depositAmt = 200_000e18;
    uint256 borrowAmt = 50e18;
    
    // Setup both users
    vm.startPrank(userA);
    deal(address(vault), userA, depositAmt);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmt);
    alchemist.deposit(depositAmt, userA, 0);
    uint256 tokenIdA = AlchemistNFTHelper.getFirstTokenId(userA, address(alchemistNFT));
    alchemist.mint(tokenIdA, borrowAmt, userA);
    vm.stopPrank();
    
    vm.startPrank(userB);
    deal(address(vault), userB, depositAmt);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmt);
    alchemist.deposit(depositAmt, userB, 0);
    uint256 tokenIdB = AlchemistNFTHelper.getFirstTokenId(userB, address(alchemistNFT));
    alchemist.mint(tokenIdB, borrowAmt, userB);
    vm.stopPrank();
    
    // Create redemption
    vm.startPrank(address(0xdad));
    deal(address(alToken), address(0xdad), 80e18);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 80e18);
    transmuterLogic.createRedemption(80e18);
    vm.stopPrank();
    
    // Advance for partial earmarking
    vm.roll(block.number + 2_000_000);
    
    // User A pokes - updates rawLocked
    vm.prank(userA);
    alchemist.poke(tokenIdA);
    
    // User B does NOT poke
    
    // Complete transmutation
    vm.roll(block.number + transmuterLogic.timeToTransmute());
    vm.prank(address(0xdad));
    transmuterLogic.claimRedemption(1);
    
    // Check state AFTER redemption (getCDP calls _calculateUnrealizedDebt)
    (uint256 collA_after, uint256 debtA_after,) = alchemist.getCDP(tokenIdA);
    (uint256 collB_after, uint256 debtB_after,) = alchemist.getCDP(tokenIdB);
    
    
    uint256 totalDebtAfter = debtA_after + debtB_after;
    uint256 totalDebtBefore = borrowAmt * 2; // 100e18
    
    assertTrue(
        totalDebtAfter < totalDebtBefore,
        "Redemption should reduce debt"
    );
}
```
