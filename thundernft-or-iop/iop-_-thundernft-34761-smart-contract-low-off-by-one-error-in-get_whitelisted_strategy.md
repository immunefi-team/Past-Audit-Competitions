# IOP \_ ThunderNFT 34761 - \[Smart Contract - Low] Off-by-one error in get\_whitelisted\_strategy

Submitted on Sat Aug 24 2024 10:33:51 GMT-0400 (Atlantic Standard Time) by @InquisitorScythe for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34761

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/execution\\_manager>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The code contains an off-by-one error in the index boundary check, which will deliver mistake revert code.

## Vulnerability Details

```rust
    /// Returns a whitelisted strategy at the index
    #[storage(read)]
    fn get_whitelisted_strategy(index: u64) -> Option<ContractId> {
        let len = storage.strategies.len();
        require(len != 0, ExecutionManagerErrors::ZeroLengthVec);
        require(index <= len, ExecutionManagerErrors::IndexOutOfBound);

        storage.strategies.get(index).unwrap().try_read()
    }
```

the second require checking should be index\<len instead of index<=len.

## Impact Details

While this bug does not result in direct financial losses or value reduction, it does impact the contract's ability to deliver its promised functionality accurately. This type of issue falls within the scope of "Failing to deliver promised returns" in terms of functional expectations, even if not in a financial sense. The severity is moderate, as it affects usability and reliability without compromising funds or causing direct losses.

## References

none

## Proof of concept

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
    let execution_manager = abi(ExecutionManager, execution_manager::CONTRACT_ID);
    execution_manager.initialize();

    execution_manager.add_strategy(ZERO_CONTRACT_ID);
    execution_manager.add_strategy(ContractId::from(0x0000000000000000000000000000000000000000000000000000000000000001));


    let cnt = execution_manager.get_whitelisted_strategy(3).unwrap();
    log(cnt);

}

#[test(should_revert="18446744073709486080")]
fn test2() {
    let execution_manager = abi(ExecutionManager, execution_manager::CONTRACT_ID);
    execution_manager.initialize();

    execution_manager.add_strategy(ZERO_CONTRACT_ID);
    execution_manager.add_strategy(ContractId::from(0x0000000000000000000000000000000000000000000000000000000000000001));


    let cnt = execution_manager.get_whitelisted_strategy(2).unwrap();
    log(cnt);

}
```

result:

```
tested -- test_contract

      test test1 ... ok (1.044537ms, 27069 gas)
[{"LogData":{"data":"0000000000000000bddc268719f6787cc71b7ccccf2ef91acce6c6e27404e1a1fa87684f824d17ff","digest":"534de284afb4ea471d255e0be03e2de718ebe38881daee6cabd076bd2d786b19","id":"3d3a7ea92ca438366572654a4770969bf874859aa16d5e6fbe46e21f4fe828c7","is":20600,"len":40,"pc":43192,"ptr":67103232,"ra":0,"rb":9517900813706399297}},{"LogData":{"data":"0000000000000005","digest":"5dee4dd60ff8d0ba9900fe91e90e0dcf65f0570d42c431f727d0300dd70dc431","id":"3d3a7ea92ca438366572654a4770969bf874859aa16d5e6fbe46e21f4fe828c7","is":21120,"len":8,"pc":33164,"ptr":67088000,"ra":0,"rb":10980104167348192313}}]
      test test2 ... FAILED (1.095737ms, 27346 gas)
[{"LogData":{"data":"0000000000000000bddc268719f6787cc71b7ccccf2ef91acce6c6e27404e1a1fa87684f824d17ff","digest":"534de284afb4ea471d255e0be03e2de718ebe38881daee6cabd076bd2d786b19","id":"3d3a7ea92ca438366572654a4770969bf874859aa16d5e6fbe46e21f4fe828c7","is":20600,"len":40,"pc":43192,"ptr":67103232,"ra":0,"rb":9517900813706399297}}]

   failures:
      - test test2, "/data/smart-contracts/contracts-v1/tests/src/main.sw":45 
        revert code: 0
        Logs: [{"LogData":{"data":"0000000000000000bddc268719f6787cc71b7ccccf2ef91acce6c6e27404e1a1fa87684f824d17ff","digest":"534de284afb4ea471d255e0be03e2de718ebe38881daee6cabd076bd2d786b19","id":"3d3a7ea92ca438366572654a4770969bf874859aa16d5e6fbe46e21f4fe828c7","is":20600,"len":40,"pc":43192,"ptr":67103232,"ra":0,"rb":9517900813706399297}}]



test result: FAILED. 1 passed; 1 failed; finished in 2.140274ms

tested -- NFT-contract


test result: OK. 0 passed; 0 failed; finished in 0ns
```

As you can see, test1 passed, but test2 failed. but they should share same revert code.
