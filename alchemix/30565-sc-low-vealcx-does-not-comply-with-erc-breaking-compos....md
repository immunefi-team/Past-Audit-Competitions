# 30565 - \[SC - Low] veALCX does not comply with ERC breaking compos...

Submitted on Apr 30th 2024 at 22:36:45 UTC by @marchev for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30565

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

veALCX is intended to be ERC721 compliant. However, it does not comply with ERC721 due to broken EIP165 implementation which is mandated by the ERC721 specification.

## Vulnerability Details

The [ERC721 specification](https://eips.ethereum.org/EIPS/eip-721) states the following:

> **Every ERC-721 compliant contract must implement the `ERC721` and `ERC165` interfaces**

This means that every compliant ERC721 token must implement EIP-165 and return `true` for all supported interfaces. veALCX (`VotingEscrow.sol`) implements both `IERC721` and `IERC721Metadata`. However, the `VotingEscrow#supportInterface()` does not work as intended:

```sol
    function supportsInterface(bytes4 _interfaceID) external pure returns (bool) {
        revert("function not supported");
    }
```

As per the [EIP-165 specification](https://eips.ethereum.org/EIPS/eip-165#how-to-detect-if-a-contract-implements-erc-165), this implementation does not comply with EIP-165 (which is mandated by ERC721):

> How to Detect if a Contract Implements ERC-165
>
> 1. The source contract makes a `STATICCALL` to the destination address with input data: `0x01ffc9a701ffc9a700000000000000000000000000000000000000000000000000000000` and gas 30,000. This corresponds to `contract.supportsInterface(0x01ffc9a7)`.
> 2. If the call fails or return false, the destination contract does not implement ERC-165.

Thus, `VotingEscrow` does not implement EIP-165 and is not ERC721 compliant.

In pursuit of thoroughness and transparency, it's crucial to reference **Finding 6.45** in [Chainsecurity's audit](https://drive.google.com/file/d/1YsO1t1-hSK1wkHajT_GAZ-u35O1Su74X/view) which reports broken/partial EIP165 support. However, the applied fix is incorrect and breaks the intended compliance with ERC721. Recognizing and addressing the currently reported issue is vital for the compliance and composability of the veALCX token.

## Impact Details

The token does not comply with the ERC721 standard, which causes composability and interoperability issues. This leads to failures when other contracts and DApps expect standard behaviors that the token does not provide, potentially causing disruptions, reducing its utility, and diminishing trust in the token's reliability.

## References

Problematic implementation:

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L174-L176>

Docs references which state that ERC721 compliance is expected:

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L16>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/CONTRACTS.md>

## Proof of Concept

Add the following test to `src/test/VotingEscrow.t.sol`:

```sol

    function test_eip165_compliance_is_broken() public {
        // As per https://eips.ethereum.org/EIPS/eip-721#specification:
        // "Every ERC-721 compliant contract must implement the ERC721 and ERC165 interfaces"
        assertEq(veALCX.supportsInterface(0x01ffc9a7), true); // ERC165
        assertEq(veALCX.supportsInterface(0x80ac58cd), true); // ERC721
        assertEq(veALCX.supportsInterface(0x5b5e139f), true); // ERC721Metadata
    }
```

Make sure the following entries are updated in `Makefile`:

```sh
# file to test 
FILE=VotingEscrow

# specific test to run
TEST=test_eip165_compliance_is_broken
```

Run the PoC via `make test_file_test`
