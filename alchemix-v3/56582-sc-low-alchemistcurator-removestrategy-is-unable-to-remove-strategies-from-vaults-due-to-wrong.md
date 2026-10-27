# 56582 sc low alchemistcurator removestrategy is unable to remove strategies from vaults due to wrong logic implementation&#x20;

**Submitted on Oct 17th 2025 at 23:15:11 UTC by @hunter0xweb3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56582
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

AlchemistCurator::removeStrategy is unable to remove strategies from vaults due to wrong logic implementation.\
So, once a strategy is enabled in the vault there is no way to remove it by using AlchemistCurator contract

## Vulnerability Details

First lets examine the process of adding an strategy `adapter` to a vault `myt` to see why removeStrategy logic flaw

Adding strategy is done with a two step process:

1. Call AlchemistCurator.sol::submitSetStrategy(adapter, myt)
2. Call AlchemistCurator.sol::setStrategy(adapter, myt)

These calls flow are for AlchemistCurator.sol::submitSetStrategy

```bash
1. AlchemistCurator.sol::submitSetStrategy(adapter, myt)
    -> AlchemistCurator.sol::_submitSetStrategy(adapter, myt) 
        data = abi.encodeCall(IVaultV2.addAdapter, adapter)
        -> vault.submit(data);
```

We can see that inside \_submitSetStrategy (called by submitSetStrategy) is a call to vault::submit

```solidity
function _submitSetStrategy(address adapter, address myt) internal {       
    IVaultV2 vault = IVaultV2(myt);                                        
    bytes memory data = abi.encodeCall(IVaultV2.addAdapter, adapter);      
@>  vault.submit(data);     
```

lib/vault-v2/src/VaultV2.sol::submit updates the timelock of the function signature of data variable declared in AlchemistCurator.sol::submitSetStrategy

```solidity
function submit(bytes calldata data) external {                                     
    //msg.sender and calldata checks
    bytes4 selector = bytes4(data);
    uint256 _timelock = selector == IVaultV2.decreaseTimelock.selector ?  
        timelock[bytes4(data[4:8])] : timelock[selector];                                      
@>    executableAt[data] = block.timestamp + _timelock; 
```

If this timelock is not updated , the following call (AlchemistCurator.sol::setStrategy) will revert\
Now that timelock is updated a call to AlchemistCurator.sol::setStrategy(address adapter, address myt) is performed

```bash
2. AlchemistCurator.sol::setStrategy(address adapter, address myt)
    AlchemistCurator.sol::_setStrategy(adapter, myt, false);
        adapterToMYT[adapter] = myt;
        IVaultV2 vault = _vault(adapter);
            >vault.addAdapter(adapter);
```

Inside AlchemistCurator.sol::\_setStrategy the adapter to myt address is updated and vault is called to add the adapter:

```solidity
    function _setStrategy(address adapter, address myt, bool remove) internal {
        adapterToMYT[adapter] = myt;
        IVaultV2 vault = _vault(adapter);
        //...
@>      vault.addAdapter(adapter);
        //...
    }
```

So, as it can be seen, the vault timelock has to be updated via vault.submit(data), with the planned data to call the vault (in this case addAdapter) before performing vault changes

The problem arises because AlchemistCurator.sol::removeStrategy doesnt update vault timelock to perform adapter removal because it doesnt call vault.submit method with vault.removeAdapter function data as argument and vault.submit is neither in the methods called inside AlchemistCurator.sol::removeStrategy

AlchemistCurator.sol::removeStrategy calls flow:

```bash
AlchemistCurator.sol::removeStrategy(adapter, myt)
    -> AlchemistCurator.sol::_setStrategy(adapter, myt, true);
        -> vault.removeAdapter(adapter);
```

As it can be seen AlchemistCurator.sol::\_setStrategy updates adapter to mty mapping and calls vault.removeAdapter

```solidity
    function _setStrategy(address adapter, address myt, bool remove) internal {
@>      adapterToMYT[adapter] = myt;                                           
        IVaultV2 vault = _vault(adapter);                                      
        if (remove) {                                                          
@>          vault.removeAdapter(adapter);                                      
        } //... snip; addAdapterCase                                       
    }   
```

However because AlchemistCurator.sol::removeStrategy(adapter, myt) didnt call vault.submit(data) with data = calldata to remove the adapter then AlchemistCurator.sol::removeStrategy(adapter, myt) calls will always fail, leaving curators unable to remove strategies from vaults

Also note that no matter if adding or removing adapter to myt the mapping adapterToMyt is updated, and it should be clear when removing the adapter, because otherwise \_vault(adapter) will return a vault even when the adapter was removed

## Impact Details

* Curators are unable to remove strategies from vaults due to AlchemistCurator::removeStrategy logic to remove adapter is wrongly implemented
* Contract fails to deliver promised returns, but doesn't lose value

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L49-L53\\>
<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L62-L66\\>
<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/lib/vault-v2/src/VaultV2.sol#L341-L348>

## Proof of Concept

## Proof of Concept

The following PoC:

1. Shows that Vault's address for a non existent strategy is address(0) (expected)
2. Adds a new strategy to a vault and shows that adapterToMYT is updated (expected)
3. Tries to remove the previously added strategy but it will fail because AlchemistCurator::removeStrategy is wrongly implemented as described above

To perform the PoC add the following test case in src/test/AlchemistCurator.t.sol

```solidity
     function testRemoveStrategyWrongLogic() public {                                                
         console.log(                                                                                
             "Before adding strategy\nadapterToMYT(mytStrategy) => ",                                
             mytCuratorProxy.adapterToMYT(address(mytStrategy))    
         );                                                                                          
                                                                                                     
         vm.startPrank(operator);                                                                    
         mytCuratorProxy.submitSetStrategy(address(mytStrategy), address(vault));                    
         _vaultFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));               
         mytCuratorProxy.setStrategy(address(mytStrategy), address(vault));                          
                                                                                                     
         console.log(                                                                                
             "\nAfter adding strategy\nadapterToMYT(mytStrategy) => ",                               
             mytCuratorProxy.adapterToMYT(address(mytStrategy))                                      
         );                                                                                          
                                                                                                     
         console.log("\nRemoving strategy");                                                         
         _vaultFastForward(abi.encodeCall(IVaultV2.removeAdapter,address(mytStrategy)));             
         mytCuratorProxy.removeStrategy(address(mytStrategy), address(vault));                       
         vm.stopPrank();                                                                             
                                                                                                     
         console.log(                                                                                
             "\nAfter removing strategy,\nadapterToMYT(mytStrategy) => ",                            
             mytCuratorProxy.adapterToMYT(address(mytStrategy))                                      
         );                                                                                                                           
     }  
```

Run test case , it will fail due to removeStrategy wrong logic

```bash
reset; forge test --mt testRemoveStrategyWrongLogic
```

Implement modifications in both functions \_setStrategy and removeStrategy in src/AlchemistCurator.sol

```solidity
      function removeStrategy(address adapter, address myt) external onlyOperator {                   
         //Mine                                                                                      
         require(adapter != address(0), "INVALID_ADDRESS");                                          
         require(myt != address(0), "INVALID_ADDRESS");                                              
         IVaultV2 vault=IVaultV2(myt);                                                               
         bytes memory data = abi.encodeCall(IVaultV2.removeAdapter, adapter);                        
         vault.submit(data); // Modification; submit to vault                                        
        _setStrategy(adapter, myt, true); // remove                                                  
      }             
      
      function _setStrategy(address adapter, address myt, bool remove) internal {                     
          adapterToMYT[adapter] = myt;                                                                
          IVaultV2 vault = _vault(adapter);                                                           
          if (remove) {                                                                               
             adapterToMYT[adapter] = address(0); // Reset adapterToMYT when removing                 
              vault.removeAdapter(adapter);                                                              
          } else {                                                                                    
              vault.addAdapter(adapter);                                                              
          }                                                                                           
          emit StrategySet(adapter, myt);                                                             
      }   
      
```

Run test case again, now function removeStrategy will work and adapter is removed from vault

```
reset; forge test --mt testRemoveStrategyWrongLogic
```

**Recomendation**

1. Implemented a function AlchemistCurator::submitRemoveStrategy similar to submitSetStrategy that calls vault.submit or
2. Implement vault.submit(data) inside removeStrategy (as in the PoC above)
