# 58324 sc high incorrect return value in deallocate function leads to permanent fund locking in mytstrategy implementations

**Submitted on Nov 1st 2025 at 09:18:23 UTC by @fullstop for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58324
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The \_deallocate function in several strategy contracts, most notably TokeAutoEthStrategy and TokeAutoUSDStrategy, incorrectly returns the requested amount instead of the actual asset amount redeemed (wethRedeemed or usdcRedeemed). This bug is triggered when the "dust-handling" logic is activated by a deallocation request that is very close to the strategy's total holdings. The VaultV2 (which trusts this return value) only withdraws the reported amount, causing the excess redeemed assets to be permanently locked and lost in the strategy contract.

## Vulnerability Details

The MYTStrategy contract requires child contracts to implement the \_deallocate function. In the TokeAutoEthStrategy and TokeAutoUSDStrategy contracts, this function contains a "dust-handling" feature designed to redeem all remaining shares if the requested deallocation is close to the total shares held.

This logic itself is sound, but its interaction with the function's return value is flawed.

Let's analyze TokeAutoEthStrategy.sol's \_deallocate function:

1. Calculates Share Difference: The contract calculates sharesNeeded based on the requested amount and compares it to actualSharesHeld.
2. Triggers Dust Logic: It enters a conditional block if the difference is small: `if (shareDiff <= 1e18)`.
3. Redeems All: Inside this block, it sets `sharesNeeded = actualSharesHeld`, forcing the strategy to redeem 100% of its shares from the rewarder and autoEth vault.
4. Calculates Actual Redemption: The contract correctly calculates the actual amount of WETH received from this 100% redemption: `uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;`.

5, The Flaw: The function then proceeds to its end, but instead of using the wethRedeemed value, it does the following:

`TokenUtils.safeApprove(address(weth), msg.sender, amount);`: It approves the original requested amount for withdrawal by the VaultV2.

`return amount;`: It reports to the VaultV2 that the original requested amount has been successfully deallocated.

This exact logic flaw is duplicated in `TokeAutoUSDStrategy.sol`'s \_deallocate function. It also has the `if (shareDiff <= 1e18)` block, calculates usdcRedeemed, but then incorrectly approves and returns the requested amount.

## Impact Details

This vulnerability leads to a permanent loss of user funds locked within the strategy contract.

Here is the attack/exploit scenario:

1. A deallocation is requested from the VaultV2 with an amount that is very close (but not equal) to the strategy's total realAssets.
2. The strategy's \_deallocate is called. The "dust" logic triggers, causing the strategy to redeem 100% of its shares, resulting in an actual redeemed amount (wethRedeemed) that is greater than the requested amount.
3. The strategy incorrectly returns amount to the VaultV2.
4. The VaultV2 trusts this return value and updates its internal accounting, believing amount was deallocated.
5. The VaultV2 calls `transferFrom(strategy, address(this), amount)` to pull the assets.
6. The VaultV2 only pulls the amount it was told, as this is all the strategy safeApproved.
7. Result: The excess funds (i.e., `wethRedeemed - amount`) are left stranded in the strategy contract. The VaultV2's accounting is consistent (from its perspective), and it has no mechanism to recover these orphaned funds. The funds are permanently lost.

## References

Vulnerable Contract 1: src/strategies/mainnet/TokeAutoEth.sol

Vulnerable Contract 2: src/strategies/mainnet/TokeAutoUSDStrategy.sol

## Proof of Concept

## Proof of Concept

A runnable Proof of Concept test, test\_reproduce\_locked\_funds\_vulnerability, was added to TokeAutoETHStrategy.t.sol and successfully passed, confirming the vulnerability.

Test Logic:

Setup: The test first allocates 1e18 (1 WETH) to the TokeAutoEthStrategy from the main VaultV2.

Trigger: It calculates the strategy's realAssets and defines an amountToDeallocate that is slightly less than realAssets (e.g., `realAssets - 3e13`), ensuring the requested amount is small enough to pass the final require check but large enough to trigger the "dust" logic.

Bug Execution: The test calls `IVaultV2(vault).deallocate(..., amountToDeallocate)`.

This forces the strategy's \_deallocate to hit the `if (shareDiff <= 1e18)` block.

The strategy redeems 100% of its shares, receiving the full wethRedeemed.

The strategy incorrectly approves and returns the requested amountToDeallocate.

Result & Validation:

The VaultV2 pulls only the requested amountToDeallocate.

The test then checks the WETH balance of the strategy contract: `uint256 lockedFunds = IERC20(testConfig.vaultAsset).balanceOf(strategy);`

The test's final assertion, `assertGt(lockedFunds, 0, "Funds should be locked in the strategy contract");`, passes, proving that funds (`wethRedeemed - amountToDeallocate`) are left locked in the strategy contract.

```
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";

    /**
     * @notice Test to reproduce the locked funds vulnerability caused by _deallocate returning an incorrect amount.
     * @dev Vulnerability Cause: _deallocate has dust-handling logic.
     * When the requested deallocation amount is very close to the total shares held,
     * it redeems ALL shares.
     * However, the function incorrectly returns the requested `amount`
     * instead of the actual `wethRedeemed`.
     * This causes the Vault to only pull back the `amount`,
     * leaving the excess redeemed assets locked in the strategy contract forever.
     */
    function test_reproduce_locked_funds_vulnerability() public {
        uint256 amountToAllocate = 1e18; // Allocate 1 WETH

        // --- 1. Allocate ---
        // AlchemistAllocator (set up in BaseStrategyTest) calls vault.allocate
        vm.startPrank(allocator); //
        bytes32 allocationId = IMYTStrategy(strategy).adapterId(); //
        bytes memory prevAllocationAmount = abi.encode(0); //
        IVaultV2(vault).allocate(strategy, prevAllocationAmount, amountToAllocate); //
        vm.stopPrank();

        // Verify assets are in the strategy
        uint256 realAssets = IMYTStrategy(strategy).realAssets();
        // Deposit might have tiny slippage/rounding
        // Allow for slippage on deposit, which was ~2.9e13 wei.
        uint256 slippageDelta = 300000000000000; // 3e14
        assertApproxEqAbs(realAssets, amountToAllocate, slippageDelta, "realAssets mismatch after allocation");

        // Ensure WETH is not on the strategy contract (it's in the Tokemak rewarder)
        assertEq(IERC20(testConfig.vaultAsset).balanceOf(strategy), 0, "Strategy contract should not hold WETH after allocation");

        // --- 2. Deallocate (almost all) ---
        // We request an amount 100 wei less than realAssets.
        // This is small enough to trigger the `shareDiff <= 1e18` logic.
        // We must request an amount that is less than what is *actually* redeemed (wethRedeemed),
        // not just less than the preview (realAssets).
        // The redemption slippage from the trace was ~2.9e13 wei.
        // Let's subtract 3e13 to be safe, which is still small enough to trigger the dust logic.
        uint256 slippageToAvoidRevert = 30000000000000; // 3e13
        uint256 amountToDeallocate = realAssets - slippageToAvoidRevert;

        vm.startPrank(allocator); //
        uint256 currentAllocation = IVaultV2(vault).allocation(allocationId); //
        prevAllocationAmount = abi.encode(currentAllocation); //

        // Vault calls strategy.deallocate. Strategy will redeem all assets
        // but return `amountToDeallocate`.
        // Vault will then `transferFrom` only `amountToDeallocate` WETH.
        IVaultV2(vault).deallocate(strategy, prevAllocationAmount, amountToDeallocate); //
        vm.stopPrank();

        // --- 3. Verify ---
        // Strategy redeemed `realAssets`, but Vault only took `amountToDeallocate`.
        // The difference (100 wei) should be locked in the strategy contract.

        uint256 lockedFunds = IERC20(testConfig.vaultAsset).balanceOf(strategy);
        
        // Use assertApproxEqAbs because realAssets itself might have rounding
        assertGt(lockedFunds, 0, "Funds should be locked in the strategy contract");

        // Strategy's realAssets() (checking the rewarder) should be 0, as all shares were withdrawn.
        uint256 finalRealAssets = IMYTStrategy(strategy).realAssets();
        assertEq(finalRealAssets, 0, "Strategy realAssets should be 0");

        // Vault's internal accounting should be correct (relative to what it thinks it deallocated)
        uint256 expectedVaultAllocation = currentAllocation - amountToDeallocate;
        assertApproxEqAbs(IVaultV2(vault).allocation(allocationId), expectedVaultAllocation, 2, "Vault allocation tracking is incorrect"); //
    }
```
