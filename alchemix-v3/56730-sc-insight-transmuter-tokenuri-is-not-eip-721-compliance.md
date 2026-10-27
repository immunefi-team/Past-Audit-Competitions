# 56730 sc insight transmuter tokenuri is not eip 721 compliance

**Submitted on Oct 19th 2025 at 23:38:25 UTC by @lirezarazavi for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56730
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`tokenURI()` is violating EIP-721 by not reverting for non-existent token id's.

## Vulnerability Details

According to the EIP-721 specification, the function should throw an error if the provided \_tokenId does not correspond to a valid NFT. However, the current implementation returns a non-empty SVG/String for non-existent tokenId values.

## Impact Details

* Violation of EIP-721
* This behavior can lead to confusion and makes it difficult for clients and applications to determine the validity of a token.

## References

* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L153-L155>
* <https://eips.ethereum.org/EIPS/eip-721#:\\~:text=///%20%40dev%20Throws%20if%20%60\\_tokenId%60%20is%20not%20a%20valid%20NFT.%20URIs%20are%20defined%20in%20RFC>

## Proof of Concept

## Proof of Concept

1. Add (copy and paste) the following function to `src/test/Transmuter.t.sol`:

```solidity
function testTokenURI_PoC() public {
        vm.prank(address(0xbeef));

        // No NFT is minted and thus totalSupply is Zero
        uint256 totalNFTMinted = transmuter.totalSupply();
        assertEq(totalNFTMinted, 0);

        // tokenURI doesn't revert for non-existent tokenId's 
        bytes memory uri_str;

        uri_str = bytes(transmuter.tokenURI(1)); // tokenId = 1
        console.log(string(uri_str)); 
        assertNotEq(uri_str.length, 0);   // tokenURI returned a SVG/string instead of reverting

        uri_str = bytes(transmuter.tokenURI(2)); 
        console.log(string(uri_str)); 
        assertNotEq(uri_str.length, 0);
    }
```

2. Run: `forge test --match-test testTokenURI_PoC`.
