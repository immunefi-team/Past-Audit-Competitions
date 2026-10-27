# 57425 sc low referral percentage updates are ignored due to append only storage in nftfactory

**Submitted on Oct 26th 2025 at 04:34:13 UTC by @Rhaydden for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57425
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Issue description

The `_set_referral_percentages` function appends new values to `used_to_percentage` instead of overwriting the existing 5 entries. `getReferralRate` indexes the vector using the referral usage count (`used`) which is capped at 4 in `_set_referral_user`. As a result, it always reads indices 0..4 from the original set and never the newly appended values at indices 5..9, etc.

This renders owner updates to referral percentages ineffective and grows storage unnecessarily.

Relevant code excerpt:

```cairo
fn _set_referral_percentages(ref self: ContractState, percentages: Span<u16>) {
    assert(percentages.len() == 5, super::Errors::WRONG_PERCENTAGES_LEN);
    for i in 0..percentages.len() {
        self.used_to_percentage.append().write(*percentages.at(i));
    };
}

fn getReferralRate(
    self: @ContractState,
    referral_user: ContractAddress,
    referral_code: felt252,
    amount: u256,
) -> u256 {
    let used = self.used_code.entry(referral_user).entry(referral_code).read();
    assert(used > 0, super::Errors::REFFERAL_CODE_NOT_USED_BY_USER);
    let rate = amount * self.used_to_percentage.at(used.into()).read().into() / SKALING_FACTOR;
    rate
}

fn _set_referral_user(ref self: ContractState, referral_code: felt252, referral_user: ContractAddress) {
    // ...
    let used_code = self.used_code.entry(referral_user).entry(referral_code).read();
    if used_code < 4 {
        self.used_code.entry(referral_user).entry(referral_code).write(used_code + 1);
    }
    // ...
}
```

How it breaks:

* Initial Setup: Owner calls initialize or setReferralPercentages with percentages \[0, 5000, 3000, 1500, 500]
  * Vec indices: 0->0, 1->5000, 2->3000, 3->1500, 4->500
* Attempted Update: Owner tries to update percentages to \[0, 6000, 4000, 2000, 1000]
  * The function appends instead of replacing
  * Vec now contains: 0->0, 1->5000, 2->3000, 3->1500, 4->500, 5->0, 6->6000, 7->4000, 8->2000, 9->1000
* Referral rate calculation: `getReferralRate` uses `used` (0..4) and therefore always accesses indices 0..4 (the old values). New values at 5..9 are never read.

### Impact

Low — Contract fails to deliver promised returns, but doesn't lose value.

* Owner cannot update referral percentages effectively; updates have no effect.
* Old percentages persist forever causing users to receive incorrect rewards.
* The vector grows indefinitely; each update attempt appends 5 more entries and wastes storage.

## Recommended mitigation steps

Overwrite indices 0..4 of `used_to_percentage` instead of appending.

Suggested approach:

* If the vector length < 5, append until size == 5.
* Else, write the new five values to indices 0..4 directly.

## Proof of Concept

Attach this PoC to `test_nftfactory.cairo`:

```ts
#[test]
fn test_referral_percentages_update_is_ignored_in_getReferralRate() {
    let account = deploy_account_mock();
    let contract = deploy_initialize(account);
    let erc20mock = deploy_erc20_mock();

    let nft_factory = INFTFactoryDispatcher { contract_address: contract };

    let fraction = 0;
    let produce_hash = ProduceHash {
        name_hash: constants::NAME().hash(),
        symbol_hash: constants::SYMBOL().hash(),
        contract_uri: constants::CONTRACT_URI().hash(),
        royalty_fraction: fraction,
    };

    let mut produce_hash_2 = produce_hash.clone();
    produce_hash_2.name_hash = constants::NAME_2().hash();

    start_cheat_caller_address_global(account);
    let signature = sign_message(produce_hash.get_message_hash(contract));
    let signature_2 = sign_message(produce_hash_2.get_message_hash(contract));
    stop_cheat_caller_address_global();

    start_cheat_caller_address(contract, constants::REFERRAL());
    let code = nft_factory.createReferralCode();

    let mut instance_info = InstanceInfo {
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
        referral_code: code,
        signature,
    };

    let mut instance_info_2 = instance_info.clone();
    instance_info_2.name = constants::NAME_2();
    instance_info_2.signature = signature_2;

    start_cheat_caller_address(contract, constants::OWNER());
    let new_percentages = array![0, 6000, 4000, 2000, 1000].span();
    nft_factory.setReferralPercentages(new_percentages);

    assert_eq!(nft_factory.usedToPercentage(6), 6000);
    assert_eq!(nft_factory.usedToPercentage(7), 4000);

    start_cheat_caller_address(contract, constants::CREATOR());

    let amount = 10000;

    nft_factory.produce(instance_info);
    assert_eq!(nft_factory.getReferralRate(constants::CREATOR(), code, amount), 5000);

    nft_factory.produce(instance_info_2);
    assert_eq!(nft_factory.getReferralRate(constants::CREATOR(), code, amount), 3000);
}
```

I made these changes to `scarb.toml`:

```diff
[package]
name = "nft"
version = "0.1.0"
edition = "2024_07"

[lib]
path = "src/lib.cairo"

# See more keys and their definitions at https://docs.swmansion.com/scarb/docs/reference/manifest.html

[dependencies]
starknet = "2.9.1"
- openzeppelin = { git = "https://github.com/OpenZeppelin/cairo-contracts.git", tag = "v0.20.0-rc.0" }
+ openzeppelin = { git = "https://github.com/OpenZeppelin/cairo-contracts.git", tag = "v0.20.0" }

[dev-dependencies]
- snforge_std = { git = "https://github.com/foundry-rs/starknet-foundry.git", tag = "v0.34.0" }
+ snforge_std = { git = "https://github.com/foundry-rs/starknet-foundry.git", tag = "v0.51.1" }
assert_macros = "2.9.1"

[[target.starknet-contract]]
sierra = true

[scripts]
test = "snforge test"
```

Run test with `scarb run test`

Logs:

```
[PASS] nft::tests::test_nftfactory::test_referral_percentages_update_is_ignored_in_getReferralRate
```

***

If you want, I can provide a minimal code patch that replaces the append behavior to correctly overwrite indices 0..4 (including handling the <5 length case).
