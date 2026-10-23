# 57307 sc low cairo factory referral percentages never update

**Submitted on Oct 25th 2025 at 05:15:44 UTC by @s8olidity for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57307
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>

## Description

### Brief / Intro

NFTFactory.\_set\_referral\_percentages appends percentages to persistent storage without clearing prior values. Subsequent admin updates silently fail, leaving referral rewards frozen at the original schedule. Operators cannot adjust tiers, breaking platform economics.

{% hint style="warning" %}
This is a logical bug that causes admin updates to referral percentages to have no practical effect because new values are appended rather than replacing the previous values.
{% endhint %}

### Vulnerability Details

* Function: NFTFactory.\_set\_referral\_percentages (src/nftfactory/nftfactory.cairo:451-457)
* Behavior: Iterates over the provided array and calls `self.used_to_percentage.append().write(...)`.
* Root cause: `used_to_percentage` is a persistent Vec. Appending adds new entries rather than replacing existing ones.
* Reads: Code that reads referral percentages (src/nftfactory/nftfactory.cairo:230-240) indexes the vector by `timesUsed` and thus still reads the original indices (0..4). The newly appended values live at indices 5..9 and are never used.
* Result: Admin attempts to adjust referral tiers via `setReferralPercentages` appear successful but do not change the system behavior.

### Impact Details

* Severity classification: Medium logic bug impact (documented here as Low report severity per reporter).
* Effect: Referral pressure cannot be adjusted. This may lead to continued payouts at outdated/higher rates, undermining platform economics and potentially causing excessive fee leakage if percentages were intended to mitigate abuse.

## Proof of Concept

### Test showing append-only update behavior

```
  // src/tests/test_nftfactory.cairo
  #[test]
  fn test_referral_percentages_update_appends_only() {
      let contract = deploy_initialize(constants::SIGNER());
      let nft_factory = INFTFactoryDispatcher { contract_address: contract };

      assert_eq!(nft_factory.usedToPercentage(0), 0);
      assert_eq!(nft_factory.usedToPercentage(1), 5000);

      let new_percentages = array![100, 200, 300, 400, 500].span();

      start_cheat_caller_address(contract, constants::OWNER());
      nft_factory.setReferralPercentages(new_percentages);
      stop_cheat_caller_address(contract);

      assert_eq!(nft_factory.usedToPercentage(0), 0);     // original value remains
      assert_eq!(nft_factory.usedToPercentage(1), 5000);  // original value remains
      assert_eq!(nft_factory.usedToPercentage(5), 100);   // new value appended at index 5
  }
```

After invoking `setReferralPercentages`, indices 0 and 1 remain unchanged while new values appear at index 5+, proving the update mechanism is ineffective.

## References

<details>

<summary>Relevant source locations</summary>

* Append-only "update": src/nftfactory/nftfactory.cairo:451-457
* Reads assume indices 0..4: src/nftfactory/nftfactory.cairo:236-240

</details>

## Suggested Fix (high-level)

* Replace the append semantics in `_set_referral_percentages` with logic that either:
  * Overwrites existing elements in `used_to_percentage` up to the provided length, and truncates any excess old entries, or
  * Clears `used_to_percentage` before writing the new sequence, then appends the new values.

Do not modify any links or code beyond the intended fix described above.
