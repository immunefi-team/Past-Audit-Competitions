# 57862 sc low incorrect balancebefore reading order in morphoyearnogwethstrategy deallocate function leads to wrong event emission

**Submitted on Oct 29th 2025 at 09:31:54 UTC by @EagleEye for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57862
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `MorphoYearnOGWETHStrategy::_deallocate` function incorrectly reads the `wethBalanceBefore` of the WETH after the withdraw from the vault is executed.

## Vulnerability Details

The `MorphoYearnOGWETHStrategy::_deallocate` function is called by the vault through `MYTStrategy::deallocate` function to deallocate assets from the vault to the strategy:

```
function _deallocate(uint256 amount) internal override returns (uint256) {
@>  vault.withdraw(amount, address(this), address(this));
@>  uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
@>      emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

The problem in that function is that it first calls `vault.withdraw` and only afterward reads the `wethBalanceBefore`. Because both variables `wethBalanceBefore` and `wethBalanceAfter` are measured after the withdrawal, they have the same value and the `wethRedeemed` variable is always equals to 0. This means that the condition `if (wethRedeemed < amount)` is always true and the `StrategyDeallocationLoss` is always emitted.

## Impact Details

The functions/systems that listen for the events from the protocol, will always receive an `StrategyDeallocationLoss` event, when there is no deallocation loss. The function doesn't lose value, but doesn't provide the correct event. It emitts always the `StrategyDeallocationLoss` event, even when there is no deallocation loss. Thefore, Low severity is appropriate for this issue.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/MorphoYearnOGWETH.sol#L49C4-L62C1>

## Recommendation

Read the WETH balance of the strategy before the withdraw is done.

## Proof of Concept

## Proof of Concept

The following test shows that the variables `wethBalanceBefore` and `wethBalanceAfter` have the same value and the event about the deallocation loss is emitted, when the redeemed amount is the expected one:

```
function testDeallocate() public{
    weth.mint(address(mockVault), 10*10**18);
    //For test purpose I made the `_deallocate` function public
    uint256 deallocated = strategy._deallocate(0.5 ether);
}
```

The result:

```
[121125] TestMorphoYearnOGWETH::testDeallocate()
    ├─ [44822] MockERC20::mint(MockVault: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a], 10000000000000000000 [1e19])
    │   └─ ← [Stop] 
    ├─ [63993] MorphoYearnOGWETHStrategy::_deallocate(500000000000000000 [5e17])
    │   ├─ [26379] MockVault::withdraw(500000000000000000 [5e17], MorphoYearnOGWETHStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9], MorphoYearnOGWETHStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9])
    │   │   ├─ [23322] MockERC20::transfer(MorphoYearnOGWETHStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9], 500000000000000000 [5e17])
    │   │   │   └─ ← [Return] true
    │   │   └─ ← [Return] 0
    │   ├─ [541] MockERC20::balanceOf(MorphoYearnOGWETHStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9]) [staticcall]
    │   │   └─ ← [Return] 500000000000000000 [5e17]
    │   ├─ [0] console::log(500000000000000000 [5e17]) [staticcall]
    │   │   └─ ← [Stop] 
    │   ├─ [541] MockERC20::balanceOf(MorphoYearnOGWETHStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9]) [staticcall]
    │   │   └─ ← [Return] 500000000000000000 [5e17]
    │   ├─ [0] console::log(500000000000000000 [5e17]) [staticcall]
    │   │   └─ ← [Stop] 
    │   ├─ emit StrategyDeallocationLoss(message: "Strategy deallocation loss.", amountRequested: 500000000000000000 [5e17], actualAmountSent: 0)
    │   ├─ [541] MockERC20::balanceOf(MorphoYearnOGWETHStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9]) [staticcall]
    │   │   └─ ← [Return] 500000000000000000 [5e17]
    │   ├─ [22690] MockERC20::approve(TestMorphoYearnOGWETH: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 500000000000000000 [5e17])
    │   │   └─ ← [Return] true
    │   └─ ← [Return] 500000000000000000 [5e17]
    └─ ← [Stop] 
```

From the result we can see that the both variable `wethBalanceBefore` and `wethBalanceAfter` have the same value and the event is emitted, because the according to the `_deallocate` function, the redeemed amount is 0, which is incorrect.
