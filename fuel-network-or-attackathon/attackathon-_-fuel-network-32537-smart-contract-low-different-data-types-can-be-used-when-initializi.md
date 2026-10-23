# Attackathon \_ Fuel Network 32537 - \[Smart Contract - Low] Different data types can be used when init

Submitted on Tue Jun 25 2024 18:46:46 GMT-0400 (Atlantic Standard Time) by @Schnilch for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32537

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Compiler bug can lead to unintended behavior in Rust SDK or sway smart contracts.
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In Sway, arrays normally can only have one data type. However, due to a bug during array initialization, different numerical data types can be used. This can result in larger numbers appearing in the array than should be possible. When such a number is returned from a function, it may overflow in the Rust SDK and consequently be handled based on an incorrect value.

## Vulnerability Details

In Sway, when an array is initialized without specifying the exact type, it is possible that the elements may be a mixture of the data types u16, u32, and u64. This is because during type checking, there is no verification that all values in an array have the same data type. The data type of the elements of the array is simply determined by the data type of the first element in the array, without checking what data types the other elements have (See 1. Reference). The bug is caught for all data types except u16, u32, and u64 because these data types are all converted into a u64 in the ir . This occurs when the IR is verified with the function verify\_store, where the data type of each element in the array is checked again (See 2. Reference).\
The problem when these three different data types occur in one array is that all elements in the array, after initialization, assume the data type of the first element. So, if the first element is, for example, a u16 and all others are u64s, u16 values larger than u16::max() can occur in the array. There isn't an overflow, but the numbers in the array are simply larger than they should be. This leads to further issues, especially when these too large u16 values are returned and used further in the Rust SDK, where they would then overflow.

## Impact Details

This bug can have various impacts depending on what the array is used for. Here is one scenario in which this bug can lead to loss of funds:

* In a Sway smart contract, during the initialization of an array, the first element is set to a number of type u16, and the second element is assigned a number representing a token amount of type u64. If now the second element is returned from the function using Rust SDK, and based on this value a transaction is executed, it results in an incorrect token amount, potentially causing a user to lose tokens.

## References

1. <https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/semantic\\_analysis/ast\\_node/expression/typed\\_expression.rs#L1833>
2. <https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-ir/src/verify.rs#L1024-L1032>

## Proof of concept

## Proof of Concept

### Setup

To execute the PoC, a new Sway project must first be created and a test template generated with Rust. The following commands can be used for this:

1. forc new array-init-bug
2. cd array-init-bug
3. cargo generate --init fuellabs/sway templates/sway-test-rs --name array-init-bug --force

In src/main.sw, the following code must now be added:

```sway
contract;

abi MyContract {
    fn test_function(num: u64) -> u16;
}

impl MyContract for Contract {
    fn test_function(num: u64) -> u16 {
        let a = [1u16, num]; //array with 2 different data types is initialized
        let b: u16 = a[1]; //all elements of the array now have the data type u16, including the element that was previously a u64
        if b <= u16::max() { //This statement should actually always be true and revert, but since the number stored in b was once in u64, b can be larger than u16::max() due to the bug, even though it is only a u16 now
            revert(1);
        }
        b //b is returned as u16
    }
}
```

The code is intended to demonstrate the bug in a Sway smart contract. To test the contract, the following test can be inserted into the file tests/harness.rs:

```rust
#[tokio::test]
async fn array_init_test() {
    let (instance, _) = get_contract_instance().await; //The contract is deployed
    
    let result = instance.methods().test_function(u64::from(u16::MAX) + 2000).call().await.unwrap(); //Here, the smart contract is called with a value greater than the max u16, which should actually lead to an error since this variable is later stored in a u16
    assert!(result.value == 1999); //This shows that the too large u16 returned by the contract is now overflowed and results in a different number.
    println!("returned value: {:#?}", result.value);

    let remaining_coins = u64::from(result.value); //As an example, it is assumed here that the number returned by the contract is used to specify how many coins the user should keep to demonstrate a loss of funds
    let coins_before = instance.account().get_coins(AssetId::zeroed()).await.unwrap()[0].amount; //This should be 1_000_000_000 as specified when the test was set up
    println!("coins_before: {:#?}", coins_before);

    instance.account().transfer(
        &Bech32Address::from_str("fuel1glsm9rc8ysh9yjt8ljkuatalvdad3rs3wpqjznd3p7daydw2gg6sftwvvr").unwrap(), //Just an address to which the coins will be sent
        coins_before - remaining_coins, //Here remaining_coins is smaller than it should be which leads to a loss of coins because the amount to send is then too large
        AssetId::zeroed(), //Default asset id
        TxPolicies::default() //Default TxPolicies
    ).await.unwrap();

    let coins_after = instance.account().get_coins(AssetId::zeroed()).await.unwrap()[0].amount;
    println!("coins: {:#?}", coins_after); 
    assert!(coins_after == 1999); //This shows that the user has now transferred too many coins
}
```

### Run the PoC

To execute the POC, the following commands must now be run:

* forc build
* cargo test array\_init\_test -- --nocapture
