# 58326 sc insight the value of the burned peapods share token may exceed expectations

**Submitted on Nov 1st 2025 at 09:45:31 UTC by @pashap9990 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58326
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/PeapodsUSDC.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Finding Description and Impact

Operators or administrators can deallocate assets from a designated strategy via `MYTStrategy::deallocate`. Furthermore, operators can set the withdrawal amount, resulting in the burning of the corresponding share tokens associated with `MYTStrategy`. Moreover, CR will shift in both directions (top and bottom) within Peapods, resulting in more Tokens of `MYTStrategy` being burned upon deallocation than expected, resulting in financial loss due to insufficient slippage management.

### Textual PoC

* `MYTStrategy` possesses 1000 Peapods share tokens. (CR = 1)
* A 500 share token burn is anticipated by the operator. The operator's transaction will be sent to the mempool, and since the CR was 0.90 prior to the deallocation transaction, 555 share tokens—more than anticipated—will be burned.

### Impact

There is "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield - Critical" going on because bad CR changes during the deallocate operation make the strategy burn a lot more share tokens than planned (555 instead of 500 in this case). These extra shares that were burned represent real user funds that are lost forever when they are withdrawn. This is theft of principal funds "in-motion" instead of yield loss, because users lose money right away when their share amount drops below what it should. Attackers or bad market conditions can take advantage of the lack of slippage protection, which causes measurable fund loss on every deallocation transaction that is impacted.

### Code Snippet

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/PeapodsUSDC.sol#L32>

## Proof of Concept

## Proof of Concept

Kindly incorporate the following PoC in `PeapodsUSDCStrategy.t.sol`

```solidity
    function testPeapodsUSDCStrategySufferFromLackOfSlippgaeControl() public {
        uint amountToAllocate = 100e18;
        uint amountToDeallocate = 50e18;
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);

        // Mock the entire withdraw function to return your desired shares amount
        vm.mockCall(
        PEAPODS_USDC_VAULT,
        abi.encodeWithSelector(
            IERC4626.withdraw.selector, 
            amountToDeallocate,
            address(strategy), // receiver
            address(strategy)  // owner
        ),
        abi.encode(60e18) // Return value: shares burned
        );
    
        // Also need to handle the token transfer that withdraw would do
        deal(USDC, strategy, amountToDeallocate);
        (bytes32[] memory strategyIds, int256 change) = IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
    }
```
