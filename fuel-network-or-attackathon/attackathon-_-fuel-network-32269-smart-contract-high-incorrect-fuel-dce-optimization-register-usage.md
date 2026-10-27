# Attackathon \_ Fuel Network 32269 - \[Smart Contract - High] Incorrect fuel dce optimization register

Submitted on Mon Jun 17 2024 08:24:27 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32269

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway optimization leading to incorrect bytecode

## Description

## Brief/Intro

The sway compiler wrongly modeled register usage of `WQAM` instruction in `def_registers`, which makes the compiler emit wrong code during optimization pass. An attacker may leverage that vulnerability to manipulate pointer to point to an arbitrary address and potentially gain arbitrary write in callee memory.

## Vulnerability Details

The `def_registers` [function](https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_lang/virtual_ops.rs#L658) and `use_registers` [function](https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_lang/virtual_ops.rs#L541) are used to define which registers in the argument will be written to or read from by the given instruction. This information is be used by `dce` [optimization](https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_generation/fuel/optimizations.rs#L212) to decide which instructions only write to "dead" registers and could be removed. In the extracted snippet shown below, we can see that `WQAM` is incorrectly thought to modify `r1` while not relying on its value, while the actual behavior using `r1` as a memory pointer.

```
pub(crate) fn use_registers(&self) -> BTreeSet<&VirtualRegister> {
    use VirtualOp::*;
    (match self {
        ...
        WQAM(_, r2, r3, r4) => vec![r2, r3, r4],
        ...
    })
    .into_iter()
    .collect()
}

pub(crate) fn def_registers(&self) -> BTreeSet<&VirtualRegister> {
    use VirtualOp::*;
    (match self {
        ...
        WQAM(r1, _, _, _) => vec![r1],
        ...
    })
    .into_iter()
    .collect()
}
```

During `dce` optimization pass, sway compiler eliminates any instructions that write to registers without any dependency. The incorrect `WQAM` modelling creates a chance where the compiler may mistake an actual useful instruction for an useless one.

To show what this means, we look at the test case below. We write a simple script that emits the `WQAM` instruction. The script is then translates into abstract instructions before going through the optimization passes. Notably, the abstract instructions before dce includes an `addi $r6 $$locbase i400` instruction before `WQAM`, which is responsible for setting up the pointer for `WQAM` output buffer.

However, after dce optimization, this instruction is removed, since the compiler thinks that `wqam` writes to `$r6`, and thus the result of `addi` is never used and could be considered "dead". This results in the compiled program never initializing the wqam output buffer ptr, and ends up using the left-over value within the register as the output destination.

Source Script

```
fn main() -> u256 {
    let c : u256 = 1;
    c % c
}
```

Abstract instructions before dce

```
;; --- Entries ---
.program:
.0                                      ; --- start of function: __entry ---
move $$locbase $sp                      ; save locals base register for __entry
cfei i520                               ; allocate 520 bytes for locals and 0 slots for call arguments.
load $$tmp data_1                       ; load initializer from data section
addi $r1 $$locbase i432                 ; calc local variable address
mcpi $r1 $$tmp i32                      ; copy initializer from data section to local variable
.2
addi $r2 $$locbase i240                 ; get offset to local
load $r3 data_0                         ; get local constant
addi $r4 $$locbase i432                 ; get offset to local
load $r5 data_0                         ; get local constant
addi $r6 $$locbase i400                 ; get offset to local
wqam $r6 $r3 $r4 $r5
...
```

Abstract instructions after dce

```
.program:
.0                                      ; --- start of function: __entry ---
move $$locbase $sp                      ; save locals base register for __entry
cfei i520                               ; allocate 520 bytes for locals and 0 slots for call arguments.
load $$tmp data_1                       ; load initializer from data section
addi $r1 $$locbase i432                 ; calc local variable address
mcpi $r1 $$tmp i32                      ; copy initializer from data section to local variable
.2
addi $r2 $$locbase i240                 ; get offset to local
load $r3 data_0                         ; get local constant
addi $r4 $$locbase i432                 ; get offset to local
load $r5 data_0                         ; get local constant
wqam $r6 $r3 $r4 $r5
...
```

## Impact Details

In the best case scenario, this would lead to unexpected failures of the script / contract due to an illegal write. It the worst case scenario, the uninitialized register and `wqam` calculation result could be controllable by caller, which would lead to a 32 byte arbitrary write within the memory space of callee contract.

## References

* `https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_lang/virtual_ops.rs#L580`
* `https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_lang/virtual_ops.rs#L697`
* `https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_generation/fuel/optimizations.rs#L247`

## Proof of concept

## Proof of Concept

This minimal sway test panics. We don't further demonstrate arbitrary write since that should be obvious from the details above. We can provide a PoC later is necessary.

```
#[test]
fn incorrect_def_modeling() -> u256 {
    let c: u256 = 1;
    c % c  // this emits a WQAM instruction
}
```
