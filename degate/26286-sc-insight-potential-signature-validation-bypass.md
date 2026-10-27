# 26286 - \[SC - Insight] Potential Signature Validation Bypass

Submitted on Nov 30th 2023 at 09:33:15 UTC by @EricTee for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26286

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/contracts/lib/SignatureUtil.sol#L32-L51>

Impacts:

* Signature Validation Bypass

## Description

## Bug Description

In `https://github.com/degatedev/protocols/blob/degate_mainnet/packages/loopring_v3/contracts/lib/SignatureUtil.sol#L32-L51`:

```
    function verifySignatures(
        bytes32          signHash,
        address[] memory signers,
        bytes[]   memory signatures
        )
        internal
        view
        returns (bool)
    {
        require(signers.length == signatures.length, "BAD_SIGNATURE_DATA");
        address lastSigner;
        for (uint i = 0; i < signers.length; i++) {
            require(signers[i] > lastSigner, "INVALID_SIGNERS_ORDER");
            lastSigner = signers[i];
            if (!verifySignature(signHash, signers[i], signatures[i])) {
                return false;
            }
        }
        return true;
    }
```

There is a potential signature validation bypass scenario happens in the above code, since there is no check that `require(signers.length != 0)`, if an attacker passes in empty array of `signers` and `signatures` values, he can bypass the for loop. As a result, the function will directly return `true`. Though this function is not used anyway in the protocol yet, but I think the awareness should be raised to the protocol team in case this function is implemented in the future.

## Impact

Potential Signature Validation Bypass

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: Signature Validation Bypass CVSS2 Score: NA

## Recommendation

Consider adding the check `require(signers.length != 0, "BAD_SIGNATURE_DATA");`:

```diff
    function verifySignatures(
        bytes32          signHash,
        address[] memory signers,
        bytes[]   memory signatures
        )
        internal
        view
        returns (bool)
    {
++      require(signers.length != 0, "BAD_SIGNATURE_DATA");
        require(signers.length == signatures.length, "BAD_SIGNATURE_DATA");
        address lastSigner;
        for (uint i = 0; i < signers.length; i++) {
            require(signers[i] > lastSigner, "INVALID_SIGNERS_ORDER");
            lastSigner = signers[i];
            if (!verifySignature(signHash, signers[i], signatures[i])) {
                return false;
            }
        }
        return true;
    }
```

## References

NA

## Proof of concept

NA
