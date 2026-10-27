# 57929 sc medium produce function doesn t check if creator is the caller allowing frontrunning attacks

**Submitted on Oct 29th 2025 at 13:52:54 UTC by @kenzo for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57929
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* **Impacts:**
  * Theft of unclaimed yield
  * Theft of unclaimed royalties

## Description

The `_produce` function in `nftfactory.cairo` validates a signature for a `ProduceHash` message that includes `info.creator_address`, but the NFT deployment and ownership assignment use `get_caller_address()` instead. There is no check that the caller matches the `creator_address` in the signed message.

The signature validation only verifies that the message was signed by the authorized signer, but does not verify that `get_caller_address() == info.creator_address`. This allows an attacker to reuse a valid signature meant for someone else's `creator_address` and deploy the NFT under their own control.

Relevant code excerpt:

```rust
// Signature validation - checks if signature is valid for the message
let message = ProduceHash {
    name_hash: metadata_name_hash,
    symbol_hash: metadata_symbol_hash,
    contract_uri_hash: contract_uri_hash,
    royalty_fraction: info.royalty_fraction,
    creator_address: info.creator_address,  // ← Signed creator_address
};

let hash = message.get_message_hash(signerAddress);
let is_valid_signature_felt = signer.is_valid_signature(hash, info.signature);
assert(
    is_valid_signature_felt == starknet::VALIDATED || is_valid_signature_felt == 1,
    super::Errors::VALIDATION_ERROR,
);

// NFT deployment uses CALLER, not creator_address
let mut nft_constructor_calldata: Array<felt252> = array![];
nft_constructor_calldata.append_serde(get_caller_address());  // ← Uses CALLER!
// ...
self.nft_info.write(
    (metadata_name_hash, metadata_symbol_hash),
    NftInfo {
        creator: get_caller_address(),  // ← Stored as CALLER!
        // ...
    },
);
```

## Impact

An attacker can hijack NFT collection deployments by impersonating legitimate creators, steal intellectual property by deploying collections under their control, drain creator funds by redirecting mint proceeds and royalties to themselves, gain full administrative control over NFT contracts including setting prices, whitelisting users, and upgrading contracts, and cause reputation damage to legitimate creators.

## Attack Path

{% stepper %}
{% step %}

### Alice requests a signature

Alice requests a signature from the backend signer for her `creator_address`. The backend creates `ProduceHash` with `creator_address: Alice` and signs the message, returning the signature to Alice.
{% endstep %}

{% step %}

### Alice submits produce()

Alice submits a transaction calling `produce()` with her signature.
{% endstep %}

{% step %}

### Bob snipes signature from mempool

Bob sees this transaction in the mempool before it's mined and extracts the signature from Alice's transaction data.
{% endstep %}

{% step %}

### Bob frontruns with Alice's signature

Bob frontruns Alice by submitting his own `produce()` transaction with the same signature but from his address and with a higher fee/priority. Since the signature is valid for Alice's `creator_address`, validation passes, but Bob's transaction gets mined first.
{% endstep %}

{% step %}

### NFT deployed to Bob

Bob calls `produce()` with Alice's signature but from his own address. He passes `instance_info.creator_address = Alice` and `instance_info.signature = Alice's signature`, but calls the function from his own address so `get_caller_address() = Bob`. Signature validation passes because the signature is valid for Alice's `creator_address`, but the NFT gets deployed with Bob as owner/creator since it uses `get_caller_address()`. Bob becomes the NFT contract owner and receives all mint proceeds and creator fees.
{% endstep %}
{% endstepper %}

## Recommendation

Add a check after signature validation to ensure the caller matches the `creator_address` in the signed message:

```cairo
fn _produce(
    ref self: ContractState, instance_info: InstanceInfo,
) -> (ContractAddress, ContractAddress) {
    // ... existing code ...
    
    let is_valid_signature_felt = signer.is_valid_signature(hash, info.signature);
    assert(
        is_valid_signature_felt == starknet::VALIDATED || is_valid_signature_felt == 1,
        super::Errors::VALIDATION_ERROR,
    );
    
    assert(
        get_caller_address() == info.creator_address,
        'Caller must match signed creator',
    );
    
    // ... rest of deployment code ...
}
```

This ensures only the creator specified in the signed message can deploy the NFT, preventing signature reuse attacks.

## Proof of Concept

(Empty)
