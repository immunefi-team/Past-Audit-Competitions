# 58751 sc medium setminimumcollateralization allows for increasing the current minimumcollateralization instantly exposing users to risk of liquidation

**Submitted on Nov 4th 2025 at 11:42:56 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58751
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`setMinimumCollateralization` never enforces that the new collateralization is lower than the current which means that it can be set to a higher number which will cause users near the previous threshold to get instantly liquidated unfairly

## Vulnerability Details

`setMinimumCollateralization` in the Alchemist is implemented as follows:

```
    function setMinimumCollateralization(uint256 value) external onlyAdmin {
        _checkArgument(value >= FIXED_POINT_SCALAR);
        minimumCollateralization = value;

        emit MinimumCollateralizationUpdated(value);
    }
```

It only enforces that the current `minimumCollateralization` is more than 100% to allow the system to be overcollateralized at all times.

It does not valdiate however that the new number is lower than the previous. This allows for increase of `minimumCollateralization` which worsens the health of all current positions and making them instantly, unfairly liquidatable. Even if the admin is trusted, this should never be allowed to happen as it can cause unfair losses to the users. Especially if the delta between the old and the new ratio is large, and `minimumCollateralization` is increased substancially, this can essentially cause losses to all users who have already borrowed from the system.

## Impact Details

An increase in `minimumCollateralization` can make some or even all of users' positions instantly liquidatable and cause loss of funds to users. An admin should never have this option. Not increasing the current LTV is a well known best practice.

## References

* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L292>

## Proof of Concept

## Proof of Concept

1. Add this test to `AlchemistV3.t.sol`

```
    /// @notice PoC: setMinimumCollateralization enables instant liquidation of healthy positions
    function test_POC_setMinimumCollateralization_enables_instant_liquidation() public {
        // ========== STEP 1: INITIAL CONFIGURATION ==========
        console.log("\n========== STEP 1: INITIAL CONFIGURATION ==========");

        // Set initial parameters to "safe" values
        uint256 initialM = 1_110_000_000_000_000_000; // 111%
        uint256 initialLB = 1_050_000_000_000_000_000; // 105%

        vm.startPrank(alOwner);
        alchemist.setMinimumCollateralization(initialM);
        alchemist.setCollateralizationLowerBound(initialLB);
        vm.stopPrank();

        console.log("minimumCollateralization (M):", initialM, "(111%)");
        console.log("collateralizationLowerBound (LB):", initialLB, "(105%)");

        // ========== STEP 2: USER CREATES POSITION ==========
        console.log("\n========== STEP 2: USER CREATES POSITION ==========");

        address user = makeAddr("user");

        // Create position with CR = 112% (safe initially)
        // This is above LB (105%) so position is safe
        // But will become liquidatable when LB rises to 115%
        uint256 collateralAmount = 11_200e18;
        uint256 debtAmount = 10_000e18;

        _magicDepositToVault(address(vault), user, collateralAmount);

        vm.startPrank(user);
        IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
        alchemist.deposit(collateralAmount, user, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        alchemist.mint(tokenId, debtAmount, user);
        vm.stopPrank();

        (uint256 collateral, uint256 debt, ) = alchemist.getCDP(tokenId);
        uint256 collateralValue = alchemist.totalValue(tokenId);
        uint256 cr = (collateralValue * FIXED_POINT_SCALAR) / debt;

        console.log("User deposited:", collateral);
        console.log("User borrowed:", debt);
        console.log("Collateralization Ratio (CR):", cr);

        // ========== STEP 3: VERIFY POSITION IS NOT LIQUIDATABLE ==========
        console.log("\n========== STEP 3: VERIFY POSITION IS SAFE ==========");

        // Try to liquidate - should fail
        vm.expectRevert(IAlchemistV3Errors.LiquidationError.selector);
        alchemist.liquidate(tokenId);
        console.log("Liquidation attempt reverted");

        // ========== STEP 4: ADMIN RAISES minimumCollateralization ==========
        console.log("\n========== STEP 4: ADMIN RAISES minimumCollateralization ==========");

        uint256 newM = 1_150_000_000_000_000_000; // 115%

        vm.startPrank(alOwner);
        alchemist.setMinimumCollateralization(newM);
        alchemist.setCollateralizationLowerBound(newM);
        vm.stopPrank();

        // ========== STEP 5: USER POSITION NOW LIQUIDATABLE ==========
        console.log("\n ========== STEP 5: USER POSITION NOW LIQUIDATABLE ==========");

        // Get current position status
        (, uint256 currentDebt, ) = alchemist.getCDP(tokenId);
        uint256 currentCR = (alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR) / currentDebt;
        console.log("currentCR:", currentCR);

        // Liquidate the position - should succeed now
        (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);

        console.log("Liquidation successful!");
        console.log("  Amount liquidated:", amountLiquidated);
        console.log("  Fee in yield:", feeInYield);
        console.log("  Fee in underlying:", feeInUnderlying);
        console.log("");

        // Verify liquidation happened
        assertTrue(amountLiquidated > 0, "Liquidation should return non-zero amount");
    }
```

2. Run the test:

```
forge test --match-test test_POC_setMinimumCollateralization_enables_instant_liquidation -vv
```

3. Observe the logs. The users previously healthy position became instantly liquidatable.

```
========== STEP 1: INITIAL CONFIGURATION ==========
  minimumCollateralization (M): 1110000000000000000 (111%)
  collateralizationLowerBound (LB): 1050000000000000000 (105%)
  
========== STEP 2: USER CREATES POSITION ==========
  User deposited: 11200000000000000000000
  User borrowed: 10000000000000000000000
  Collateralization Ratio (CR): 1120000000000000000
  
========== STEP 3: VERIFY POSITION IS SAFE ==========
  Liquidation attempt reverted
  
========== STEP 4: ADMIN RAISES minimumCollateralization ==========
  
 ========== STEP 5: USER POSITION NOW LIQUIDATABLE ==========
  currentCR: 1120000000000000000
  Liquidation successful!
    Amount liquidated: 2276000000000000000000
    Fee in yield: 36000000000000000000
    Fee in underlying: 0
```
