# Attackathon \_ Fuel Network 33227 - \[Smart Contract - High] Lack of overflow protection in the pow fu

Submitted on Mon Jul 15 2024 12:05:53 GMT-0400 (Atlantic Standard Time) by @Schnilch for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33227

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

For the data types u8, u16, and u32, there is no overflow protection in the pow function implemented in the standard library. If the pow function is used to calculate coin amounts, it can result in the loss of coins.

## Vulnerability Details

For the data types `u8` `u16`, `u32`, `u64`, and `u256`, the pow function is implemented in the standard library in `math.sw`. However, the pow function for the data types `u8`, `u16`, and `u32` does not have overflow protections. An overflow of a `u64` and a `u256` is handled by the VM. Overflow protection for the other three data types would need to be implemented directly in the pow function in the standard library because the VM does not handle it. Due to the lack of these protections, the pow function of these data types are susceptible to overflow (see 1. reference). For `u16` and `u32`, the result would simply be larger than the data type as long as it still fits into a `u64`, since the VM converts `u16` and `u32` into u64's. For `u8`, there would be a simple wraparound, and once the result exceeds 255, it would start back at 0.

## Impact Details

This bug can have many different impacts based on what is calculated with the pow function. Especially if a coin amount is calculated with the pow function or if the result is further used to calculate a coin amount, it can easily lead to a loss of coins. Additionally, with the data types `u16` and `u32`, if the result exceeds the maximum of the data type, the value is still stored without wrapping. If this number is then returned and further used in the Rust SDK, it would wrap there, resulting in an incorrect value. This can again lead to a loss of coins if tokens are transfers based on this value.

## References

1. <https://github.com/FuelLabs/sway/blob/ebc2ee6bf5d488e0ff693bfc8680707d66cd5392/sway-lib-std/src/math.sw#L114-L139>

## Proof of concept

## Proof of Concept

For the PoC of this bug, a new fuel project with a Rust test suite is needed. This can be created with the following commands:

1. `forc new pow-bug`
2. `cd pow-bug`
3. `cargo generate --init fuellabs/sway templates/sway-test-rs --name pow-bug --force` Now the following code needs to be inserted into the `main.sw` file:

```rust
contract;

//Setup
use std::{
    asset::mint_to,
    constants::DEFAULT_SUB_ID
};

abi PowBug {
    fn demonstrate_bug_u8(recipient: Identity, a: u8) -> AssetId;
    fn demonstrate_bug_u32(a: u32) -> u32;
}

impl PowBug for Contract {
    fn demonstrate_bug_u8(recipient: Identity, a: u8) -> AssetId { //This function is used to demonstrate overflow of a u8 with the pow function.
        //Here is an example calculation on which coins are minted, which can lead to an 
        //incorrect amount of minted coins in case of an overflow in the pow function.
        let result_u8 = a.pow(2);
        let coins_to_mint = result_u8.as_u64() * 100;

        //This check is there to show that in the case of an overflow, the result is wrapped 
        //and not simply stored as an oversized number in the variable like with u32 or u16.
        if result_u8 > u8::max() {
            revert(1337);
        }

        mint_to(recipient, DEFAULT_SUB_ID, coins_to_mint); //The coins are minted here
        AssetId::new(ContractId::this(), DEFAULT_SUB_ID) //The AssetId of the minted coins is returned so that it doesn't have to be calculated again in the Rust SDK
    }

    fn demonstrate_bug_u32(a: u32)  -> u32{ //This function is used to demonstrate overflow of a u32 with the pow function.
        //This is simply another calculation using the pow function to demonstrate the bug in it. 
        //Based on this result, the Rust SDK transfers coins later.
        let coins_to_keep = a.pow(4);

        //This shows that with u32, in the case of overflow using the pow function, it doesn't wrap around, 
        //and the result is simply larger than the maximum of the data type.
        if coins_to_keep <= u32::max() {
            revert(1337)
        }

        coins_to_keep //The result is returned to the Rust SDK where it will be wrapped
    }
}
```

Now the following code needs to be inserted into `harness.rs`:

```rust
use std::str::FromStr;

use fuels::{
    prelude::*, 
    types::{
        ContractId,
        Identity,
        AssetId
    }
};

// Load abi from json
abigen!(Contract(
    name = "MyContract",
    abi = "out/debug/pow-bug-abi.json"
));

async fn get_contract_instance() -> (MyContract<WalletUnlocked>, ContractId) {
    // Launch a local network and deploy the contract
    let mut wallets = launch_custom_provider_and_get_wallets(
        WalletsConfig::new(
            Some(1),             /* Single wallet */
            Some(1),             /* Single coin (UTXO) */
            Some(10_000_000_000), /* Amount per coin */
        ),
        None,
        None,
    )
    .await
    .unwrap();
    let wallet = wallets.pop().unwrap();

    let id = Contract::load_from(
        "./out/debug/pow-bug.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();

    let instance = MyContract::new(id.clone(), wallet);

    (instance, id.into())
}

#[tokio::test]
async fn pow_bug() {
    //Setup
    let (instance, _id) = get_contract_instance().await;

    //This part of the code demonstrates overflow in u8 when the number 20 is passed as a parameter. 
    //In the smart contract, 20^2 is calculated, resulting in 400, which should cause an overflow. 
    //However, since there is no overflow protection, the incorrect result is multiplied by 100, 
    //minting an incorrect amount of coins.
    println!("------------u8 pow function overflow----------------");
    let asset_id = instance.methods().demonstrate_bug_u8(Identity::Address(instance.account().address().into()), 20)
        .append_variable_outputs(1)
        .call()
        .await
        .unwrap()
        .value;
    
    let balance = instance.account().get_asset_balance(&asset_id).await.unwrap();
    println!("balance: {:#?}", balance); //In the console, you can see that the user received some incorrect amount of coins due to not reverting during the overflow

    //This part of the code demonstrates the overflow with a u32 using the pow function. The parameter with 1000 
    //is raised to the power of four in the contract, resulting in a number larger than the maximum u32. However, 
    //since there is no overflow protection, the too-large number returned by the contract is wrapped here in the 
    //rust sdk and is now too small which leads to the user keeping fewer coins than he should.
    println!("------------u32 pow function overflow----------------");
    let coins_to_keep = instance.methods().demonstrate_bug_u32(1000).call().await.unwrap().value;
    println!("coins_to_keep: {:#?}", coins_to_keep);
    
    let coins_before = instance.account().get_coins(AssetId::zeroed()).await.unwrap()[0].amount;
    println!("coins_before: {:#?}", coins_before); //This shows how many coins the user had before the transfer

    instance.account().transfer(
        &Bech32Address::from_str("fuel1glsm9rc8ysh9yjt8ljkuatalvdad3rs3wpqjznd3p7daydw2gg6sftwvvr").unwrap(), //Just an address to which the coins will be sent
        coins_before - TryInto::<u64>::try_into(coins_to_keep).unwrap(), //Only the amount of coins that the contract returned is retained, which, due to the overflow in this case, results in too many coins being sent and too few being retained.
        AssetId::zeroed(),
        TxPolicies::default()
    ).await.unwrap();

    let coins_after = instance.account().get_coins(AssetId::zeroed()).await.unwrap()[0].amount;
    println!("coins after: {:#?}", coins_after); //That shows that really only the amount of coins was retained that the contract returned as value.
}
```

The PoC can be started with the following command: `cargo test pow_bug -- --nocapture`

In the PoC, only the overflow of u8 and u32 is shown because the overflow behavior of u16 is certainly the same as that of u32.
