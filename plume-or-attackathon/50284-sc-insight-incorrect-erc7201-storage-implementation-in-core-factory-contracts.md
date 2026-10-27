# 50284 sc insight incorrect erc7201 storage implementation in core factory contracts

**Submitted on Jul 23rd 2025 at 12:17:16 UTC by @AasifUsmani for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50284
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* **Impacts:**
  * Protocol insolvency
  * Temporary freezing of funds for at least 1 hour

## Description

### Brief/Intro

The `ArcTokenFactory` and `RestrictionsFactory` contracts claim to implement ERC7201 namespaced storage via `@custom:storage-location erc7201:` annotations but use incorrect storage slot calculations that violate the ERC7201 specification. This creates potential storage collision risks and is a standards compliance issue in core Plume Chain infrastructure contracts.

## Vulnerability Details

### Root Cause Analysis

Both factory contracts contain identical violations of the ERC7201 standard.

1. ArcTokenFactory.sol (simplified excerpt):

```solidity
/// @custom:storage-location erc7201:arc.factory.storage
struct FactoryStorage {
    // ... storage fields
}

// ❌ VIOLATION: Uses simple keccak256 instead of ERC7201 formula
bytes32 private constant FACTORY_STORAGE_LOCATION = keccak256("arc.factory.storage");

function _getFactoryStorage() private pure returns (FactoryStorage storage fs) {
    bytes32 position = FACTORY_STORAGE_LOCATION;
    assembly {
        fs.slot := position
    }
}
```

2. RestrictionsFactory.sol (simplified excerpt):

```solidity
/// @custom:storage-location erc7201:restrictions.factory.storage
struct FactoryStorage {
    // ... storage fields  
}

// ❌ VIOLATION: Uses simple keccak256 instead of ERC7201 formula
bytes32 private constant FACTORY_STORAGE_LOCATION = keccak256("restrictions.factory.storage");

function _getFactoryStorage() private pure returns (FactoryStorage storage fs) {
    bytes32 position = FACTORY_STORAGE_LOCATION;
    assembly {
        fs.slot := position
    }
}
```

### ERC7201 Standard Requirements

According to EIP-7201, the correct storage slot calculation must be:

```solidity
// Correct ERC7201 implementation:
bytes32 private constant FACTORY_STORAGE_LOCATION = 
    keccak256(abi.encode(uint256(keccak256("arc.factory.storage")) - 1)) & ~bytes32(uint256(0xff));
```

The ERC7201 formula ensures collision resistance through:

* Double hashing: keccak256(abi.encode(uint256(keccak256(namespace)) - 1))
* Byte masking: & \~bytes32(uint256(0xff)) ensures the last byte is 0x00
* Offset subtraction: -1 prevents direct collision with the inner hash

## Impact Details

### Storage Collision Risks

Because the contracts use a plain keccak256(namespace) as the storage slot:

* It can collide with storage slots used by common libraries (e.g., OpenZeppelin `AccessControlUpgradeable`, `UUPSUpgradeable`, `Initializable`).
* During upgrades, overlapping slots could corrupt state leading to broken behavior or loss of funds.

### System-Level Impact

These are core infrastructure contracts:

* ArcTokenFactory manages creation and upgrades of ARC tokens.
* RestrictionsFactory manages restriction module deployments.

Storage corruption in either can:

* Break token creation/upgrades
* Corrupt restriction module mappings
* Cause system-wide degradation and require emergency migrations

### Standards Compliance Issue

Contracts claim ERC7201 compliance via annotations but do not implement its slot calculation correctly. This misleads developers/auditors and breaks tooling that expects ERC7201 namespaces.

## Required Actions

{% stepper %}
{% step %}

### Correct storage slot calculation

Immediately correct the storage slot calculations to follow the ERC-7201 formula (double-hash + offset + byte mask) for each `@custom:storage-location erc7201:` namespace.
{% endstep %}

{% step %}

### Align annotations with actual implementation

If ERC7201 is not being implemented, remove or correct the `@custom:storage-location erc7201:` annotations so documentation and code are consistent.
{% endstep %}
{% endstepper %}

## References

1. ArcTokenFactory: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenFactory.sol#L62>
2. RestrictionsFactory: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/RestrictionsFactory.sol#L43>
3. EIP - 7201 documentation and mandates: <https://eips.ethereum.org/EIPS/eip-7201>

## Proof of Concept

<details>

<summary>PoC solidity test demonstrating slot differences (click to expand)</summary>

To run the PoC, create a new file in `arc/tests/EIPTests.t.sol` and copy/paste the following. Run: `forge test --mt test_StorageSlotCalculationComparison -vvvv --via-ir`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import "forge-std/console.sol";

contract ERC7201ViolationTest is Test {
    
    function test_StorageSlotCalculationComparison() public view {
        // Current implementation (INCORRECT)
        bytes32 currentArcFactory = keccak256("arc.factory.storage");
        bytes32 currentRestrictionsFactory = keccak256("restrictions.factory.storage");
        
        // Correct ERC7201 implementation
        bytes32 correctArcFactory = keccak256(
            abi.encode(uint256(keccak256("arc.factory.storage")) - 1)
        ) & ~bytes32(uint256(0xff));
        
        bytes32 correctRestrictionsFactory = keccak256(
            abi.encode(uint256(keccak256("restrictions.factory.storage")) - 1)
        ) & ~bytes32(uint256(0xff));
        
        console.log("=== Storage Slot Comparison ===");
        console.log("ArcTokenFactory:");
        console.logBytes32(currentArcFactory);
        console.log("Should be:");
        console.logBytes32(correctArcFactory);
        
        console.log("RestrictionsFactory:");
        console.logBytes32(currentRestrictionsFactory);
        console.log("Should be:");
        console.logBytes32(correctRestrictionsFactory);
        
        // Demonstrate they are different
        assert(currentArcFactory != correctArcFactory);
        assert(currentRestrictionsFactory != correctRestrictionsFactory);
        
        // Show ERC7201 slots always end in 0x00
        assert(uint8(uint256(correctArcFactory)) == 0);
        assert(uint8(uint256(correctRestrictionsFactory)) == 0);
        
        console.log("ERC7201 violation confirmed: slots do not match standard");
    }
}
```

</details>
