# 57911 sc medium signature are malleable in signatureverifier sol&#x20;

* Report ID: #57911
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* Submitted on: Oct 29th 2025 at 12:38:02 UTC by @Bx4 for Audit Comp | Belong

{% hint style="warning" %}
Signature malleability is not prevented by the imported signature checking library. This may allow signature reuse or manipulation even if funds are not directly lost.
{% endhint %}

## Description

### Brief / Intro

In `SignatureVerifier.sol`, the protocol does not implement checks to prevent signature malleability.

### Vulnerability Details

`SignatureVerifier.sol` imports `SignatureCheckerLib` from Solady. That library uses its `isValidSignatureNow` implementation to verify signatures.

According to the Solady documentation, the imported library does not prevent malleability, and consumers must implement their own checks against malleability. The current implementation does not add such checks.

Malleability allows an existing signature to be manipulated and potentially reused.

## Impact Details

Signature malleability can be abused by attackers for personal gains. The contract therefore fails to fully guarantee the integrity of verified signatures, even if no direct loss of funds occurs.

## References

* <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/utils/SignatureVerifier.sol#L4>
* <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/utils/SignatureVerifier.sol#L28>
* Solady SignatureCheckerLib note: <https://github.com/Vectorized/solady/blob/73f13dd1483707ef6b4d16cb0543570b7e1715a8/src/utils/SignatureCheckerLib.sol#L23>

## Proof of Concept

<details>

<summary>View PoC</summary>

```solidity
/// WARNING! Do NOT use signatures as unique identifiers:
/// - Use a nonce in the digest to prevent replay attacks on the same contract.
/// - Use EIP-712 for the digest to prevent replay attacks across different chains and contracts.
///   EIP-712 also enables readable signing of typed data for better user safety.
> /// This implementation does NOT check if a signature is non-malleable.
library SignatureCheckerLib {
```

From the snippet above we can see that the library imported does not protect signatures against malleability and the current implementation does not add any prevention against malleability.

</details>
