# 57454 sc low referral percentages schedule stuck on first configuration

Submitted on Oct 26th 2025 at 10:54:30 UTC by @OxPrince for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)

* Report ID: #57454
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* Impacts:
  * Protocol insolvency

## Description

### Brief/Intro

The factory exposes `setReferralPercentages`, but the implementation never overwrites the five referral tiers that were first stored during `initialize`. Each subsequent call just appends another five entries to `used_to_percentage`, while every lookup continues to read only the original slots. As a result, once an aggressive commission schedule is live there is no way to rotate back to a safer set of percentages, and referral partners keep receiving the inflated split indefinitely.

### Vulnerability Details

* `_set_referral_percentages` iterates over the supplied span and calls `append()` on every entry (src/nftfactory/nftfactory.cairo:451-456), so the storage vector keeps growing instead of being overwritten.
* Referral math indexes the vector by usage count: `usedToPercentage` and `getReferralRate` read `self.used_to_percentage.at(timesUsed)` (src/nftfactory/nftfactory.cairo:200-245).
* `_set_referral_user` caps `used_code` at 4 (src/nftfactory/nftfactory.cairo:431-433), meaning only indices 0–4 are ever consulted. Those slots are permanently populated with the very first schedule because nothing truncates or rewrites them.

### Impact Details

If the owner ever raises the referral percentages—whether deliberately for a campaign or because a privileged key is misused—the protocol cannot later lower them. Referral operators can continue routing sales through their codes and siphon the same heightened commission on every mint. Because the payout comes directly from revenue that should fund the platform and creators, sustained operation under inflated rates can quickly drain treasury inflows and threaten solvency.

{% stepper %}
{% step %}

### Reproduction steps

* Call `initialize` with a schedule such as `[9000, 9000, 9000, 9000, 9000]`.
  {% endstep %}

{% step %}

* Later call `setReferralPercentages` with safer values, e.g. `[500, 400, 300, 200, 100]`.
  {% endstep %}

{% step %}

* Query `usedToPercentage(1)` or run a sale through `getReferralRate`; the contract still returns `9000`, confirming the old payout remains active.
  {% endstep %}
  {% endstepper %}

## References

Add any relevant links to documentation or code

## Proof of Concept

<details>

<summary>Rust test PoC — demonstrates that setReferralPercentages appends instead of overwriting</summary>

```rust
#[test]
fn test_setReferralPercentages_update_should_override() {
    let contract = deploy();

    let nft_factory = INFTFactoryDispatcher { contract_address: contract };

    start_cheat_caller_address(contract, constants::OWNER());

    let initial = array![9000, 9000, 9000, 9000, 9000].span();
    nft_factory.setReferralPercentages(initial);

    assert_eq!(nft_factory.usedToPercentage(0), *initial[0]);
    assert_eq!(nft_factory.usedToPercentage(4), *initial[4]);

    let updated = array![500, 400, 300, 200, 100].span();
    nft_factory.setReferralPercentages(updated);

    // Expected behavior: latest schedule visible at indices 0-4.
    assert_eq!(nft_factory.usedToPercentage(0), *updated[0]);
    assert_eq!(nft_factory.usedToPercentage(4), *updated[4]);
}
```

</details>
