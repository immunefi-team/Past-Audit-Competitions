# #37276 \[SC-Medium] Redstone's price feed is used incorrectly.

**Submitted on Dec 1st 2024 at 11:03:07 UTC by @jasonxiale for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37276
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/oracle-contract/src/main.sw>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In current implementation for `oracle-contract.get_price`, while function `is_pyth_price_stale_or_outside_confidence` return true in [main.sw#L88](https://github.com/Hydrogen-Labs/fluid-protocol/blob/78ab7bdd243b414b424fca6e1eb144218f36a18a/contracts/oracle-contract/src/main.sw#L88), Redstone price feed will be used.

However there is a flaw when using Redstone price feed that might lead stale price being used.

## Vulnerability Details

According to `oracle-contract.get_price`, when redstone is used, `redstone.read_timestamp` will be used as the price 's publish timestamp.

And `redstone.read_timestamp` will be checked against with `current_time` in [main.sw#L107](https://github.com/Hydrogen-Labs/fluid-protocol/blob/78ab7bdd243b414b424fca6e1eb144218f36a18a/contracts/oracle-contract/src/main.sw#L107)

**The issue is that `redstone.read_timestamp` uses Unix timestamp, but Fule's `timestamp` function returns TAI64 format timestamp**

1. Quoting from [timestamp's implementation](https://github.com/FuelLabs/sway/blob/6d9065b8d762a39eb475562426a2d4ed17d92d00/sway-lib-std/src/block.sw#L37-L61)

````rust
 63 /// Get the TAI64 timestamp of a block at a given `block_height`.
 64 ///
 65 /// # Additional Information
 66 ///
 67 /// The TAI64 timestamp begins at 2^62 seconds before 1970, and ends at 2^62 seconds after 1970,
 68 /// with a TAI second defined as the duration of 9192631770 periods of the radiation corresponding
 69 /// to the transition between the two hyperfine levels of the ground state of the cesium atom.
 70 ///
 71 /// # Arguments
 72 ///
 73 /// * `block_height`: [u32] - The height of the block to get the timestamp of.
 74 ///
 75 /// # Returns
 76 ///
 77 /// * [u64] - The TAI64 timestamp of the block at `block_height`.
 78 ///
 79 /// # Examples
 80 ///
 81 /// ```sway
 82 /// use std::block::timestamp_of_block;
 83 ///
 84 /// fn foo() {
 85 ///     let timestamp_of_block_100 = timestamp_of_block(100u32);
 86 ///     log(timestamp_of_block_100);
 87 /// }
 88 /// ```
 89 pub fn timestamp_of_block(block_height: u32) -> u64 {
 90     asm(timestamp, height: block_height) {
 91         time timestamp height;
 92         timestamp: u64
 93     }
 94 }
````

From above code, we can see that `timestamp()` returns \[The TAI64 timestamp of the current block.] (<https://github.com/FuelLabs/sway/blob/6d9065b8d762a39eb475562426a2d4ed17d92d00/sway-lib-std/src/block.sw#L47>)

2. According to [redstone\_adapter.read\_timestamp](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/187f358b9e48544133034af89da8630061c7df79/packages/fuel-connector/sway/contract_adapter/src/redstone_adapter.sw#L71-L74), the function uses `storage.timestamp` as return value.
3. And `storage.timestamp` is written by [redstone\_adapter.sw#L139](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/187f358b9e48544133034af89da8630061c7df79/packages/fuel-connector/sway/contract_adapter/src/redstone_adapter.sw#L139) in `redstone_adapter.overwrite_prices` function.
4. `redstone_adapter.overwrite_prices` is called by `redstone_adapter.write_prices` in [redstone\_adapter.sw#L96](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/187f358b9e48544133034af89da8630061c7df79/packages/fuel-connector/sway/contract_adapter/src/redstone_adapter.sw#L96)
5. Now we'll check how `timestamp` is returned in [redstone\_adapter.sw#L95](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/187f358b9e48544133034af89da8630061c7df79/packages/fuel-connector/sway/contract_adapter/src/redstone_adapter.sw#L95)
6. In [redstone\_adapter.process\_payload](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/187f358b9e48544133034af89da8630061c7df79/packages/fuel-connector/sway/contract_adapter/src/redstone_adapter.sw#L142-L151), a `config` var is created with `config.block_timestamp` set to `get_unix_timestamp()` [get\_unix\_timestamp()](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/main/packages/fuel-connector/sway/common/src/timestamp.sw#L6) is defined as:

```rust
pub fn get_unix_timestamp() -> u64 {
    timestamp() - TAI64_UNIX_ADJUSTMENT
}
```

**From here, we know that `config.block_timestamp` is UNIX time.**

7. Back to `redstone_adapter.process_payload`, in [redstone\_adapter.sw#L150](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/187f358b9e48544133034af89da8630061c7df79/packages/fuel-connector/sway/contract_adapter/src/redstone_adapter.sw#L150), `process_input` is called, and `process_input` is defined in [processor.process\_input](https://github.com/redstone-finance/redstone-fuel-sdk/blob/5aa4ac58f2ebfc6d0a7ffd4b8bdbb509fa87df50/src/core/processor.sw#L28-L39).

```rust
 28 pub fn process_input(bytes: Bytes, config: Config) -> (Vec<u256>, u64) {
 29     config.check_parameters();
 30 
 31     let payload = Payload::from_bytes(bytes);
 32     let timestamp = config.validate_timestamps(payload); <<<--- Here we only care timestamp
 33 
 34     let matrix = get_payload_result_matrix(payload, config);
 35     let results = get_feed_values(matrix, config);
 36 
 37     config.validate_signer_count(results);
 38     (results.aggregated(), timestamp)
 39 }
```

8. In [processor.sw#L32](https://github.com/redstone-finance/redstone-fuel-sdk/blob/5aa4ac58f2ebfc6d0a7ffd4b8bdbb509fa87df50/src/core/processor.sw#L32), `timestamp` is returned by `config.validate_timestamps` function.
9. `config.validate_timestamps` is defined in [config\_validation.sw#L27-L43](https://github.com/redstone-finance/redstone-fuel-sdk/blob/5aa4ac58f2ebfc6d0a7ffd4b8bdbb509fa87df50/src/core/config_validation.sw#L27-L43)

```rust
 27     fn validate_timestamps(self, payload: Payload) -> u64 {
 28         let first_timestamp = payload.data_packages.get(0).unwrap().timestamp;
 29         validate_timestamp(first_timestamp, self.block_timestamp * 1000);
 30 
... 
41 
 42         first_timestamp
 43     }
```

10. In [config\_validation.sw#L28-L29](https://github.com/redstone-finance/redstone-fuel-sdk/blob/5aa4ac58f2ebfc6d0a7ffd4b8bdbb509fa87df50/src/core/config_validation.sw#L28-L29), the payload's timestamp and self.block\_timestamp(which is UNIX timestamp) are passed to `validate_timestamp` function.
11. According to [validate\_timestamp's implemention](https://github.com/redstone-finance/redstone-fuel-sdk/blob/5aa4ac58f2ebfc6d0a7ffd4b8bdbb509fa87df50/src/core/validation.sw#L7-L21), we can get see that `payload's timestamp` in step9 should be UNIX timestamp format.

```rust
  7 pub fn validate_timestamp(timestamp: u64, block_timestamp: u64) {
  8     if (block_timestamp > timestamp) {
  9         require(
 10             block_timestamp - timestamp <= MAX_DATA_TIMESTAMP_DELAY_SECONDS * 1000,
 11             RedStoneError::TimestampOutOfRange((false, block_timestamp, timestamp)),
 12         );
 13     }
 14 
 15     if (timestamp > block_timestamp) {
 16         require(
 17             timestamp - block_timestamp <= MAX_DATA_TIMESTAMP_AHEAD_SECONDS * 1000,
 18             RedStoneError::TimestampOutOfRange((true, block_timestamp, timestamp)),
 19         );
 20     }
 21 }
```

## Impact Details

As I described above, when `is_pyth_price_stale_or_outside_confidence` returns true, Redstone can't work as expected.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

The following POC is used to demonstrate that when `is_pyth_price_stale_or_outside_confidence` returns true, Redstone's oracle will be used.

Please put the following patch in `contracts/oracle-contract/tests/authorization.rs` and run

```bash
cargo test test_get_price_pyth_from_redstone -- --nocapture
...
running 1 test
xxx_xxx redstone RedstoneCore { contract_id: Bech32ContractId { hrp: "fuel", hash: 6d4aa3cbdfd84bc2702b388838ecd680a0a63a7ae7701c28e5ac0e72ca33b36f }, account: WalletUnlocked { wallet: Wallet { address: Bech32Address { hrp: "fuel", hash: bdaad6a89e073e177895b3e5a9ccd15806749eda134a6438dae32fc5b6601f3f } }, private_key: 0000000000000000000000000000000000000000000000000000000000000003 }, log_decoder: LogDecoder { log_formatters: {}, decoder_config: DecoderConfig { max_depth: 45, max_tokens: 10000 } }, encoder_config: EncoderConfig { max_depth: 45, max_tokens: 10000 } }
thread 'test_get_price_pyth_from_redstone' panicked at /opt/work/fluid-protocol/test-utils/src/interfaces/oracle.rs:54:14:
called `Result::unwrap()` on an `Err` value: Transaction(Reverted { reason: "ContractNotFound", revert_id: 0, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: 265bda0cb99a723686a3db36b184e8b7282a6cc838420fec68fb74e082127ed5, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 10968, param1: 10480, param2: 10497, pc: 12408, is: 12408 }, Call { id: 265bda0cb99a723686a3db36b184e8b7282a6cc838420fec68fb74e082127ed5, to: 140a4d3150c6488998e120dbe24bafe9175a11be7a8e9c222ef93f0e59b0b7ad, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 6998, param1: 67107264, param2: 67106240, pc: 62657, is: 62657 }, ReturnData { id: 140a4d3150c6488998e120dbe24bafe9175a11be7a8e9c222ef93f0e59b0b7ad, ptr: 67103680, len: 28, digest: e5be52b1788a3651027e1eacefe4fe7047f12dcb153688b2ed85aa5e292f6ec6, pc: 64693, is: 62657, data: Some(00000000000000000000000900...) }, Panic { id: 265bda0cb99a723686a3db36b184e8b7282a6cc838420fec68fb74e082127ed5, reason: PanicInstruction { reason: ContractNotFound, instruction: CALL { target_struct: 0x10, fwd_coins: 0x0, asset_id_addr: 0x11, fwd_gas: 0x12 } (bytes: 2d 40 04 52) }, pc: 46752, is: 12408, contract_id: None }, ScriptResult { result: Panic, gas_used: 10654 }] })
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
test test_get_price_pyth_from_redstone ... FAILED

failures:

failures:
    test_get_price_pyth_from_redstone

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 2 filtered out; finished in 1.88s

error: test failed, to rerun pass `--test authorization`
```
