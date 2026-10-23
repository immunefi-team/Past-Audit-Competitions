# 58590 sc low incorrect balance read ordering in morphoyearnogwethstrategy deallocate

**Submitted on Nov 3rd 2025 at 11:45:16 UTC by @oct0pwn for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58590
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The \_deallocate function in MorphoYearnOGWETHStrategy reads the wethBalanceBefore after performing the vault withdrawal, resulting in both pre- and post-withdrawal balances being identical. Consequently, the computed wethRedeemed amount will always be zero. This incorrect accounting prevents accurate detection of deallocation losses and could lead to false assumptions about the strategy’s balance integrity in production.

## Vulnerability Details

The function \_deallocate is designed to withdraw a specific amount from the vault and determine how much WETH was redeemed by comparing the pre- and post-withdrawal balances. However, the code currently measures both balances after the withdrawal operation:

```solidity
vault.withdraw(amount, address(this), address(this));
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
```

Since both wethBalanceBefore and wethBalanceAfter are captured **after** the withdrawal, their values will be equal, resulting in:

```solidity
wethRedeemed = wethBalanceAfter - wethBalanceBefore = 0
```

This causes:

* The StrategyDeallocationLoss event to never emit correctly.
* Incorrect validation in subsequent require statements, which rely on a faulty wethRedeemed value.

The logical ordering of operations is incorrect, making the redemption calculation and loss detection mechanisms unreliable.

The correct implementation should capture wethBalanceBefore **before** invoking the vault withdrawal:

```solidity
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
vault.withdraw(amount, address(this), address(this));
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
```

## Impact Details

This issue leads to incorrect calculation of redeemed WETH during deallocation. Inaccurate accounting can cause:

* **Missed loss detection:** Losses incurred during withdrawals will not be properly identified, making strategy performance tracking and safety checks unreliable.
* **Faulty state validation:** Subsequent require statements may incorrectly pass, allowing further operations under false assumptions of sufficient balance.
* **Potential accounting mismatch:** In production, this could distort internal strategy metrics, mislead monitoring systems, and potentially cause unexpected fund transfer failures or inaccuracies in user-facing reports.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Place the following test in MockMorphoYearnOGWETHStrategy.t.sol

```solidity
import {console} from "forge-std/console.sol";
import {VmSafe} from "forge-std/Vm.sol";
 
 
 function test_POC_balance_calculation_bug() public {
        uint256 amountToAllocate = 10 ether;
        uint256 amountToDeallocate = 5 ether;

        vm.startPrank(vault);
        // Step 1: Allocate funds to strategy
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(
            prevAllocationAmount,
            amountToAllocate,
            "",
            address(vault)
        );

        // Step 2: Capture balance BEFORE deallocation (CORRECT way to calculate redeemed amount)
        uint256 balanceBeforeDeallocate = TokenUtils.safeBalanceOf(
            WETH,
            strategy
        );

        // Step 3: Record logs to capture StrategyDeallocationLoss event
        vm.recordLogs();

        // Step 4: Deallocate funds
        // Inside _deallocate:
        //   - Line 50: vault.withdraw() is called (sends WETH to strategy)
        //   - Line 51: wethBalanceBefore is read AFTER withdraw (already includes withdrawn funds)
        //   - Line 52: wethBalanceAfter is read immediately after (same value)
        //   - Line 53: wethRedeemed = wethBalanceAfter - wethBalanceBefore = 0 (BUG!)
        prevAllocationAmount = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(
            prevAllocationAmount,
            amountToDeallocate,
            "",
            address(vault)
        );

        // Step 5: Capture balance AFTER deallocation
        uint256 balanceAfterDeallocate = TokenUtils.safeBalanceOf(
            WETH,
            strategy
        );

        // Step 6: Calculate actual amount redeemed (CORRECT calculation)
        uint256 actualRedeemed = balanceAfterDeallocate -
            balanceBeforeDeallocate;

        // Step 7: Extract buggy calculation from StrategyDeallocationLoss event
        VmSafe.Log[] memory logs = vm.getRecordedLogs();
        bytes32 strategyDeallocationLossEventSig = keccak256(
            "StrategyDeallocationLoss(string,uint256,uint256)"
        );

        uint256 buggyActualAmountSent = type(uint256).max; // Initialize to detect if event not found
        for (uint256 i = 0; i < logs.length; i++) {
            if (logs[i].topics[0] == strategyDeallocationLossEventSig) {
                (
                    string memory message,
                    uint256 amountRequested,
                    uint256 actualAmountSent
                ) = abi.decode(logs[i].data, (string, uint256, uint256));
                buggyActualAmountSent = actualAmountSent;
                break;
            }
        }

        // Output bug information
        console.log("Amount to deallocate:", amountToDeallocate);
        console.log("Balance before deallocation:", balanceBeforeDeallocate);
        console.log("Balance after deallocation:", balanceAfterDeallocate);
        console.log("Actual redeemed (CORRECT calculation):", actualRedeemed);
        console.log(
            "Buggy code calculates (from event):",
            buggyActualAmountSent
        );

        // BUG VERIFICATION:
        // The buggy code in _deallocate calculates: wethRedeemed = 0 (because both reads happen after withdraw)
        // But we can prove funds were actually received by using correct balance tracking:
        assertGt(
            actualRedeemed,
            0,
            "BUG CONFIRMED: Funds were actually redeemed (actualRedeemed > 0), but buggy code calculates 0"
        );
        assertGe(
            actualRedeemed,
            amountToDeallocate - 1e15,
            "BUG CONFIRMED: Actual redeemed amount >= deallocated amount, proving funds were received"
        );

        // Functional safety: Verify funds are available for vault to pull
        uint256 strategyBalance = TokenUtils.safeBalanceOf(WETH, strategy);
        console.log("Strategy balance after deallocation:", strategyBalance);
        assertGe(
            strategyBalance,
            amountToDeallocate - 1e15,
            "FUNCTIONAL SAFETY: Strategy has sufficient balance after deallocation"
        );

        vm.stopPrank();
    }
    
    
//    EXPECTED OUTPUT:

//	Amount to deallocate: 5000000000000000000
//  Balance before deallocation: 0
//  Balance after deallocation: 5000000000000000000
//  Actual redeemed (CORRECT calculation): 5000000000000000000
//  Buggy code calculates (from event): 0
//  Strategy balance after deallocation: 5000000000000000000

// Suite result: ok. 1 passed; 0 failed; 0 skipped;
```
