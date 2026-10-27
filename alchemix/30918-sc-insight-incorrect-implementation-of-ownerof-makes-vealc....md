# 30918 - \[SC - Insight] Incorrect implementation of ownerOf makes veALC...

Submitted on May 8th 2024 at 04:15:13 UTC by @marchev for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30918

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

As per the ERC721 specification, NFTs assigned to zero address are considered invalid, and `ownerOf()` queries about them must revert. However, the implementation of `VotingEscrow#ownerOf()` returns `address(0)` for such NFTs and not comply with that and thus makes veALCX not compliant with ERC721.

## Vulnerability Details

The [ERC721 specification](https://eips.ethereum.org/EIPS/eip-721) states the following:

```
    /// @notice Find the owner of an NFT
    /// @dev NFTs assigned to zero address are considered invalid, and queries
    ///  about them do throw.
    /// @param _tokenId The identifier for an NFT
    /// @return The address of the owner of the NFT
    function ownerOf(uint256 _tokenId) external view returns (address);
```

This means that every compliant ERC721 token must implement the `ownerOf()` function so that it throws if a `_tokenId` is passed that belongs to `address(0)`. However, `VotingEscrow`'s implementation looks like this:

```sol
    /// @inheritdoc IVotingEscrow
    function ownerOf(uint256 _tokenId) public view override(IERC721, IVotingEscrow) returns (address) {
        return idToOwner[_tokenId];
    }
```

For a non-existent `_tokenId`, this implementation will return `address(0)` instead of reverting. This behavior is not compliant with the ERC721 specification.

## Impact Details

Due to the incorrect implementation of the `ownerOf()` function, the veALCX token does not conform to the ERC721 standard. This leads to problems with composability and interoperability. Other protocols that integrate with the veALCX token may malfunction since they expect standard behaviors the token does not support, potentially leading to operational disruptions.

## References

Problematic implementation:

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L230-L232>

Docs references which state that ERC721 compliance is expected:

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L16>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/CONTRACTS.md>

## Proof of Concept

Add the following test to `src/test/VotingEscrow.t.sol`:

```sol

	function test_erc721_compliance_is_broken_due_to_incorrect_owner_of_implementation() public {
        hevm.expectRevert();
        address noSuchOwner = veALCX.ownerOf(31337); // Invalid tokenId must throw
    }
```

Make sure the following entries are updated in `Makefile`:

```sh
# file to test 
FILE=VotingEscrow

# specific test to run
TEST=test_erc721_compliance_is_broken_due_to_incorrect_owner_of_implementation
```

Run the PoC via `make test_file_test`
