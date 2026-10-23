# 58345 sc low operators in alchemistallocator sol can allocate higher than dao defined limits

**Submitted on Nov 1st 2025 at 12:29:29 UTC by @TyroneX for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58345
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Operator role in AlchemistAllocator.sol can allocate higher than DAO defined limits, potentially messing up internal accounting

## Vulnerability Details

The `AlchemistAllocator` contract allows two users call `AlchemistAllocator::allocate`. Admin and operator. It also defines caps set by both the vault and DAO. Admins can deposit upto vault cap however operators are restricted to DAO defined caps. However, a wrong comparison logic allows operators bypass this check and deposit up to vault cap. Also, the `adjusted` variable, which tracks the cap, is never used for any check.

## Impact Details

Operator addresses can allocate to vault, values above DAO defined limits. Potentially messing up internal accounting and allowing operators overexpose funds to risky strategies.

## References

```solidity
    function allocate(address adapter, uint256 amount) external {
        require(msg.sender == admin || operators[msg.sender], "PD");   
        bytes32 id = IMYTStrategy(adapter).adapterId();               
        uint256 absoluteCap = vault.absoluteCap(id);       
        uint256 relativeCap = vault.relativeCap(id);       
        // FIXME get this from the StrategyClassificationProxy for the respective risk class
        uint256 daoTarget = type(uint256).max;   
        uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
@>>         adjusted = adjusted > daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.allocate(adapter, oldAllocation, amount);
    }
```

**Recommended Mitigation:** Cap adjusted for operator at DAO target and add check

```diff
    function allocate(address adapter, uint256 amount) public {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        // mock vault caps
        uint256 absoluteCap = vault.absoluteCap(id);
        uint256 relativeCap = vault.relativeCap(id);
        // simulate real limit (rather than type(uint256).max)
        uint256 daoTarget = type(uint256).max; 
        uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
-            adjusted = adjusted > daoTarget ? adjusted : daoTarget; 
+            adjusted = adjusted < daoTarget ? adjusted : daoTarget; 
        }

+       require(amount <= adjusted, "AllocationExceedsAllowed");
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.allocate(adapter, oldAllocation, amount);
    }
```

## Proof of Concept

## Proof of Concept

**Proof of Concept:** Place the following inside MockAlchemistAllocator in AlchemistAllocator.t.sol to simulate `allocate` function with real DAO target

```solidity
    function allocateWithDaoCap(address adapter, uint256 amount) public {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        uint256 absoluteCap = vault.absoluteCap(id);
        uint256 relativeCap = vault.relativeCap(id);
        // simulate real limit (rather than type(uint256).max)
        uint256 daoTarget = 10 ether;
        uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            adjusted = adjusted > daoTarget ? adjusted : daoTarget; // buggy logic
        }

        require(amount <= adjusted, "AllocationExceedsAllowed");

        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.allocate(adapter, oldAllocation, amount);
    }
```

Place this in AlchemistAllocatorTest contract

```solidity

    function testAllocateIgnoresDaoTargetCap() public {
        _magicDepositToVault(address(vault), user1, 500 ether);

        // Call with operator
        vm.startPrank(operator);

        bytes32 id = mytStrategy.adapterId();

        // expected behavior: should cap at daoTarget (10 ether)
        // current logic ignores this and allows 15 ether
        allocator.allocateWithDaoCap(address(mytStrategy), 15 ether);

        uint256 allocated = vault.allocation(id);
        assertGt(allocated, 10 ether, "Allocation exceeded DAO cap");

        vm.stopPrank();
    }
```

This passes showing operators can deposit higher than dao limits

```rust
Ran 1 test for src/test/AlchemistAllocator.t.sol:AlchemistAllocatorTest
[PASS] testAllocateIgnoresDaoTargetCap() (gas: 580424)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 14.81ms (2.99ms CPU time)

Ran 1 test suite in 171.49ms (14.81ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
