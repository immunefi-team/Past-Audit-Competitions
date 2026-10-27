# IOP \_ ThunderNFT 34760 - \[Smart Contract - Low] Off-by-one error in get\_supported\_asset

Submitted on Sat Aug 24 2024 10:22:44 GMT-0400 (Atlantic Standard Time) by @InquisitorScythe for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34760

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/asset\\_manager>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The code contains an off-by-one error in the index boundary check, which will deliver mistake revert code.

## Vulnerability Details

```rust
    #[storage(read)]
    fn get_supported_asset(index: u64) -> Option<AssetId> {
        let len = storage.assets.len();
        require(len != 0, AssetManagerErrors::ZeroLengthVec);
        require(index <= len, AssetManagerErrors::IndexOutOfBound);

        storage.assets.get(index).unwrap().try_read()
    }
```

the second require checking should be `index<len` instead of `index<=len`.

## Impact Details

While this bug does not result in direct financial losses or value reduction, it does impact the contract's ability to deliver its promised functionality accurately. This type of issue falls within the scope of "Failing to deliver promised returns" in terms of functional expectations, even if not in a financial sense. The severity is moderate, as it affects usability and reliability without compromising funds or causing direct losses.

## References

none

## Proof of concept

## Proof of Concept

```
contract;

 use interfaces::{
    thunder_exchange_interface::{ThunderExchange},
    royalty_manager_interface::*,
    asset_manager_interface::*,
    execution_manager_interface::ExecutionManager,
    execution_strategy_interface::*,
    pool_interface::Pool,
};

use libraries::{
    msg_sender_address::*,
    constants::*,
    order_types::*,
    ownable::*,
};

use std::{
    block::timestamp,
    auth::*,
    call_frames::*,
    context::*,
    contract_id::ContractId,
    logging::log,
    revert::require,
    storage::storage_map::*,
    asset::*
};

#[test(should_revert="18446744073709486080")]
fn test1() {
    let asset_mngr = abi(AssetManager, asset_manager::CONTRACT_ID);
    asset_mngr.initialize();
    asset_mngr.add_asset(ZERO_ASSET_ID);
    asset_mngr.add_asset(AssetId::from(0x0000000000000000000000000000000000000000000000000000000000000001));

    let cnt = asset_mngr.get_count_supported_assets();
    log(cnt);

    let assetid = asset_mngr.get_supported_asset(3).unwrap();
    log(assetid);
}

#[test(should_revert="18446744073709486080")]
fn test2() {
    let asset_mngr = abi(AssetManager, asset_manager::CONTRACT_ID);
    asset_mngr.initialize();
    asset_mngr.add_asset(ZERO_ASSET_ID);
    asset_mngr.add_asset(AssetId::from(0x0000000000000000000000000000000000000000000000000000000000000001));

    let cnt = asset_mngr.get_count_supported_assets();
    log(cnt);

    let assetid = asset_mngr.get_supported_asset(2).unwrap();
    log(assetid);
}

```

result:

```
tested -- test_contract

      test test1 ... ok (1.055128ms, 29791 gas)
[{"LogData":{"data":"0000000000000000bddc268719f6787cc71b7ccccf2ef91acce6c6e27404e1a1fa87684f824d17ff","digest":"534de284afb4ea471d255e0be03e2de718ebe38881daee6cabd076bd2d786b19","id":"d6a10379a8e9b4aeb0b9646dd5c8cccd157532db227a1b2b8aeb18c5a512d6da","is":21760,"len":40,"pc":44720,"ptr":67103232,"ra":0,"rb":9517900813706399297}},{"LogData":{"data":"0000000000000002","digest":"cd04a4754498e06db5a13c5f371f1f04ff6d2470f24aa9bd886540e5dce77f70","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":8,"pc":16252,"ptr":67086976,"ra":0,"rb":1515152261580153489}},{"LogData":{"data":"0000000000000005","digest":"5dee4dd60ff8d0ba9900fe91e90e0dcf65f0570d42c431f727d0300dd70dc431","id":"d6a10379a8e9b4aeb0b9646dd5c8cccd157532db227a1b2b8aeb18c5a512d6da","is":22392,"len":8,"pc":34940,"ptr":67082624,"ra":0,"rb":8518707422325009122}}]
      test test2 ... FAILED (1.074996ms, 30068 gas)
[{"LogData":{"data":"0000000000000000bddc268719f6787cc71b7ccccf2ef91acce6c6e27404e1a1fa87684f824d17ff","digest":"534de284afb4ea471d255e0be03e2de718ebe38881daee6cabd076bd2d786b19","id":"d6a10379a8e9b4aeb0b9646dd5c8cccd157532db227a1b2b8aeb18c5a512d6da","is":21760,"len":40,"pc":44720,"ptr":67103232,"ra":0,"rb":9517900813706399297}},{"LogData":{"data":"0000000000000002","digest":"cd04a4754498e06db5a13c5f371f1f04ff6d2470f24aa9bd886540e5dce77f70","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":8,"pc":16252,"ptr":67086976,"ra":0,"rb":1515152261580153489}}]

   failures:
      - test test2, "/data/smart-contracts/contracts-v1/tests/src/main.sw":45 
        revert code: 0
        Logs: [{"LogData":{"data":"0000000000000000bddc268719f6787cc71b7ccccf2ef91acce6c6e27404e1a1fa87684f824d17ff","digest":"534de284afb4ea471d255e0be03e2de718ebe38881daee6cabd076bd2d786b19","id":"d6a10379a8e9b4aeb0b9646dd5c8cccd157532db227a1b2b8aeb18c5a512d6da","is":21760,"len":40,"pc":44720,"ptr":67103232,"ra":0,"rb":9517900813706399297}},{"LogData":{"data":"0000000000000002","digest":"cd04a4754498e06db5a13c5f371f1f04ff6d2470f24aa9bd886540e5dce77f70","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":8,"pc":16252,"ptr":67086976,"ra":0,"rb":1515152261580153489}}]



test result: FAILED. 1 passed; 1 failed; finished in 2.130124ms
```

As you can see, test1 passed, but test2 failed. but they should share same revert code.
