# #38344 \[SC-Low] Old validated messages can not pass proof check when new validators are set

**Submitted on Dec 31st 2024 at 13:24:54 UTC by @security for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38344
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/consortium/Consortium.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Protocol insolvency

## Description

### Brief/Intro

During the proof verification process, the signatures provided are checked against the **current** validator set. However, the message may have been validated using a previous validator set, as validator rotations occur frequently for security reasons. Verification should instead be performed against the validator set active during the epoch when the message was validated, not the current epoch.

### Vulnerability Details

Imagine a deposit is initiated for bridging to the destination chain, where the message is validated using the current validator set. The signatures of this set are included in the proof for withdrawal on the destination chain.

If the validator set is updated before the message is delivered to the destination chain, proof verification will fail. This happens because the proof contains signatures from the earlier validator set (e.g., epoch 500), whereas the current validator set belongs to a new epoch (e.g., epoch 501). This mismatch causes the proof verification to revert with the error `NotEnoughSignatures`.\
[Reference](https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/consortium/Consortium.sol#L240)

The exact mechanism for off-chain validator rotations is beyond the scope of this discussion, so the details of how validator sets are updated remain unclear.

### Impact Details

Frequent validator rotations render messages validated by earlier validator sets unverifiable. This leads to:

* Griefing scenarios where an attacker disrupts the protocol or user activity without financial gain (e.g., causing losses exceeding $1,000).
* Contract fails to deliver promised returns, but doesn't lose value
* Protocol insolvency

## References

## Proof of Concept

## PoC

The absence of the epoch number in the proof prevents the protocol from identifying the validator set that validated the message during proof verification.

Adding the epoch number to the proof enables accurate verification by ensuring alignment between the validator set used for validation and the one used for proof verification. This prevents failures resulting from validator rotations and improves protocol reliability.

```diff
    function _checkProof(
        bytes32 _payloadHash,
        bytes calldata _proof,
+       uint256 epoch
    ) internal view virtual {
        ConsortiumStorage storage $ = _getConsortiumStorage();
-       if ($.epoch == 0) {
+       if (epoch == 0) {
            revert NoValidatorSet();
        }
        // decode proof
        bytes[] memory signatures = abi.decode(_proof, (bytes[]));

-       address[] storage validators = $.validatorSet[$.epoch].validators;
+       address[] storage validators = $.validatorSet[epoch].validators;
        uint256 length = validators.length;
        if (signatures.length != length) {
            revert LengthMismatch();
        }

        uint256 weight = 0;
-       uint256[] storage weights = $.validatorSet[$.epoch].weights;
+       uint256[] storage weights = $.validatorSet[epoch].weights;
        for (uint256 i; i < length; ++i) {
            // each signature preset R || S values
            // V is missed, because validators use Cosmos SDK keyring which is not signing in eth style
            // We only check signatures which are the expected 64 bytes long - we are expecting
            // a signatures array with the same amount of items as there are validators, but not all
            // validators will need to sign for a proof to be valid, so validators who have not signed
            // will have their corresponding signature set to 0 bytes.
            // In case of a malformed signature (i.e. length isn't 0 bytes but also isn't 64 bytes)
            // this signature will be discarded.
            if (signatures[i].length == 64) {
                // split signature by R and S values
                bytes memory sig = signatures[i];
                bytes32 r;
                bytes32 s;

                // load the first 32 bytes (r) and the second 32 bytes (s) from the sig
                assembly {
                    r := mload(add(sig, 0x20)) // first 32 bytes (offset 0x20)
                    s := mload(add(sig, 0x40)) // next 32 bytes (offset 0x40)
                }

                if (r != bytes32(0) && s != bytes32(0)) {
                    // try recover with V = 27
                    (address signer, ECDSA.RecoverError err, ) = ECDSA
                        .tryRecover(_payloadHash, 27, r, s);

                    // revert if bad signature
                    if (err != ECDSA.RecoverError.NoError) {
                        continue;
                    }

                    // if signer doesn't match try V = 28
                    if (signer != validators[i]) {
                        (signer, err, ) = ECDSA.tryRecover(
                            _payloadHash,
                            28,
                            r,
                            s
                        );
                        if (err != ECDSA.RecoverError.NoError) {
                            continue;
                        }

                        if (signer != validators[i]) {
                            continue;
                        }
                    }
                    // signature accepted

                    unchecked {
                        weight += weights[i];
                    }
                }
            }
        }
-       if (weight < $.validatorSet[$.epoch].weightThreshold) {
+       if (weight < $.validatorSet[epoch].weightThreshold) {
            revert NotEnoughSignatures();
        }
    }
```
