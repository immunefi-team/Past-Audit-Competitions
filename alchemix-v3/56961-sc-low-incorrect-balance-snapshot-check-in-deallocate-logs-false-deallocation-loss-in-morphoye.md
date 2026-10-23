# 56961 sc low incorrect balance snapshot check in deallocate logs false deallocation loss in morphoyearnogweth strategy

**Submitted on Oct 22nd 2025 at 08:19:04 UTC by @Bluedragon for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56961
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary:

The `_deallocate` function in `MorphoYearnOGWETH.sol` measures WETH redeemed by taking a balance snapshot incorrectly (it reads the same balance twice after calling `vault.withdraw`). As a result, `wethRedeemed` is always zero, which causes the strategy to emit a `StrategyDeallocationLoss` event on successful deallocations.

## Vulnerability Details:

In the `_deallocate` function, the code captures the WETH balance snapshot after calling `vault.withdraw`, leading to an incorrect calculation of `wethRedeemed`. Thus, the strategy always believes there is a loss during deallocation, even when there isn't and emits the `StrategyDeallocationLoss` event incorrectly.

Code Snippet:

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        // @audit-insight incorrect logic: should check balance before and after withdraw
        // The wethRedeemed will always be 0, thus incorrect deallocation log is emitted.
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        console.log("WETH Redeemed:", wethRedeemed);
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

## Impact

* The strategy incorrectly emits `StrategyDeallocationLoss` events when no loss occurred.
* May cause unnecessary reverts or alerts in event monitoring services.
* Breaks accounting and invariants that depend on accurate deallocation events and reporting.

## Proof of Concept

## Proof of Concept:

1. Add the following test case to the `MorphoYearnOGWETHStrategy.t.sol`
2. Run the test using the command `forge test --mt test_bluedragon_startegy_deallocation_always_fails_due_to_incorrect_logic -vvvv`
3. Observe that the `StrategyDeallocationLoss` event is emitted even when there is no actual loss.

**POC Test Case:**

```solidity
    function test_bluedragon_startegy_deallocation_always_fails_due_to_incorrect_logic() public {
        uint256 amountToAllocate = 10000e18;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        uint256 amountToDeallocate = IMYTStrategy(strategy).realAssets();
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }
```

**Logs:**

```rs
│   │   ├─ emit Withdraw(sender: MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], receiver: MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], owner: MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], assets: 9999999999999999999999 [9.999e21], shares: 9858094034274911428821 [9.858e21])
    │   │   └─ ← [Return] 9858094034274911428821 [9.858e21]
    │   ├─ [534] 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2::balanceOf(MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1]) [staticcall]
    │   │   └─ ← [Return] 9999999999999999999999 [9.999e21]
    │   ├─ [534] 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2::balanceOf(MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1]) [staticcall]
    │   │   └─ ← [Return] 9999999999999999999999 [9.999e21]
@-> │   ├─ emit StrategyDeallocationLoss(message: "Strategy deallocation loss.", amountRequested: 9999999999999999999999 [9.999e21], actualAmountSent: 0)
    │   ├─ [534] 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2::balanceOf(MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1]) [staticcall]
    │   │   └─ ← [Return] 9999999999999999999999 [9.999e21]
    │   ├─ [24420] 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2::approve(MockMYTVault: [0x522B3294E6d06aA25Ad0f1B8891242E335D3B459], 9999999999999999999999 [9.999e21])
    │   │   ├─ emit Approval(owner: MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], spender: MockMYTVault: [0x522B3294E6d06aA25Ad0f1B8891242E335D3B459], value: 9999999999999999999999 [9.999e21])
    │   │   └─ ← [Return] true
    │   ├─ emit Deallocate(amount: 9999999999999999999999 [9.999e21], strategy: MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1])
    │   └─ ← [Return] [0xb0d2d9d9301af6eca734344f3f06b0a7c5e71828c144699c71395cc98682e276], -9999999999999999999999 [-9.999e21]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    └─ ← [Stop]
```

## Recommended Mitigation:

Correct the snapshot order. Capture `wethBalanceBefore` before calling `vault.withdraw`, and `wethBalanceAfter` after the withdraw.
