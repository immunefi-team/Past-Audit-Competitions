# 57804 sc insight unbounded percentages cause underflow and dos in mint payment flow

**Submitted on Oct 29th 2025 at 00:31:01 UTC by @Rhaydden for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57804
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

### Issue description

The factory allows setting unbounded `platform_commission` and referral percentages. These values are later used to compute `fees` and splits during mint. If they exceed the denominators, it underflows at runtime and the transaction reverts, causing a DoS on minting.

There's no upper bound on `platform_commission` in factory setters as seen here: <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nftfactory/nftfactory.cairo#L442-L449>

```cairo
fn _set_factory_parameters(ref self: ContractState, factory_parameters: FactoryParameters) {
    assert(factory_parameters.signer.is_non_zero(), super::Errors::ZERO_ADDRESS);
    assert(factory_parameters.platform_address.is_non_zero(), super::Errors::ZERO_ADDRESS);
    assert(
        factory_parameters.platform_commission.is_non_zero(), super::Errors::ZERO_AMOUNT,
    );
    self.factory_parameters.write(factory_parameters);
}
```

Also there's no upper bound on referral percentage entries.

```cairo
pub const SKALING_FACTOR: u256 = 10000; // 100 %

fn _set_referral_percentages(ref self: ContractState, percentages: Span<u16>) {
    assert(percentages.len() == 5, super::Errors::WRONG_PERCENTAGES_LEN);

    for i in 0..percentages.len() {
        self.used_to_percentage.append().write(*percentages.at(i));
    };
}
```

These are the underflow points in the NFT payment flow: <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nft/nft.cairo#L417-L436>

```cairo
fn _check_price(self: @ContractState, price: u256, payment_token: ContractAddress) -> (u256, u256) {
    assert(payment_token == self.nft_parameters.payment_token.read(), super::Errors::EXPECTED_TOKEN_ERROR);

    let (platform_commission, _) = INFTFactoryDispatcher { contract_address: self.factory.read() }.platformParams();

    let fees = (price * platform_commission) / DefaultConfig::FEE_DENOMINATOR.into();
    let amount_to_creator = price - fees;               // underflow if fees > price
    return (fees, amount_to_creator);
}

fn _pay(ref self: ContractState, amount: u256, fees: u256, amount_to_creator: u256) {
    // ...
    let mut fees_to_platform = fees;
    let mut referral_fees = 0;

    if referral_code.is_non_zero() {
        referral_fees = factory.getReferralRate(creator, referral_code, fees);
        fees_to_platform = fees_to_platform - referral_fees;   // underflow if referral_fees > fees
    }
    // ...
}
```

* Setting `platform_commission` above `DefaultConfig::FEE_DENOMINATOR` will make `fees` > `price` and cause underflow on `price - fees` in `_check_price`.
* Setting a referral percentage entry above `SKALING_FACTOR` will make `referral_fees` > `fees` and cause underflow on `fees - referral_fees` in `_pay`.

## Impact

Deterministic reverts in mint functions due to underflow in `price - fees` or `fees - referral_fees`. This prevents users from minting NFTs.

{% hint style="warning" %}
High - Temporary freezing of NFTs for at least 24 hours
{% endhint %}

## Recommended mitigation steps

* In `_set_factory_parameters`, enforce:
  * `0 < platform_commission <= DefaultConfig::FEE_DENOMINATOR`
* In `_set_referral_percentages`, enforce for each entry:
  * `0 <= percentages[i] <= SKALING_FACTOR` (10000)

{% hint style="info" %}
Validate inputs at the point of setting factory parameters and referral percentages so runtime arithmetic cannot underflow during mint/payment flows.
{% endhint %}

## Proof of Concept

### Proof of concept description

Two starknet foundry tests demonstrate the underflow and expected revert.

Attach this PoC to `test_nft.cairo`:

```cairo
fn deploy_factory_nft_receiver_erc20_custom(
    signer: ContractAddress,
    is_referral: bool,
    transferrable: bool,
    platform_commission: u256,
    percentages: Span<u16>,
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
        platform_commission,
        max_array_size: 2,
    };

    start_cheat_caller_address(factory, constants::OWNER());
    nft_factory.initialize(*nft_class.class_hash, *receiver_class.class_hash, factory_parameters, percentages);

    start_cheat_caller_address(factory, constants::REFERRAL());
    let referral = if is_referral { nft_factory.createReferralCode() } else { '' };

    let royalty_fraction = if is_referral { constants::FRACTION() } else { 0 };

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
        collection_expires: constants::EXPIRES(),
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
#[should_panic]
fn test_underflow_platform_commission_in_check_price() {
    let signer = deploy_account_mock();
    let percentages = array![0, 5000, 3000, 1500, 500].span();
    let (_, _nft, _, erc20mock) = deploy_factory_nft_receiver_erc20_custom(
        signer, false, true, 20000, percentages,
    );

    let receiver = signer;
    let token_id: u256 = 0;
    let price: u256 = constants::MINT_PRICE();
    let token_uri = constants::TOKEN_URI();

    let dynamic_price_hash = DynamicPriceHash { receiver, token_id: token_id, price: price, token_uri };
    start_cheat_caller_address_global(signer);
    let signature: Span<felt252> = sign_message(dynamic_price_hash.get_message_hash(_nft)).into();
    stop_cheat_caller_address_global();

    let dynamic_params = DynamicPriceParameters { receiver, token_id, price, token_uri, signature };
    let mut dynamic_params_array = array![dynamic_params];

    start_cheat_caller_address(_nft, signer);
    INFTDispatcher { contract_address: _nft }.mintDynamicPrice(dynamic_params_array, erc20mock);
}

#[test]
#[should_panic]
fn test_underflow_referral_percentage_in_pay() {
    let signer = deploy_account_mock();
    let percentages = array![0, 15000, 15000, 15000, 15000].span();
    let (_, _nft, _, erc20mock) = deploy_factory_nft_receiver_erc20_custom(
        signer, true, true, 1000, percentages,
    );

    let receiver = signer;
    let token_id: u256 = 0;
    let price: u256 = constants::MINT_PRICE();
    let token_uri = constants::TOKEN_URI();

    let dynamic_price_hash = DynamicPriceHash { receiver, token_id: token_id, price: price, token_uri };
    start_cheat_caller_address_global(signer);
    let signature: Span<felt252> = sign_message(dynamic_price_hash.get_message_hash(_nft)).into();
    stop_cheat_caller_address_global();

    let dynamic_params = DynamicPriceParameters { receiver, token_id, price, token_uri, signature };
    let mut dynamic_params_array = array![dynamic_params];

    start_cheat_caller_address(_nft, signer);
    INFTDispatcher { contract_address: _nft }.mintDynamicPrice(dynamic_params_array, erc20mock);
}
```

Due to compatibility issues, paste this in the `scarb.toml` file:

```toml
[package]
name = "nft"
version = "0.1.0"
edition = "2024_07"

[lib]
path = "src/lib.cairo"

# See more keys and their definitions at https://docs.swmansion.com/scarb/docs/reference/manifest.html

[dependencies]
starknet = "2.10.1"
openzeppelin = { git = "https://github.com/OpenZeppelin/cairo-contracts.git", tag = "v0.20.0" }

[dev-dependencies]
snforge_std = { git = "https://github.com/foundry-rs/starknet-foundry.git", tag = "v0.50.0" }
assert_macros = "2.10.1"

[[target.starknet-contract]]
sierra = true

[scripts]
test = "snforge test"
```

Run test with:

```
snforge test underflow
```

<details>

<summary>Logs</summary>

```
  Collected 2 test(s) from nft package
  Running 2 test(s) from src/
  [PASS] nft::tests::test_nft::test_underflow_referral_percentage_in_pay (...)
  [PASS] nft::tests::test_nft::test_underflow_platform_commission_in_check_price (...)
  Tests: 2 passed, 0 failed, 0 ignored, 45 filtered out
```

</details>
