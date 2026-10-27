# Attackathon \_ Fuel Network 32390 - \[Smart Contract - Low] Unchecked Virtual Immediate Construction O

Submitted on Thu Jun 20 2024 04:55:58 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32390

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway compilation leading to incorrect bytecode

## Description

## Brief/Intro

Directly constructing `VirtualImmediates` without using the constructor functions with checks causes silent truncation of immediates and incorrect program behavior.

## Vulnerability Details

`VirtualImmediates` are constructed when converting `ir` to `Ops`. The `ir` does not impose limits on immediates, so the values must be checked at this stage. Normally `VirtualImmediates` should be constructed through the `VirtualImmediate::new` constructors, which checks that the provided immediate is within the allowed range. However, there are some instances which directly builds the `VirtualImmediate`.

For example, `compile_mem_copy_bytes` casts a `u64` `bytes_len` to `u32`, then builds a `VirtualImmediate18` from it. If the `bytes_len` does not fit in 18 bits, the compiler would later silently truncate its value, causing the `MCP` to copy fewer bytes than it should.

```
fn compile_mem_copy_bytes(
    &mut self,
    instr_val: &Value,
    dst_val_ptr: &Value,
    src_val_ptr: &Value,
    byte_len: u64,
) -> Result<(), CompileError> {
    if byte_len == 0 {
        // A zero length MCP will revert.
        return Ok(());
    }

    let owning_span = self.md_mgr.val_to_span(self.context, *instr_val);

    let dst_reg = self.value_to_register(dst_val_ptr)?;
    let src_reg = self.value_to_register(src_val_ptr)?;

    let len_reg = self.reg_seqr.next();
    self.cur_bytecode.push(Op {
        opcode: Either::Left(VirtualOp::MOVI(
            len_reg.clone(),
            VirtualImmediate18 {
                value: byte_len as u32,
            },
        )),
        comment: "get length for mcp".into(),
        owning_span: owning_span.clone(),
    });

    self.cur_bytecode.push(Op {
        opcode: Either::Left(VirtualOp::MCP(dst_reg, src_reg, len_reg)),
        comment: "copy memory with mem_copy".into(),
        owning_span,
    });

    Ok(())
}
```

## Impact Details

As usual, it is hard to precisely estimate the impact of incorrect code generation because it depends on what code the user writes. The best case scenario would be contracts that run into those bugs getting bricked, and the worst case scenario would be loss of funds due to incorrect program logic.

## References

These are the places where I think the `VirtualImmediates` could overflow. There are more raw construction of `VirtualImmediates` in the compiler, but most immediates are checked right before `VirtualImmediates` are built from it, so they are not affected.

* `https://github.com/FuelLabs/sway/blob/6e9130bbf22da7971fd5178c2a4167dc46c43b20/sway-core/src/asm_generation/fuel/fuel_asm_builder.rs#L138`
* `https://github.com/FuelLabs/sway/blob/6e9130bbf22da7971fd5178c2a4167dc46c43b20/sway-core/src/asm_generation/fuel/fuel_asm_builder.rs#L150`
* `https://github.com/FuelLabs/sway/blob/6e9130bbf22da7971fd5178c2a4167dc46c43b20/sway-core/src/asm_generation/fuel/fuel_asm_builder.rs#L1312`
* `https://github.com/FuelLabs/sway/blob/6e9130bbf22da7971fd5178c2a4167dc46c43b20/sway-core/src/asm_generation/fuel/fuel_asm_builder.rs#L1433`

## Proof of concept

This test would fail because `MCP` length is truncated, and the slice is not properly copied.

```
#[test]
fn immediate_truncation() -> () {
    let mut a = [0u8; 0x40000];
    a[0x3ffff] = 1;
    let b = a;
    assert(a[0x3ffff] == 1);
    assert(b[0x3ffff] == 1);
    ()
}
```
