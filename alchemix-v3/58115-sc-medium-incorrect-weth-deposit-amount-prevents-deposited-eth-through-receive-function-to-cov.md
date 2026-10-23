# 58115 sc medium incorrect weth deposit amount prevents deposited eth through receive function to cover strategy loss&#x20;

**Submitted on Oct 30th 2025 at 18:25:20 UTC by @Tadev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58115
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The StargateEthPoolStrategy contract defines the `_deallocate` function as follows:

```
 function _deallocate(uint256 amount) internal override returns (uint256) {
        // Compute LP needed ∝ TVL to withdraw `amount` underlying
        // For Stargate, LP tokens are 1:1 with underlying
        // So we can just redeem the amount directly
        uint256 lpBalance = lp.balanceOf(address(this));
        uint256 lpNeeded = amount; // 1:1 ratio

        // Cap at available LP balance
        if (lpNeeded > lpBalance) {
            lpNeeded = lpBalance;
        }

        // Redeem LP to native ETH, then wrap back to WETH
        lp.approve(address(pool), lpNeeded);
        uint256 ethBalanceBefore = address(this).balance;
        pool.redeem(lpNeeded, address(this));
        uint256 ethBalanceAfter = address(this).balance;
        uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;
        if (ethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss which includes rounding loss.", amount, ethRedeemed);
        }
        
        if (ethRedeemed + ethBalanceBefore >= amount) {
            weth.deposit{value: ethRedeemed}();
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

First, it checks the `lpBalance` and sets `lpNeeded` to provided `amount`. Then it caps `lpNeeded` to only redeem from the pool what it is possible to redeem. Then redeem happens and `ethRedeemed` is calculated using post and pre ETH balances.

The problem arises in the following snippet :

```
  if (ethRedeemed + ethBalanceBefore >= amount) {
            weth.deposit{value: ethRedeemed}();
        }
```

if `ethRedeemed + ethBalanceBefore >= amount` , then it means the contract has enough ETH to successfully send back `amount` to the vault. The contract has a `receive` function and can receive ETH, so `ethBalanceBefore` can be a non zero value. If this check passes, then `amount` should be converted to WETH, not `ethRedeemed`.

Indeed, the next line :

```
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

would fail if `ethRedeemed < amount` and only `ethRedeemed` is converted to WETH.

Note that the issue is also present in the MoonwellWETHStrategy contract.

## Vulnerability Details

The snippet:

```
  if (ethRedeemed + ethBalanceBefore >= amount) {
            weth.deposit{value: ethRedeemed}();
        }
```

is wrong and should be:

```
  if (ethRedeemed + ethBalanceBefore >= amount) {
            weth.deposit{value: amount}();
        }
```

Indeed, if `ethBalanceBefore > 0` and `pool.redeem` incurs a strategy loss, any ETH stored in the contract should be used to cover the loss as per the current design.

Overall, having the following code is problematic:

```
        if (ethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss which includes rounding loss.", amount, ethRedeemed);
        }
        
        if (ethRedeemed + ethBalanceBefore >= amount) {
            // @audit MEDIUM REPORTED should be value: amount
            weth.deposit{value: ethRedeemed}();
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

This means:

* if `ethRedeemed < amount` , then we admit there is a strategy loss
* if `ethRedeemed + ethBalanceBefore >= amount` , then we convert to WETH only `ethRedeemed`
* at the last line, we expect the check to pass but it will never pass because `TokenUtils.safeBalanceOf(address(weth), address(this))` is equal to `ethRedeemed` , which is less than `amount`

## Impact Details

The impact of this issue can be considered as medium as it results in the strategy adapter contract being unable to cover strategy loss with any ETH it holds. This also means any ETH sent to the contract with the `receive` function will be stuck in the contract.

Note that the issue is also present in the MoonwellWETHStrategy contract.

## Proof of Concept

## Proof of Concept

Please copy paste the following test in StargateEthStrategy.t.sol:

```
   function testIncorrectEthWrapping() public {
        uint256 amountToAllocate = 10e18;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        // allocate 10e18 to the strategy
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");

        // someones sends 2 ETH to the strategy contract directly
        vm.deal(strategy, 2e18);

        // deallocate 11 ETH
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, 11e18, "", address(vault));
        vm.stopPrank();
    }
```

This tests will revert but it should pass. Indeed, in the case where ether was sent through the `receive` function, this ether should be usable to cover strategy loss and allow to withdraw the requested amount.

Note that in the test, I tried to simulated strategy loss by deallocating a greater amount than what was allocated first. This allows to cap at available LP balance and simulate a strategy loss.

The original code does:

```
  if (ethRedeemed + ethBalanceBefore >= amount) {
            weth.deposit{value: ethRedeemed}();
        }
```

but it should do :

```
  if (ethRedeemed + ethBalanceBefore >= amount) {
            weth.deposit{value: amount}();
        }
```
