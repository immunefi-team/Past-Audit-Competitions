# 56658 sc insight transmuter s tokenuri does not revert for nonexistent tokenids

**Submitted on Oct 19th 2025 at 02:53:26 UTC by @tygra for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56658
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

NFTs the Transmuter mints to represent redemptions have a `tokenURI` method, but this differs from the EIP-721 spec as it does not revert for invalid tokenIds.

## Vulnerability Details

The method in question, `tokenURI` calls NFTMetadataGenerator.generateTokenURI in order to return an SVG to describe NFT identified by the requested tokenID.

```
// Transmuter.sol
    function tokenURI(uint256 id) public view override returns (string memory) {
        return NFTMetadataGenerator.generateTokenURI(id, "Transmuter V3 Position");
    }

// NFTMetadataGenerator.sol
    function generateTokenURI(uint256 tokenId, string memory title) internal pure returns (string memory) {
        string memory svg = generateSVG(tokenId, title);
        string memory json = generateJSONString(tokenId, svg);
        return string(abi.encodePacked("data:application/json;base64,", json));
    }

```

Nowhere is the `tokenURI` validated, consequently the contract will return an SVG for nonexistent tokenIDs.

This differs from EIP-721's definition of contracts that implement a `tokenURI` method such that Transmuter.sol fails to deliver the expected behaviour for an EIP-721 compliant contract that includes token metadata.

```
    /// @notice A distinct Uniform Resource Identifier (URI) for a given asset.
    /// @dev Throws if `_tokenId` is not a valid NFT. URIs are defined in RFC
    ///  3986. The URI may point to a JSON file that conforms to the "ERC721
    ///  Metadata JSON Schema".
    function tokenURI(uint256 _tokenId) external view returns (string);
```

## Impact Details

There is no risk of any customer funds being affected. The risk of not reverting for invalid tokenIDs is that onchain or offchain programs attempting to fetch the metadata for a given tokenID can incorrectly believe a tokenID exists due to metadata always being returned.

## References

<https://eips.ethereum.org/EIPS/eip-721>

## Proof of Concept

## Proof of Concept

Add this test to Transmuter.t.sol

```
    function test_tokenURI() public {
        string memory str = transmuter.tokenURI(1e16);
        console.log(str);
    }
```

It returns the following metadata even though token #1e16 does not exist

{"name": "AlchemistV3 Position #10000000000000000", "description": "Position token for Alchemist V3", "image": "data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAzMDAgNDg0IiB3aWR0aD0iMzAwIiBoZWlnaHQ9IjQ4NCI+PHJlY3Qgd2lkdGg9IjMwMCIgaGVpZ2h0PSI0ODQiIGZpbGw9IiNkNGMzYjciIC8+PGNpcmNsZSBjeD0iMTUwIiBjeT0iMTUwIiByPSIxMjAiIGZpbGw9Im5vbmUiIHN0cm9rZT0iIzBhM2E2MCIgc3Ryb2tlLXdpZHRoPSI0IiAvPjx0ZXh0IHg9IjE1MCIgeT0iMTIwIiBmb250LWZhbWlseT0iQXJpYWwiIGZvbnQtc2l6ZT0iMjQiIGZpbGw9IiMwYTNhNjAiIHRleHQtYW5jaG9yPSJtaWRkbGUiPlRyYW5zbXV0ZXIgVjMgUG9zaXRpb248L3RleHQ+PHRleHQgeD0iMTUwIiB5PSIxNTAiIGZvbnQtZmFtaWx5PSJBcmlhbCIgZm9udC1zaXplPSIyOCIgZmlsbD0iIzBhM2E2MCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC13ZWlnaHQ9ImJvbGQiPlBvc2l0aW9uPC90ZXh0Pjx0ZXh0IHg9IjE1MCIgeT0iMTkwIiBmb250LWZhbWlseT0ibW9ub3NwYWNlIiBmb250LXNpemU9IjI0IiBmaWxsPSIjMGEzYTYwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIj4jMTAwMDAwMDAwMDAwMDAwMDA8L3RleHQ+PC9zdmc+"}
