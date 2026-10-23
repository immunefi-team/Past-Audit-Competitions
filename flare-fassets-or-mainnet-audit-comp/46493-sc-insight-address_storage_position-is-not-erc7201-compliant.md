# #46493 \[SC-Insight] ADDRESS\_STORAGE\_POSITION is not ERC7201 compliant

**Submitted on May 31st 2025 at 15:40:59 UTC by @kenzo for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46493
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/governance/implementation/AddressUpdatable.sol>
* **Impacts:**

## Description

## Vulnerability Details

`ADDRESS_STORAGE_POSITION` in the `AddressUpdatable` contract doesn't follow EIP-7201 exactly the formula prescribed by ERC-7201:

keccak256(abi.encode(uint256(keccak256(bytes(id))) - 1)) & \~bytes32(uint256(0xff)).

We can see it is calculated as:

```
 bytes32 internal constant ADDRESS_STORAGE_POSITION =
        keccak256("flare.diamond.AddressUpdatable.ADDRESS_STORAGE_POSITION");
```

## Impact

The omitted part of the ERC7201 formula: & \~bytes32(uint256(0xff)) guarantees the last byte is 0x00 and this ensures proper storage alignment in future upgrades. The current version ignores this and the created slots don't end with 00 as the rightmost bytes. The protocol is expected to be ERC-7201 compatible, several libraries are prone to storage collision if not follow the specification. Failure to comply with ERC712 can lead to integration issues.

## Tools Used

Manual Review

## Recommendations

Used the exact ERC7201 formula to calculate the required storage slot in the AddressUpdatable Contract.

## Proof of Concept

Step 1: The AddressUpdatable contract calculates the storage slot ADDRESS\_STORAGE\_POSITION using a simple keccak256 hash of a static string.

```
bytes32 internal constant ADDRESS_STORAGE_POSITION = keccak256("flare.diamond.AddressUpdatable.ADDRESS_STORAGE_POSITION");
```

This is a straightforward hash without additional transformations.

Step 2: What the standard requires: ERC-7201 specifies a more complex formula for storage slot calculation to ensure proper alignment and uniqueness:

```
Compute the initial hash: keccak256(id).

Subtract 1 from the result: uint256(keccak256(id)) - 1.

Encode the result: abi.encode(...).

Hash again: keccak256(...).

Mask the last byte to end with 0x00: & ~bytes32(0xff).
```

Correct Formula:

```
keccak256(abi.encode(uint256(keccak256("flare.diamond.AddressUpdatable.ADDRESS_STORAGE_POSITION")) - 1)) & ~bytes32(0xff)
```

Key Difference: The current implementation skips these steps, resulting in a slot that doesn’t align with ERC-7201 requirements.

Step 3: Issue 1: Misalignment During Upgrades The ERC-7201 standard ensures the storage slot ends with 0x00 for proper alignment in upgradeable contracts.

The current slot lacks this property, which can cause data misalignment when upgrading the contract, leading to corrupted or inaccessible storage.

Issue 2: Non-Compliance with ERC-7201 Systems or tools expecting ERC-7201-compliant contracts may fail to interact correctly with AddressUpdatable, breaking integrations or compatibility.

Issue 3: Increased Risk of Storage Collisions The simpler keccak256 hash provides less uniqueness compared to the ERC-7201 formula, increasing the likelihood of storage slot conflicts with other variables or contracts.

Example Scenario: During a contract upgrade, a new version assumes storage slots follow ERC-7201 alignment (ending in 0x00).

The current misaligned slot causes the new contract to read or write to incorrect storage locations, resulting in unexpected behavior or data loss.

Solution: Update the storage slot calculation to comply with ERC-7201.

```
bytes32 internal constant ADDRESS_STORAGE_POSITION = keccak256(abi.encode(uint256(keccak256("flare.diamond.AddressUpdatable.ADDRESS_STORAGE_POSITION")) - 1)) & ~bytes32(0xff);
```
