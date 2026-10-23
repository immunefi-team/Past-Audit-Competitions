# #35908 \[SC-Low] If the collateral token''s decimal is <= the base token decimal in a market, \`collat

**Submitted on Oct 12th 2024 at 17:50:20 UTC by @SeveritySquad for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35908
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw>
* **Impacts:**
  * Temporary (1 hr) freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

On the line to get the scale which is later used to calculate the value of assets, in \` collateral\_value\_to\_sell()\` & \`available\_to\_borrow()\`, the operation to subtract the base decimal from the collateral decimal will cause the entire function to revert if the base decimal is greater than the collateral decimal, this will likely occur in markets where the some of the collaterals have lesser decimals than the base token of the market.

## Vulnerability Details

using \` collateral\_value\_to\_sell()\` as an instance:

* <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw#L882C3-L888C11> \`\`\`rust let scale = u256::from(10\_u64).pow( collateral\_configuration .decimals - storage .market\_configuration .read() .base\_token\_decimals, ); \`\`\` Supposing the base decimal is greater than the collateral, this will lead to a steady revert for such collateral (In sway overflow and underflow will lead to a revert for safety).

using \`available\_to\_borrow()\` as an instance :

* <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw#L707C1-L712C24>

\`\`\`rust let scale = u256::from(10\_u64).pow( collateral\_configuration.decimals + price\_exponent - market\_configuration.base\_token\_decimals, );

```
        borrow_limit +&#x3D; amount * price / scale; // decimals: base_token_decimals
        index +&#x3D; 1;
```

\`\`\`

This will lead to a wrong \`borrow\_limit\` being calculated. eg.

* if the base decimal is 8.
* and the collateral decimal is 6.
* due to the \`price\_exponet\` the subtraction might not revert here but
* the scale generated will be far large than supposed causing the \`borrow\_limit\` to be lesser than supposed, causing a wrong amount to be used.

## Impact Details

The functions are not used internally but are used by outside actors to calculate their borrows and collateral to sell for liquidators, liquidations require speed to gain profit and prevent a bad debt if time is wasted the opportunity will be taken by another liquidator and gas fees will be wasted for bots due to the revert when calculating the collateral to sell via the API, and in worst scenarios if quick liquidations are needed to avoid the market from entering a bad debt the revert will waste time causing the market to enter bad debt.

## Mitigation

use an if-else block checking which is greater in order to get the right scale for the amount of value calculated and in case if where the scale is \`0\` there is no division needed.

## Proof of Concept

## Proof of Concept

To test this simply. make a simple change to the \`tokens.json\` file making the value of ETH less than the value of \`USDC's\` decimal(in the context of the tests it is the base token)

* <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/tests/tokens.json#L19> \`\`\`diff
* ```
  &quot;decimals&quot;: 9,
  ```
* "decimals": 5, \`\`\`

You can see that \`liquidation::absorb\_and\_liquidate\` will always fail via this reason. \`\`\`bash local\_tests::scenarios::liquidation::absorb\_and\_liquidate' panicked at contracts/market/tests/local\_tests/scenarios/liquidation.rs:232:10: called \`Result::unwrap()\` on an \`Err\` value: transaction reverted: ArithmeticOverflow, \`\`\`

This is just an example scenario to show that the way the scale is calculated can lead to issues. \`\`\`rust use crate::utils::{print\_case\_title, setup, TestData}; use chrono::Utc; use fuels::{ accounts::ViewOnlyAccount, programs::{ calls::{CallHandler, CallParameters}, responses::CallResponse, }, types::{transaction::TxPolicies, transaction\_builders::VariableOutputPolicy}, }; use market::PriceDataUpdate; use market\_sdk::{ convert\_i256\_to\_i128, convert\_i256\_to\_u64, convert\_u256\_to\_u128, format\_units\_u128, is\_i256\_negative, parse\_units, };

const AMOUNT\_COEFFICIENT: u64 = 10u64.pow(0); const SCALE\_6: f64 = 10u64.pow(6) as f64; const SCALE\_9: f64 = 10u64.pow(9) as f64;

\#\[tokio::test] async fn absorb\_and\_liquidate() { let TestData { wallets, alice, alice\_account, bob, bob\_account, chad, market, assets, usdc, eth, oracle, price\_feed\_ids, publish\_time, prices, usdc\_contract, .. } = setup(None).await;

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

}

\#\[tokio::test] async fn all\_assets\_liquidated() { let TestData { wallets, alice, alice\_account, bob, bob\_account, chad, market, assets, usdc, eth, oracle, price\_feed\_ids, publish\_time, prices, usdc\_contract, .. } = setup(None).await;

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
let alice_mint_amount &#x3D; parse_units(20000 * AMOUNT_COEFFICIENT, usdc.decimals);
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
let bob_mint_log_amount &#x3D; format!(&quot;{} ETH&quot;, bob_supply_amount as f64 / SCALE_9);
print_case_title(1, &quot;Bob&quot;, &quot;supply_collateral&quot;, bob_mint_log_amount.as_str());
println!(&quot;💸 Bob + {bob_mint_log_amount}&quot;);
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
println!(&quot;Bob can borrow {max_borrow_amount} USDC&quot;);
let log_amount &#x3D; format!(&quot;{} USDC&quot;, max_borrow_amount as f64 / SCALE_6);
print_case_title(2, &quot;Bob&quot;, &quot;withdraw_base&quot;, &amp;log_amount.as_str());
let bob_withdraw_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .withdraw_base(
        &amp;[&amp;oracle.instance],
        max_borrow_amount.try_into().unwrap(),
        &amp;price_data_update,
    )
    .await;
assert!(bob_withdraw_res.is_ok());

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

// Check asset balance
let balance &#x3D; alice.get_asset_balance(&amp;eth.asset_id).await.unwrap();
assert!(balance &#x3D;&#x3D; 10_999_999_997 * AMOUNT_COEFFICIENT);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;eth)
    .await
    .unwrap();
```

}

\#\[tokio::test] async fn is\_liquidatable\_internal\_uses\_correct\_index() { let TestData { wallets, alice, alice\_account, bob, bob\_account, chad, market, assets, usdc, uni, oracle, price\_feed\_ids, publish\_time, prices, usdc\_contract, uni\_contract, .. } = setup(Some(100\_000\_000)).await;

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
// 💰 Amount: 10K USDC
let amount &#x3D; parse_units(10000 * AMOUNT_COEFFICIENT, usdc.decimals);
let log_amount &#x3D; format!(&quot;{} USDC&quot;, amount as f64 / SCALE_6);
print_case_title(0, &quot;Alice&quot;, &quot;supply_base&quot;, log_amount.as_str());
println!(&quot;💸 Alice + {log_amount}&quot;);

// Transfer of 10K USDC to the Alice&#x27;s wallet
usdc_contract.mint(alice_account, amount).await.unwrap();
let balance &#x3D; alice.get_asset_balance(&amp;usdc.asset_id).await.unwrap();
assert!(balance &#x3D;&#x3D; amount);

// Alice calls supply_base
market
    .with_account(&amp;alice)
    .await
    .unwrap()
    .supply_base(usdc.asset_id, amount)
    .await
    .unwrap();

// Сheck supply balance equal to 10K USDC
let (supply_balance, _) &#x3D; market.get_user_supply_borrow(alice_account).await.unwrap();
assert!(supply_balance &#x3D;&#x3D; amount as u128);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;uni)
    .await
    .unwrap();
market.debug_increment_timestamp().await.unwrap();

// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #1 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Bob 🧛
// 🤙 Call: supply_collateral
// 💰 Amount: 1K UNI ~ $5K
let amount &#x3D; parse_units(1000 * AMOUNT_COEFFICIENT, uni.decimals);
let log_amount &#x3D; format!(&quot;{} UNI&quot;, amount as f64 / SCALE_9);
print_case_title(1, &quot;Bob&quot;, &quot;supply_collateral&quot;, log_amount.as_str());
println!(&quot;💸 Bob + {log_amount}&quot;);

// Transfer of 1K UNI to the Bob&#x27;s wallet
uni_contract.mint(bob_account, amount).await.unwrap();

let balance &#x3D; bob.get_asset_balance(&amp;uni.asset_id).await.unwrap();
assert!(balance &#x3D;&#x3D; amount);

// Bob calls supply_collateral
market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .supply_collateral(uni.asset_id, amount)
    .await
    .unwrap();

// Сheck supply balance equal to 1K UNI
let res &#x3D; market
    .get_user_collateral(bob_account, uni.asset_id)
    .await
    .unwrap()
    .value;
assert!(res &#x3D;&#x3D; amount);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;uni)
    .await
    .unwrap();
market.debug_increment_timestamp().await.unwrap();

// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #2 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Bob 🧛
// 🤙 Call: withdraw_base
// 💰 Amount: 2K USDC
let amount &#x3D; parse_units(2500 * AMOUNT_COEFFICIENT, usdc.decimals);
let log_amount &#x3D; format!(&quot;{} USDC&quot;, amount as f64 / SCALE_6);
print_case_title(2, &quot;Bob&quot;, &quot;withdraw_base&quot;, log_amount.as_str());

// Bob calls withdraw_base
market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .withdraw_base(&amp;[&amp;oracle.instance], amount, &amp;price_data_update)
    .await
    .unwrap();

// USDC balance check
let balance &#x3D; bob.get_asset_balance(&amp;usdc.asset_id).await.unwrap();
assert!(balance &#x3D;&#x3D; amount);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;uni)
    .await
    .unwrap();

for _ in 0..6 {
    market.debug_increment_timestamp().await.unwrap();
}

// Calculate liqudiation point, wrong present value and correct present value
let collateral_configurations &#x3D; market.get_collateral_configurations().await.unwrap().value;
let uni_config &#x3D; collateral_configurations
    .iter()
    .find(|config| config.asset_id &#x3D;&#x3D; uni.asset_id);

let market_basics &#x3D; market.get_market_basics_with_interest().await.unwrap();

let liquidation_factor &#x3D; format_units_u128(
    convert_u256_to_u128(uni_config.unwrap().liquidate_collateral_factor),
    18,
);

let borrow_factor &#x3D; format_units_u128(
    convert_u256_to_u128(uni_config.unwrap().borrow_collateral_factor),
    18,
);

let uni_price &#x3D; oracle.price(uni.price_feed_id).await.unwrap().value;
let uni_price &#x3D; uni_price.price as f64 / 10u64.pow(uni.price_feed_decimals as u32) as f64;

let borrow_limit &#x3D; borrow_factor * uni_price * 1000_f64;
let liquidation_point &#x3D; liquidation_factor * uni_price * 1000_f64;

let base_supply_index &#x3D; format_units_u128(
    convert_u256_to_u128(market_basics.value.base_supply_index),
    15,
);

let base_borrow_index &#x3D; format_units_u128(
    convert_u256_to_u128(market_basics.value.base_borrow_index),
    15,
);

let user_principal &#x3D; market
    .get_user_basic(bob_account)
    .await
    .unwrap()
    .value
    .principal;

let user_principal &#x3D;
    convert_i256_to_i128(&amp;user_principal) as f64 / 10u64.pow(usdc.decimals as u32) as f64;

let wrong_present_value &#x3D; base_supply_index * user_principal;
let correct_present_value &#x3D; base_borrow_index * user_principal;

println!(&quot;\n&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; INFO &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&quot;);
println!(
    &quot;📈 Borrow limit: {borrow_limit:?}\n📈 Liquidation point: {liquidation_point:?}\n📈 User principal: {user_principal:?}\n📈 Wrong present value: {wrong_present_value:?}\n📈 Correct present value: {correct_present_value:?}&quot;,
);
print!(&quot;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&quot;);

// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #3 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Chad 🧛
// 🤙 Call: absorb
// 🔥 Target: Bob
print_case_title(3, &quot;Chad&quot;, &quot;absorb&quot;, &quot;Bob&quot;);

// &#x60;is_liquidatable&#x60; accrues iterest first, so this must return &#x60;true&#x60;
assert!(
    market
        .is_liquidatable(&amp;[&amp;oracle.instance], bob_account)
        .await
        .unwrap()
        .value
        &#x3D;&#x3D; true
);

// This should work
market
    .with_account(&amp;chad)
    .await
    .unwrap()
    .absorb(&amp;[&amp;oracle.instance], vec![bob_account], &amp;price_data_update)
    .await
    .unwrap();

// Check if absorb was ok
let (_, borrow) &#x3D; market.get_user_supply_borrow(bob_account).await.unwrap();
assert!(borrow &#x3D;&#x3D; 0);

let amount &#x3D; market
    .get_user_collateral(bob_account, uni.asset_id)
    .await
    .unwrap()
    .value;
assert!(amount &#x3D;&#x3D; 0);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;uni)
    .await
    .unwrap();
```

}

\`\`\`
