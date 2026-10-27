# 57838 sc insight missing produce name sanitization allows breaking snip 12 standard compliance

**Submitted on Oct 29th 2025 at 06:35:55 UTC by @Another for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57838
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>

## Description

### Brief/Intro

The NFTFactory contract does not properly sanitize user-provided NFT names and symbols, allowing malicious or malformed input that can break SNIP-12 compliance. This can lead to interoperability issues with wallets, explorers, and other StarkNet infrastructure that expect properly formatted token identifiers.

### Vulnerability Details

The `_produce` function allows the user to provide the name and symbol for the NFT. The function only checks that names and symbols are non-empty but does not validate allowed characters. This enables creation of NFTs with names and symbols that are not SNIP-12 compliant.

Relevant snippet:

{% code title="nftfactory.cairo (excerpt)" %}

```
```

{% endcode %}

```cairo
        fn _produce(
            ref self: ContractState, instance_info: InstanceInfo,
        ) -> (ContractAddress, ContractAddress) {
            let info = instance_info.clone();

            assert(info.name.len().is_non_zero(), super::Errors::EMPTY_NAME);
            assert(info.symbol.len().is_non_zero(), super::Errors::EMPTY_SYMBOL);

            let metadata_name_hash: felt252 = info.name.hash();
            let metadata_symbol_hash: felt252 = info.symbol.hash();
            let contract_uri_hash: felt252 = info.contract_uri.hash();

            assert(
                self
                    .nft_info
                    .entry((metadata_name_hash, metadata_symbol_hash))
                    .nft_address
                    .read()
                    .is_zero(),
                super::Errors::NFT_EXISTS,
            );

            let message = ProduceHash {
                name_hash: metadata_name_hash,
                symbol_hash: metadata_symbol_hash,
                contract_uri: contract_uri_hash,
                royalty_fraction: info.royalty_fraction,
            };

            let hash = message.get_message_hash(get_contract_address());
            let is_valid_signature_felt = ISRC6Dispatcher {
                contract_address: self.factory_parameters.signer.read(),
            }
```

(unchanged links and file paths preserved)

According to SNIP-12 (StarkNet Improvement Proposal 12), token identifiers should follow additional constraints beyond non-empty strings, for example:

* No empty name (already enforced)
* Name can't be enclosed in parentheses
* Name can't contain the comma (,) character (since it is used as a delimiter in the enum type)
* ...and other character/formatting restrictions defined by SNIP-12

### Impact Details

A malicious user can create NFTs with names/symbols containing special or delimiter characters that break SNIP-12 assumptions. Consequences include:

* Interoperability issues with wallets, explorers, and other tooling expecting SNIP-12 compliant identifiers
* Display or parsing issues in frontends, potentially enabling UI/UX issues or spoofing/confusion

{% hint style="warning" %}
The contract currently allows creation of tokens with problematic characters (beyond just non-empty checks). This can be weaponized to cause downstream issues across StarkNet tooling that assumes SNIP-12 compliance.
{% endhint %}

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nftfactory/nftfactory.cairo#L251-L285>
* <https://github.com/starknet-io/SNIPs/blob/906b19074038005dc26d3574b6853a7c68d14e02/SNIPS/snip-12.md#type-identification>

## Proof of Concept

<details>

<summary>Test demonstrating creation of NFT with special characters (add to test_nftfactory.cairo)</summary>

```cairo
#[test]
fn test_produce_with_special_characters() {
    let account = deploy_account_mock();
    let contract = deploy_initialize(account);
    let erc20mock = deploy_erc20_mock();

    let nft_factory = INFTFactoryDispatcher { contract_address: contract };

    let problematic_name = ByteArray::from("Test\x***,NFT"); 
    let problematic_symbol = ByteArray::from("T\x01\x02\x03"); 

    let fraction = 0;
    let produce_hash = ProduceHash {
        name_hash: problematic_name.hash(),
        symbol_hash: problematic_symbol.hash(),
        contract_uri: constants::CONTRACT_URI().hash(),
        royalty_fraction: fraction,
    };

    start_cheat_caller_address_global(account);
    let signature = sign_message(produce_hash.get_message_hash(contract));
    stop_cheat_caller_address_global();

    let instance_info = InstanceInfo {
        name: problematic_name,
        symbol: problematic_symbol,
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

    start_cheat_caller_address(contract, constants::CREATOR());

    // This should work currently, demonstrating the vulnerability
    let (nft, receiver) = nft_factory.produce(instance_info);

    // Verify the NFT was created with problematic characters
    let nft_info = nft_factory.nftInfo(problematic_name, problematic_symbol);
    assert_eq!(nft_info.nft_address, nft);
} 
```

</details>
