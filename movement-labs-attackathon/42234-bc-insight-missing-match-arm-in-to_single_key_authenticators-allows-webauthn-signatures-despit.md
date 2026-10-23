# #42234 \[BC-Insight] Missing Match Arm in to\_single\_key\_authenticators() Allows WebAuthn Signatures Despite WEBAUTHN\_SIGNATURE Being Disabled

**Submitted on Mar 21st 2025 at 22:43:33 UTC by @jovi for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42234
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Summary

A mismatch between the top‐level `TransactionAuthenticator` variants (which can be multi‐agent or fee payer–based) and the helper function `to_single_key_authenticators()` leads to **WebAuthn** signatures being **silently ignored** when the `WEBAUTHN_SIGNATURE` feature is off. Instead of rejecting transactions that contain WebAuthn signatures, the validator code never even *detects* them, effectively bypassing the feature gating.

## Vulnerability Details

### `TransactionAuthenticator` Enum

The chain supports several transaction authentication modes, including single Ed25519, multi‐Ed25519, multi‐agent, or fee‐payer transactions:

```rust
#[derive(Clone, Debug, Eq, PartialEq, Hash, Serialize, Deserialize)]
pub enum TransactionAuthenticator {
    /// Single Ed25519 signature
    Ed25519 {
        public_key: Ed25519PublicKey,
        signature: Ed25519Signature,
    },
    /// K-of-N multisignature
    MultiEd25519 {
        public_key: MultiEd25519PublicKey,
        signature: MultiEd25519Signature,
    },
    /// Multi-agent transaction.
    MultiAgent {
        sender: AccountAuthenticator,
        secondary_signer_addresses: Vec<AccountAddress>,
        secondary_signers: Vec<AccountAuthenticator>,
    },
    /// Optional Multi-agent transaction with a fee payer.
    FeePayer {
        sender: AccountAuthenticator,
        secondary_signer_addresses: Vec<AccountAddress>,
        secondary_signers: Vec<AccountAuthenticator>,
        fee_payer_address: AccountAddress,
        fee_payer_signer: AccountAuthenticator,
    },
    SingleSender {
        sender: AccountAuthenticator,
    },
}
```

### `to_single_key_authenticators()` Snippet

Below is the function used to convert high‐level signers into a list of `SingleKeyAuthenticator` objects:

```rust
pub fn to_single_key_authenticators(&self) -> Result<Vec<SingleKeyAuthenticator>> {
    let account_authenticators = self.all_signers();
    let mut single_key_authenticators: Vec<SingleKeyAuthenticator> =
        Vec::with_capacity(MAX_NUM_OF_SIGS);

    for account_authenticator in account_authenticators {
        match account_authenticator {
            AccountAuthenticator::Ed25519 { public_key, signature } => {
                let authenticator = SingleKeyAuthenticator {
                    public_key: AnyPublicKey::ed25519(public_key.clone()),
                    signature: AnySignature::ed25519(signature.clone()),
                };
                single_key_authenticators.push(authenticator);
            },
            AccountAuthenticator::MultiEd25519 { public_key, signature } => {
                let public_keys = MultiKey::from(public_key);
                let signatures: Vec<AnySignature> = signature
                    .signatures()
                    .iter()
                    .map(|sig| AnySignature::ed25519(sig.clone()))
                    .collect();
                let signatures_bitmap = aptos_bitvec::BitVec::from(signature.bitmap().to_vec());
                let authenticator = MultiKeyAuthenticator {
                    public_keys,
                    signatures,
                    signatures_bitmap,
                };
                single_key_authenticators
                    .extend(authenticator.to_single_key_authenticators()?);
            },
            AccountAuthenticator::SingleKey { authenticator } => {
                single_key_authenticators.push(authenticator);
            },
            AccountAuthenticator::MultiKey { authenticator } => {
                single_key_authenticators
                    .extend(authenticator.to_single_key_authenticators()?);
            },
        };
    }
    Ok(single_key_authenticators)
}
```

**Problem:** If a FeePayer/MultiAgent authenticator appears, there is no `match` arm for it. If the code compiles and runs without error, that variant may be silently dropped or never converted, effectively bypassing the check for WebAuthn.

### Underlying Gating Check

When `WEBAUTHN_SIGNATURE` is disabled, the validator code attempts to reject any WebAuthn usage:

```rust
if !self.features().is_enabled(FeatureFlag::WEBAUTHN_SIGNATURE) {
    if let Ok(sk_authenticators) = transaction
        .authenticator_ref()
        .to_single_key_authenticators()
    {
        for authenticator in sk_authenticators {
            if let AnySignature::WebAuthn { .. } = authenticator.signature() {
                return VMValidatorResult::error(StatusCode::FEATURE_UNDER_GATING);
            }
        }
    } else {
        return VMValidatorResult::error(StatusCode::INVALID_SIGNATURE);
    }
}
```

But since `to_single_key_authenticators()` never produces any item with `WebAuthn` (due to missing match logic), **no** WebAuthn signature is “found.” The transaction is incorrectly allowed.

### Impact

* Users can include WebAuthn signatures (or other unhandled signatures) in multi‐agent or fee payer transactions, *even with* `WEBAUTHN_SIGNATURE` off.
* It circumvents a chain policy that is meant to block WebAuthn.

## Proof of Concept

1. **Disable** `WEBAUTHN_SIGNATURE` via on‐chain config or local node settings.
2. **Submit** a multi‐agent or fee‐payer transaction carrying an unknown or hypothetical `WebAuthn` variant for the signers.
3. Observe that `to_single_key_authenticators()` does not parse or flag this, and the loop checking for `AnySignature::WebAuthn { .. }` never sees it.
4. The transaction is accepted instead of returning `FEATURE_UNDER_GATING`.
