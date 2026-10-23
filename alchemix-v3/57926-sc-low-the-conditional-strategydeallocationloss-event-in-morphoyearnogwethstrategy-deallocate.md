# 57926 sc low the conditional strategydeallocationloss event in morphoyearnogwethstrategy deallocate gets logged all the time due a misplacement in variable declaration

**Submitted on Oct 29th 2025 at 13:39:53 UTC by @TyroneX for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57926
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

A misplacement in syntax makes a conditional event trigger all the time, which could lead to issues in on-chain/off-chain mechanisms listening to it

## Vulnerability Details

In the MorphoYearnOGWETHStrategy contract, `StrategyDeallocationLoss` event is logged all the time due to `wethBalanceBefore` and `wethBalanceAfter` being the same value. This happened cause `wethBalanceBefore` is declared after the withdrawal is triggered rather than before. This means `wethBalanceBefore` and `wethBalanceAfter` are the same value hence, `wethRedeemed`, a variable which depends on the difference of both always returns 0. This in turn makes `StrategyDeallocationLoss`, which depends on `wethRedeemed`, always return true. Hence the event always triggers leaving any onchain/offchain variables listening to it susceptible to error.

## Impact Details

The function executes properly regardless, and no immediate funds are at risk; however, other mechanisms that listen to the `StrategyDeallocationLoss` may be at risk as the third parameter, `wethRedeemed` would always return 0.

## References

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
@>      uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
@>      uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
@>          emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
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

## Proof of Concept

## Proof of Concept

Place the following into MorphoYearnOGWETHStrategyTest.t.sol

```solidity
    contract MockMorphoYearnOGWETHStrategy is MorphoYearnOGWETHStrategy {

        constructor(
            address _myt,
            StrategyParams memory _params,
            address _vault,
            address _weth,
            address _permit2Address
        ) MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address) {}

        // expose deallocate
        function test_deallocate(uint256 amount) external returns (uint256 amountDeallocated) {
            return _deallocate(amount);
        }
    }
```

Place this in MorphoYearnOGWETHStrategyTest contract

```solidity
    // from MYTStrategy
    event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

    function test_deallocate_always_emits_log(uint256 amount) public {
        amount = bound(amount, 1e18, testConfig.vaultInitialDeposit);

        vm.startPrank(vault);

        // Fund strategy
        deal(WETH, strategy, amount);

        // Allocate
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amount, "", address(vault));

        // deallocate
        // vm.expectEmit(address(strategy));
        vm.expectEmit(false, false, false, false, address(strategy));
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, 0);
        // foundrys ecpectEmit has a hard time comparing strings but stack trace shows that event was truly emitted
        MockMorphoYearnOGWETHStrategy(strategy).test_deallocate(amount);

        vm.stopPrank();
    }
    
```

This will result in an error due to Foundry's vm.expectEmit, which struggles to compare dynamic types like strings and bytes (when comparing the actual emitted string and the expected emitted string). But from the stack trace, we can see that an event was truly logged at that point. This occurs for every possible value of amount

```rust
    │   │   ├─ emit UpdateLastTotalAssets(: 10004799211647145971406 [1e22])
    │   │   └─ ← [Return] 983837784620636161868 [9.838e20]
    │   ├─ emit Allocate(amount: 998000000000000001290 [9.98e20], strategy: MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1])
    │   └─ ← [Return] [0xb0d2d9d9301af6eca734344f3f06b0a7c5e71828c144699c71395cc98682e276], 998000000000000001290 [9.98e20]
    ├─ [0] VM::expectEmit(false, false, false, false, MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1])
    │   └─ ← [Return]
@>>  ├─ emit StrategyDeallocationLoss(message: "Strategy deallocation loss.", amountRequested: 998000000000000001290 [9.98e20], actualAmountSent: 0)
    ├─ [210297] MockMorphoYearnOGWETHStrategy::test_deallocate(998000000000000001290 [9.98e20])
    │   ├─ [209082] 0xE89371eAaAC6D46d4C3ED23453241987916224FC::withdraw(998000000000000001290 [9.98e20], MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], MockMorphoYearnOGWETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1])
```
