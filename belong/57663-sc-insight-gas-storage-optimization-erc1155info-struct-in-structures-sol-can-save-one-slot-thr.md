# 57663 sc insight gas storage optimization erc1155info struct in structures sol can save one slot through field reordering

* **Submitted on:** Oct 27th 2025 at 23:19:41 UTC by @chief\_hunter888 for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* **Report ID:** #57663
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/Structures.sol>

## Description

### Brief Summary

The `Structures.sol` file defines multiple custom data structures (`AccessTokenInfo`, `ERC1155Info`, `CustomerInfo`, `PromoterInfo`, etc.) that contain a mix of fields: address, bool, and smaller integer types (uint96, uint128, uint24, etc.).

Problem: In the `Structures.sol` struct layout, several structs are not optimally packed for efficient storage utilisation, resulting in unnecessary consumption of additional 32-byte storage slots. This increases gas cost for storing these structs. Solidity packs contiguous smaller types (<32 bytes) together if adjacent in declaration order and combined size ≤ 32 bytes. Dynamic types and large types (address, uint256, string, bytes, mapping) start a new slot.

Solution: Move the `bool transferable` in `ERC1155Info` next to the static fields (addresses) so it can be packed into an existing slot rather than occupying its own slot. This reduces the number of storage slots used per `ERC1155Info` struct from 8 to 7, saving one 32-byte slot (≈ \~20,000 gas on initial write).

## Context: Solidity Storage Basics - How to Optimize Solidity Storage

Each storage slot = 32 bytes (256 bits). Solidity packs contiguous smaller types (<32 bytes) together into the same slot if:

* They are adjacent in declaration order.
* Their combined size ≤ 32 bytes.

Group smaller types together before introducing larger ones (`address`, `uint256`, `string`, `bytes`, `mapping`, etc. always start a new slot).

Below are the relevant structs and the suggested reordering for `ERC1155Info`.

***

## 3. `ERC1155Info`

Current declaration in `Structures.sol`:

```solidity
struct ERC1155Info {
    string name;
    string symbol;
    address defaultAdmin;
    address manager;
    address minter;
    address burner;
    string uri;
    bool transferable;
}
```

### Improvement Suggestion

Dynamic fields (`string`) always occupy their own slots, so packing is limited. The `bool transferable` could be grouped before the dynamic fields (`string`) to share a slot with `address burner` (or any adjacent address). Reorder to place `bool transferable` adjacent to the addresses so it fits into the remaining bytes of an address slot instead of forcing a new slot.

Suggested optimized layout:

```solidity
struct ERC1155Info {
    string name;
    string symbol;
    address defaultAdmin;
    bool transferable;
    address manager;
    address minter;
    address burner;
    string uri;
}
```

Rationale: The four addresses (4 × 20 bytes = 80 bytes) still occupy four slots, but the `bool` (1 byte) can now be packed into the slot containing an address, saving one full slot compared to the original layout where the `bool` was declared after dynamic fields and forced into its own slot.

✅ Saves 1 storage slot vs current layout.

***

## `ERC1155Info` Finding Summary Details

Finding: The struct `ERC1155Info` — Can save 1 storage slot

Current Layout (rough):

* Slot 1-2: string name, string symbol (dynamic)
* Slot 3: address defaultAdmin (20 bytes) → 12 bytes wasted
* Slot 4: address manager (20 bytes) → 12 bytes wasted
* Slot 5: address minter (20 bytes) → 12 bytes wasted
* Slot 6: address burner (20 bytes) → 12 bytes wasted
* Slot 7+: string uri (dynamic)
* Slot 8+: bool transferable (1 byte) → 31 bytes wasted

Optimized: Move `bool transferable` next to any address to pack them together (20 + 1 = 21 bytes in one slot).

Improved storage layout (example):

* Slot 1-2: string name, string symbol (dynamic)
* Slot 3: address defaultAdmin (20 bytes) + bool (1 byte) → 11 bytes wasted
* Slot 4: address manager (20 bytes) → 12 bytes wasted
* Slot 5: address minter (20 bytes) → 12 bytes wasted
* Slot 6: address burner (20 bytes) → 12 bytes wasted
* Slot 7+: string uri (dynamic)

One full slot saved.

Example docstring for the optimized struct:

```solidity
/// @title ERC1155Info
/// @notice Initialization/configuration data for a CreditToken (ERC-1155) collection.
struct ERC1155Info {
    string name;
    string symbol;
    address defaultAdmin;
    bool transferable;
    address manager;
    address minter;
    address burner;
    string uri;
}
```

***

## Summary of Potential Storage Improvements

| Struct                   | Slots Saved (Est.) | Change                           |
| ------------------------ | ------------------ | -------------------------------- |
| `NftMetadata`            | 0                  | optimal                          |
| `AccessTokenInfo`        | 0                  | Optimal                          |
| `ERC1155Info`            | **1 slot**         | Move `bool` above dynamic fields |
| `VestingWalletInfo`      | 0                  | Optimal                          |
| `StaticPriceParameters`  | 0                  | Optimal                          |
| `DynamicPriceParameters` | 0                  | Optimal                          |
| `VenueRules`             | 0                  | Optimal                          |
| `VenueInfo`              | 0                  | Optimal                          |
| `CustomerInfo`           | 0                  | Optimal                          |
| `PromoterInfo`           | 0                  | Optimal                          |

***

## Impact Details

Reordering the fields of `ERC1155Info` achieves a saving of one 32-byte storage slot. Each eliminated slot corresponds to a reduction of approximately:

* \~20,000 gas when first written (cold SSTORE), and
* \~5,000 gas for subsequent updates (warm writes differ depending on state transitions and EVM specifics).

This improvement is non-invasive, requires no logic changes, and directly enhances contract performance for storage-heavy operations (deployments, updates to these structs, or repeated writes).

***

## Proof of Concept

The optimized `Structures.sol` file (showing the adjusted `ERC1155Info` and full file for context):

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.27;

struct NftMetadata {
    /// @notice The name of the NFT collection.
    string name;
    /// @notice The symbol representing the NFT collection.
    string symbol;
}

/// @title AccessTokenInfo
/// @notice Initialization/configuration data for an AccessToken (ERC-721) collection.
/// @dev
/// - `paymentToken` can be a token address or the NativeCurrency pseudo-address
///   (0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE).
/// - `feeNumerator` is used for ERC-2981 royalty configuration.
/// - `signature` is validated off-chain by a platform signer.
struct AccessTokenInfo {
    /// @notice ERC-20 used for payments, or NativeCurrency pseudo-address for native NativeCurrency.
    address paymentToken;
    /// @notice ERC-2981 royalty numerator (denominator defined by receiver).
    uint96 feeNumerator;
    /// @notice Whether transfers between users are allowed.
    bool transferable;
    /// @notice Collection-wide supply cap.
    uint256 maxTotalSupply;
    /// @notice Public mint price.
    uint256 mintPrice;
    /// @notice Whitelist mint price.
    uint256 whitelistMintPrice;
    /// @notice Optional collection expiration timestamp (seconds since epoch).
    uint256 collectionExpire;
    /// @notice Collection name and symbol stored as NftMetadata struct.
    NftMetadata metadata;
    /// @notice Contract-level metadata URI.
    string contractURI;
    /// @notice Backend signature authorizing creation with the provided fields.
    bytes signature;
}

/// @title ERC1155Info
/// @notice Initialization/configuration data for a CreditToken (ERC-1155) collection.
struct ERC1155Info {
    string name;
    string symbol;
    address defaultAdmin;
    bool transferable;
    address manager;
    address minter;
    address burner;
    string uri;
}

/// @title VestingWalletInfo
/// @notice Parameters configuring a vesting wallet schedule and metadata.
struct VestingWalletInfo {
    /// @notice Vesting start timestamp (TGE) in seconds since epoch.
    uint64 startTimestamp;
    /// @notice Cliff duration in seconds added to `startTimestamp` for the linear section to begin.
    uint64 cliffDurationSeconds;
    /// @notice Linear vesting duration in seconds counted from `cliff`.
    uint64 durationSeconds;
    /// @notice ERC-20 token being vested.
    address token;
    /// @notice Recipient of vested token releases.
    address beneficiary;
    /// @notice Total tokens allocated to this vesting schedule (must equal TGE + linear + tranches).
    uint256 totalAllocation;
    /// @notice One-off amount vested immediately at `startTimestamp`.
    uint256 tgeAmount;
    /// @notice Amount linearly vested over `durationSeconds` starting at `cliff`.
    uint256 linearAllocation;
    /// @notice Human-readable description of the vesting schedule.
    string description;
}

/// @title StaticPriceParameters
/// @notice Mint payload for static-priced mints validated by a platform signer.
struct StaticPriceParameters {
    /// @notice Token id to mint.
    uint256 tokenId;
    /// @notice Whether receiver is eligible for whitelist pricing.
    bool whitelisted;
    /// @notice Token metadata URI.
    string tokenUri;
    /// @notice Backend signature validating the payload.
    bytes signature;
}

/// @title DynamicPriceParameters
/// @notice Mint payload for dynamic-priced mints validated by a platform signer.
struct DynamicPriceParameters {
    /// @notice Token id to mint.
    uint256 tokenId;
    /// @notice Explicit price for this mint.
    uint256 price;
    /// @notice Token metadata URI.
    string tokenUri;
    /// @notice Backend signature validating the payload.
    bytes signature;
}

/// @title StakingTiers
/// @notice Tier levels derived from staked LONG balance.
enum StakingTiers {
    NoStakes,
    BronzeTier,
    SilverTier,
    GoldTier,
    PlatinumTier
}

/// @title PaymentTypes
/// @notice Venue-allowed payment currencies.
enum PaymentTypes {
    NoType,
    USDC,
    LONG,
    Both
}

/// @title BountyTypes
/// @notice Venue-allowed promoter bounty schemes.
enum BountyTypes {
    NoType,
    VisitBounty,
    SpendBounty,
    Both
}

/// @title LongPaymentTypes
/// @notice Venue-allowed Long payment options.
enum LongPaymentTypes {
    NoType,
    AutoStake,
    AutoConvert
}

/// @title VenueRules
/// @notice Venue-level configuration for payment and bounty types.
struct VenueRules {
    PaymentTypes paymentType;
    BountyTypes bountyType;
    LongPaymentTypes longPaymentType;
}

/// @title VenueInfo
/// @notice Signed payload authorizing a venue deposit and metadata update.
struct VenueInfo {
    VenueRules rules;
    address venue;
    uint256 amount;
    bytes32 referralCode;
    string uri;
    bytes signature;
}

/// @title CustomerInfo
/// @notice Signed payload authorizing a customer payment to a venue (and optional promoter attribution).
struct CustomerInfo {
    // Backend configurable
    bool paymentInUSDC;
    uint128 visitBountyAmount;
    uint24 spendBountyPercentage;
    // Actors
    address customer;
    address venueToPayFor;
    address promoter;
    // Amounts
    uint256 amount;
    bytes signature;
}

/// @title PromoterInfo
/// @notice Signed payload authorizing distribution of promoter payouts in USDC or LONG.
struct PromoterInfo {
    bool paymentInUSDC;
    address venue;
    uint256 amountInUSD;
    bytes signature;
}
```

***

## Summary

* Reordering `bool transferable` to sit next to address fields in `ERC1155Info` saves one storage slot per struct instance.
* This is a low-risk, non-functional change that reduces gas cost for writes and updates related to `ERC1155Info`.
* No logic changes required — only declaration order adjustment to improve packing.
