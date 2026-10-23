# Attackathon \_ Fuel Network 32491 - \[Smart Contract - Low] Incorrect PushA PopA Mask Calculation

Submitted on Sun Jun 23 2024 22:48:44 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32491

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway compilation leading to incorrect bytecode

## Description

## Brief/Intro

`emit_pusha_popa` only handles the first `def_reg` of each instruction, and could lead to corruption of register in function caller.

## Vulnerability Details

When calling a function, we need to store caller registers onto stack and restore it later, so that it doesn't get overwritten by the callee. The compiler does a small optimization by storing only the registers that callee modifies to reduce the amount of stack memory writes.

However, the compiler incorrectly assumes each instruction only modifies one register, so if there are more than one register modified, the remaining registers will not be pushed and popped from stack. Without a push and pop for modified registers, caller register can be modified unexpectedly and cause incorrect execution result.

```
let reg = match &op.opcode {
    Either::Right(ControlFlowOp::PushAll(label)) => {
        active_sets.insert(*label);
        None
    }
    Either::Right(ControlFlowOp::PopAll(label)) => {
        active_sets.swap_remove(label);
        None
    }

    Either::Left(alloc_op) => alloc_op.def_registers().into_iter().next(),
    Either::Right(ctrl_op) => ctrl_op.def_registers().into_iter().next(),
};

if let Some(reg) = reg {
    for active_label in active_sets.clone() {
        reg_sets
            .entry(active_label)
            .and_modify(|regs: &mut BTreeSet<AllocatedRegister>| {
                regs.insert(reg.clone());
            })
            .or_insert_with(|| {
                BTreeSet::from_iter(std::iter::once(reg).cloned())
            });
    }
}

```

For example, we compile this code

```
#[storage(read)]
fn setup() -> () {
    let a: u64 = 1;
    let b: u64 = 1;
    let c: u64 = 1;
    //call a few times to avoid inline
    store_read();
    let r = asm(r, a: a, b: b, c: c, d: store_read()) {
        movi r i0;
        add r a b;	// r = a + b = 2
        add r r c;	// r = a + b + c = 3        c value is overwritten by store_read, so we get 2 instead
        add r r d;	// r = a + b + c + d = 3	d returns 0
        r
    };
    assert(r == 3);
    ()
}

#[storage(read)]
fn store_read() -> u64 {
    let a = asm(slot, a, b, c) {
        movi c i32;
        aloc c;
        move slot hp;
        srw a b slot;	// somehow make b allocate to $r3
        movi c i0;
        add a a slot;
        sub a a slot;
        add a a b;
        add a a c;
        a
    };
    a - a	// return 0 and make sure a is not dced
}
```

And the allocated abstract instruction after compiling the code is this

```
.program:
.14                                     ; --- start of function: setup_23 ---
pshl i31                                ; Save registers 16..40
pshh i524288                            ; Save registers 40..64
move $$locbase $sp                      ; save locals base register for setup_23
cfei i32                                ; allocate 32 bytes for locals and 0 slots for call arguments.
move $r0 $$reta                         ; save reta
.56
sw   $$locbase $one i0                  ; store value
sw   $$locbase $one i1                  ; store value
sw   $$locbase $one i2                  ; store value
mova $$reta .57                         ; set new return addr
fncall .16                              ; call store_read_24
.57
lw   $r1 $$locbase i0                   ; load value
lw   $r2 $$locbase i1                   ; load value
lw   $r3 $$locbase i2                   ; load value                        // use $r3 to hold value of c
mova $$reta .58                         ; set new return addr
fncall .16                              ; call store_read_24                // the call unexpectedly modifies $r3
.58
move $r4 $$retv                         ; copy the return value
add  $r1 $r1 $r2                        ; add r a b
add  $r1 $r1 $r3                        ; add r r c                         // use the modified $r3 instead of correct one
add  $r1 $r1 $r4                        ; add r r d
sw   $$locbase $r1 i3                   ; store value
lw   $r1 $$locbase i3                   ; load value
movi $r2 i3                             ; initializer constant into register
eq   $r1 $r1 $r2
eq   $r1 $r1 $zero
jnzi $r1 .59
.60
move $$retv $zero                       ; set return value
ji  .15
.59
load $r0 data_1                         ; literal instantiation
rvrt $r0
.15
cfsi i32                                ; free 32 bytes for locals and 0 slots for extra call arguments.
move $$reta $r0                         ; restore reta
poph i524288                            ; Restore registers 40..64
popl i31                                ; Restore registers 16..40
jmp $$reta                              ; return from call

.program:
.16                                     ; --- start of function: store_read_24 ---
pshl i23                                ; Save registers 16..40             // the mask does not include $r3
pshh i524288                            ; Save registers 40..64
move $$locbase $sp                      ; save locals base register for store_read_24
cfei i8                                 ; allocate 8 bytes for locals and 0 slots for call arguments.
move $r0 $$reta                         ; save reta
.61
movi $r4 i32                            ; movi c i32
aloc $r4                                ; aloc c
move $r1 $hp                            ; move slot hp
srw  $r2 $r3 $r1                        ; srw a b slot                      // $r3 is modified here
movi $r4 i0                             ; movi c i0
add  $r2 $r2 $r1                        ; add a a slot
sub  $r2 $r2 $r1                        ; sub a a slot
add  $r2 $r2 $r3                        ; add a a b
add  $r2 $r2 $r4                        ; add a a c
sw   $$locbase $r2 i0                   ; store value
lw   $r1 $$locbase i0                   ; load value
lw   $r2 $$locbase i0                   ; load value
sub  $r1 $r1 $r2
move $$retv $r1                         ; set return value
.17
cfsi i8                                 ; free 8 bytes for locals and 0 slots for extra call arguments.
move $$reta $r0                         ; restore reta
poph i524288                            ; Restore registers 40..64
popl i23                                ; Restore registers 16..40
jmp $$reta                              ; return from call
```

The `store_read_24` function uses `pshl i23` to store caller registers, which does not include the `$r3` register. But `$r3` register in modified by `srw $r2 $r3 $r1`. This causes caller registers to be modified after the call. And any usage of the `$r3` register after this will have incorrect value in it.

## Impact Details

As usual, it is hard to come up with a precise impact estimation of incorrect code generation because it depends on what code the user writes. The best case scenario would be contracts that run into those bugs getting bricked, and the worst case scenario would be that incorrect program behaviors lead to loss of funds.

## References

* `https://github.com/FuelLabs/sway/blob/c186d93155d0ef9f4432773be08c7da6d0fdf6de/sway-core/src/asm_generation/fuel/allocated_abstract_instruction_set.rs#L63`
* `https://github.com/FuelLabs/sway/blob/c186d93155d0ef9f4432773be08c7da6d0fdf6de/sway-core/src/asm_lang/allocated_ops.rs#L341`

## Proof of concept

## Proof of Concept

This test would fail because `c` in the asm block of `setup` is overwritten by `store_read` `srw a b slot` unexpectedly.

```
abi IncorrectPushaPopa {
    #[storage(read)]
    fn incorrect_pusha_popa() -> ();
}

impl IncorrectPushaPopa for Contract {
    #[storage(read)]
    fn incorrect_pusha_popa() -> () {
        setup();
        ()
    }
}

#[storage(read)]
fn setup() -> () {
    let a: u64 = 1;
    let b: u64 = 1;
    let c: u64 = 1;
    //call a few times to avoid inline
    store_read();
    let r = asm(r, a: a, b: b, c: c, d: store_read()) {
        movi r i0;  
        add r a b;  // r = a + b = 2 
        add r r c;  // r = a + b + c = 3        c value is overwritten by store_read, so we get 2 instead
        add r r d;  // r = a + b + c + d = 3    d returns 0
        r
    };
    assert(r == 3);
    ()
}

#[storage(read)]
fn store_read() -> u64 {
    let a = asm(slot, a, b, c) {
        movi c i32;
        aloc c;
        move slot hp;
        srw a b slot;   // somehow make b allocate to $r3
        movi c i0;
        add a a slot;
        sub a a slot;
        add a a b;
        add a a c;
        a
    };
    a - a   // return 0 and make sure a is not dced
}

#[test]
fn incorrect_pusha_popa() -> () {
    let c = abi(IncorrectPushaPopa, CONTRACT_ID);
    c.incorrect_pusha_popa();
    ()	
}
```
