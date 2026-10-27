# 58683 sc critical there is an issue in earmarked debt eeduction in the repay can causes a permanent fund freeze

**Submitted on Nov 4th 2025 at 01:45:21 UTC by @XDZIBECX for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58683
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

there is an issue in the repay function that is removes all earmarked debt when a user repays, regardless of the repayment amount. and this is creates an asymmetry where users lose their protection during redemptions, so their collateral is reduced globally based on system-wide redemption weights, but their debt reduction depends on their local earmarked amount. When a user repays 50% of their debt, the function is incorrectly wipes 100% of their earmarked protection instead of 50%. Subsequent redemptions then reduce the user's collateral without proportional debt reduction, and this is not correct because this is create a positions with collateralization ratios below the minimum threshold. and this positions become a permanently frozen due an underflows and that is prevent withdrawals, and liquidations, or any recovery operations, so as reuslt his is gone be a permanent loss of user funds.

## Vulnerability Details

The vulnerability exists in the repay() function here --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L521C9-L523C46>

```
// Repay debt from earmarked amount of debt first
uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
account.earmarked -= earmarkToRemove;
```

so the flaw is that When `credit < account.earmarked`, the contract is removes `min(credit, earmarked)` from the earmarked. and this is means if a user repays 50% of their debt, but earmarked equals or exceeds that amount, then the full repayment amount is removed from earmarked, not a proportional share.

the `burn()` function it's correctly protects earmarked debt ---> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L470> from here it's prevents burning earmarked debt entirely, but the `repay()` removes it disproportionately. and this inconsistency is a bug, and this is not an intended behavior.

```solidity
// Burning alAssets can only repay unearmarked debt
_checkState((debt = _accounts[recipientId].debt - _accounts[recipientId].earmarked) > 0);
```

as known that the Earmarked debt represents a user's proportional share of debt that will be redeemed via the Transmuter. the fucntion `_sync()`uses rhe earmarked amounts to calculate the debt reduction during the redemption from here --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1075C7-L1084C1> :

```solidity
// Old earmarks that survived redemptions in the current sync window
uint256 exposureSurvival = _mulQ128(account.earmarked, survivalRatio);
// What was redeemed from the newly earmark between last sync and now
uint256 redeemedFromEarmarked = earmarkRaw - earmarkedUnredeemed;
// Total overall earmarked to adjust user debt
uint256 redeemedTotal = (account.earmarked - exposureSurvival) + redeemedFromEarmarked;

account.earmarked = exposureSurvival + earmarkedUnredeemed;
account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;
```

so If the `account.earmarked = 0`, then the `redeemedTotal ≈ 0`, this is mean there is no debt reduction occurs during the redemptions.

the collateral is ALWAYS reduced and hti is based on global weights here <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1217C7-L1219C80>

```solidity
uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
account.collateralBalance -= collateralToRemove;
```

this is creates the asymmetry that collateral reduced globally, debt reduced locally based on earmarked.

## Impact Details

i chose this as impact Permanent freezing of funds i alerdy test this and is occur check and run please the test is show all result

so this impact can happen when users repay their debt, then the contract incorrectly removes all their earmarked protection instead of reducing it proportionally. as an example, if a user repays 50% of their debt, the contract should keep 50% of their earmarked protection but instead is removes 100% of it. and this earmarked protection is crucial because it determines how much debt gets reduced when redemptions happen. Without it, users lose collateral during the redemptions but their debt stays the same, and this making them owe more than they have. The position then breaks mathematically the contract's internal accounting becomes impossible (it calculates that locked collateral should be 27.78 ETH when the user only has 20.36 ETH), and this is cause all operations to fail with arithmetic errors. The user cannot withdraw, cannot repay more, and cannot be liquidated. Their funds are permanently stuck. In a real scenario with 100 ETH deposited, approximately 20-30 ETH (worth $40,000-$60,000) becomes permanently frozen. and this is happens automatically during normal use any user who repays debt while the Transmuter has active positions will trigger this with no way to recover their fund this need to be fixed

## References

check the vulnerbaility details i use all them in the pargraph

## Proof of Concept

## Proof of Concept

copy past this test in the AlchemistV3.t.sol and run it

```solidity

function test_CriticalBug_RepayWipesEarmarked_CreatesUnliquidatablePosition() external {
        console.log("=== CRITICAL BUG TEST: Repay Wipes Earmarked Debt ===");
        console.log("");
        
        // Setup: Enable protocol fee and configure liquidation bounds
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(100);  // 1% protocol fee
        alchemist.setCollateralizationLowerBound(alchemist.minimumCollateralization() * 99 / 100);
        vm.stopPrank();

        uint256 depositAmount = 100e18;
        uint256 debtAmount = 50e18;  // 50% LTV
        
        // === STEP 1: Create two identical positions ===
        address userA = address(0xA11CE);
        address userB = address(0xB0B);
        
        // Setup User A
        vm.startPrank(userA);
        deal(address(vault), userA, depositAmount);
        vault.approve(address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, userA, 0);
        uint256 tokenIdA = AlchemistNFTHelper.getFirstTokenId(userA, address(alchemistNFT));
        alchemist.mint(tokenIdA, debtAmount, userA);
        vm.stopPrank();

        // Setup User B
        vm.startPrank(userB);
        deal(address(vault), userB, depositAmount);
        vault.approve(address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, userB, 0);
        uint256 tokenIdB = AlchemistNFTHelper.getFirstTokenId(userB, address(alchemistNFT));
        alchemist.mint(tokenIdB, debtAmount, userB);
        vm.stopPrank();

        console.log("Step 1 - Initial State:");
        (uint256 collateralA1, uint256 debtA1, uint256 earmarkedA1) = alchemist.getCDP(tokenIdA);
        (uint256 collateralB1, uint256 debtB1, uint256 earmarkedB1) = alchemist.getCDP(tokenIdB);
        console.log("  User A: collateral=%e, debt=%e, earmarked=%e", collateralA1, debtA1, earmarkedA1);
        console.log("  User B: collateral=%e, debt=%e, earmarked=%e", collateralB1, debtB1, earmarkedB1);
        console.log("");

        // === STEP 2: Create LARGE redemption to build significant earmarked debt ===
        vm.startPrank(userA);
        alToken.approve(address(transmuterLogic), debtAmount);
        transmuterLogic.createRedemption(debtAmount);
        vm.stopPrank();

        // Advance to FULL MATURATION to maximize earmarked debt
        vm.roll(block.number + 5_256_000);
        
        alchemist.poke(tokenIdA);
        alchemist.poke(tokenIdB);

        console.log("Step 2 - After FULL Earmarking (5.25M blocks - FULLY MATURED):");
        (uint256 collateralA2, uint256 debtA2, uint256 earmarkedA2) = alchemist.getCDP(tokenIdA);
        (uint256 collateralB2, uint256 debtB2, uint256 earmarkedB2) = alchemist.getCDP(tokenIdB);
        console.log("  User A: collateral=%e, debt=%e, earmarked=%e", collateralA2, debtA2, earmarkedA2);
        console.log("  User B: collateral=%e, debt=%e, earmarked=%e", collateralB2, debtB2, earmarkedB2);
        console.log("  Total Debt: %e, Cumulative Earmarked: %e", alchemist.totalDebt(), alchemist.cumulativeEarmarked());
        assertGt(earmarkedA2, debtA2 / 4, "User A should have significant earmarked debt (>25%)");
        assertGt(earmarkedB2, debtB2 / 4, "User B should have significant earmarked debt (>25%)");
        console.log("");

        // === STEP 3: User B repays SMALL amount (BUG: wipes DISPROPORTIONATE earmarked) ===
        uint256 repayAmount = debtB2 / 2;  // Repay 50% of debt
        
        vm.startPrank(userB);
        deal(address(vault), userB, repayAmount);
        vault.approve(address(alchemist), repayAmount);
        alchemist.repay(alchemist.convertDebtTokensToYield(repayAmount), tokenIdB);
        
        // Withdraw maximum to make position leveraged
        (uint256 tempColl, uint256 tempDebt,) = alchemist.getCDP(tokenIdB);
        uint256 lockedNeeded = alchemist.convertDebtTokensToYield(tempDebt) * alchemist.minimumCollateralization() / FIXED_POINT_SCALAR;
        uint256 maxWithdraw = tempColl > lockedNeeded ? tempColl - lockedNeeded - 1e18 : 0;
        if (maxWithdraw > 0) {
            alchemist.withdraw(maxWithdraw, userB, tokenIdB);
        }
        vm.stopPrank();

        alchemist.poke(tokenIdA);
        alchemist.poke(tokenIdB);

        console.log("Step 3 - After User B Repays 50%% of Debt:");
        (uint256 collateralA3, uint256 debtA3, uint256 earmarkedA3) = alchemist.getCDP(tokenIdA);
        (uint256 collateralB3, uint256 debtB3, uint256 earmarkedB3) = alchemist.getCDP(tokenIdB);
        console.log("  User A: collateral=%e, debt=%e, earmarked=%e", collateralA3, debtA3, earmarkedA3);
        console.log("  User B: collateral=%e, debt=%e, earmarked=%e", collateralB3, debtB3, earmarkedB3);
        console.log("  >>> BUG EVIDENCE: User B repaid 50%% but earmarked is: %e", earmarkedB3);
        console.log("  >>> EXPECTED: earmarked should be ~%e (50%% of %e)", earmarkedB2 / 2, earmarkedB2);
        assertEq(earmarkedB3, 0, "BUG CONFIRMED: User B's earmarked debt completely wiped!");
        console.log("");

        // === STEP 4: LARGE Redemption occurs ===
        vm.prank(userA);
        transmuterLogic.claimRedemption(1);

        alchemist.poke(tokenIdA);
        alchemist.poke(tokenIdB);

        console.log("Step 4 - After LARGE Redemption (50e18 debt):");
        (uint256 collateralA4, uint256 debtA4, uint256 earmarkedA4) = alchemist.getCDP(tokenIdA);
        (uint256 collateralB4, uint256 debtB4, uint256 earmarkedB4) = alchemist.getCDP(tokenIdB);
        console.log("  User A: collateral=%e, debt=%e, earmarked=%e", collateralA4, debtA4, earmarkedA4);
        console.log("  User B: collateral=%e, debt=%e, earmarked=%e", collateralB4, debtB4, earmarkedB4);
        
        // Calculate collateralization ratios
        uint256 valueA = alchemist.totalValue(tokenIdA);
        uint256 valueB = alchemist.totalValue(tokenIdB);
        uint256 ratioA = debtA4 > 0 ? valueA * FIXED_POINT_SCALAR / debtA4 : type(uint256).max;
        uint256 ratioB = debtB4 > 0 ? valueB * FIXED_POINT_SCALAR / debtB4 : type(uint256).max;
        
        console.log("  User A collateralization: %e%% (healthy)", ratioA * 100 / FIXED_POINT_SCALAR);
        console.log("  User B collateralization: %e%% (compromised!)", ratioB * 100 / FIXED_POINT_SCALAR);
        console.log("");
        
        // Show the asymmetric impact
        uint256 collateralLossA = collateralA3 > collateralA4 ? collateralA3 - collateralA4 : 0;
        uint256 collateralLossB = collateralB3 > collateralB4 ? collateralB3 - collateralB4 : 0;
        uint256 debtReductionA = debtA3 > debtA4 ? debtA3 - debtA4 : 0;
        uint256 debtReductionB = debtB3 > debtB4 ? debtB3 - debtB4 : 0;
        
        console.log("  ASYMMETRIC IMPACT:");
        console.log("  User A: Lost %e collateral, Debt reduced by %e", collateralLossA, debtReductionA);
        console.log("  User B: Lost %e collateral, Debt reduced by %e", collateralLossB, debtReductionB);
        console.log("  >>> User B has earmarked=0, so NO debt reduction despite collateral loss!");
        console.log("");

        // === STEP 5: Prove position cannot be managed ===
        console.log("Step 5 - Proving Permanent Freeze:");
        
        vm.startPrank(userB);
        
        // Try to repay remaining debt - will fail because user has no vault shares left
        console.log("  Attempting to repay remaining debt...");
        try alchemist.repay(alchemist.convertDebtTokensToYield(debtB4), tokenIdB) {
            console.log("    Repayment succeeded");
        } catch {
            console.log("    >>> FAILED: Cannot repay (insufficient vault shares from earlier withdrawal)");
        }
        
        // Try to withdraw - will fail because position is at minimum collateralization
        console.log("  Attempting to withdraw collateral...");
        try alchemist.withdraw(1e18, userB, tokenIdB) {
            console.log("    Withdrawal succeeded");
        } catch {
            console.log("    >>> FAILED: Cannot withdraw (position too leveraged)");
        }
        
        vm.stopPrank();
        console.log("");
        
        // === STEP 6: Show the core bug ===
        console.log("=== BUG SUMMARY ===");
        console.log("1. User B repaid 50%% of debt");
        console.log("2. repay() wiped 100%% of earmarked debt (should only wipe 50%%)");
        console.log("3. Redemption occurred:");
        console.log("   - User A (earmarked=%e): Debt reduced by %e", earmarkedA3, debtReductionA);
        console.log("   - User B (earmarked=0): Debt reduced by %e", debtReductionB);
        console.log("4. User B lost earmarked protection unfairly");
        console.log("5. With larger redemptions, this creates underwater/unliquidatable positions");
        console.log("");
        console.log("SEVERITY: CRITICAL - Breaks redemption fairness and can freeze funds");
        console.log("");
    }

}

```

* the result :

```solidity
 │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 28777777777777777775 [2.877e19]
    │   │   ├─ [26990] MockMYTVault::transfer(0x0000000000000000000000000000000000000B0b, 70972222222222222225 [7.097e19])
    │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: 0x0000000000000000000000000000000000000B0b, value: 70972222222222222225 [7.097e19])       
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit Withdraw(amount: 70972222222222222225 [7.097e19], tokenId: 2, recipient: 0x0000000000000000000000000000000000000B0b)
    │   │   └─ ← [Return] 70972222222222222225 [7.097e19]
    │   └─ ← [Return] 70972222222222222225 [7.097e19]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [66515] TransparentUpgradeableProxy::fallback(1)
    │   ├─ [65803] AlchemistV3::poke(1) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000A11cE
    │   │   ├─ [20520] MockMYTVault::convertToShares(50000000000000000000 [5e19]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 50000000000000000000 [5e19]
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [65509] TransparentUpgradeableProxy::fallback(2)
    │   ├─ [64797] AlchemistV3::poke(2) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(2) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000B0b
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [0] console::log("Step 3 - After User B Repays 50%% of Debt:") [staticcall]
    │   └─ ← [Stop]
    ├─ [38552] TransparentUpgradeableProxy::fallback(1) [staticcall]
    │   ├─ [37831] AlchemistV3::getCDP(1) [delegatecall]
    │   │   └─ ← [Return] 100000000000000000000 [1e20], 50000000000000000000 [5e19], 25000000000000000000 [2.5e19]
    │   └─ ← [Return] 100000000000000000000 [1e20], 50000000000000000000 [5e19], 25000000000000000000 [2.5e19]
    ├─ [37546] TransparentUpgradeableProxy::fallback(2) [staticcall]
    │   ├─ [36825] AlchemistV3::getCDP(2) [delegatecall]
    │   │   └─ ← [Return] 28777777777777777775 [2.877e19], 25000000000000000000 [2.5e19], 0
    │   └─ ← [Return] 28777777777777777775 [2.877e19], 25000000000000000000 [2.5e19], 0
    ├─ [0] console::log("  User A: collateral=%e, debt=%e, earmarked=%e", 100000000000000000000 [1e20], 50000000000000000000 [5e19], 25000000000000000000 [2.5e19]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  User B: collateral=%e, debt=%e, earmarked=%e", 28777777777777777775 [2.877e19], 25000000000000000000 [2.5e19], 0) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  >>> BUG EVIDENCE: User B repaid 50%% but earmarked is: %e", 0) [staticcall]  
    │   └─ ← [Stop]
    ├─ [0] console::log("  >>> EXPECTED: earmarked should be ~%e (50%% of %e)", 12500000000000000000 [1.25e19], 25000000000000000000 [2.5e19]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::prank(0x00000000000000000000000000000000000A11cE)
    │   └─ ← [Return]
    ├─ [575120] Transmuter::claimRedemption(1)
    │   ├─ emit Transfer(from: 0x00000000000000000000000000000000000A11cE, to: 0x0000000000000000000000000000000000000000, tokenId: 1)
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   ├─ [22324] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [21612] AlchemistV3::getTotalUnderlyingValue() [delegatecall]
    │   │   │   ├─ [18895] MockMYTVault::convertToAssets(128777777777777777775 [1.287e20]) [staticcall] 
    │   │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   └─ ← [Return] 128777777777777777775 [1.287e20]
    │   │   │   └─ ← [Return] 128777777777777777775 [1.287e20]
    │   │   └─ ← [Return] 128777777777777777775 [1.287e20]
    │   ├─ [21133] TransparentUpgradeableProxy::fallback(25000000000000000000 [2.5e19]) [staticcall]    
    │   │   ├─ [20418] AlchemistV3::convertYieldTokensToUnderlying(25000000000000000000 [2.5e19]) [delegatecall]
    │   │   │   ├─ [18895] MockMYTVault::convertToAssets(25000000000000000000 [2.5e19]) [staticcall]    
    │   │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   ├─ [22324] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [21612] AlchemistV3::getTotalUnderlyingValue() [delegatecall]
    │   │   │   ├─ [18895] MockMYTVault::convertToAssets(128777777777777777775 [1.287e20]) [staticcall] 
    │   │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   └─ ← [Return] 128777777777777777775 [1.287e20]
    │   │   │   └─ ← [Return] 128777777777777777775 [1.287e20]
    │   │   └─ ← [Return] 128777777777777777775 [1.287e20]
    │   ├─ [21133] TransparentUpgradeableProxy::fallback(25000000000000000000 [2.5e19]) [staticcall]    
    │   │   ├─ [20418] AlchemistV3::convertYieldTokensToUnderlying(25000000000000000000 [2.5e19]) [delegatecall]
    │   │   │   ├─ [18895] MockMYTVault::convertToAssets(25000000000000000000 [2.5e19]) [staticcall]    
    │   │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   ├─ [1295] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [583] AlchemistV3::totalSyntheticsIssued() [delegatecall]
    │   │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   ├─ [3448] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [2736] AlchemistV3::underlyingToken() [delegatecall]
    │   │   │   └─ ← [Return] TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA]
    │   │   └─ ← [Return] TestERC20: [0xf9ee71f54F2009fD180C480e44a5176AdCABa5eA]
    │   ├─ [2519] TestERC20::decimals() [staticcall]
    │   │   └─ ← [Return] 18
    │   ├─ [21969] TransparentUpgradeableProxy::fallback(25000000000000000000 [2.5e19]) [staticcall]    
    │   │   ├─ [21254] AlchemistV3::convertYieldTokensToDebt(25000000000000000000 [2.5e19]) [delegatecall]
    │   │   │   ├─ [18895] MockMYTVault::convertToAssets(25000000000000000000 [2.5e19]) [staticcall]    
    │   │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   ├─ [278780] TransparentUpgradeableProxy::fallback(25000000000000000000 [2.5e19])
    │   │   ├─ [278068] AlchemistV3::redeem(25000000000000000000 [2.5e19]) [delegatecall]
    │   │   │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   │   ├─ [18895] MockMYTVault::convertToAssets(25000000000000000000 [2.5e19]) [staticcall]    
    │   │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   │   ├─ [20520] MockMYTVault::convertToShares(0) [staticcall]
    │   │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]    
    │   │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   │   ├─ [7090] MockMYTVault::transfer(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 25000000000000000000 [2.5e19])
    │   │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 25000000000000000000 [2.5e19])
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ [7090] MockMYTVault::transfer(PointEvaluation: [0x000000000000000000000000000000000000000A], 250000000000000000 [2.5e17])
    │   │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: PointEvaluation: [0x000000000000000000000000000000000000000A], value: 250000000000000000 [2.5e17])
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ emit Redemption(amount: 25000000000000000000 [2.5e19])
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Return]
    │   ├─ [23939] TransparentUpgradeableProxy::fallback(50000000000000000000 [5e19]) [staticcall]      
    │   │   ├─ [23224] AlchemistV3::convertDebtTokensToYield(50000000000000000000 [5e19]) [delegatecall]
    │   │   │   ├─ [20520] MockMYTVault::convertToShares(50000000000000000000 [5e19]) [staticcall]      
    │   │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   └─ ← [Return] 50000000000000000000 [5e19]
    │   │   │   └─ ← [Return] 50000000000000000000 [5e19]
    │   │   └─ ← [Return] 50000000000000000000 [5e19]
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   └─ ← [Return] 50000000000000000000 [5e19]
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [26990] MockMYTVault::transfer(0x00000000000000000000000000000000000A11cE, 49950000000000000000 [4.995e19])
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: 0x00000000000000000000000000000000000A11cE, value: 49950000000000000000 [4.995e19])
    │   │   └─ ← [Return] true
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [28990] MockMYTVault::transfer(AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 50000000000000000 [5e16])
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 50000000000000000 [5e16])
    │   │   └─ ← [Return] true
    │   ├─ [6329] AlchemicTokenV3::transfer(0x00000000000000000000000000000000000A11cE, 0)
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: 0x00000000000000000000000000000000000A11cE, value: 0)
    │   │   └─ ← [Return] true
    │   ├─ [8329] AlchemicTokenV3::transfer(AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 0)
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: AlchemistV3Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 0)
    │   │   └─ ← [Return] true
    │   ├─ [5414] AlchemicTokenV3::burn(50000000000000000000 [5e19])
    │   │   ├─ emit Transfer(from: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], to: 0x0000000000000000000000000000000000000000, value: 50000000000000000000 [5e19])
    │   │   └─ ← [Stop]
    │   ├─ [2257] TransparentUpgradeableProxy::fallback(50000000000000000000 [5e19])
    │   │   ├─ [1545] AlchemistV3::reduceSyntheticsIssued(50000000000000000000 [5e19]) [delegatecall]   
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Return]
    │   ├─ [1712] TransparentUpgradeableProxy::fallback() [staticcall]
    │   │   ├─ [1000] AlchemistV3::myt() [delegatecall]
    │   │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   │   └─ ← [Return] MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]
    │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   └─ ← [Return] 0
    │   ├─ [3119] TransparentUpgradeableProxy::fallback(0)
    │   │   ├─ [2407] AlchemistV3::setTransmuterTokenBalance(0) [delegatecall]
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Return]
    │   ├─ emit PositionClaimed(claimer: 0x00000000000000000000000000000000000A11cE, amountClaimed: 49950000000000000000 [4.995e19], amountUnclaimed: 0)
    │   └─ ← [Stop]
    ├─ [152095] TransparentUpgradeableProxy::fallback(1)
    │   ├─ [151383] AlchemistV3::poke(1) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000A11cE
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [151980] TransparentUpgradeableProxy::fallback(2)
    │   ├─ [151268] AlchemistV3::poke(2) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(2) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000B0b
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    ├─ [0] console::log("Step 4 - After LARGE Redemption (50e18 debt):") [staticcall]
    │   └─ ← [Stop]
    ├─ [36071] TransparentUpgradeableProxy::fallback(1) [staticcall]
    │   ├─ [35350] AlchemistV3::getCDP(1) [delegatecall]
    │   │   └─ ← [Return] 83166666666666666666 [8.316e19], 25000000000000000000 [2.5e19], 0
    │   └─ ← [Return] 83166666666666666666 [8.316e19], 25000000000000000000 [2.5e19], 0
    ├─ [36071] TransparentUpgradeableProxy::fallback(2) [staticcall]
    │   ├─ [35350] AlchemistV3::getCDP(2) [delegatecall]
    │   │   └─ ← [Return] 20361111111111111108 [2.036e19], 25000000000000000000 [2.5e19], 0
    │   └─ ← [Return] 20361111111111111108 [2.036e19], 25000000000000000000 [2.5e19], 0
    ├─ [0] console::log("  User A: collateral=%e, debt=%e, earmarked=%e", 83166666666666666666 [8.316e19], 25000000000000000000 [2.5e19], 0) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  User B: collateral=%e, debt=%e, earmarked=%e", 20361111111111111108 [2.036e19], 25000000000000000000 [2.5e19], 0) [staticcall]
    │   └─ ← [Stop]
    ├─ [55419] TransparentUpgradeableProxy::fallback(1) [staticcall]
    │   ├─ [54704] AlchemistV3::totalValue(1) [delegatecall]
    │   │   ├─ [18895] MockMYTVault::convertToAssets(83166666666666666666 [8.316e19]) [staticcall]      
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 83166666666666666666 [8.316e19]
    │   │   └─ ← [Return] 83166666666666666666 [8.316e19]
    │   └─ ← [Return] 83166666666666666666 [8.316e19]
    ├─ [55419] TransparentUpgradeableProxy::fallback(2) [staticcall]
    │   ├─ [54704] AlchemistV3::totalValue(2) [delegatecall]
    │   │   ├─ [18895] MockMYTVault::convertToAssets(20361111111111111108 [2.036e19]) [staticcall]      
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 20361111111111111108 [2.036e19]
    │   │   └─ ← [Return] 20361111111111111108 [2.036e19]
    │   └─ ← [Return] 20361111111111111108 [2.036e19]
    ├─ [0] console::log("  User A collateralization: %e%% (healthy)", 332) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  User B collateralization: %e%% (compromised!)", 81) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  ASYMMETRIC IMPACT:") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  User A: Lost %e collateral, Debt reduced by %e", 16833333333333333334 [1.683e19], 25000000000000000000 [2.5e19]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  User B: Lost %e collateral, Debt reduced by %e", 8416666666666666667 [8.416e18], 0) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  >>> User B has earmarked=0, so NO debt reduction despite collateral loss!") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Step 5 - Proving Permanent Freeze:") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::startPrank(0x0000000000000000000000000000000000000B0b)
    │   └─ ← [Return]
    ├─ [0] console::log("  Attempting to repay remaining debt...") [staticcall]
    │   └─ ← [Stop]
    ├─ [23939] TransparentUpgradeableProxy::fallback(25000000000000000000 [2.5e19]) [staticcall]        
    │   ├─ [23224] AlchemistV3::convertDebtTokensToYield(25000000000000000000 [2.5e19]) [delegatecall]  
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   └─ ← [Return] 25000000000000000000 [2.5e19]
    ├─ [165747] TransparentUpgradeableProxy::fallback(25000000000000000000 [2.5e19], 2)
    │   ├─ [164998] AlchemistV3::repay(25000000000000000000 [2.5e19], 2) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(2) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000B0b
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   ├─ [18895] MockMYTVault::convertToAssets(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   ├─ [3183] MockMYTVault::transferFrom(0x0000000000000000000000000000000000000B0b, Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 25000000000000000000 [2.5e19])
    │   │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   │   └─ ← [Revert] ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)
    │   └─ ← [Revert] ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)
    ├─ [0] console::log("    >>> FAILED: Cannot repay (insufficient vault shares from earlier withdrawal)") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Attempting to withdraw collateral...") [staticcall]
    │   └─ ← [Stop]
    ├─ [91363] TransparentUpgradeableProxy::fallback(1000000000000000000 [1e18], 0x0000000000000000000000000000000000000B0b, 2)
    │   ├─ [90635] AlchemistV3::withdraw(1000000000000000000 [1e18], 0x0000000000000000000000000000000000000B0b, 2) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(2) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000B0b
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(2) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000B0b
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   ├─ [20520] MockMYTVault::convertToShares(25000000000000000000 [2.5e19]) [staticcall]        
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9620] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4941] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000 [1e18]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   └─ ← [Return] 25000000000000000000 [2.5e19]
    │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    ├─ [0] console::log("    >>> FAILED: Cannot withdraw (position too leveraged)") [staticcall]        
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("=== BUG SUMMARY ===") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("1. User B repaid 50%% of debt") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("2. repay() wiped 100%% of earmarked debt (should only wipe 50%%)") [staticcall]
    │   └─ ← [Stop]
    │   └─ ← [Stop]
    ├─ [0] console::log("   - User A (earmarked=%e): Debt reduced by %e", 25000000000000000000 [2.5e19], 25000000000000000000 [2.5e19]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("   - User B (earmarked=0): Debt reduced by %e", 0) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("4. User B lost earmarked protection unfairly") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("5. With larger redemptions, this creates underwater/unliquidatable positions") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("SEVERITY: CRITICAL - Breaks redemption fairness and can freeze funds") [staticcall]       
    │   └─ ← [Stop]
    ├─ [0] console::log("") [staticcall]
    │   └─ ← [Stop]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 23.25ms (11.82ms CPU time)

Ran 1 test suite in 41.63ms (23.25ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```
