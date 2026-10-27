# 58004 sc high protocol insolvency from cumulativeearmarked during forcerepay&#x20;

**Submitted on Oct 29th 2025 at 23:15:45 UTC by @ihtishamsudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58004
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `_forceRepay()` internal function in AlchemistV3.sol fails to update the global `cumulativeEarmarked` state variable when processing forced debt repayments during liquidations, while correctly updating the per-account `earmarked` debt.

## Vulnerability Details

The vulnerability exists in the `_forceRepay()` function which is called during liquidation operations when underwater accounts have earmarked debt that must be forcibly repaid. Unlike the public `repay()` function which correctly updates both account-level and global debt tracking, `_forceRepay()` only updates the account-level `earmarked` amount without decrementing the global `cumulativeEarmarked` variable.

```solidity
function _forceRepay(address accountId, uint256 amount) internal {
    Account storage account = accounts[accountId];
    
    // ... validation and debt calculation ...
    
    uint256 earmarkToRemove = amount > account.earmarked ? account.earmarked : amount;
    account.earmarked -= earmarkToRemove; //@audit not updating global earmark
    
    // ... rest of function ...
}
```

## Impact Details

A permanent discrepancy between individual account debt tracking and the protocol's global debt accounting. causes the protocol to maintain inflated global debt records that never decrease, leading to accounting corruption that compounds with each liquidation. Also `breaks` the `fundamental invariant` that `cumulativeEarmarked` should equal the sum of all account earmarked debts\`

## References

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L738>

## Proof of Concept

## Proof of Concept

Cooked a poc for it!

Include this test in AlchemistV3.t.sol and execute it with `forge test --mt testVulnerability_CumulativeEarmarkedNotUpdatedInForceRepay -vv`

```solidity
function testVulnerability_CumulativeEarmarkedNotUpdatedInForceRepay() external {
        // Setup: Create one account that will be liquidated
        uint256 depositAmount = 100_000e18;
        uint256 mintAmount = 50_000e18;
        
        // Account 1 (0xbeef) - will be liquidated
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId1, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Create a redemption to earmark the debt
        deal(address(alToken), address(0xdead), mintAmount);
        vm.startPrank(address(0xdead));
        IERC20(alToken).approve(address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.roll(vm.getBlockNumber() + 5_256_000 / 2); // Only roll halfway so not all debt is claimed yet
        vm.stopPrank();

        // Poke to earmark the debt without claiming the redemption
        alchemist.poke(tokenId1);

        // Check state after earmark but before claim
        (, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenId1);
        uint256 cumulativeEarmarkedBefore = alchemist.cumulativeEarmarked();
        
        // Verify debt is earmarked
        assertGt(earmarkedBefore, 0, "Account should have earmarked debt");
        assertEq(cumulativeEarmarkedBefore, earmarkedBefore, "Cumulative should match account earmark");

        // Crash the yield token price to make account undercollateralized
        // 90% drop to force liquidation
        _manipulateYieldTokenPrice(9000);

        // Liquidate account - this will trigger _forceRepay for earmarked debt
        // The bug: _forceRepay reduces account.earmarked but does NOT reduce cumulativeEarmarked
        vm.prank(someWhale);
        alchemist.liquidate(tokenId1);

        // Get state after liquidation
        (, uint256 debtAfter, uint256 earmarkedAfter) = alchemist.getCDP(tokenId1);
        uint256 cumulativeEarmarkedAfter = alchemist.cumulativeEarmarked();

        // Account earmarked should have decreased after force repay
        assertLt(earmarkedAfter, earmarkedBefore, "Account earmarked should decrease after liquidation");
        
        // BUG DEMONSTRATION: cumulativeEarmarked did NOT decrease proportionally
        // It should have decreased by the same amount as the account's earmarked
        uint256 earmarkedRepaid = earmarkedBefore - earmarkedAfter;
        uint256 expectedCumulativeEarmarked = cumulativeEarmarkedBefore - earmarkedRepaid;
        
        // This demonstrates the bug: cumulativeEarmarked was not updated
        assertGt(
            cumulativeEarmarkedAfter,
            expectedCumulativeEarmarked,
            "cumulativeEarmarked should have decreased by the force-repaid amount"
        );

        // Calculate the discrepancy
        uint256 earmarkedDiscrepancy = cumulativeEarmarkedAfter - earmarkedAfter;
   
        console.log("Account earmarked BEFORE liquidation:", earmarkedBefore);
        console.log("Account earmarked AFTER liquidation:", earmarkedAfter);
        console.log("Amount force-repaid from account:", earmarkedRepaid);
        console.log("");
        console.log("Cumulative earmarked BEFORE:", cumulativeEarmarkedBefore);
        console.log("Cumulative earmarked AFTER (actual):", cumulativeEarmarkedAfter);
        console.log("Cumulative earmarked AFTER (expected):", expectedCumulativeEarmarked);
        console.log("");
        console.log("DISCREPANCY (lost tracking):", earmarkedDiscrepancy);
    }
```

* **Test Logs**

```bash
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testVulnerability_CumulativeEarmarkedNotUpdatedInForceRepay() (gas: 3209706)
Logs:
  Account earmarked BEFORE liquidation: 25000000000000000000000
  Account earmarked AFTER liquidation: 0
  Amount force-repaid from account: 25000000000000000000000
  
  Cumulative earmarked BEFORE: 25000000000000000000000
  Cumulative earmarked AFTER (actual): 25000000000000000000000
  Cumulative earmarked AFTER (expected): 0
  
  DISCREPANCY (lost tracking): 25000000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 33.76ms (7.47ms CPU time)

Ran 1 test suite in 42.69ms (33.76ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
