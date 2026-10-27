# 57787 sc high asset can be transferred to strategies even when the killswitch enabled without posibility to use this funds for allocation

**Submitted on Oct 28th 2025 at 21:17:21 UTC by @zeroK for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57787
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

One of the core components in Alchemix V3 is the MYT (Meta Yield Token) vault, which is implemented as a Morpho V2 vault. Morpho V2 vaults are designed with multiple adapters (strategies) that interact with third-party protocols to generate yield. alchemix V3 follows the same pattern, the MYT vault manages assets like WETH or USDC, while its strategies (adapters) allocate and deallocate funds across external protocols such as Euler to earn yield. each strategy includes a kill switch mechanism, which is intended to prevent any further actions (e.g., deposits, withdrawals, or allocations) when enabled. However, in practice, this mechanism does not fully prevent all actions as intended, when the `allocate()` function is invoked, assets (WETH or USDC) are still transferred from the MYT vault to the adapter, even if the strategy’s kill switch is enabled, this occurs because the VaultV2.sol contract performs the transfer before the kill switch condition is enforced(even if it enforced, the transaction will not revert) as a result, the transferred funds become stuck within the strategy contract, since they are never deposited into a third-party protocol (the internal `_allocate()` function is not executed for that amount), Furthermore, these stuck funds cannot be recovered, as subsequent `deallocate()` calls will fail due to insufficient allowance or the lack of actual deposits in third-party vaults.

While only the operator can trigger this action, it’s important to note that the Alchemix team’s expectation is that no asset movement or allocation should occur when the kill switch is active, the team responded with below while discussing how allocate/deallocate should behave when killSwitch is true:

```
it is expected when allocate/deallocate are called when killSwitch is true that they will simply do nothing. So if something else happens, then that would be a bug.

```

Therefore, this behavior poses a permanent fund locking risk and should be treated as a critical/high issue that require immediate attention.

## Vulnerability Details

operators or admin invoke the allocate function from alchemistAllocator.sol below:

```
    // Overriden vault actions
    function allocate(address adapter, uint256 amount) external {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        uint256 absoluteCap = vault.absoluteCap(id);
        uint256 relativeCap = vault.relativeCap(id);
        // FIXME get this from the StrategyClassificationProxy for the respective risk class
        uint256 daoTarget = type(uint256).max;
        uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            // caller is operator
            adjusted = adjusted > daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
        bytes memory oldAllocation = abi.encode(vault.allocation(id)); 
        vault.allocate(adapter, oldAllocation, amount); 
    }



```

this function in return invokes the vault(morpho vault v2) allocate function:

```solidity
    function allocate(address adapter, bytes memory data, uint256 assets) external {
        require(isAllocator[msg.sender], ErrorsLib.Unauthorized());
        allocateInternal(adapter, data, assets);
    }

    function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

        accrueInterest();

        SafeERC20Lib.safeTransfer(asset, adapter, assets); //@audit transfer asset to adapter even if adapter killswitch is available
        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender); // this is where the mytStrategy call happens

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];
            _caps.allocation = (int256(_caps.allocation) + change).toUint256();

            require(_caps.absoluteCap > 0, ErrorsLib.ZeroAbsoluteCap());
            require(_caps.allocation <= _caps.absoluteCap, ErrorsLib.AbsoluteCapExceeded());
            require(
                _caps.relativeCap == WAD || _caps.allocation <= firstTotalAssets.mulDivDown(_caps.relativeCap, WAD),
                ErrorsLib.RelativeCapExceeded()
            );
        }
        emit EventsLib.Allocate(msg.sender, adapter, assets, ids, change);
    }


```

as shown(highlighted with @audit) the vault transfer tokens(weth or usdc) into the adapter which in our case its eular strategy adapter which implement mytStrategy.sol, then the adapter.allocate function invoked:

```solidity
    function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            return (ids(), int256(0));
        } // return ids() and 0, no revert
        require(assets > 0, "Zero amount");

        uint256 oldAllocation = abi.decode(data, (uint256));
        uint256 amountAllocated = _allocate(assets);
        uint256 newAllocation = oldAllocation + amountAllocated;
        
        emit Allocate(amountAllocated, address(this));
        return (ids(), int256(newAllocation) - int256(oldAllocation));
    }


```

currently the eular strategy holds USDC or WETH, but it get stuck because the \_allocate function never get invoked which is responsible for depositing the asset to eular:

```solidity
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(usdc), address(vault), amount);
        vault.deposit(amount, address(this)); // deposit amount into the eular vault(not morpho vault)
        return amount;
    }
```

however this function never invoked, and as mentioned it holds the asset which it can not be used to allocate, even if the deallocate get invoked while killlSwitch is true, it revert due to reason that the strategies gives approve to vault to transfer asset back only when \_deallocate invoked, and if the killSwitch sets to false, the deallocate will revert since the strategies does not deposited allocated amount into the third parties(e.g eular) which lead to withdraw revert.

## Impact Details

the killSwitch mechanism can lead to transfer asset into adapter without any use cases which lead to stuck of allocation asset in the strategies forever.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L29-L66>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/EulerUSDCStrategy.sol#L27-L46>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/MYTStrategy.sol#L260-L263>

## Proof of Concept

## Proof of Concept

run the test below in `alchemistAcummulator.t.sol`:

```solidity
      function testAllocate() public {
        console.log("\n--- run forge test --match-test testAllocate -vvvv ---");
        console.log("\n--- start tracking from mockVault.allocate trace ---\n");

        require(vault.adaptersLength() == 1, "adaptersLength is must be 1");
        _magicDepositToVault(address(vault), user1, 150 ether);
        
        vm.startPrank(admin);
        allocator.allocate(address(mytStrategy), 50 ether);

        mytStrategy.setKillSwitch(true);
        allocator.allocate(address(mytStrategy), 100 ether);

        vm.expectRevert();
        // mytStrategy.setKillSwitch(false);  uncomment to see the function revert due to invalid funds 
        allocator.deallocate(address(mytStrategy),  100 ether);

        console.log("\n--- END OF POC ---\n");

        vm.stopPrank();
    }

```
