# 58120 sc low incorrect balance measurement in morphoyearnogweth strategy leads to incorrect deallocation loss registering

**Submitted on Oct 30th 2025 at 19:18:19 UTC by @ByteKnight for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58120
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `_deallocate()` function in the *MorphoYearnOGWETHStrategy* contract has a logic flaw where both `wethBalanceBefore` and `wethBalanceAfter` variables are measured after the withdrawal operation, causing `wethRedeemed` variable to always be calculated as 0. While this doesn't result in loss of funds, it causes all deallocations to emit incorrect `StrategyDeallocationLoss` events with `actualAmountSent: 0` which breaks the intended validation logic and forces off-chain infrastructure to incorrectly log all successful deallocations as failures and present misleading loss information in monitoring dashboards and user interfaces.

## Vulnerability Details

The balance measurements are both taken after the withdrawal, causing the delta calculation to always be zero:

```sol
function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
    @>  uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    @>  uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        require(
            TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount,
            "Strategy balance is less than the amount needed"
        );
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

As you can see, function first withdraws the allocated WETH tokens from the vault and then measures WETH balance of the calling contract. And right after that it calculates the `wethRedeemed` by subtracting the WETH before withdraw from after, which in these case are equal and then emits the `StrategyDeallocationLoss` event, even thou the withdraw might be successful and without any loss or even with bigger amount.

## Impact Details

This logic flaw leads to a situation where the `StrategyDeallocationLoss` event will be emitted for each deallocation operation with false information that strategy loss is 100% which isn't true and :

* Any off-chain monitoring systems will incorrectly think all deallocations are failing
* Off-chain loss tracking and accounting will be completely wrong
* Users and administrators cannot trust the event data for monitoring strategy performance
* Every successful deallocation generates a false alert
* Any real loss in strategy won't be correctly logged and addressed by entitled party

## References

* Buggy code in contract: [MorphoYearnOGWETHStrategy.sol](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/MorphoYearnOGWETH.sol#L49-L61)

## Proof of Concept

## Proof of Concept

To verify this flaw actually exists, do the following steps:

1. Add the test below to the existing test suite in **MorphoYearnOGWETHStrategy.t.sol** (located at `src/test/strategies/MorphoYearnOGWETHStrategy.t.sol`):

* [Test file](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/test/strategies/MorphoYearnOGWETHStrategy.t.sol)

```sol
/**
     * @notice Fuzz test demonstrating the balance measurement issue in _deallocate()
     * @dev This test proves that the MorphoYearnOGWETH._deallocate() function contains a
     *      logic flow where both wethBalanceBefore and wethBalanceAfter are measured AFTER the
     *      withdrawal, causing wethRedeemed to always be calculated as 0.
     *
     *      This test confirms the issue by:
     *      1. Successfully allocating WETH to the strategy
     *      2. Successfully deallocating the requested amount
     *      3. Proving that despite the successful withdrawal, the event shows actualAmountSent = 0
     */
    function testFuzz_strategy_deallocate_incorrect_loss_reporting(uint256 amountToAllocate) public {
        // ============================================================================
        // SETUP: Prepare a successful allocation scenario
        // ============================================================================

        // Bound to reasonable values
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);

        vm.startPrank(vault);

        // Give the strategy some WETH tokens to work with
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);

        // ============================================================================
        // STEP 1: Allocate funds into the MorphoYearnOGWETH vault
        // ============================================================================

        // This should succeed - we're depositing WETH into the Morpho Yearn vault
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));

        // Verify allocation succeeded (strategy now holds assets in the vault)
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");

        // ============================================================================
        // STEP 2: Deallocate funds (this is where the issues occurs)
        // ============================================================================

        // Calculate how much we can withdraw (accounting for any yield/loss)
        uint256 amountToDeallocate = IMYTStrategy(strategy).previewAdjustedWithdraw(amountToAllocate);
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);

        // Start recording all events emitted during deallocation
        vm.recordLogs();

        // Execute the deallocation - this will succeed and funds are withdrawn
        // but the balance measurement flow will cause incorrect event emission
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));

        // ============================================================================
        // STEP 3: Check the issue - verify the event shows incorrect data
        // ============================================================================

        // Retrieve all emitted events
        Vm.Log[] memory logs = vm.getRecordedLogs();

        // Search for the StrategyDeallocationLoss event
        for (uint256 i = 0; i < logs.length; i++) {
            if (logs[i].topics[0] == keccak256("StrategyDeallocationLoss(string,uint256,uint256)")) {
                // Decode the event: (message, requestedAmount, actualAmountSent)
                (, , uint256 actualAmountSent) = abi.decode(logs[i].data, (string, uint256, uint256));

                // Check that despite the withdrawal succeeding, actualAmountSent shows 0
                assertEq(actualAmountSent, 0);
            }
        }
        vm.stopPrank();
    }
```

2. Run the test by executing the following script in terminal and pasting your fork url:

```
forge test --fork-url <YOUR_FORK_URL> --mt testFuzz_strategy_deallocate_incorrect_loss_reporting
```
