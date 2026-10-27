# Attackathon \_ Fuel Network 32979 - \[Smart Contract - Low] operations with StorageVec incorrectly rev

Submitted on Mon Jul 08 2024 18:21:13 GMT-0400 (Atlantic Standard Time) by @cyberthirst for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32979

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

Certain operations with `StorageVec` can revert. The type `StorageVec` has 0 size (due to optimization reasons when manipulating storage). The 0 size is, however, problematic because some functions in `lib-std` assume the invariant that types they operate on are non-zero. Because this invariant is broken, the functions revert.

## Vulnerability Details

The `StorageVec` is defined as `pub struct StorageVec<V> {}`, and the type has 0 size. Certain functions in `lib-std` like `load_vec` in `storage_vec.sw` take the size of the inner type via `let size_V_bytes = __size_of::<V>();` and perform further computations with the size.

So if we have eg a `StorageVec<StorageVec<u64>` then the inner type has 0 size. If we call `load_vec` on such nested storage vec, then the code [1](https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-lib-std/src/storage/storage_vec.sw/#L912) will revert:

```
 if size_V_bytes < 8 {
                    let len_bytes = len * size_V_bytes;
                    let new_vec = alloc_bytes(len_bytes);
                    let mut i = 0;
                    while i < len {
                        // The stored vec is offset with 1 word per element, remove the padding for elements less than the size of a word
                        // (size_of_word * element)
                        ptr
                            .add_uint_offset((8 * i))
                            .copy_bytes_to(new_vec.add::<V>(i), size_V_bytes);
                        i += 1;
                    }
```

This is because the type size is 0, and thus, we didn't allocate any memory (because `len_bytes` will evaluate to 0). But when the nested vec has a non-zero `len`, then `copy_bytes_to` will revert due to the access of unallocated memory.

## Impact Details

The PoC shows that if the nested `StorageVec` is used in a `withdraw` function, then the funds can be permanently blocked because the code would always revert.

However, using nested vectors is rather an unlikely pattern in smart contracts and as such we rate it with `low` impact.

## References

## Proof of concept

## Proof of Concept

The whole PoC is provided as a zip in Google Drive: <https://drive.google.com/file/d/1ExpSThwZ3jk5SrUM664wMCC\\_qo6v--cP/view?usp=sharing>

Unzip the archive and run `forc test`.

The PoC demonstrates that the operations on the nested storage vector incorrectly revert and might cause blocking of user funds:

```
contract;

use std::storage::storage_vec::*;

storage {
    storage_vec: StorageVec<StorageVec<b256>> = StorageVec {},
}

abi MyContract {
    #[storage(read, write)]
    fn init ();

    #[storage(read)]
    fn whitdraw() -> bool;
}

impl MyContract for Contract {

    #[storage(read, write)]
    fn init () {
        storage.storage_vec.push(StorageVec {});
    }

    #[storage(read)]
    fn whitdraw() -> bool {
        let v = storage.storage_vec.load_vec();
        // withdraw the user funds based on the vectors contents
        return true;
    } 
}


#[test]
fn test() {
    let caller = abi(MyContract, CONTRACT_ID);
    caller.init();

    assert(caller.whitdraw() == true);
}
```
