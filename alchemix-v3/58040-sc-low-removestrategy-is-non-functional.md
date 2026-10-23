# 58040 sc low removestrategy is non functional

**Submitted on Oct 30th 2025 at 07:49:46 UTC by @teoslaf1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58040
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

The `removeStrategy()` function in `AlchemistCurator` is completely non-functional and will always revert. This prevents the protocol from removing strategies from vaults, even in emergency situations where a strategy becomes compromised or malicious.

***

## Vulnerability Details

### Root Cause

The `removeStrategy()` function calls `vault.removeAdapter()` directly without first submitting the operation to the timelock queue. However, `removeAdapter()` in VaultV2 has a `timelocked()` modifier that requires the operation to be submitted via `vault.submit()` first.

### Vulnerable Code

```solidity
// AlchemistCurator.sol
function removeStrategy(address adapter, address myt) external onlyOperator {
    require(adapter != address(0), "INVALID_ADDRESS");
    require(myt != address(0), "INVALID_ADDRESS");
    _setStrategy(adapter, myt, true); //  Calls removeAdapter directly
}

function _setStrategy(address adapter, address myt, bool remove) internal {
    adapterToMYT[adapter] = myt;
    IVaultV2 vault = _vault(adapter);
    if (remove) {
        vault.removeAdapter(adapter); //  This will always revert!
    } else {
        vault.addAdapter(adapter);
    }
    emit StrategySet(adapter, myt);
}
```

### VaultV2 Implementation

```solidity
// VaultV2.sol
function removeAdapter(address account) external {
    timelocked(); //  Requires timelock submission
    if (isAdapter[account]) {
        // ... removal logic
    }
    emit EventsLib.RemoveAdapter(account);
}

function timelocked() internal {
    bytes4 selector = bytes4(msg.data);
    require(executableAt[msg.data] != 0, ErrorsLib.DataNotTimelocked()); //  Always fails
    require(block.timestamp >= executableAt[msg.data], ErrorsLib.TimelockNotExpired());
    require(!abdicated[selector], ErrorsLib.Abdicated());
}
```

***

## Impact

1. Cannot remove strategies: All attempts to remove strategies will revert with `DataNotTimelocked()`
2. Compromised strategies cannot be removed: If a strategy is hacked or becomes malicious, it cannot be removed
3. No emergency response: Protocol cannot respond to security incidents involving strategies
4. Permanent lock-in: Strategies are permanently locked in vaults once added
5. Deprecated protocols: Cannot remove strategies from deprecated or sunset protocols
6. Risk accumulation: Bad strategies accumulate over time with no removal mechanism

## Proof of Concept

## Proof of Concept

Add this to AlchemistCurator.t.sol

```solidity
	function testRemoveStrategyAlwaysReverts() public {
		_submitAndSetStrategy(address(mytStrategy), address(vault));
		
		vm.prank(operator);
		vm.expectRevert(abi.encodeWithSignature("DataNotTimelocked()"));
		mytCuratorProxy.removeStrategy(address(mytStrategy), address(vault));
	}
```
