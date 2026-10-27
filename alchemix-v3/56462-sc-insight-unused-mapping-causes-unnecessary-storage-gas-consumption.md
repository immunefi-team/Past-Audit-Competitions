# 56462 sc insight unused mapping causes unnecessary storage gas consumption

**Submitted on Oct 16th 2025 at 10:13:25 UTC by @Purpledragon for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56462
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Summary:

The mapping `_redemptions` and struct `RedemptioInfo` are declared in `AlchemistV3.sol` and `IAlchemistV3.sol` correspondingly.

```solidity
AlchemistV3.sol:Line140
mapping(uint256 => RedemptionInfo) private _redemptions; 

IAlchemistV3.sol:Line64
struct RedemptionInfo {
    uint256 earmarked;
    uint256 debt;
    uint256 earmarkWeight;
}
```

However, both the mapping and struct is never used anywhere in the contract logic. There are no reads, writes, or references to `_redemptions` and `RedemptionInfo` in the code.

## Impact

* **Unnecessary Storage Allocation:** Declaring a mapping in contract storage reserves a slot in the contract's storage layout, which increases deployment gas costs.
* **Wasted Gas:** Although mappings themselves do not consume gas until written to, their presence in the storage layout still increases the contract's complexity and deployment cost.

## Recommendation

Remove the unused `_redemptions` mapping and `RedemptionInfo` from the contract to optimize storage usage and reduce deployment gas costs. This will also improve code clarity and maintainability.

## Proof of Concept

## Proof of Concept

For Mapping:

1. Navigate to the `AlchemistV3.sol` contract
2. Scroll to the `Line64` to observe the `_redemptions` mapping is declared, but not used else where in the codebase i.e no read or write operation is performed on this mapping.

For Struct:

1. Navigate to the `IAlchemistV3.sol` contract
2. Scroll to the `Line140` to observe the `RedemptionInfo` struct is declared, but not used else where in the codebase i.e no read or write operation is performed on this struct.
