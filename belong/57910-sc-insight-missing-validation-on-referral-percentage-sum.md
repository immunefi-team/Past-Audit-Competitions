# 57910 sc insight missing validation on referral percentage sum

**Submitted on Oct 29th 2025 at 12:37:53 UTC by @kenzo for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57910
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>

## Description

The `_set_referral_percentages` function in `nftfactory.cairo` only validates that the input array has exactly 5 elements. It does not check:

* Whether each percentage value is <= 10000 (100%)
* Whether the sum of all percentages is <= 10000 (100%)

This allows the factory owner to set invalid percentages like `[15000, 0, 0, 0, 0]` or `[5000, 4000, 3000, 2000, 1000]` (sums to 15000).

When these percentages are used in `getReferralRate`, the calculation `rate = amount * percentage / SKALING_FACTOR` can return a value larger than the input `amount`. In the `_pay` function, this causes `fees_to_platform = fees - referral_fees` to underflow and panic, or incorrectly allocate more funds than available.

## Impact

A compromised or malicious factory owner can set percentages and drain creator/platform funds by over-allocating referral fees.

## Recommendation

Add validation in `_set_referral_percentages` to check both individual values and the sum. Example implementation:

{% code title="\_set\_referral\_percentages (suggested validation)" %}

```cairo
fn _set_referral_percentages(ref self: ContractState, percentages: Span<u16>) {
    assert(percentages.len() == 5, super::Errors::WRONG_PERCENTAGES_LEN);

    let mut sum: u32 = 0;
    for i in 0..percentages.len() {
        let percentage = *percentages.at(i);
        assert(percentage <= SKALING_FACTOR.try_into().unwrap(), 'Percentage exceeds 100%');
        sum += percentage.into();
    }
    assert(sum <= SKALING_FACTOR.try_into().unwrap(), 'Percentages exceed 100%');

    // Clear existing values before setting new ones
    self.used_to_percentage.clear();
    for i in 0..percentages.len() {
        self.used_to_percentage.append().write(*percentages.at(i));
    };
}
```

{% endcode %}

## Proof of Concept

Add these tests to `test_nftfactory.cairo` and run with:

* snforge test test\_setReferralPercentages\_sum\_exceeds\_100\_percent
* snforge test test\_setReferralPercentages\_individual\_exceeds\_100\_percent

```rust
#[test]
fn test_setReferralPercentages_sum_exceeds_100_percent() {
    let contract = deploy();
    let nft_factory = INFTFactoryDispatcher { contract_address: contract };

    let percentages = array![5000, 4000, 3000, 2000, 1000].span();

    start_cheat_caller_address(contract, constants::OWNER());
    nft_factory.setReferralPercentages(percentages);
    stop_cheat_caller_address(contract);

    assert_eq!(nft_factory.usedToPercentage(0), 5000);
    assert_eq!(nft_factory.usedToPercentage(1), 4000);
    assert_eq!(nft_factory.usedToPercentage(2), 3000);
    assert_eq!(nft_factory.usedToPercentage(3), 2000);
    assert_eq!(nft_factory.usedToPercentage(4), 1000);
}

#[test]
fn test_setReferralPercentages_individual_exceeds_100_percent() {
    let contract = deploy();
    let nft_factory = INFTFactoryDispatcher { contract_address: contract };

    let percentages = array![15000, 0, 0, 0, 0].span();

    start_cheat_caller_address(contract, constants::OWNER());
    nft_factory.setReferralPercentages(percentages);
    stop_cheat_caller_address(contract);

    assert_eq!(nft_factory.usedToPercentage(0), 15000);
}
```
