# 57473 sc low inverted comparison operator allows operators admin level allocation privileges

**Submitted on Oct 26th 2025 at 14:32:24 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57473
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Operators gain admin-equivalent allocation capabilities

## Description

## Summary

The `allocate()` function in `AlchemistAllocator` uses an inverted comparison operator (`>` instead of `<`) when applying the `daoTarget` cap for operators. This bug will grant operators the same allocation privileges as admins once `daoTarget` is properly implemented, bypassing intended governance restrictions.

## Description

The `AlchemistAllocator` contract implements a privilege separation model where admins have unrestricted allocation rights, while operators should be subject to additional DAO-governed caps. The contract has two symmetric functions: `allocate()` and `deallocate()`.

**Wrong implementation in `allocate()`:** <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L34C8-L40C10>

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
        // caller is operator
    @>>    adjusted = adjusted > daoTarget ? adjusted : daoTarget;  
    }
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}
```

**Correct implementation in `deallocate()`:** <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L56C7-L62C10>

```solidity
function deallocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);
    uint256 daoTarget = type(uint256).max;
    uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
    if (msg.sender != admin) {
        // caller is operator
   @>>     adjusted = adjusted < daoTarget ? adjusted : daoTarget; 
    }
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.deallocate(adapter, oldAllocation, amount);
}
```

### Key Differences

| Function       | Operator Check                                | Result                                    |
| -------------- | --------------------------------------------- | ----------------------------------------- |
| `allocate()`   | `adjusted > daoTarget ? adjusted : daoTarget` | Takes **maximum** of both values (wrong)  |
| `deallocate()` | `adjusted < daoTarget ? adjusted : daoTarget` | Takes **minimum** of both values (correc) |

### Current State vs Future Impact

**Currently:** The bug is dormant because `daoTarget` is hardcoded to `type(uint256).max`, making the check ineffective.

**When implemented:** Per the FIXME comment, `daoTarget` will be fetched from `StrategyClassificationProxy`. Once this happens, the inverted comparison will immediately allow operators to bypass governance-imposed caps.

## Impact

### Access Control Bypass

When `daoTarget` is properly implemented, operators will be able to allocate funds up to `max(adjusted, daoTarget)` instead of the intended `min(adjusted, daoTarget)`, effectively granting them admin-level privileges.

## Mitigation

Change the comparison operator in `allocate()` from `>` to `<`:

```diff
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
-       adjusted = adjusted > daoTarget ? adjusted : daoTarget;
+       adjusted = adjusted < daoTarget ? adjusted : daoTarget;
    }
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}
```

This change aligns `allocate()` with the correct logic already implemented in `deallocate()`.

## Proof of Concept

## Proof of Concept

### 1.paste the following test in `AllchemistAllocator.t.sol`

```solidity
    function testInvertedComparisonBug() public  {
        // Simulate future state when daoTarget is implemented
        uint256 absoluteCap = 2000e18;
        uint256 relativeCap = 2000e18;
        uint256 daoTarget = 1000e18; // Governance-set operator limit

        // Current buggy logic in allocate()
        uint256 adjustedBuggy = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        adjustedBuggy = adjustedBuggy > daoTarget ? adjustedBuggy : daoTarget;

        // Correct logic from deallocate()
        uint256 adjustedCorrect = absoluteCap < relativeCap ? absoluteCap : relativeCap;
        adjustedCorrect = adjustedCorrect < daoTarget ? adjustedCorrect : daoTarget;

        // Demonstrate the access control bypass
        assert(adjustedBuggy == 2000e18);   // Bug: Operator can allocate same as admin
        assert(adjustedCorrect == 1000e18); // Expected: Operator restricted to daoTarget

        console.log("Buggy allocate() allows operator to allocate:", adjustedBuggy); // same as admin
        console.log("Correct deallocate() restricts operator to:", adjustedCorrect); // capped correctly
        console.log("Difference (privilege escalation):", adjustedBuggy - adjustedCorrect);
    }

```

### 2. run it via `forge test --mt testInvertedComparisonBug -vvv`

### Logs

```
Logs:
  Buggy allocate() allows operator to allocate: 2000000000000000000000
  Correct deallocate() restricts operator to: 1000000000000000000000
  Difference (privilege escalation): 1000000000000000000000

```
