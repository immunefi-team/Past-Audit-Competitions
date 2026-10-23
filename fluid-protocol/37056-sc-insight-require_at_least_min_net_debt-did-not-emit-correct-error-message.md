# #37056 \[SC-Insight] \`require\_at\_least\_min\_net\_debt\` did not emit correct error message

**Submitted on Nov 24th 2024 at 03:16:10 UTC by @InquisitorScythe for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37056
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/borrow-operations-contract/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

in `require_at_least_min_net_debt`, if it did not pass the check, it will emit a error message "Borrow Operations: net debt must be greater than 0", it did not consider the value of `MIN_NET_DEBT` in the error message.

## Vulnerability Details

in `contracts/borrow-operations-contract/src/main.sw`

```
fn require_at_least_min_net_debt(_net_debt: u64) {
    require(
        _net_debt >= MIN_NET_DEBT,
        "Borrow Operations: net debt must be greater than 0", // @audit: error message
    );
}
```

but MIN\_NET\_DEBT is 500 usdf defined in `fluid_math.sw`

```
// min debt is 500 USDF for staging 
pub const MIN_NET_DEBT: u64 = 500_000_000_000; 
```

It is suggested to emit correct error message like :

```
fn require_at_least_min_net_debt(_net_debt: u64) {
    require(
        _net_debt >= MIN_NET_DEBT,
        "Borrow Operations: net debt must be greater than 500", // @audit: error message
    );
}
```

It is even better to use format string to fill `MIN_NET_DEBT` into the error message, but I could not find out how to do it

## Impact Details

Deliver wrong message when the checks fails, makes it hard to debug and confuse the users.

## References

None

## Proof of Concept

## Proof of Concept

change the function `fails_open_trove_under_min_usdf_required` in `contracts/borrow-operations-contract/tests/failure`like:

```
#[tokio::test]
async fn fails_open_trove_under_min_usdf_required() {
    let (contracts, admin, _) = setup_protocol(2, false, false).await;

    token_abi::mint_to_id(
        &contracts.asset_contracts[0].asset,
        5_000 * PRECISION,
        Identity::Address(admin.address().into()),
    )
    .await;

    let coll_amount = 1_200 * PRECISION;
    let debt_amount = 400 * PRECISION;
    // 100 USDF < 500 USDF
    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        pyth_price_feed(1),
    )
    .await;

    let res = borrow_operations_abi::open_trove(
        &contracts.borrow_operations,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.asset_contracts[0].asset,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.sorted_troves,
        &contracts.asset_contracts[0].trove_manager,
        &contracts.active_pool,
        coll_amount,
        debt_amount,
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await;

    if let Err(error) = res {
        assert!(
            error
                .to_string()
                .contains("net debt must be greater than 500"),
            "Incorrect error message: {}",
            error
        );
    }
}
```

and then run `cargo test fails_open_trove_under_min_usdf_required -- --nocapture`, the output is like :

```
running 1 test
Deploying core contracts...
Initializing core contracts...
thread 'fails_open_trove_under_min_usdf_required' panicked at contracts/borrow-operations-contract/tests/failure.rs:279:9:
Incorrect error message: transaction reverted: AsciiString { data: "Borrow Operations: net debt must be greater than 0" }, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: 25fbb8fb8ceb5e847bfc1c6f0f215b9b2aab5ce6462cce8671b46adaf99cd199, amount: 1200000000000, asset_id: 9a464b41647f372808cb8f20f8f7aa8eef27acdfd9e6aaa166b1a1ae8b35d7f4, gas: 50472, param1: 10480, param2: 10498, pc: 15624, is: 15624 }, Call { id: 25fbb8fb8ceb5e847bfc1c6f0f215b9b2aab5ce6462cce8671b46adaf99cd199, to: ab8d58390cc175702ab3059fbf77e43d48a36c17d9b2fa530968806a3a31fbd8, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 34972, param1: 67104704, param2: 67103680, pc: 131448, is: 131448 }, Call { id: ab8d58390cc175702ab3059fbf77e43d48a36c17d9b2fa530968806a3a31fbd8, to: c672c33f1411c7c77d6d4f169b89c9e18060c79a8eb434c0d160cd9b4d1b74d8, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 30993, param1: 67101056, param2: 67100032, pc: 183393, is: 183393 }, ReturnData { id: c672c33f1411c7c77d6d4f169b89c9e18060c79a8eb434c0d160cd9b4d1b74d8, ptr: 67097472, len: 28, digest: a9dd65fcff83b6bccf652da94d61edd8632ff002061afc77e498266005f6be0c, pc: 185429, is: 183393, data: Some(00000000000000000000000900...) }, ReturnData { id: ab8d58390cc175702ab3059fbf77e43d48a36c17d9b2fa530968806a3a31fbd8, ptr: 67095680, len: 8, digest: 59f603c39018dc65fbf3007d91985355b0e27df2993aab3c4a9e4b5ea36c5996, pc: 148732, is: 131448, data: Some(000000003b9aca00) }, Call { id: 25fbb8fb8ceb5e847bfc1c6f0f215b9b2aab5ce6462cce8671b46adaf99cd199, to: 82d6fe3974112bf9994034b4a425998e635a89521fc6747e1b38ba016b717cb8, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 25254, param1: 67094656, param2: 67093632, pc: 131600, is: 131600 }, ReturnData { id: 82d6fe3974112bf9994034b4a425998e635a89521fc6747e1b38ba016b717cb8, ptr: 67090171, len: 8, digest: af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc, pc: 157732, is: 131600, data: Some(0000000000000000) }, Call { id: 25fbb8fb8ceb5e847bfc1c6f0f215b9b2aab5ce6462cce8671b46adaf99cd199, to: 371f0b98ae91d7e9198dab7cd5714e22645813a787f11d0c80375849619d1f06, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 15206, param1: 67089147, param2: 67088123, pc: 132624, is: 132624 }, ReturnData { id: 371f0b98ae91d7e9198dab7cd5714e22645813a787f11d0c80375849619d1f06, ptr: 0, len: 0, digest: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855, pc: 151880, is: 132624, data: Some() }, Call { id: 25fbb8fb8ceb5e847bfc1c6f0f215b9b2aab5ce6462cce8671b46adaf99cd199, to: 183ba321f9edc4afa185a7ca9001eeba214a52753ea2d1392f1593e1776492d2, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 8880, param1: 67084987, param2: 67083963, pc: 133912, is: 133912 }, Mint { sub_id: 0000000000000000000000000000000000000000000000000000000000000000, contract_id: 183ba321f9edc4afa185a7ca9001eeba214a52753ea2d1392f1593e1776492d2, val: 2000000000, pc: 175720, is: 133912 }, Transfer { id: 183ba321f9edc4afa185a7ca9001eeba214a52753ea2d1392f1593e1776492d2, to: 371f0b98ae91d7e9198dab7cd5714e22645813a787f11d0c80375849619d1f06, amount: 2000000000, asset_id: 0a9bf6faca63498a8e53497a1d9cdb4e7a772d1e65d93999a0d1550a1e47db2c, pc: 175992, is: 133912 }, LogData { id: 183ba321f9edc4afa185a7ca9001eeba214a52753ea2d1392f1593e1776492d2, ra: 0, rb: 17462098202904023478, ptr: 67080315, len: 80, digest: 75fba1c44ab4fce453cea9982bd94199a56df408c4d7cbf498c04ca3a828ee7a, pc: 171716, is: 133912, data: Some(0a9bf6faca63498a8e53497a1d...) }, ReturnData { id: 183ba321f9edc4afa185a7ca9001eeba214a52753ea2d1392f1593e1776492d2, ptr: 0, len: 0, digest: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855, pc: 154544, is: 133912, data: Some() }, LogData { id: 25fbb8fb8ceb5e847bfc1c6f0f215b9b2aab5ce6462cce8671b46adaf99cd199, ra: 0, rb: 10098701174489624218, ptr: 67079291, len: 58, digest: c1130903e915571f01a555bdde08cda22ec45f1c491f0e4fa93c8dcace05a5d1, pc: 46492, is: 15624, data: Some(0000000000000032426f72726f...) }, Revert { id: 25fbb8fb8ceb5e847bfc1c6f0f215b9b2aab5ce6462cce8671b46adaf99cd199, ra: 18446744073709486080, pc: 46500, is: 15624 }, ScriptResult { result: Revert, gas_used: 48315 }]
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
test fails_open_trove_under_min_usdf_required ... FAILED

failures:

failures:
    fails_open_trove_under_min_usdf_required
```
