# 58393 sc low wrong order in balance querying instructions in morphoyearnogwethstrategy deallocate function leads to always emit strategydeallocationloss event&#x20;

**Submitted on Nov 1st 2025 at 21:49:35 UTC by @hunter0xweb3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58393
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

MorphoYearnOGWETHStrategy::\_deallocate(amount) function perform two weth.balanceOf(address(this)) queries to check if enough weth amount was withdrawn from a vault, and emits an event if amount withdrawn from the vault was not enough.\
However, the two weth.balanceOf(address(this)) queries are performed in wrong order leading to MorphoYearnOGWETHStrategy always emit that amount withdrawn was not enough, even when enough amount was withdrawn from the vault.

## Vulnerability Details

The issue occurs because the weth balanceBefore query is performed after withdrawn from the vault instead of querying balance before

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
@>      uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
@>        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
@>        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
```

So when it compares balanceBefore and balanceAfter variables values they are the same\
Because of this the value of wethRedeemed variable will always be zero, so conditional branch code will always be executed, leading \_deallocate function to always emit StrategyDeallocationLoss event, even when enough weth was withdraw from the vault

## Impact Details

* Wrong balance comparission before and after withdrawn from the vault
* Emit StrategyDeallocationLoss event is always emited generating incorrect emitted information for offchain listeners and integrators
* Contract fails to deliver promised returns, but doesn't lose value

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/MorphoYearnOGWETH.sol#L49-L56>

**Recommendation** Perform wethBalanceBefore before call to vault withdraw

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
      uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        vault.withdraw(amount, address(this), address(this));
       uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
```

## Proof of Concept

## Proof of Concept

The following Proof of concept performs a successfull allocation and deallocation (enough weth was withdraw from vault), however \_deallocate function will emit StrategyDeallocationLoss leading emitted event integrity loss

Create in src/test/strategies/MorphoYearnOGWETHStrategy.t.sol The following test case:

```
     function test_MorphoYearnOGWETHDeallocateWrongBalanceComparission(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate/2;

        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        
        // Records logs emitted by deallocate
        vm.recordLogs(); 
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        
        // Because of wrong before/ after balance querying
        // StrategyDeallocationLoss is always emitted
        VmSafe.Log[] memory logs = vm.getRecordedLogs();
        bytes32 eventSignature = keccak256(
            "StrategyDeallocationLoss(string,uint256,uint256)"
        );
        uint eventCount;
        for (uint256 i = 0; i < logs.length; i++) {
            if(logs[i].topics[0] == eventSignature ){
                eventCount+=1;
            }
        }
        assertEq(eventCount,1);
        vm.stopPrank();
    }  
```

Exec test with

```bash
forge test --mc MorphoYearnOGWETHStrategy  --mt test_MorphoYearnOGWETHDeallocateWrongBalanceComparission -vv
```

Observe event is always emitted (even when redeemed weth is >= amount argument) due wrong balance order querying in MorphoYearnOGWETHStrategy::\_deallocate function
