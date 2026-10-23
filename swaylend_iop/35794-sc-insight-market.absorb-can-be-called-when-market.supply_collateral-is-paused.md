# #35794 \[SC-Insight] \`Market.absorb\` can be called when \`Market.supply\_collateral\` is paused

**Submitted on Oct 8th 2024 at 14:25:27 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35794
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In current implementation, \`storage.pause\_config.supply\_paused\` is used to enable/disable \`Market.supply\_collateral\` and \`Market.supply\_base\` function, and \`storage.pause\_config.absorb\_paused\` is used to enable/disable \`Market.absorb\` function.

And function \`Market.absorb\` only checks if \`storage.pause\_config.absorb\_paused\` in [main.sw#L598](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L598).

**The issue is that in a case that \`Market.supply\_collateral\` is paused, but \`Market.absorb\` is not paused, if the collateral token's price goes down, the borrower can't call \`Market.supply\_collateral\` to add collateral token, nor call \`Market.supply\_base\` to repay his debt.**

At the same time, other users can call \`Market.absorb\` to absorb his collateral assets, which causes the borrower loses assets

## Vulnerability Details

As the following code shows: [Market.supply\_collateral](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L260-L297) uses \`storage.pause\_config.supply\_paused\` to enable/disable the function in [main.sw#L262](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L261) \`\`\`Rust 258 // ## 3.1 Supply Collateral 259 #\[payable, storage(write)] 260 fn supply\_collateral() { 261 // Only allow supplying collateral if paused flag is not set 262 require(!storage.pause\_config.supply\_paused.read(), Error::Paused); <<<--- pause\_config.supply\_paused is used ... 297 } \`\`\`

[Market.supply\_base](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L398-L445) also uses \`storage.pause\_config.supply\_paused.read\` to enable/disable the function in [main.sw#L400](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L400) \`\`\`Rust 398 fn supply\_base() { 399 // Only allow supplying if paused flag is not set 400 require(!storage.pause\_config.supply\_paused.read(), Error::Paused); <<<--- pause\_config.supply\_paused is used

```
 ...
```

445 } \`\`\`

But [Market.absort](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L594-L612) only checks \`storage.pause\_config.absorb\_paused\` in [main.sw#L598](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L598) \`\`\`Rust 594 fn absorb(accounts: Vec\<Identity>, price\_data\_update: PriceDataUpdate) { 595 reentrancy\_guard(); 596 597 // Check that the pause flag is not set 598 require(!storage.pause\_config.absorb\_paused.read(), Error::Paused); <<<--- only pause\_config.absorb\_paused is checked ... 612 } \`\`\`

## Impact Details

Borrower might lose assets becase he can't recovery from his bad debt

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Please put the following code in \`swaylend-monorepo/contracts/market/tests/local\_tests/scenarios/liquidation.rs\` and run \`\`\`bash cargo test --release local\_tests::scenarios::liquidation::absorb\_and\_liquidate\_after\_pause\_supply -- --nocapture

... running 1 test Price for USDC = 1 Price for ETH = 3500 Price for UNI = 5 Price for BTC = 70000 💸 Alice + 3000 USDC supply\_collateral is paused by the admin ETH price drops: $3500 -> $1750 Bob fails to repay his debt test local\_tests::scenarios::liquidation::absorb\_and\_liquidate\_after\_pause\_supply ... ok \`\`\`

As the result shows, the borrower(Bob) can't call \`Market.supply\_base\` to repay his debt, but Chad can call \`Market.absorb\` to liquidate Bob's debt

\`\`\`Rust #\[tokio::test] async fn absorb\_and\_liquidate\_after\_pause\_supply() { let TestData { admin, wallets, alice, alice\_account, bob, bob\_account, chad, market, assets, usdc, eth, oracle, price\_feed\_ids, publish\_time, prices, usdc\_contract, .. } = setup().await;

```
let price_data_update &#x3D; PriceDataUpdate {
    update_fee: 0,
    price_feed_ids,
    publish_times: vec![publish_time; assets.len()],
    update_data: oracle.create_update_data(&amp;prices).await.unwrap(),
};

let alice_supply_amount &#x3D; parse_units(3000 * AMOUNT_COEFFICIENT, usdc.decimals);
let alice_mint_amount &#x3D; parse_units(4000 * AMOUNT_COEFFICIENT, usdc.decimals);
let alice_supply_log_amount &#x3D; format!(&quot;{} USDC&quot;, alice_supply_amount as f64 / SCALE_6);

println!(&quot;💸 Alice + {alice_supply_log_amount}&quot;);
usdc_contract
    .mint(alice_account, alice_mint_amount)
    .await
    .unwrap();
let balance &#x3D; alice.get_asset_balance(&amp;usdc.asset_id).await.unwrap();
assert!(balance &#x3D;&#x3D; alice_mint_amount);

let alice_supply_res &#x3D; market
    .with_account(&amp;alice)
    .await
    .unwrap()
    .supply_base(usdc.asset_id, alice_supply_amount)
    .await;
assert!(alice_supply_res.is_ok());

market.debug_increment_timestamp().await.unwrap();

let bob_supply_amount &#x3D; parse_units(1 * AMOUNT_COEFFICIENT, eth.decimals);
let bob_supply_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .supply_collateral(eth.asset_id, bob_supply_amount)
    .await;
assert!(bob_supply_res.is_ok());

let bob_user_collateral &#x3D; market
    .get_user_collateral(bob_account, eth.asset_id)
    .await
    .unwrap()
    .value;
assert!(bob_user_collateral &#x3D;&#x3D; bob_supply_amount);

market.debug_increment_timestamp().await.unwrap();

let max_borrow_amount &#x3D; market
    .available_to_borrow(&amp;[&amp;oracle.instance], bob_account)
    .await
    .unwrap();
let log_amount &#x3D; format!(&quot;{} USDC&quot;, max_borrow_amount as f64 / SCALE_6);

let bob_borrow_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .withdraw_base(
        &amp;[&amp;oracle.instance],
        max_borrow_amount.try_into().unwrap(),
        &amp;price_data_update,
    )
    .await;
assert!(bob_borrow_res.is_ok());

let balance &#x3D; bob.get_asset_balance(&amp;usdc.asset_id).await.unwrap();
assert!(balance &#x3D;&#x3D; max_borrow_amount as u64);


println!(&quot;supply_collateral is paused by the admin&quot;);
let pause_config &#x3D; PauseConfiguration {
    supply_paused: true,
    withdraw_paused: true,
    absorb_paused: false,
    buy_paused: false,
};

market
    .with_account(&amp;admin)
    .await
    .unwrap()
    .pause(&amp;pause_config)
    .await
    .unwrap();


let res &#x3D; oracle.price(eth.price_feed_id).await.unwrap().value;
let new_price &#x3D; (res.price as f64 * 0.5) as u64;
let prices &#x3D; Vec::from([(
    eth.price_feed_id,
    (
        new_price,
        eth.price_feed_decimals,
        res.publish_time,
        res.confidence,
    ),
)]);

oracle.update_prices(&amp;prices).await.unwrap();

let price_data_update &#x3D; PriceDataUpdate {
    update_fee: 0,
    price_feed_ids: vec![eth.price_feed_id],
    publish_times: vec![tai64::Tai64::from_unix(Utc::now().timestamp().try_into().unwrap()).0],
    update_data: oracle.create_update_data(&amp;prices).await.unwrap(),
};

println!(
    &quot;ETH price drops: ${}  -&gt; ${}&quot;,
    res.price as f64 / 10_u64.pow(eth.price_feed_decimals) as f64,
    new_price as f64 / 10_u64.pow(eth.price_feed_decimals) as f64
);
let res &#x3D; oracle.price(eth.price_feed_id).await.unwrap().value;
assert!(new_price &#x3D;&#x3D; res.price);


println!(&quot;Bob fails to repay his debt&quot;);
let bob_balance &#x3D; bob.get_asset_balance(&amp;usdc.asset_id).await.unwrap();
let bob_supply_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .supply_base(usdc.asset_id, bob_balance)
    .await;

//println!(&quot;{:?}&quot;, bob_supply_res);
assert!(bob_supply_res.is_err());


assert!(
    market
        .is_liquidatable(&amp;[&amp;oracle.instance], bob_account)
        .await
        .unwrap()
        .value
);

let chad_absorb_bob_res &#x3D; market
    .with_account(&amp;chad)
    .await
    .unwrap()
    .absorb(&amp;[&amp;oracle.instance], vec![bob_account], &amp;price_data_update)
    .await;
assert!(chad_absorb_bob_res.is_ok());

let (_, borrow) &#x3D; market.get_user_supply_borrow(bob_account).await.unwrap();
assert!(borrow &#x3D;&#x3D; 0);

let amount &#x3D; market
    .get_user_collateral(bob_account, eth.asset_id)
    .await
    .unwrap()
    .value;
assert!(amount &#x3D;&#x3D; 0);
```

} \`\`\`
