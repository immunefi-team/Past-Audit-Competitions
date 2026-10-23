# #36117 \[SC-High] Permanent freezing of tokens when user sends extra tokens as update fee

**Submitted on Oct 20th 2024 at 16:42:11 UTC by @savi0ur for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #36117
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw>
* **Impacts:**

## Description

## Bug Description

Anyone can update the price of an assets using \`update\_price\_feeds\_if\_necessary\` function.

<https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw#L1269-L1273> \`\`\`sway #\[payable, storage(read)] fn update\_price\_feeds\_if\_necessary(price\_data\_update: PriceDataUpdate) { reentrancy\_guard(); update\_price\_feeds\_if\_necessary\_internal(price\_data\_update) } \`\`\`

<https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw#L1441-L1469> \`\`\`sway #\[payable, storage(read)] fn update\_price\_feeds\_if\_necessary\_internal(price\_data\_update: PriceDataUpdate) { let contract\_id = storage.pyth\_contract\_id.read(); require( contract\_id != ContractId::zero(), Error::OracleContractIdNotSet, );

```
// check if the payment is sufficient
require(
    msg_amount() &gt;&#x3D; price_data_update
        .update_fee &amp;&amp; msg_asset_id() &#x3D;&#x3D; AssetId::base(),
    Error::InvalidPayment,
); //@audit-issue there is no transfer of remaining tokens back to caller when msg_amount() &gt; update_fee

let oracle &#x3D; abi(PythCore, contract_id.bits());
oracle
    .update_price_feeds_if_necessary {
        asset_id: AssetId::base().bits(),
        coins: price_data_update.update_fee,
    }(
        price_data_update
            .price_feed_ids,
        price_data_update
            .publish_times,
        price_data_update
            .update_data,
    );
```

} \`\`\`

As we can see, in \`update\_price\_feeds\_if\_necessary\_internal\` function, its accepting \`msg\_amount() >= update\_fee\`, if \`msg\_amount\` is strictly greater than \`update\_fee\`, their delta i.e., \`msg\_amount() - update\_fee\` will remain in the market contract instead of returning it back to caller i.e., \`msg\_sender()\`. Due to this, caller lose those extra tokens and it gets stuck permanently in the contract.

## Impact

Permanent freezing of tokens when user sends extra tokens as update fee while updating price using \`update\_price\_feeds\_if\_necessary\` function.

## Recommendation

Either return the remaining tokens back to the caller at the end of function OR make sure \`update\_price\_feeds\_if\_necessary\_internal\` function accepts \`msg\_amount()\` exactly equal to \`update\_fee\`.

## References

* <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw#L1269-L1273>
* <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw#L1441-L1469>

## Proof Of Concept

**Steps to Run:**

* Open terminal and run \`cd swaylend-monorepo\`
* Paste following rust code in \`contracts/market/tests/local\_tests/scenarios/price\_changes.rs\`
* Run test using \`cargo test --package market --test integration\_tests -- local\_tests::scenarios::price\_changes::excess\_fee\_not\_returned --exact --show-output\`

\`\`\`rust #\[tokio::test] async fn excess\_fee\_not\_returned() { let TestData { alice, market, assets, eth, oracle, price\_feed\_ids, publish\_time, prices, .. } = setup(None).await;

```
let price_data_update &#x3D; PriceDataUpdate {
    update_fee: 100,
    price_feed_ids,
    publish_times: vec![publish_time; assets.len()],
    update_data: oracle.create_update_data(&amp;prices).await.unwrap(),
};

let alice_balance_before &#x3D; alice.get_asset_balance(&amp;eth.asset_id).await.unwrap();
println!(&quot;Alice Balance Before: {}&quot;, alice_balance_before);

// Prepare calls for multi_call_handler
let tx_policies &#x3D;
    fuels::types::transaction::TxPolicies::default().with_script_gas_limit(1_000_000);

// Params for update_price_feeds_if_necessary
let extra_fee &#x3D; 20;
let call_params_update_price &#x3D; fuels::programs::calls::CallParameters::default()
    .with_amount(price_data_update.update_fee + extra_fee);

println!(
    &quot;Alice is sending &#x60;extra fee ({}) + update_fee ({}) &#x3D; {}&#x60; to update_price_feeds_if_necessary&quot;,
    extra_fee, price_data_update.update_fee, extra_fee + price_data_update.update_fee
);
// Update price feeds if necessary
let _update_balance_call &#x3D; market
    .instance
    .with_account(alice.clone())
    .methods()
    .update_price_feeds_if_necessary(price_data_update.clone())
    .with_contracts(&amp;[&amp;oracle.instance])
    .with_tx_policies(tx_policies)
    .call_params(call_params_update_price)
    .unwrap()
    .call()
    .await;

// println!(&quot;res: {:#?}&quot;, _update_balance_call);
let alice_balance_after &#x3D; alice.get_asset_balance(&amp;eth.asset_id).await.unwrap();
println!(&quot;Alice Balance After: {}&quot;, alice_balance_after);
println!(
    &quot;Difference in balance: {}&quot;,
    alice_balance_before - alice_balance_after
);
assert!(alice_balance_after &lt; alice_balance_before - price_data_update.update_fee - extra_fee);
println!(
    &quot;NOTE: Its taking complete &#x60;extra fee ({}) + update_fee ({}) &#x3D; {}&#x60;, even though update fee is just {}&quot;, 
    extra_fee, price_data_update.update_fee, 
    extra_fee + price_data_update.update_fee, 
    price_data_update.update_fee
);
```

} \`\`\`

**Console Output:**

\`\`\`shell ---- local\_tests::scenarios::price\_changes::excess\_fee\_not\_returned stdout ---- Price for UNI = 5 Price for BTC = 70000 Price for ETH = 3500 Price for USDC = 1 Alice Balance Before: 10000000000 Alice is sending \`extra fee (20) + update\_fee (100) = 120\` to update\_price\_feeds\_if\_necessary Alice Balance After: 9999999879 Difference in balance: 121 NOTE: Its using complete \`extra fee (20) + update\_fee (100) = 120\`, even though update fee is just 100

successes: local\_tests::scenarios::price\_changes::excess\_fee\_not\_returned \`\`\`

## Proof of Concept

## Proof of Concept
