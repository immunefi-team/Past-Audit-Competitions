# 57941 sc high incorrect handling of deallocate return val causes any interest gains in a strategy to become unclaimable and permanently locked

**Submitted on Oct 29th 2025 at 14:36:49 UTC by @niroh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57941
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Morpho VaultV2 relies on adapters::deallocate() implementation to return the amount of change required in their current (old) allocation as recorded in the vault, so that the vault records the correct allocation for the adapter after the operation. The vault adds/subtracts that value from its current record of the adapter allocation:

```solidity
//FROM VaultV2 deallocateInternal()
(bytes32[] memory ids, int256 change) = IAdapter(adapter).deallocate(data, assets, msg.sig, msg.sender);

for (uint256 i; i < ids.length; i++) {
    Caps storage _caps = caps[ids[i]];
    require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());
    _caps.allocation = (int256(_caps.allocation) + change).toUint256();
}
```

MytStrategy currently handles this by returning the negative of the return value of \_deallocate (which is implemented by the derived strategy contract) that represent the amount deallocated.

```solidity
 uint256 oldAllocation = abi.decode(data, (uint256));
uint256 amountDeallocated = _deallocate(assets);
uint256 newAllocation = oldAllocation - amountDeallocated;
emit Deallocate(amountDeallocated, address(this));
return (ids(), int256(newAllocation) - int256(oldAllocation));
```

Note that the returned value is always amountDeallocated \* -1.

## Vulnerability Details

MythStategy's implementation fails to address the case where the assets in the adapter increased since they were allocated. For example:

1. 1000 USDC are allocated to a strategy (the VaultV2 records 1000 allocation for the strategy)
2. over time the strategy's underlying vault balance gains interest and now there are 1200 USDC in the strategy.
3. With MythStrategy's implementation, there is no way to deallocate the 1200 USDC, or even part of it because any call to deallocate of more than 1000 will revert. For example if Deallocate(strategy, 1100) is called, the strategy will be able to withdraw that amount, but will return 1200 qs the deallocated amount. Consequently, this line in MytStrategy will revert for underflow:`uint256 newAllocation = oldAllocation - amountDeallocated;`

The root cause is that MytStrategy calculates newAllocation as oldAllocation-amountDeallocated, which is incorrect. The derived strategy should return the amount of allocation it actually has post-operation (in this case 0) from the underlying vault, and that should be used as the newAllocation, so MytStrategy::deallocate returns newAllocation (current Real Allocation Left In The Strategy) - oldAllocation (which could be positive or negative). This is also aligned with how the MorphoVaultV1Adapter is implemented in the vault-v2 lib.

With the current implementation only the amount originally allocated can be deallocated, making any gained interest permanently locked.

Note: The root cause of this issue is a combination of the code in MythStrategy and the implementation of \_deallocate in all derived strategies in scope. A solution will require changing both the code in MytStrategy (using the return value from \_deallocate at the newAllocation) and the derived strategies (returning the remaining allocation from \_deallocate instead of the deallocated amount)

## Impact Details

Permanent lock of strategy gains above the originally allocated amount.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/MYTStrategy.sol#L119>

## Proof of Concept

## Proof of Concept

To run:

1. Copy the code below into the AaveV3ARBUSDCStrategyTest contract in v3-poc/src/test/strategies/AaveV3ARBUSDCStrategy.t.sol
2. Add this import: `import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";`
3. Run with FOUNDRY\_PROFILE=default forge test --fork-url <https://arbitrum.gateway.tenderly.co> --match-test testUnableWithdrawInterest -vvv

```solidity
function testUnableWithdrawInterest() public {
        uint256 amountToAllocate = 1000e6;
        uint256 interest = 200e6;
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        console.log("Initial allocation: %s. InitialRealAssets: %s", amountToAllocate, initialRealAssets);
        vm.stopPrank();

        //Simulate interest accrual
        //by sending more aTokens to the strategy from some whale
        vm.prank(0x4c2D98889bacB55B6E21d951C8929fd27C322f7B);
        IERC20(AAVE_V3_USDC_ATOKEN).transfer(strategy,interest);

        
        uint256 afterInterestRealAssets = IMYTStrategy(strategy).realAssets();
        console.log("Real assets with interest %s", afterInterestRealAssets);
        


       bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        //try to deallocate with even just half of the interest
        uint256 amountToDeallocate = amountToAllocate + (interest / 2);
        vm.startPrank(vault);
       vm.expectRevert(stdError.arithmeticError);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();

        //AUDIT COMMENT:
        //The call to deallocate reverts with coverflow/underflow.
        //Comment out the expectRevert line to see the trace. From the trace its clear that the underflow happens immediately after AaveV3ARBUSDCStrategy::_dealocate returns,
        //in line 130 of MytStrategy:  uint256 newAllocation = oldAllocation - amountDeallocated;
    }
```
