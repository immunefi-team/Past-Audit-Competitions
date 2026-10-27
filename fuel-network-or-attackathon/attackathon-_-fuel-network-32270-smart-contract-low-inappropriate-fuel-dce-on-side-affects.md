# Attackathon \_ Fuel Network 32270 - \[Smart Contract - Low] Inappropriate fuel dce on side affects

Submitted on Mon Jun 17 2024 08:24:54 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32270

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway optimization leading to unexpected bytecode

## Description

## Brief/Intro

The sway compiler may remove expressions that potentially has side affects during dce pass, and eventually change behavior of program.

## Vulnerability Details

The `has_side_effect` [function](https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_lang/virtual_ops.rs#L328) is used to check whether an operation may have untrackable effects. Any operation with untrackable effect should never be removed by `dce` [optimization](https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_generation/fuel/optimizations.rs#L239). However, the function doesn't `has_side_effect` function doesn't correctly consider that all arithmetic opcodes may panic on illegal operands or overflows.

```
pub(crate) fn has_side_effect(&self) -> bool {
    use VirtualOp::*;
    (match self {
        ...
        DIV(_, _, _) => self.def_registers().iter().any(|vreg| matches!(vreg, VirtualRegister::Constant(_))),
        ...
    })
    .into_iter()
    .collect()
}
```

Thus if `dce` considers the output of an arithmetic instruction to be "dead", it would remove the instruction regardless of whether a panic may happen due to calculation.

To show what this means, we look at the test case below. Since the result of `let c = 1 / 0` is never used, `dce` removed the `div $r70 $one $zero` instruction, changing the behavior of the program.

Source Script

```
fn main() -> () {
    let c = 1 / 0;
}
```

Abstract instructions before dce

```
.program:
.2                                      ; --- start of function: strange_panic ---
move $$locbase $sp                      ; save locals base register for strange_panic
cfei i0                                 ; allocate 0 bytes for locals and 0 slots for call arguments.
.5
div $r70 $one $zero
ret $zero                               ; returning unit as zero
```

Abstract instructions after dce

```
.program:
.2                                      ; --- start of function: strange_panic ---
move $$locbase $sp                      ; save locals base register for strange_panic
cfei i0                                 ; allocate 0 bytes for locals and 0 slots for call arguments.
.5
ret $zero                               ; returning unit as zero
```

## Impact Details

This would potentially change the behavior of programs. The example provided is relatively harmless, but there could be cases where developers rely on a `let c = a + b` panic to check whether `a + b` overflows, but otherwise leaves `c` unused. The incorrect elimination of such instructions would result in removal of such checks.

## References

* `https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_lang/virtual_ops.rs#L328`
* `https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_generation/fuel/optimizations.rs#L239`

## Proof of concept

## Proof of Concept

```
#[test]
fn incorrect_neglect_of_side_effects() -> () {
    let c = 1 / 0;
}
```
