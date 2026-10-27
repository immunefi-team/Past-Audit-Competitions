# 57656 sc insight incorrect supply cap check uses token id instead of total supply in base mint&#x20;

* **Submitted on:** Oct 27th 2025 at 21:57:31 UTC by @iehnnkta
* **Report ID:** #57656
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo>
* **Impacts:** Permanent freezing of NFTs

## Description

### Brief / Intro

The `_base_mint` function enforces `max_total_supply` by comparing `token_id + 1` to the cap rather than the actual minted count. This couples the cap to the token ID range, causing legitimate mints to revert and making the configured cap unreachable when IDs aren’t 0-indexed and contiguous.

## Vulnerability Details

* Root cause: admission control ties the cap to `token_id` (`token_id + 1`) instead of the real supply (`total_supply + 1`).
* According to the docs, the `tokenId` can be anything. In many codebases token IDs may be derived from user addresses (or other non-contiguous values). If token IDs are non-sequential or large, the current check can permanently DoS the minting process.
* Even if token IDs are sequential in some deployments, this enforcement is inconsistent with how supply is normally tracked, causing operational mismatches (e.g., a first user mints a high token ID and prevents subsequent valid mints).

## Impact

{% hint style="danger" %}

* Denial-of-Service on minting: valid mints can be blocked even when supply is below the cap if IDs aren’t in the range 0..max\_total\_supply-1.
* Operational inconsistencies: sales/airdrops/logic keyed off `total_supply` may stall or misreport because enforcement uses `token_id` instead of the actual minted count.
  {% endhint %}

## References

* Vulnerable code location: <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nft/nft.cairo#L352-L355>

## Mitigation

Replace the `token_id`-based cap check with a `total_supply`-based check. Example patch:

{% code title="Suggested fix (Cairo snippet)" %}

```
```

{% endcode %}

\`\`\`cairo fn \_base\_mint( ref self: ContractState, token\_id: u256, recipient: ContractAddress, token\_uri: felt252, ) { assert( - token\_id + 1 <= self.nft\_parameters.max\_total\_supply.read(), + self.nft\_node.total\_supply.read() + 1 <= self.nft\_parameters.max\_total\_supply.read(), super::Errors::TOTAL\_SUPPLY\_LIMIT, );

```
self.nft_node.total_supply.write(self.nft_node.total_supply.read() + 1);
self.nft_node.metadata_uri.write(token_id, token_uri);

self.erc721.safe_mint(recipient, token_id, array![].span());
```

}

````
Ensure `total_supply` is the authoritative counter for minted NFTs and is incremented atomically when minting.

## Proof of Concept

<details>
<summary>Test case demonstrating the issue (expandable)</summary>

Include the below test case function in `test_nft.cairo`:

```rust
#[test]
#[should_panic(expected: 'Total supply limit reached')]
fn test_mintStaticPrice_token_id_above_cap_reverts() {
    let signer = deploy_account_mock();
    let (_, _nft, _, erc20mock) = deploy_factory_nft_receiver_erc20(signer, false, true);
    let nft = INFTDispatcher { contract_address: _nft };

    let receiver = signer;
    // here we took token_id as some large number greater than max total supply, but in actual it will be u256 conversion of receiver address
    let token_id: u256 = 100000; // well above typical MAX_TOTAL_SUPPLY
    let whitelisted: bool = false;
    let token_uri = constants::TOKEN_URI();

    let static_price_hash = StaticPriceHash { receiver, token_id, whitelisted, token_uri };
    start_cheat_caller_address_global(signer);
    let signature: Span<felt252> = sign_message(static_price_hash.get_message_hash(_nft)).into();
    stop_cheat_caller_address_global();

    let static_params = StaticPriceParameters { receiver, token_id, whitelisted, token_uri, signature };
    let mut static_params_array = array![static_params];

    start_cheat_caller_address(_nft, signer);
    // Should revert because implementation incorrectly checks token_id+1 <= max_total_supply,
    // rejecting high token IDs even when total_supply is still below the cap.
    nft.mintStaticPrice(static_params_array, erc20mock, constants::MINT_PRICE());
}
````

Observed test run:

```bash
$ snforge test test_mintStaticPrice_token_id_above_cap_reverts

   Compiling snforge_scarb_plugin v0.51.1 (git+https://github.com/foundry-rs/starknet-foundry.git?tag=v0.51.1#6f5a02e54c76c2e740c0756568448265e12a6f2d)
    Finished `release` profile [optimized] target(s) in 0.07s
   Compiling test(nft_unittest) nft v0.1.0 (/Users/sainikethan/Documents/Immunefi/checkin-contracts/Scarb.toml)

    Finished `dev` profile target(s) in 4 seconds

Collected 1 test(s) from nft package
Running 1 test(s) from src/
[PASS] nft::tests::test_nft::test_mintStaticPrice_token_id_above_cap_reverts (l1_gas: ~0, l1_data_gas: ~4608, l2_gas: ~5078152)
Tests: 1 passed, 0 failed, 0 ignored, 45 filtered out
```
