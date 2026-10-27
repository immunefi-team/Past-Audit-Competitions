# #35708 \[SC-Insight] Adding too many collaterals will halt the protocol operation

**Submitted on Oct 4th 2024 at 08:10:20 UTC by @SeveritySquad for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35708
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield
  * Permanent freezing of unclaimed royalties
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \`Market\` contract allows adding allowed collateral configuration by the owner of the protocol. Once collateral is added it cannot be removed (just paused). Certain protocol function loop through all collateral configurations regardless if they are paused or not. Once the amount of collateral configurations reaches certain threshold the loop that is going through all the collateral configurations will be too large and will hit the Fuel TX gas limit stopping the protocol operation.

## Vulnerability Details

The following \`Market\` methods are looping through all the collateral configurations:

* \`get\_collateral\_configurations()\`
* \`get\_all\_user\_collateral()\`
* \`get\_all\_totals\_collateral()\`
* \`available\_to\_borrow()\`
* \`is\_borrow\_collateralized()\`
* \`is\_liquidatable\_internal()\`
* \`absorb\_internal()\`

Let's take for example the method \`withdraw\_base()\` which internally calls \`is\_borrow\_collateralized()\`, inside we can find the following loop: \`\`\` let len = storage.collateral\_configurations\_keys.len();

```
while index &lt; len {
    [...]
}
```

\`\`\` For each entry in \`collateral\_configuration\_keys\` the loop adds exactly \`26508\` gas to the TX. The Fuel transaction gas limit is \`30 000 000\`, this means that after adding exactly \`1129\` collateral configuration the loop becomes unexecutable due to the above TX gas limit.

As there is no method for removing entries from \`collateral\_configuration\_keys\`, there will be no way to fix the protocol once the limit is reached.

## Impact Details

Once the collateral amount is reached, the protocols \`withdraw\_base()\` will become un-executable and hence all the base assets will be permanently locked.

Another side effect is that as the collateral configurations are added the gas costs of all the methods which need to cycle through the \`collateral\_configuration\_keys\` \`StorageVec\` increases permanently and hence impacts the cost to the users of the protocol.

Despite reaching the limit having grave consequences for the protocol we chose the severity Medium as the limit is quite large and hence difficult to reach in practice.

## Solution Proposal

Add method to remove collateral configuration from the \`Market\` permanently. This will however have an impact on the currently collateralized loans as it may create bad debt.

Alternative solution would be to create a \`Vec\` holding a list of supplied collateral assets for each user, and when there is a need the protocol will cycle only through the user's individual list of supplied collateral assets.

## References

The problematic loop: <https://github.com/Swaylend/swaylend-monorepo/blob/4f1491b86b10121b0ffa7ca68149cf4e3c641684/contracts/market/src/main.sw#L1333>

## Proof of Concept

## Proof of Concept

Use the following test to print the gas cost of a withdrawal: \`\`\` #\[tokio::test] async fn poc1\_test() { let TestData { wallets, bob, bob\_account, alice, alice\_account, market, assets, usdc, oracle, price\_feed\_ids, publish\_time, prices, usdc\_contract, uni, uni\_contract, .. } = setup().await;

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
// 💰 Amount: 5000.00 USDC
let alice_supply_amount &#x3D; parse_units(5000 * AMOUNT_COEFFICIENT, usdc.decimals);
let alice_mint_amount &#x3D; parse_units(10_000 * AMOUNT_COEFFICIENT, usdc.decimals);
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
// 💰 Amount: 40.00 UNI ~ $200.00
let bob_mint_amount &#x3D; parse_units(50 * AMOUNT_COEFFICIENT, uni.decimals);
let bob_supply_amount &#x3D; parse_units(40 * AMOUNT_COEFFICIENT, uni.decimals);
let bob_mint_log_amount &#x3D; format!(&quot;{} UNI&quot;, bob_mint_amount as f64 / SCALE_9);
print_case_title(1, &quot;Bob&quot;, &quot;supply_collateral&quot;, bob_mint_log_amount.as_str());
println!(&quot;💸 Bob + {bob_mint_log_amount}&quot;);
uni_contract
    .mint(bob_account, bob_mint_amount)
    .await
    .unwrap();
let bob_balance &#x3D; bob.get_asset_balance(&amp;uni.asset_id).await.unwrap();
assert!(bob_balance &#x3D;&#x3D; bob_mint_amount);
let bob_supply_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .supply_collateral(uni.asset_id, bob_supply_amount)
    .await;
assert!(bob_supply_res.is_ok());

let bob_user_collateral &#x3D; market
    .get_user_collateral(bob_account, uni.asset_id)
    .await
    .unwrap()
    .value;
assert!(bob_user_collateral &#x3D;&#x3D; bob_supply_amount);


market.debug_increment_timestamp().await.unwrap();
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #2 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Bob 🧛
// 🤙 Call: withdraw_base
// 💰 Amount: 150.00 USDC
let amount_to_fail &#x3D; parse_units(1000 * AMOUNT_COEFFICIENT, usdc.decimals);
let withdraw_base_fail &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .withdraw_base(&amp;[&amp;oracle.instance], amount_to_fail, &amp;price_data_update)
    .await;
assert!(withdraw_base_fail.is_err());

let amount &#x3D; parse_units(70 * AMOUNT_COEFFICIENT, usdc.decimals);
let log_amount &#x3D; format!(&quot;{} USDC&quot;, amount as f64 / SCALE_6);
print_case_title(2, &quot;Bob&quot;, &quot;withdraw_base&quot;, &amp;log_amount.as_str());
let bob_withdraw_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .withdraw_base(&amp;[&amp;oracle.instance], amount, &amp;price_data_update)
    .await;
assert!(bob_withdraw_res.is_ok());
println!(&quot;Gas used for bob_withdraw_res: {}&quot;, bob_withdraw_res.unwrap().gas_used);
let balance &#x3D; bob.get_asset_balance(&amp;usdc.asset_id).await.unwrap();
assert!(balance &#x3D;&#x3D; amount);
```

} \`\`\`

Token configurations are added by adding more entries into the \`tokens.json\` file.

With 5 tokens the test lists the following: \`\`\` Gas used for bob\_withdraw\_res: 187425 \`\`\` With 4 token the test lists the following: \`\`\` Gas used for bob\_withdraw\_res: 160923 \`\`\`

This means that each collateral configuration adds \`26508\` gas.
