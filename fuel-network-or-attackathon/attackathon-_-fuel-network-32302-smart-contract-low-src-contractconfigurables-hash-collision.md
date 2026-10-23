# Attackathon \_ Fuel Network 32302 - \[Smart Contract - Low] Src ContractConfigurables hash collision

Submitted on Mon Jun 17 2024 17:31:27 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32302

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway-standards/tree/348f7175df4c012b23c86cdb18aab79025ca1f18>

Impacts:

* Incorrect library function behaviors

## Description

## Brief/Intro

`src12` `Hash` trait implementation for `ContractConfigurables` allows collision between different configurations.

## Vulnerability Details

`ContractConfigurables` is a vector of `(offset: u64, data: Vec<u8>)` and represents the interchangeable config for contracts. The `hash` method is often used to generate a single identity value that could be used to track or retrieve the configurations or contract from storage. An example of this can be found in the example [contracts](https://github.com/FuelLabs/sway-standards/blob/2206de75bdb5fcabfc90d33dab17878a4d7efe2f/examples/src12-contract-factory/with_configurables/src/with_configurables.sw#L187).

However, there are a few flaws in the hash implementation.

1. Due to the `u64` offset casted as a `raw_ptr` in the `asm` block, we're doing an arbitrary memory read on `offset_ptr.copy_bytes_to(buffer, 4);`. This is clearly incorrect because `offset` should be a pointer into the bytecode, and not the current vm memory. Hashing based on the data in vm memory doesn't make sense here.
2. If we fix the type casting and correctly use address storing offset instead of treating offset as a pointer, the hash still suffers from collision because only the first 4 bytes of the `u64` offset is included. Fuel data is stored in big endian, which means the offset should never execeed `u32::max`, and the first 4 bytes used in hashing will always be 0.
3. The encoding of the `Vec<u8>` data also allows for more collisions. Assuming both incorrect casting and offset length are fixed, the two configs `[(0, [0]), (10, [0, 0, 0, 0, 0, 0, 0, 11, 0])]` and `[(0, [0, 0, 0, 0, 0, 0, 0, 0, 10]), (11, [0])]` will still have the same hash.
4. Without sorting the configuration entries, functionally equivalent configurations with entry orders swapped could have different hashes.

```
impl Hash for ContractConfigurables {
    fn hash(self, ref mut state: Hasher) {
        // Iterate over every configurable
        let mut configurable_iterator = 0;
        while configurable_iterator < self.len() {
            let (offset, data) = self.get(configurable_iterator).unwrap();
            let buffer = alloc_bytes(data.len() + 4);
            let offset_ptr = asm(input: offset) {
                input: raw_ptr
            };

            // Overwrite the configurable data into the buffer
            offset_ptr.copy_bytes_to(buffer, 4);
            data.ptr().copy_bytes_to(buffer.add::<u8>(4), data.len());

            state.write(Bytes::from(raw_slice::from_parts::<u8>(buffer, data.len() + 4)));
            configurable_iterator += 1;
        }
    }
}
```

## Impact Details

Depending on usage, this may lead to out-of-bound reads, hash collision for different configs, or multiple hashes for functionally identical configs. If `ContractConfigurables` is used as the key in factory contracts, it might allow attacks to overwrite entries for other contracts. The exact impact depends on functionality of contracts involved.

## References

* `https://github.com/FuelLabs/sway-standards/blob/87b39dfcd2a0fcad9d7092b0c9937ae66e3f9299/standards/src/src12.sw#L110`
* `https://github.com/FuelLabs/sway-standards/blob/2206de75bdb5fcabfc90d33dab17878a4d7efe2f/examples/src12-contract-factory/with_configurables/src/with_configurables.sw#L99`

## Proof of concept

## Proof of Concept

This PoC demonstrates that offset is incorrectly treated as a pointer, and could easily have hash collisions or different hashes for the same content.

```
#[test]
fn src12_contractconfigurables_hash_collision() -> () {
    let reserved_buf_ptr = alloc_bytes(8);
    let reserved_buf_ptr_int = asm(ptr: reserved_buf_ptr) {
        ptr: u64
    };
    let mut configurables1 = ContractConfigurables::new();
    let mut configurables2 = ContractConfigurables::new();
    let mut data1 = Vec::<u8>::new();
    let mut data2 = Vec::<u8>::new();
    configurables1.push((reserved_buf_ptr_int, data1));
    configurables2.push((reserved_buf_ptr_int + 4, data2));

    reserved_buf_ptr.write::<u64>(1311768465173141112);             //0x1234567812345678
    let hash1 = sha256(configurables1);
    let hash2 = sha256(configurables2);
    assert(hash1 == hash2);

    reserved_buf_ptr.write::<u64>(1311768465173141113);             //0x1234567812345679
    let hash1 = sha256(configurables1);
    let hash2 = sha256(configurables2);
    assert(hash1 != hash2);
    ()
}
```
