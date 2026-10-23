# #35724 \[SC-Low] Users can withdraw collateral even when the admin pauses the contract.

**Submitted on Oct 5th 2024 at 00:13:01 UTC by @SeveritySquad for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35724
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/abis/market\\_abi/src/abi.sw>
* **Impacts:**
  * Users can interact with some critical functionalities the protocol when paused

## Description

## Brief/Intro

the admin is allowed to pause the contract or certain collaterals whenever there might be an issue in the system to avoid exploits or for protocol-known reasons, however, users will still be able to withdraw their collateral when the contract markets functionalities are paused.

## Vulnerability Details

with the

* \`pause\_collateral\_asset()\`
* \`pause()\`

The admin is allowed to stop functionalities of the protocol. For instance on the call to supply\_collateral() \`\`\`sway // ## 3.1 Supply Collateral #\[payable, storage(write)] fn supply\_collateral() { // Only allow supplying collateral if paused flag is not set require(!storage.pause\_config.supply\_paused.read(), Error::Paused); // code let asset\_id: AssetId = msg\_asset\_id(); let collateral\_configuration = storage.collateral\_configurations.get(asset\_id).read();//@audit ere require(!collateral\_configuration.paused, Error::Paused); // code // Log user supply collateral event log(UserSupplyCollateralEvent { account: caller, asset\_id, amount, }); } \`\`\` if ensures that collateral cannot be supplied if the admin pauses the market, however, the opposite is the case for \`withdraw\_collateral()\` \`\`\`sway #\[payable, storage(write)] fn withdraw\_collateral( asset\_id: AssetId, amount: u64, price\_data\_update: PriceDataUpdate, ) { // no checks against withdrawal when the protocol is paused. reentrancy\_guard();

```
    // Get the caller&#x27;s account and calculate the new user and total collateral
    let caller &#x3D; msg_sender().unwrap();
    let user_collateral &#x3D; storage.user_collateral.get((caller, asset_id)).try_read().unwrap_or(0) - amount;
    let total_collateral &#x3D; storage.totals_collateral.get(asset_id).try_read().unwrap_or(0) - amount;

    // Update the storage values (total collateral, user collateral)
    storage.totals_collateral.insert(asset_id, total_collateral);
    storage
        .user_collateral
        .insert((caller, asset_id), user_collateral);

    // Update price data
    update_price_feeds_if_necessary_internal(price_data_update);

    // Note: no accrue interest, BorrowCollateralFactor &lt; LiquidationCollateralFactor covers small changes
    // Check if the user is borrow collateralized
    require(is_borrow_collateralized(caller), Error::NotCollateralized);

    transfer(caller, asset_id, amount);

    // Log user withdraw collateral event
    log(UserWithdrawCollateralEvent {
        account: caller,
        asset_id,
        amount,
    });
}
```

\`\`\`

## Mitigation

Apply same checks in \`supply\_collateral()\` for the \`withraw\_collateral()\`

## Impact Details

Here, users will be able to bypass the sanctions put by the admin on the market allowing for exploits to continue even tho the contract is paused.

## References

* <https://github.com/Swaylend/swaylend-monorepo/blob/63e7b1163216d1400b9436c8d256f0f0e043e225/contracts/market/src/main.sw#L304C1-L338C6>

## Proof of Concept

\`\`\`rust use std::str::FromStr;

use fuels::accounts::ViewOnlyAccount; use market::PriceDataUpdate; use market\_sdk::parse\_units;

const AMOUNT\_COEFFICIENT: u64 = 10u64.pow(0); const SCALE\_6: f64 = 10u64.pow(6) as f64; const SCALE\_9: f64 = 10u64.pow(9) as f64; use crate::utils::{setup, TestData}; use fuels::types::{ContractId, U256}; use market::{CollateralConfiguration, PauseConfiguration}; use market\_sdk::get\_market\_config;

\#\[tokio::test] async fn test\_withdraw\_when\_paused() { let TestData { admin, admin\_account, alice, alice\_account, bob, bob\_account, market, assets, usdc, oracle, price\_feed\_ids, publish\_time, prices, usdc\_contract, uni, uni\_contract, .. } = setup().await;

```
let asset_id &#x3D; assets[&quot;ETH&quot;].asset_id;

let mock_collateral_config &#x3D; CollateralConfiguration {
    asset_id: assets[&quot;USDC&quot;].asset_id.into(),
    price_feed_id: assets[&quot;USDC&quot;].price_feed_id,
    decimals: assets[&quot;USDC&quot;].decimals.try_into().unwrap(),
    borrow_collateral_factor: U256::from(18), // decimals: 18
    liquidate_collateral_factor: U256::from(18), // decimals: 18
    liquidation_penalty: U256::from(18),      // decimals: 18
    supply_cap: 10,                           // decimals: asset decimals
    paused: false,
};

let admin_add_collat_res &#x3D; market
    .with_account(&amp;admin)
    .await
    .unwrap()
    .add_collateral_asset(&amp;mock_collateral_config)
    .await;

// make sure add_collateral_asset was ok
assert!(admin_add_collat_res.is_ok());

// &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D; Step #1 &#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;&#x3D;
// 👛 Wallet: Bob 🧛
// 🤙 Call: supply_collateral
// 💰 Amount: 40.00 UNI ~ $200.00
let bob_mint_amount &#x3D; parse_units(50 * AMOUNT_COEFFICIENT, uni.decimals);
let bob_supply_amount &#x3D; parse_units(40 * AMOUNT_COEFFICIENT, uni.decimals);
let bob_mint_log_amount &#x3D; format!(&quot;{} UNI&quot;, bob_mint_amount as f64 / SCALE_9);
// print_case_title(1, &quot;Bob&quot;, &quot;supply_collateral&quot;, bob_mint_log_amount.as_str());
println!(&quot;💸 Bob + {bob_mint_log_amount}&quot;);

// mint bob an amount to supply
uni_contract
    .mint(bob_account, bob_mint_amount)
    .await
    .unwrap();
let bob_balance &#x3D; bob.get_asset_balance(&amp;uni.asset_id).await.unwrap();
assert!(bob_balance &#x3D;&#x3D; bob_mint_amount);

// allow bob to supply collateral
let bob_supply_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .supply_collateral(uni.asset_id, bob_supply_amount)
    .await;
assert!(bob_supply_res.is_ok());

// makes checks to ensure bob deposit was successful
let bob_user_collateral &#x3D; market
    .get_user_collateral(bob_account, uni.asset_id)
    .await
    .unwrap()
    .value;
assert!(bob_user_collateral &#x3D;&#x3D; bob_supply_amount);

// allowing admin to pause the contract.

let admin_pause_collat_res &#x3D; market
    .with_account(&amp;admin)
    .await
    .unwrap()
    .pause_collateral_asset(uni.asset_id)
    .await;

assert!(admin_pause_collat_res.is_ok());

// get bobs withdrawable balance.
let bob_withdraw_amount &#x3D; market
    .get_user_collateral(bob_account, uni.asset_id)
    .await
    .unwrap()
    .value;

// get the price feed to enable withdraw.
let price_data_update &#x3D; PriceDataUpdate {
    update_fee: 0,
    price_feed_ids,
    publish_times: vec![publish_time; assets.len()],
    update_data: oracle.create_update_data(&amp;prices).await.unwrap(),
};

// bob is not supposed to be able to withdraw when the market is paused, however he can.
let withdraw_collateral_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .withdraw_collateral(
        &amp;[&amp;oracle.instance],
        uni.asset_id,
        bob_withdraw_amount.try_into().unwrap(),
        &amp;price_data_update,
    )
    .await;

assert!(withdraw_collateral_res.is_ok());
```

}

\`\`\`
