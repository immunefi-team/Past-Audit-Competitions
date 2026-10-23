# 57989 sc low broken isvalidsignature leads to fund freezing&#x20;

**Submitted on Oct 29th 2025 at 20:26:42 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57989
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Wrong isValidSignature leads to permenant freeze of funds in strategies don't have direct withdraws or require swaps

## Description

## Summary

The `isValidSignature()` function in `MYTStrategy` attempts to delegate signature verification to Permit2 by calling `IPermit2(permit2Address).isValidSignature()`, which does not exist in the Permit2 contract. This causes all Permit2 operations where the strategy contract acts as the token owner to revert, resulting in a complete denial of service of any Permit2-based functionality

## Description

The current implementation of `isValidSignature()` in `MYTStrategy.sol`:

```solidity
function isValidSignature(bytes32 _hash, bytes memory _signature) public view returns (bytes4) {
    return IPermit2(permit2Address).isValidSignature(_hash, _signature);
}
```

This implementation is fundamentally broken because:

1. **Permit2 does not have an `isValidSignature` function** - The Permit2 contract (`SignatureTransfer` and `AllowanceTransfer`) does not expose any `isValidSignature` function. The interface `IPermit2` defined in the contract is incorrect.

`SignatureTransfer.sol`: <https://github.com/Uniswap/permit2/blob/main/src/SignatureTransfer.sol>

`AllowanceTransfer.sol`: <https://github.com/Uniswap/permit2/blob/main/src/AllowanceTransfer.sol>

2. **Backwards delegation logic** - When Permit2 calls `permitTransferFrom()` with a smart contract as the owner, it internally calls that contract's `isValidSignature()` for ERC-1271 validation. The strategy contract should validate signatures itself, not delegate back to Permit2.
3. **Incorrect understanding of ERC-1271 pattern** - The ERC-1271 standard requires the implementing contract to verify signatures locally and return the magic value `0x1626ba7e` if valid. The current implementation attempts to call an external contract instead.

### **How Permit2 Actually Works:**

When `ISignatureTransfer(permit2).permitTransferFrom(permit, details, owner, signature)` is called it initiaite an intenral call to `signatureVerify` lib which validate the sig:

`_permitTransfer`: <https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/SignatureTransfer.sol#L65>

```solidity
 function _permitTransferFrom(
        PermitTransferFrom memory permit,
        SignatureTransferDetails calldata transferDetails,
        address owner,
        bytes32 dataHash,
        bytes calldata signature
    ) private {
        uint256 requestedAmount = transferDetails.requestedAmount;
// @Cropped
@>>        signature.verify(_hashTypedData(dataHash), owner);

        ERC20(permit.permitted.token).safeTransferFrom(owner, transferDetails.to, requestedAmount);
    }

```

`Signature::verify`: <https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/libraries/SignatureVerification.sol#L42C8-L44C101>

```solidity
// Inside Permit2's SignatureVerification.verify()
if (owner.code.length > 0) {
    // Owner is a contract - call its isValidSignature
    bytes4 magicValue = IERC1271(owner).isValidSignature(hash, signature);
    if (magicValue != 0x1626ba7e) revert InvalidContractSignature();
}
```

When the owner is the strategy contract, Permit2 calls the strategy's `isValidSignature()`, which then tries to call Permit2's non-existent `isValidSignature()`, causing the transaction to revert.

## Impact

1. **PermenantFreezeOfFunds**- Based on protocol team intentions `permit2` gonna used with `0x` in strategies that don't have the ability to instantly withdraw, or require swaps.leaving the funds allocated forever
2. **100% failure rate** - Any call to `permitTransferFrom()` where the strategy is the token owner will always revert
3. **No workaround available** - MytStrategy is not upgradeable and The function will revert before any token transfer logic executes
4. **Affects all future Permit2 features** - Any planned signature-based transfers, gasless transactions, or whitelisted allocator operations using Permit2 will be permanently broken
5. **Blocks ZeroX integration** -Third-party protocols attempting to integrate with the strategy via Permit2 will fail

## Mitigation

* Replace the current implementation with proper ERC-1271 signature verification

```solidity
import {ECDSA} from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

/// @notice ERC-1271 signature verification for Permit2 compatibility
/// @dev Permit2 calls this function to validate signatures when the strategy is the token owner
function isValidSignature(bytes32 _hash, bytes memory _signature) 
    public 
    view 
    returns (bytes4) 
{
    // Recover the signer from the signature
    address recovered = ECDSA.recover(_hash, _signature);
    
    // Check if the signer is authorized (owner or whitelisted allocator)
    if (recovered == owner() || whitelistedAllocators[recovered]) {
        return 0x1626ba7e; // ERC-1271 magic value indicating valid signature
    }
    
    return 0xffffffff; // Invalid signature
}
```

## Proof of Concept

## Proof of Concept

> Note Permit2AddressMainnet= 0x000000000022d473030f116ddee9f6b43ac78ba3

### 1. Import the following test in `MYTStrategy.t.sol`

```solidity
    function test_isValidSignature_always_reverts() public {
        console.log("=== Test: isValidSignature Always Reverts ===");
        console.log("");

        // Step 1: Create a valid signature
        console.log("Step 1: Creating a valid ECDSA signature");
        (address signer, uint256 signerKey) = makeAddrAndKey("signer");
        console.log("  Signer address:", signer);
        console.log("  Signer private key:", signerKey);

        bytes32 messageHash = keccak256("Test message");
        console.log("  Message hash:");
        console.logBytes32(messageHash);

        (uint8 v, bytes32 r, bytes32 s) = vm.sign(signerKey, messageHash);
        bytes memory signature = abi.encodePacked(r, s, v);
        console.log("  Signature length:", signature.length, "bytes");
        console.log("  Signature (r,s,v):");
        console.logBytes(signature);
        console.log("");

        // Step 2: Verify the signature is valid using ecrecover
        console.log("Step 2: Verifying signature is valid (using ecrecover)");
        address recovered = ecrecover(messageHash, v, r, s);
        console.log("  Recovered signer:", recovered);
        console.log("  Expected signer:", signer);
        console.log("  Signature is valid:", recovered == signer);
        console.log("");

        // Step 3: Call isValidSignature and expect revert
        console.log("Step 3: Calling strategy.isValidSignature()");
        console.log("  Expected behavior: Should validate signature and return True or False "); // wrong return value type expected bytes4
        console.log("  Actual behavior: Will revert because Permit2 has no isValidSignature function");
        console.log("");

        console.log("=== CALLING isValidSignature ===");

        // This will revert because:
        // 1. strategy.isValidSignature() tries to call IPermit2(permit2).isValidSignature()
        // 2. Permit2 contract has no isValidSignature function
        // 3. Transaction reverts with function selector not found
        vm.expectRevert();
        bytes4 result = strategy.isValidSignature(messageHash, signature);

        console.log("");
        console.log("=== Result ===");
        console.log(" Test passed: isValidSignature reverted as expected");
        console.log(" Issue confirmed: Function is broken and will always revert");
        console.log("");
        console.log("Impact: ANY Permit2 operation will fail");
    }

```

### 2.Run it via `forge test --mc MYTStrategyTest --mt test_isValidSignature_always_revert -vvv`

### Logs

```
Logs:
  === Test: isValidSignature Always Reverts ===
  
  Step 1: Creating a valid ECDSA signature
    Signer address: 0x6E12D8C87503D4287c294f2Fdef96ACd9DFf6bd2
    Signer private key: 49099792800763675079532137679706322989817545357788440619111868498148356080914
    Message hash:
  0xceb871db69754f583338155828aa4ed5d9c89918bd9227275b0a71515469254a
    Signature length: 65 bytes
    Signature (r,s,v):
  0xf6b8f7e21198d0985300c7c9a6fb8e2906b480a9e3a21ecd2bc71b8abd5ac8c60d4ad19ad738931049694e8da4452458e246c7836a1f047823848a330c17725a1c
  
  Step 2: Verifying signature is valid (using ecrecover)
    Recovered signer: 0x6E12D8C87503D4287c294f2Fdef96ACd9DFf6bd2
    Expected signer: 0x6E12D8C87503D4287c294f2Fdef96ACd9DFf6bd2
    Signature is valid: true
  
  Step 3: Calling strategy.isValidSignature()
    Expected behavior: Should validate signature and return True or False 
    Actual behavior: Will revert because Permit2 has no isValidSignature function
  
  === CALLING isValidSignature ===
  
  === Result ===
   Test passed: isValidSignature reverted as expected
   Issue confirmed: Function is broken and will always revert
  
  Impact: ANY Permit2 operation will fail


```
