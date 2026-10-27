# #35876 \[SC-High] Users will lose funds on calls to critical functions if the prices are not updated

**Submitted on Oct 11th 2024 at 15:51:15 UTC by @SeveritySquad for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35876
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The internal \`update\_price\_feeds\_if\_necessary\_internal()\` transfers the fees for updating the Pyth price-feeds to the oracle, however, the fees will be left stuck in Pyth in scenarios where the prices were never even updated.

## Vulnerability Details

The \`update\_price\_feeds\_if\_necessary\_internal()\` is called by:

* \`absorb()\`.
* \`withdraw\_base()\`.
* \`withdraw\_collateral()\`.

Using \`withdraw\_collateral()\` as an example, on the call to it, \`\`\`rust #\[payable, storage(write)] fn withdraw\_collateral( // some code // Update price data //@audit here the price is updated if neccessary. update\_price\_feeds\_if\_necessary\_internal(price\_data\_update);

```
    // Note: no accrue interest, BorrowCollateralFactor &lt; LiquidationCollateralFactor covers small changes
    // Check if the user is borrow collateralized
    require(is_borrow_collateralized(caller), Error::NotCollateralized);

    transfer(caller, asset_id, amount);
```

// code for logging } \`\`\` internally the function calls \`update\_price\_feeds\_if\_necessary\_internal()\` which further makes a call to Pyth's \`update\_price\_feeds\_if\_necessary()\` function, before the call to Pyth's oracle it ensures that the user sent the amount of fee for the price feeds to be updated and forwards that fee on the call to Pyth's oracle \`update\_price\_feeds\_if\_necessary()\`.

* <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw#L1441C1-L1469C2> \`\`\`rust #\[payable, storage(read)] fn update\_price\_feeds\_if\_necessary\_internal(price\_data\_update: PriceDataUpdate) { // some code // check if the payment is sufficient require( msg\_amount() >= price\_data\_update .update\_fee && msg\_asset\_id() == AssetId::base(), Error::InvalidPayment, );

  let oracle = abi(PythCore, contract\_id.bits()); oracle .update\_price\_feeds\_if\_necessary { asset\_id: AssetId::base().bits(), coins: price\_data\_update.update\_fee, }( price\_data\_update .price\_feed\_ids, price\_data\_update .publish\_times, price\_data\_update .update\_data, ); }

\`\`\` The issue here lies in the fact that Pyth's oracle \`update\_price\_feeds\_if\_necessary()\` which only calls \`update\_price\_feeds()\` if the latest publish time is less than the input published time, and fees are only required in \`update\_price\_feeds()\` whenever a price feed is updated.

* <https://github.com/pyth-network/pyth-crosschain/blob/9c761626440da0c731d5e41e9b6d31aa5e909bf3/target\\_chains/fuel/contracts/pyth-contract/src/main.sw#L292C1-L300C10> \`\`\`rust while i < price\_feed\_ids.len() { if latest\_publish\_time(price\_feed\_ids.get(i).unwrap()) < publish\_times.get(i).unwrap() { update\_price\_feeds(update\_data); return; }

  ```
        i +&#x3D; 1;
    }
  ```

\`\`\`

* <https://github.com/pyth-network/pyth-crosschain/blob/9c761626440da0c731d5e41e9b6d31aa5e909bf3/target\\_chains/fuel/contracts/pyth-contract/src/main.sw#L420C2-L421C71>

\`\`\`rust #\[storage(read, write), payable] fn update\_price\_feeds(update\_data: Vec\<Bytes>) { // code for price feed updates. let required\_fee = total\_fee(total\_number\_of\_updates, storage.single\_update\_fee); require(msg\_amount() >= required\_fee, PythError::InsufficientFee); \`\`\`

## Impact Details

In scenarios when the price feeds are not updated in Pyth's oracle, the fees for updating those feeds will be left stuck in Pyth with no way to retrieve it, and given the number of transactions and collaterals that would be used the amount the users will lose for price feeds that were never updated will increase exponentially. Also, Pyth fee can increase in the future causing the costs to rise, and concerning this

* <https://github.com/Swaylend/swaylend-monorepo/issues/158>

## Mitigation

Use the Pyth's \`PythInfo::latest\_publish\_time()\` -> <https://github.com/pyth-network/pyth-crosschain/blob/9c761626440da0c731d5e41e9b6d31aa5e909bf3/target\\_chains/fuel/contracts/pyth-contract/src/main.sw#L563C8-L563C27> to get the latest publish times, possibly looping through to check the price feeds that need not be updated and only forwarding to the Pyth oracle the amount of fees required for the feeds that will be updated.

## Proof of Concept

## Proof of Concept

\`\`\`bash cargo test local\_tests::scenarios::multicall\_withdraw\_supply::multicall\_withdraw\_supply\_test \`\`\` We update the price feeds, before with call \`withdraw\_base()\` with that same price feed update data, but the fee is still lost to the Pyth oracle \`\`\`rust use crate::utils::{print\_case\_title, setup, TestData}; use fuels::{ accounts::ViewOnlyAccount, programs::{ calls::{CallHandler, CallParameters}, responses::CallResponse, }, types::{transaction::TxPolicies, transaction\_builders::VariableOutputPolicy}, }; use market::PriceDataUpdate; use market\_sdk::parse\_units;

const AMOUNT\_COEFFICIENT: u64 = 10u64.pow(0); const SCALE\_6: f64 = 10u64.pow(6) as f64; const SCALE\_9: f64 = 10u64.pow(9) as f64;

\#\[tokio::test] async fn multicall\_withdraw\_supply\_test() { let TestData { wallets, alice, alice\_account, bob, bob\_account, market, usdc, usdc\_contract, eth, oracle, price\_feed\_ids, publish\_time, prices, assets, .. } = setup().await;

```
let price_data_update &#x3D; PriceDataUpdate {
    update_fee: 1,
    price_feed_ids,
    publish_times: vec![publish_time; assets.len()],
    update_data: oracle.create_update_data(&amp;prices).await.unwrap(),
};

// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #0 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Alice 🧛
// 🤙 Call: supply_base
// 💰 Amount: 1000.00 USDC
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
let bob_mint_log_amount &#x3D; format!(&quot;{} ETH&quot;, bob_supply_amount as f64 / SCALE_9);
print_case_title(1, &quot;Bob&quot;, &quot;supply_collateral&quot;, bob_mint_log_amount.as_str());
let bob_supply_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .supply_collateral(eth.asset_id, bob_supply_amount)
    .await;
assert!(bob_supply_res.is_ok());

market.debug_increment_timestamp().await.unwrap();

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;eth)
    .await
    .unwrap();

// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #2 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Bob 🧛
// 🤙 Call: withdraw_base then supply_base using multicall
// 💰 Amount: 500.00 USDC
let bob_mint_amount &#x3D; parse_units(500 * AMOUNT_COEFFICIENT, usdc.decimals);
usdc_contract
    .mint(bob_account, bob_mint_amount)
    .await
    .unwrap();
let balance &#x3D; bob.get_asset_balance(&amp;usdc.asset_id).await.unwrap();
assert!(balance &#x3D;&#x3D; bob_mint_amount);
let bob_withdraw_amount &#x3D; parse_units(100 * AMOUNT_COEFFICIENT, usdc.decimals);
let bob_withdraw_log_amount &#x3D; format!(&quot;{} USDC&quot;, bob_withdraw_amount as f64 / SCALE_6);
print_case_title(
    2,
    &quot;Bob&quot;,
    &quot;withdraw_base then supply_base&quot;,
    bob_withdraw_log_amount.as_str(),
);

// simulate a call to update_price_feeds_if_necessary() so that call with same price data update
// won&#x27;t udpate the price feed again. since latest_publish_time(price_feed_ids.get(i).unwrap()) &#x3D;&#x3D; publish_times.get(i)
// check https://github.com/pyth-network/pyth-crosschain/blob/9c761626440da0c731d5e41e9b6d31aa5e909bf3/target_chains/fuel/contracts/pyth-contract/src/main.sw#L292C1-L300C10
// for more info

let alice_updates_feeds_recently_res &#x3D; market
    .with_account(&amp;alice)
    .await
    .unwrap()
    .update_price_feeds_if_necessary(&amp;[&amp;oracle.instance], &amp;price_data_update.clone())
    .await;
assert!(alice_updates_feeds_recently_res.is_ok());

let tx_policies &#x3D; TxPolicies::default().with_script_gas_limit(1_000_000);

// Withdraw base
let withdraw_base_call &#x3D; market
    .instance
    .methods()
    .withdraw_base(bob_withdraw_amount.into(), price_data_update.clone())
    .with_contracts(&amp;[&amp;oracle.instance])
    .with_tx_policies(tx_policies)
    .call_params(CallParameters::default().with_amount(price_data_update.update_fee))
    .unwrap();

// Supply base
let supply_base_call &#x3D; market
    .instance
    .methods()
    .supply_base()
    .with_tx_policies(tx_policies)
    .call_params(
        CallParameters::default()
            .with_amount(bob_withdraw_amount)
            .with_asset_id(usdc.asset_id),
    )
    .unwrap();

let multi_call_handler &#x3D; CallHandler::new_multi_call(bob.clone())
    .add_call(withdraw_base_call)
    .add_call(supply_base_call)
    .with_variable_output_policy(VariableOutputPolicy::Exactly(2));

// Submit tx
let submitted_tx &#x3D; multi_call_handler.submit().await.unwrap();

// Wait for response
let _: CallResponse&lt;((), ())&gt; &#x3D; submitted_tx.response().await.unwrap();

// Check asset balance
let balance &#x3D; bob.get_asset_balance(&amp;usdc.asset_id).await.unwrap();
assert!(balance &#x3D;&#x3D; bob_mint_amount);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;usdc)
    .await
    .unwrap();
// after the call the fee will be stuck in the pyth oracle.
```

}

\`\`\`
