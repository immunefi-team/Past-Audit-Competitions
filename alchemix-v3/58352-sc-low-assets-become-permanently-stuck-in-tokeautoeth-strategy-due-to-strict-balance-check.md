# 58352 sc low assets become permanently stuck in tokeautoeth strategy due to strict balance check

**Submitted on Nov 1st 2025 at 13:43:40 UTC by @mohitisimmortal for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58352
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Description

When deallocating, the `TokeAutoEth strategy adapter` expects to receive the exact amount of WETH back from the Tokemak AutoETH vault. However, the vault consistently returns slightly less (e.g., 49.99 WETH for a 50 WETH deallocation) due to internal rounding/slippage.

The adapter then fails this check:

```solidity
TokeAutoEth.sol::
    function _deallocate(uint256 amount) internal override returns (uint256) {
        ...
>>        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        ...
    }
```

* Because the returned amount is just below the requested amount, the transaction reverts. This prevents any deallocation from succeeding once assets are allocated.

ex -

allocate(100 weth)

deallocate(50 weth)

auto eth vault only sends 49.99 weth, and then that require st not passes, as 49.99 is not >= 50 weth.

## Impact

* All assets allocated into the TokeAutoEth strategy become permanently frozen.

## Recommendation

Allow a small tolerance/slippage buffer when verifying returned assets, e.g.: `require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount-SLIPPAGE_TOLERANCE`

Where SLIPPAGE\_TOLERANCE is 0.5%–2%.

## Proof of Concept

## Proof of Concept

1. Add below test into `TokeAutoETHStrategy.t.sol`:

```solidity
function test_Revert_during_deallocate_due_to_less_assets_sends_to_adapter() public {
        uint256 amountToAllocate = 100 ether;
        uint256 amountToDeallocate = 50 ether;
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 90 ether, "Initial real assets is 0"); // even though assets are greater than 90%, still fails to deallocate 50% amount

        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        
        vm.expectRevert("Strategy balance is less than the amount needed");
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }
```

2. run by `forge test --mt test_Revert_during_deallocate_due_to_less_assets_sends_to_adapter -vvvv`
3. output:

```solidity
emit StrategyDeallocationLoss(message: "Strategy deallocation loss.", amountRequested: 50000000000000000000 [5e19], actualAmountSent: 49998537973349730260 [4.999e19])

0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2::balanceOf(MockTokeAutoEthStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1]) [staticcall]
    │   │   └─ ← [Return] 49998537973349730260 [4.999e19]
    │   └─ ← [Revert] Strategy balance is less than the amount needed
```
