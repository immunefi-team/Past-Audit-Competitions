# #35758 \[SC-Critical] Loss of yield to the protocol due to incorrect interest rate applied

**Submitted on Oct 6th 2024 at 21:02:19 UTC by @SeveritySquad for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35758
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Theft of unclaimed yield
  * Permanent freezing of unclaimed yield
  * Loss of yield

## Description

## Brief/Intro

When a liquidator wants to liquidate a position he calls the \`absorb()\` method. In that method, the position of a given user is checked if it is liquidatable. This check is done however erroneously with Supply Rate rather than Borrow Rate. This prevents the liquidation to happen until the collateral value drops in value more, hence when it can finally be liquidated, the yield that the protocol earns from buying the collateral is less than it should.

## Vulnerability Details

The problem lies in the \`is\_liquidatable\_internal()\` function, in how the present value is calculated here: \`\`\` let present: u256 = present\_value(principal.wrapping\_neg()).try\_into().unwrap(); \`\`\` The supplied \`principal\` value to the \`present\_value()\` is turned from negative to positive. If the value is positive then inside \`present\_value\` the Supply Rate is applied instead of Borrow Rate: \`\`\` fn present\_value(principal: I256) -> I256 { let market\_basic = storage.market\_basic.read(); if principal >= I256::zero() { let present\_value = present\_value\_supply( market\_basic .base\_supply\_index, principal .try\_into() .unwrap(), ); I256::try\_from(present\_value).unwrap() } else { let present\_value = present\_value\_borrow( market\_basic .base\_borrow\_index, principal .wrapping\_neg() .try\_into() .unwrap(), ); I256::neg\_try\_from(present\_value).unwrap() } } \`\`\` And Supply Rate is always smaller than Borrow Rate. This means that the liquidatee \`present\` value is lower when compared against the collateral value, hence the collateral must drop in value even more to reach the liquidation threshold.

## Impact Details

Due to lower rate (Supply Rate instead of Borrow Rate) applied to calculate the \`present\` value of a position the liquidation can happen only when the collateral drops more in price. The result is that the amount that the protocol receives from liquidation is lower than it should according to the Borrow Rate. The difference is lost to the protocol and the lenders in terms of yield that is not obtained, hence we chose the impact to be High according the Impacts in Scope.

## Solution Proposal

The Present value should be calculated as it is done in \`is\_borrow\_collateralized()\`. The following line: \`\`\` let present: u256 = present\_value(principal.wrapping\_neg()).try\_into().unwrap(); \`\`\` should be changed to: \`\`\` let present = present\_value(principal); \`\`\` and the following line: \`\`\` let borrow\_amount = present \* base\_token\_price / base\_token\_price\_scale; \`\`\` to: \`\`\` let borrow\_amount = u256::try\_from(present.wrapping\_neg()).unwrap() \* base\_token\_price / base\_token\_price\_scale; \`\`\`

## References

Problematic line: <https://github.com/Swaylend/swaylend-monorepo/blob/d7fec5cd27bafa4b0d04a0690e71a2751fb66979/contracts/market/src/main.sw#L1379>

## Proof of Concept

## Proof of Concept

The PoC: \`\`\` #\[tokio::test] async fn poc\_not\_liquidatable() { let TestData { wallets, alice, alice\_account, bob, bob\_account, chad, market, assets, usdc, eth, oracle, price\_feed\_ids, publish\_time, prices, usdc\_contract, .. } = setup().await;

```
let price_data_update &#x3D; PriceDataUpdate {
    update_fee: 0,
    price_feed_ids,
    publish_times: vec![publish_time; assets.len()],
    update_data: oracle.create_update_data(&amp;prices).await.unwrap(),
};

// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #0 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Alice 🧛
// 🤙 Call: supply_base
// 💰 Amount: 3000.00 USDC
let alice_supply_amount &#x3D; parse_units(3000 * AMOUNT_COEFFICIENT, usdc.decimals);
let alice_mint_amount &#x3D; parse_units(4000 * AMOUNT_COEFFICIENT, usdc.decimals);
let alice_supply_log_amount &#x3D; format!(&quot;{} USDC&quot;, alice_supply_amount as f64 / SCALE_6);
print_case_title(0, &quot;Alice&quot;, &quot;supply_base&quot;, alice_supply_log_amount.as_str());
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
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #1 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Bob 🧛
// 🤙 Call: supply_collateral
// 💰 Amount: 1.00 ETH
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

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;eth)
    .await
    .unwrap();

market.debug_increment_timestamp().await.unwrap();
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #2 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Bob 🧛
// 🤙 Call: withdraw_base
// 💰 Amount: &lt;MAX HE CAN BORROW&gt;
let max_borrow_amount &#x3D; market
    .available_to_borrow(&amp;[&amp;oracle.instance], bob_account)
    .await
    .unwrap();
let log_amount &#x3D; format!(&quot;{} USDC&quot;, max_borrow_amount as f64 / SCALE_6);
print_case_title(2, &quot;Bob&quot;, &quot;withdraw_base&quot;, &amp;log_amount.as_str());
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
market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;eth)
    .await
    .unwrap();

// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #3 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Admin 🗿
// 🤙 Drop of ETH price
// 💰 Amount: -50%
print_case_title(3, &quot;Admin&quot;, &quot;Drop of ETH price&quot;, &quot;-50%&quot;);
let res &#x3D; oracle.price(eth.price_feed_id).await.unwrap().value;
let new_price &#x3D; (res.price as f64 * 0.99) as u64;
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
    &quot;🔻 ETH price drops: ${}  -&gt; ${}&quot;,
    res.price as f64 / 10_u64.pow(eth.price_feed_decimals) as f64,
    new_price as f64 / 10_u64.pow(eth.price_feed_decimals) as f64
);
let res &#x3D; oracle.price(eth.price_feed_id).await.unwrap().value;
assert!(new_price &#x3D;&#x3D; res.price);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;eth)
    .await
    .unwrap();
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #4 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Chad 🧛
// 🤙 Call: absorb
// 🔥 Target: Bob
print_case_title(4, &quot;Chad&quot;, &quot;absorb&quot;, &quot;Bob&quot;);

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

// Check if absorb was ok
let (_, borrow) &#x3D; market.get_user_supply_borrow(bob_account).await.unwrap();
assert!(borrow &#x3D;&#x3D; 0);

let amount &#x3D; market
    .get_user_collateral(bob_account, eth.asset_id)
    .await
    .unwrap()
    .value;
assert!(amount &#x3D;&#x3D; 0);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;eth)
    .await
    .unwrap();

// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #5 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Alice 🧛
// 🤙 Call: buy_collateral
// 💰 Amount: &lt;MAX HE CAN BUY&gt;
let reserves &#x3D; market
    .with_account(&amp;alice)
    .await
    .unwrap()
    .get_collateral_reserves(eth.asset_id)
    .await
    .unwrap()
    .value;
assert!(!is_i256_negative(&amp;reserves));

let amount &#x3D; market
    .collateral_value_to_sell(
        &amp;[&amp;oracle.instance],
        eth.asset_id,
        convert_i256_to_u64(&amp;reserves),
    )
    .await
    .unwrap()
    .value;

let log_amount &#x3D; format!(&quot;{} USDC&quot;, amount as f64 / SCALE_6);
print_case_title(5, &quot;Alice&quot;, &quot;buy_collateral&quot;, log_amount.as_str());

// Prepare calls for multi_call_handler
let tx_policies &#x3D; TxPolicies::default().with_script_gas_limit(1_000_000);

// Params for update_price_feeds_if_necessary
let call_params_update_price &#x3D;
    CallParameters::default().with_amount(price_data_update.update_fee);

// Update price feeds if necessary
let update_balance_call &#x3D; market
    .instance
    .methods()
    .update_price_feeds_if_necessary(price_data_update.clone())
    .with_contracts(&amp;[&amp;oracle.instance])
    .with_tx_policies(tx_policies)
    .call_params(call_params_update_price)
    .unwrap();

// Params for buy_collateral
let call_params_base_asset &#x3D; CallParameters::default()
    .with_amount(amount as u64)
    .with_asset_id(usdc.asset_id);

// Buy collateral with base asset
usdc_contract
    .mint(alice_account, amount.try_into().unwrap())
    .await
    .unwrap();

let buy_collateral_call &#x3D; market
    .instance
    .methods()
    .buy_collateral(eth.asset_id, 1u64.into(), alice_account)
    .with_contracts(&amp;[&amp;oracle.instance])
    .with_tx_policies(tx_policies)
    .call_params(call_params_base_asset)
    .unwrap();

let mutli_call_handler &#x3D; CallHandler::new_multi_call(alice.clone())
    .add_call(update_balance_call)
    .add_call(buy_collateral_call)
    .with_variable_output_policy(VariableOutputPolicy::Exactly(2));

// Sumbit tx
let submitted_tx &#x3D; mutli_call_handler.submit().await.unwrap();

// Wait for response
let _: CallResponse&lt;((), ())&gt; &#x3D; submitted_tx.response().await.unwrap();
let alice_balance &#x3D; alice.get_asset_balance(&amp;eth.asset_id).await.unwrap();
assert!(alice_balance &#x3D;&#x3D; 10_999_999_997 * AMOUNT_COEFFICIENT);

// check reserves
let reserves &#x3D; market
    .with_account(&amp;alice)
    .await
    .unwrap()
    .get_collateral_reserves(eth.asset_id)
    .await
    .unwrap()
    .value;
let normalized_reserves: u64 &#x3D; convert_i256_to_i128(&amp;reserves).try_into().unwrap();
assert!(normalized_reserves &#x3D;&#x3D; 0);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;eth)
    .await
    .unwrap();
```

} \`\`\` We can see that the test fails at: \`\`\` failures: local\_tests::scenarios::severity\_squad\_pocs::poc\_not\_liquidatable \`\`\` because the position is not liquidatable.
