# 57346 sc low alchemistallocator compares incompatible units asset wei vs wad percentage&#x20;

**Submitted on Oct 25th 2025 at 12:02:53 UTC by @teoslaf1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57346
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Protocol insolvency
  * Contract fails to deliver promised returns, but doesn't lose value
  * logic flaw in fund-management layer

## Description

## Summary

The `AlchemistAllocator` contract contains a logic error where it directly compares `absoluteCap` (measured in asset wei) with `relativeCap` (measured in WAD percentage units). This is mathematically meaningless. While the result is currently unused, it indicates broken cap enforcement logic.

## Vulnerability Details

```solidity
// Line 35 - In allocate():
uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;

// Line 57 - In deallocate():
uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
```

### Why This Is Wrong

The comparison mixes two incompatible units:

**`absoluteCap`** - Asset amount in wei:

```solidity
200 ether = 200 * 10^18 = 200,000,000,000,000,000,000 wei
```

**`relativeCap`** - Percentage in WAD format:

```solidity
1e18 = 1,000,000,000,000,000,000 (represents 100%)
0.2e18 = 200,000,000,000,000,000 (represents 20%)
```

## Understanding WAD Units

### WAD Constant Definition

**Source:** `lib/vault-v2/src/libraries/ConstantsLib.sol`

```solidity
uint256 constant WAD = 1e18;
```

### How RelativeCap Should Be Used

**From Morpho VaultV2 documentation** (`lib/vault-v2/src/VaultV2.sol` lines 46-54):

```solidity
/// CAPS
/// @dev The relative cap is relative to firstTotalAssets, not realAssets.
/// @dev The relative cap unit is WAD.
```

**Proper enforcement** (from VaultV2 line 585):

```solidity
require(
    _caps.relativeCap == WAD ||
    _caps.allocation <= firstTotalAssets.mulDivDown(_caps.relativeCap, WAD),
    ErrorsLib.RelativeCapExceeded()
);
```

This converts relativeCap to assets: `(totalAssets * relativeCap) / WAD`

## Impact

The allocator’s cap enforcement logic is inactive and incorrect. Although the adjusted variable is never used, leaving the logic dormant, its presence in a core fund-allocation function still has meaningful impact. It shows an unfinished or misunderstood design in a component meant to enforce per-strategy caps, causing false assumptions for operators and off-chain automation that may rely on allocator-level checks.

Allocations currently proceed without any pre-validation, relying solely on the vault’s internal limits. This can lead to unexpected reverts, wasted gas, and confusion around how caps are enforced. Even if unused today, the code represents a broken and misleading safety layer that could reintroduce risk if activated in the future.

### Example Calculation

**Scenario:**

* Vault has 1000 ETH total
* `absoluteCap = 200 ETH`
* `relativeCap = 0.2e18` (20%)

**Correct calculation:**

```solidity
relativeCapInAssets = (1000 ETH * 0.2e18) / 1e18 = 200 ETH
effectiveCap = min(200 ETH, 200 ETH) = 200 ETH
```

**Wrong calculation (current code):**

```solidity
adjusted = 200e18 > 0.2e18 ? 200e18 : 0.2e18 = 200e18
// This is meaningless! Comparing 200 ETH with 20%
```

## Recommendations

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    bytes32 id = IMYTStrategy(adapter).adapterId();

    // Get current state
    uint256 currentAllocation = vault.allocation(id);
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);

    // correct: Convert relativeCap to asset terms
    uint256 totalAssets = vault.totalAssets();
    uint256 relativeCapInAssets = (totalAssets * relativeCap) / 1e18;

    // correct: Compare in same units
    uint256 effectiveCap = absoluteCap < relativeCapInAssets ? absoluteCap : relativeCapInAssets;

    // correct: Enforce the cap
    require(currentAllocation + amount <= effectiveCap, "Allocation exceeds cap");

    // For operators, further restrict by daoTarget
    if (msg.sender != admin) {
        // TODO: Implement StrategyClassificationProxy
        // uint256 daoTarget = strategyClassifier.getIndividualCap(id);
        // require(currentAllocation + amount <= daoTarget, "Exceeds DAO target");
    }

    bytes memory oldAllocation = abi.encode(currentAllocation);
    vault.allocate(adapter, oldAllocation, amount);
}
```

## Proof of Concept

## Proof of Concept

Add this to AlchemistAllocator.t.sol

```solidity
    function test_PROOF_comparing_incompatible_units() public view {
        bytes32 allocationId = mytStrategy.adapterId();
        
        // Get cap values from vault
        uint256 absoluteCap = vault.absoluteCap(allocationId);  // 200 ether (asset amount)
        uint256 relativeCap = vault.relativeCap(allocationId);  // 1e18 (100% in WAD)
        
        // Verify the values
        assertEq(absoluteCap, 200 ether, "absoluteCap should be 200 ether");
        assertEq(relativeCap, 1e18, "relativeCap should be 1e18 (100%)");
        
        // This is what AlchemistAllocator does (line 35):
        // uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        bool comparison = absoluteCap > relativeCap;
        
        // Prove the comparison happens
        assertTrue(comparison, "absoluteCap (200e18) > relativeCap (1e18) = true");
        
        // But this comparison is meaningless:
        // - absoluteCap is in asset wei (200 * 10^18)
        // - relativeCap is in WAD percentage (1 * 10^18 = 100%)
        // It's like comparing "200 meters" with "100 percent"
        
        // correct approach: Convert relativeCap to assets first
        uint256 totalAssets = 100 ether;
        uint256 relativeCapInAssets = (totalAssets * relativeCap) / 1e18;
        assertEq(relativeCapInAssets, 100 ether, "100% of 100 ether = 100 ether");
        
        // Now you can compare in same units
        uint256 effectiveCap = absoluteCap < relativeCapInAssets ? absoluteCap : relativeCapInAssets;
        assertEq(effectiveCap, 100 ether, "min(200, 100) = 100 ether");
    }
```
