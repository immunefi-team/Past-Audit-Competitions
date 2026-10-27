# 57255 sc low allowed minting of nfts after collection expiry date

Submitted on Oct 24th 2025 at 18:34:43 UTC by @chinepun for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)

* Report ID: #57255
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo>
* Impacts:
  * Unauthorized minting of NFTs

## Description

### Brief / Intro

The smart contract allows anyone to mint tokens after the NFT collection's expiry timestamp has passed. In other words, users can still mint tokens after the configured expiration timestamp.

### Vulnerability Details

The `NftParameters` structure contains a field `collection_expires` which tracks the expiration timestamp for the NFT collection. See:

* NftParameters: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/interface.cairo#L3-L17>
* `collection_expires` field: <https://github.com/immunefi-team/audit-comp-belong/blob/0cbcde6fd80dbc55a9e3403c8e5a74827dea19e2/src/nft/interface.cairo#L14>

However, during minting (via `_mint_static_price_batch` or `_mint_dynamic_price_batch`), the `_base_mint` function does not check whether the collection has expired before minting:

* Minting flow: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo#L386-L401>
* `_mint_static_price_batch`: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo#L328-L384>
* `_mint_dynamic_price_batch`: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo#L279-L326>
* `_base_mint`: <https://github.com/immunefi-team/audit-comp-belong/blob/0cbcde6fd80dbc55a9e3403c8e5a74827dea19e2/src/nft/nft.cairo#L386C12-L386C22>

Excerpt from `_base_mint` (no expiration check):

```cairo
assert(
    token_id + 1 <= self.nft_parameters.max_total_supply.read(),
    super::Errors::TOTAL_SUPPLY_LIMIT,
);// it checks here that nft total supply is not exceeded after mint
// this updates metadata of tokens after mint
self.nft_node.total_supply.write(self.nft_node.total_supply.read() + 1);
self.nft_node.metadata_uri.write(token_id, token_uri);

self.erc721.safe_mint(recipient, token_id, array![].span());// this does the actual minting
```

As shown, `_base_mint` validates total supply but does not verify `collection_expires` before performing the mint.

### Impact Details

Because expiration is not enforced, NFTs can be minted after the intended expiration date. This may allow continuous minting until `max_total_supply` is reached, potentially crashing the collection's market price or allowing unauthorized minting beyond the intended sale period.

### References

* Relevant mint logic: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo#L386-L401>

## Proof of Concept

The repository already contains tests covering mint behavior: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/tests/test\\_nft.cairo#L508-L574>

You can reproduce the issue by changing the collection expiry used in tests to an expired timestamp (0) and running the test. The test still passes, demonstrating minting occurs despite an expired collection.

{% stepper %}
{% step %}

### Step: Modify test to set collection\_expires to 0

Edit `src/tests/test_nft.cairo` to set `collection_expires: 0`:

```diff
diff --git a/src/tests/test_nft.cairo b/src/tests/test_nft.cairo
index 4d3eea6..4e55105 100644
--- a/src/tests/test_nft.cairo
+++ b/src/tests/test_nft.cairo
@@ -124,8 +124,7 @@ fn deploy_factory_nft_receiver_erc20(
         max_total_supply: constants::MAX_TOTAL_SUPPLY(),
         mint_price: constants::MINT_PRICE(),
         whitelisted_mint_price: constants::WL_MINT_PRICE(),
-        //collection_expires: constants::EXPIRES(),
+        collection_expires: 0,
         referral_code: referral,
         signature,
     };
```

{% endstep %}

{% step %}

### Step: Run the specific test

Run the test:

```
scarb test test_mintDynamicPrice_referral
```

{% endstep %}

{% step %}

### Step: Observe passing test (minting despite expiry)

Expected test output shows the mint succeeded even though the collection expiry was set to 0 (expired):

```
Running 1 test(s) from src/
platfrom_balance_after + referral_balance_after should be 10 % from price: 10000000
platfrom_balance_after should be 5 % from price: 5000000
referral_balance_after should be 5 % from price: 5000000
creator_balance_after should be without 10 % from price: 90000000
[PASS] nft::tests::test_nft::test_mintDynamicPrice_referral (gas: ~4875)
Tests: 1 passed, 0 failed, 0 skipped, 0 ignored, 46 filtered out
```

This demonstrates NFTs were minted although `collection_expires` was set to an expired timestamp.
{% endstep %}
{% endstepper %}

## Recommendations (not prescriptive)

* Add a check in the minting flow (e.g., in `_base_mint` or the public mint wrapper functions) that asserts the current timestamp is less than or equal to `collection_expires` before proceeding to mint.
* Ensure unit tests include cases that verify minting reverts when `collection_expires` is in the past.

(Note: recommendations are general guidance based on the reported behavior; they do not add new information beyond the report.)
