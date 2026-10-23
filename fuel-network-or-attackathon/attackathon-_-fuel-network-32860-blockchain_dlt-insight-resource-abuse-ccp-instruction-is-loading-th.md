# Attackathon \_ Fuel Network 32860 - \[Blockchain\_DLT - Insight] Resource Abuse CCP instruction is load

Submitted on Fri Jul 05 2024 06:27:13 GMT-0400 (Atlantic Standard Time) by @ret2happy for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32860

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-vm/tree/0e46d324da460f2db8bcef51920fb9246ac2143b>

Impacts:

* Temporary freezing of network transactions by delaying one block by 3000% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments
* Causing network processing nodes to process transactions from the mempool beyond set parameters (e.g. prevents processing transactions from the mempool)
* Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

CCP instruction executes the contract loading operation before charging GAS. The attacker could perform resource-costing operation for contract loading with low GAS. Such undercharge instruction would cause additional burden of the validators, increasing node pressure.

## Vulnerability Details

In the `CCP` instruction, it loaded the contract before charing the dependent GAS as `fuel-vm/src/interpreter/blockchain.rs#L792-L820` shows: \[1]

```
let contract = super::contract::contract(self.storage, &contract_id)?; // load contract before charging
let contract_bytes = contract.as_ref().as_ref();
let contract_len = contract_bytes.len();
let charge_len = core::cmp::max(contract_len as u64, length);
let profiler = ProfileGas {
    pc: self.pc.as_ref(),
    is: self.is,
    current_contract: self.current_contract,
    profiler: self.profiler,
};
dependent_gas_charge_without_base(
    self.cgas,
    self.ggas,
    profiler,
    self.gas_cost,
    charge_len,
)?;

// Owner checks already performed above
copy_from_slice_zero_fill(
    self.memory,
    self.owner,
    contract.as_ref().as_ref(),
    dst_addr,
    offset as usize,
    length,
)?;
```

If the contract is very large, it would be resources consuming without charing the large dependent GAS. We should firstly get the contract size using the storage API i.e., `contract_size` and charge the dependent GAS before load the contract.

## Impact Details

Attacker is allowed to use CCP instruction to load very large contract with very cheap GAS. It would lead to the network stuck or cease the node processing.

## References

\[1] <https://github.com/FuelLabs/fuel-vm/blob/2604237c9ff4a755e48b40b2c006711d22cff19f/fuel-vm/src/interpreter/blockchain.rs#L792-L820>

## Proof of concept

## Proof of Concept

We could simply debug the CCP instruction using a test program:

```
#[test]
fn code_copy_b_gt_vm_max_ram() {
    let reg_a = 0x20;
    // test overflow add
    let code_copy = vec![
        op::slli(reg_a, RegId::ONE, MAX_MEM_SHL),
        op::subi(reg_a, reg_a, 31),
        op::ccp(RegId::ZERO, reg_a, RegId::ZERO, RegId::ZERO),
    ];

    check_expected_reason_for_instructions(code_copy, MemoryOverflow);
}
```

We could check whether the GAS is charged after the contract is loaded in the vulnerability details \[1].
