# 58773 sc medium in stargate incorrect allocation cap accounting leading to unnecessary dos

**Submitted on Nov 4th 2025 at 12:53:01 UTC by @aman for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58773
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

While allocating deposits into the `Stargate` strategy, the implementation removes the dust amount from the provided deposit value and only deposits the remaining amount into the strategy.However, the `vault still records the full provided amount` (including the removed dust) in the `allocation caps mapping`.\
As a result, the stored allocation value becomes `inaccurate`, leading to a situation where the system incorrectly reports that the allocation cap has been reached even though the actual deposited amount is lower.

## Vulnerability Details

To understand this behavior, let’s first examine the Vault’s allocation flow: `allocate -> allocateInternal`

```solidity
/v3-poc/lib/vault-v2/src/VaultV2.sol:570
570:     function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
571:         (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);
572: 
573:         for (uint256 i; i < ids.length; i++) {
574:             Caps storage _caps = caps[ids[i]];
575:             _caps.allocation = (int256(_caps.allocation) + change).toUint256();
576: 
577:             require(_caps.absoluteCap > 0, ErrorsLib.ZeroAbsoluteCap());
578:             require(_caps.allocation <= _caps.absoluteCap, ErrorsLib.AbsoluteCapExceeded());
579:             require(
580:                 _caps.relativeCap == WAD || _caps.allocation <= firstTotalAssets.mulDivDown(_caps.relativeCap, WAD),
581:                 ErrorsLib.RelativeCapExceeded()
582:             );
583:         }
584:         emit EventsLib.Allocate(msg.sender, adapter, assets, ids, change);
585:     }
586: 
```

At `line 571`, the value stored in `change` is added to `caps.allocation`.\
Now, let’s take a closer look at the `Stargate` strategy:

```solidity
/v3-poc/src/strategies/optimism/StargateEthPoolStrategy.sol:42
42:     function _allocate(uint256 amount) internal override returns (uint256) {
43:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "not enough WETH");
44:         // unwrap to native ETH for Pool Native
45:         weth.withdraw(amount);
46:         uint256 amountToDeposit = (amount / 1e12) * 1e12;
47:         uint256 dust = amount - amountToDeposit;
48:         if (dust > 0) {
49:             emit StrategyAllocationLoss("Strategy allocation loss due to rounding.", amount, amountToDeposit);
50:         }
51:         pool.deposit{value: amountToDeposit}(address(this), amountToDeposit);
52:         return amount;
53:     }
54: 
```

Here we can observe that at `line 47`, the dust is subtracted from the provided amount before depositing into the strategy, while at `line 52`, the function returns the original amount (including the dust).

By combining both code snippets, we can summarize the behavior as follows:

1. **Admin allocates:** `3999999999999999996`
2. **After removing dust:** `3999999000000000000`
3. The `_allocate` function still returns `3999999999999999996`, and the vault records `caps.allocation = 3999999999999999996`.
4. Even after a full `deallocation`, the vault still shows `caps.allocation = 999999999996`, which corresponds exactly to the `dust` amount stored in step 3.

## Impact Details

Due to incorrect `caps.allocation` calculation, the vault reports inaccurate cap values, which can cause unexpected reverts and potential DoS during allocation.

## References

[StargateEthPoolStrategy.sol#L46-L52](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/optimism/StargateEthPoolStrategy.sol#L46-L52)

## Proof of Concept

## Proof of Concept

Add Following file to `test/strategies` Dir with the name `POC.t.sol` :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {AlchemistAllocator} from "../../AlchemistAllocator.sol";

import {StargateEthPoolStrategy} from "../../strategies/optimism/StargateEthPoolStrategy.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {console} from "forge-std/console.sol";


contract MockStargateEthPoolStrategy is StargateEthPoolStrategy {
    constructor(address _myt, StrategyParams memory _params, address _weth, address _pool, address _permit2Address)
        StargateEthPoolStrategy(_myt, _params, _weth, _pool, _permit2Address)
    {}
}

contract MockAlchemistAllocator is AlchemistAllocator {
    constructor(address _myt, address _admin, address _operator) AlchemistAllocator(_myt, _admin, _operator) {}
}


contract POCTest is BaseStrategyTest {
    address public constant STARGATE_ETH_POOL = 0xe8CDF27AcD73a434D661C84887215F7598e7d0d3;
    address public constant WETH = 0x4200000000000000000000000000000000000006;
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "StargateEthPool",
            protocol: "StargateEthPool",
            riskClass: IMYTStrategy.RiskClass.HIGH,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 10e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockStargateEthPoolStrategy(vault, params, WETH, STARGATE_ETH_POOL, OPTIMISM_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 141_751_698;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("OPTIMISM_RPC_URL");
    }

    function test_allocation_caps() public {
        // set 1 : Allocator and strategy setup
        vm.startPrank(admin);
        allocator = address(new MockAlchemistAllocator(address(vault), admin, operator));
        vm.stopPrank();
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        IVaultV2(vault).setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(strategy)));
        IVaultV2(vault).addAdapter(address(strategy));
        vm.stopPrank();
        // allocate and deallocate with dust amount to check allocation caps mapping in vault
        uint256 amountToAllocate=3999999999999999996;
         uint256 amountToDeallocate = 3999999000000000000; //this amount will got deposited in Stragate Strategy
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        // allocation before
        bytes32 strategyId = IMYTStrategy(strategy).adapterId();
        uint256 allocationBefore = IVaultV2(vault).allocation(strategyId);
        console.log("Allocation before: %s", allocationBefore);
        vm.startPrank(admin);
        MockAlchemistAllocator(address(allocator)).allocate(address(strategy), amountToAllocate); // it will only allocate the amount after removing dust
        vm.stopPrank();
        // allocation 
        uint256 allocationAfter= IVaultV2(vault).allocation(strategyId); // the allocation caps stores the complete amount passed in allocation
        console.log("Allocation After allocate: %s", allocationAfter);
        vm.startPrank(admin);
        MockAlchemistAllocator(address(allocator)).deallocate(address(strategy), amountToDeallocate); // even after deallocation we have dust amount in allocation caps mapping
        vm.stopPrank();
         allocationAfter= IVaultV2(vault).allocation(strategyId);
        console.log("Allocation After deallocate: %s", allocationAfter);
    }

}
```

Run With Command : `forge test --match-test test_allocation_caps -vvv --decode-internal`
