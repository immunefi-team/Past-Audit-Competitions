# 56887 sc low incorrect balance tracking in morphoyearnogwethstrategy deallocate function leads to wrong loss event emission resend&#x20;

**Submitted on Oct 21st 2025 at 14:36:10 UTC by @Snuggle for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56887
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `_deallocate` function in `src/strategies/mainnet/MorphoYearnOGWETH.sol` contains a logic bug where the balance before withdrawal is captured AFTER the withdrawal has already occurred. This leads to incorrect loss event emissions where `wethRedeemed` will always be 0, making the loss detection mechanism ineffective for tracking actual losses.

## Vulnerability Details

* The `_deallocate` function captures the balance before withdrawal AFTER the withdrawal has already happened:

```49:61:src/strategies/mainnet/morphoyearnogweth.sol
    function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

* Line 50: `vault.withdraw()` is called first
* Line 51: `wethBalanceBefore` is captured AFTER withdrawal (should be before)
* Line 52: `wethBalanceAfter` is captured immediately after, so it equals `wethBalanceBefore`
* Line 53: `wethRedeemed` will always be 0 because both balances are identical
* Lines 57-58: Redundant require statements similar to Euler strategies

## Impact Details

* **Incorrect loss event emission**: `wethRedeemed` will always be 0, so loss events will never trigger when they should
* **Production confusion**: Wrong events can mislead operators, monitoring systems, and users about actual strategy performance
* **Broken loss tracking**: The loss detection mechanism cannot properly track actual losses during deallocation
* **Monitoring issues**: Dashboards and alerts may show incorrect data, leading to wrong operational decisions
* **Gas inefficiency**: Redundant require statements (lines 57-58) perform identical checks
* **Code maintainability**: Logic is confusing and error-prone
* **Severity**: Low (wrong events can cause significant production confusion and operational issues)

## References

* `src/strategies/mainnet/MorphoYearnOGWETH.sol`:
  * Incorrect balance capture after withdrawal

```51:51:src/strategies/mainnet/morphoyearnogweth.sol
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
```

* Redundant require statements

```57:58:src/strategies/mainnet/morphoyearnogweth.sol
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

## Recommendation

Fix the balance tracking logic and remove redundant require statements:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    vault.withdraw(amount, address(this), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

This fixes the balance tracking logic and eliminates redundant gas consumption while maintaining security guarantees.

## Link to Proof of Concept

<https://gist.github.com/Snuggle20040629/4fba5a5c59dc463581707549233f4d4d>

## Proof of Concept

## Proof of Concept

### Runnable PoC Test

A complete runnable PoC has been implemented in `src/test/strategies/MorphoYearnOGWETHStrategy.t.sol`:

```solidity
function test_deallocate_balance_tracking_bug() public {
    uint amountToAllocate = 1e18;
    uint amountToDeallocate = 1;
    vm.startPrank(vault);
    deal(WETH, strategy, amountToAllocate);
    bytes memory prevAllocationAmount = abi.encode(0);
    IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
    uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
    require(initialRealAssets > 0, "Initial real assets is 0");
    bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
    
    // Record all events during the deallocate call
    vm.recordLogs();

    IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
  
    vm.stopPrank();
    
    // Get all recorded logs
    Vm.Log[] memory logs = vm.getRecordedLogs();
    bytes32 strategyDeallocationLossEventSignature = keccak256("StrategyDeallocationLoss(string,uint256,uint256)");
    
    uint256 eventCount = 0;
    bool foundWrongEvent = false;
    for (uint256 i = 0; i < logs.length; i++) {
        if (logs[i].topics[0] == strategyDeallocationLossEventSignature) {
            eventCount++;
            
            // Decode the event data to get wethRedeemed
            (string memory message, uint256 amount, uint256 wethRedeemed) = abi.decode(logs[i].data, (string, uint256, uint256));
            
            console.log("Found StrategyDeallocationLoss event:");
            console.log("  Message:", message);
            console.log("  Amount:", amount);
            console.log("  wethRedeemed:", wethRedeemed);
            
            // The bug: wethRedeemed should be the actual loss, but it's always 0
            if (wethRedeemed == 0) {
                console.log("BUG CONFIRMED: wethRedeemed is 0 (should show actual loss)");
                foundWrongEvent = true;
            }
        }
    }
    
    // Assert that we found the wrong event
    assertTrue(foundWrongEvent, "StrategyDeallocationLoss event with wethRedeemed = 0 not found");
    console.log("Bug proven: Wrong event emitted with wethRedeemed = 0");
}
```

### Running the PoC

Execute the test with mainnet fork to demonstrate the bug:

```bash
$env:FOUNDRY_PROFILE="default"; $env:MAINNET_RPC_URL="https://mainnet.gateway.tenderly.co"; forge test --fork-url https://mainnet.gateway.tenderly.co --match-test test_deallocate_balance_tracking_bug -vv
```

### Expected Output

```
[PASS] test_deallocate_balance_tracking_bug() (gas: 1209699)
Logs:
  Found StrategyDeallocationLoss event:
    Message: Strategy deallocation loss.
    Amount: 1
    wethRedeemed: 0
  BUG CONFIRMED: wethRedeemed is 0 (should show actual loss)
  Bug proven: Wrong event emitted with wethRedeemed = 0
```
