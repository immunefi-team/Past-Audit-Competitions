# 58497 sc low the amount of weth redeemed is not calculated properly in morphoyearnogweth

**Submitted on Nov 2nd 2025 at 19:25:54 UTC by @Josh4324 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58497
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The amount of WETH redeemed is not accurately calculated; this will make the WETH redeemed always zero, and the contract will always emit `StrategyDeallocationLoss(` when it didn't deallocate at a loss.

## Vulnerability Details

The root cause of the issue is that the `vault.withdraw` function is called before the `wethBalanceBefore` value is queried. This means that `wethBalanceBefore` and `wethBalanceAfter` is always equal and `uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;` will always be zero.

```sol
    function _deallocate(uint256 amount) internal override returns (uint256) {
@->        vault.withdraw(amount, address(this), address(this));
@->        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
@->        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this)); 
@->        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }

```

Since `wethRedeemed` is always zero, this will always be emitted.

```sol
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
```

## Impact Details

The current impact is that any off-chain system that is listening to this `StrategyDeallocationLoss` event will always think that the contract is at a loss.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol?utm\\_source=immunefi#L50>

## Proof of Concept

## Proof of Concept

Add to `test/strategy/MorphoYearnOGWETHStrategy.t.sol`

```sol
interface IERC20Lite {
    function balanceOf(address) external returns (uint);
}
```

```sol
    // Declare the event so the test can emit an expected log
    event StrategyDeallocationLoss(string message, uint256 amount, uint256 loss);
    function test_redeem_poc(uint256 amountToAllocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        uint amountToDeallocate = amountToAllocate / 2;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));

        // Get Balance of Strategy before deallocation
        uint balanceBefore = IERC20Lite(WETH).balanceOf(address(strategy));

        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);

        vm.expectEmit(true, true, true, true);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amountToDeallocate, 0);

        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();

        // Get Balance of Strategy before deallocation
        uint balanceAfter = IERC20Lite(WETH).balanceOf(address(strategy));

        console2.log("STRATEGY BALANCE Before deallocation: ", balanceBefore);

        console2.log("STRATEGY BALANCE After  deallocation: ", balanceAfter);

        console2.log("Balance Redeemed                    : ", balanceAfter - balanceBefore);
    }
```

***OUTPUT***

```bash
   ├─ [534] 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2::balanceOf(MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1])
    │   └─ ← [Return] 499500000000000002919 [4.995e20]
    ├─ [0] console::log("STRATEGY BALANCE Before deallocation: ", 0) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("STRATEGY BALANCE After  deallocation: ", 499500000000000002919 [4.995e20]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Balance Redeemed                    : ", 499500000000000002919 [4.995e20]) [staticcall]
    │   └─ ← [Stop]
    └─ ← [Stop]
```
