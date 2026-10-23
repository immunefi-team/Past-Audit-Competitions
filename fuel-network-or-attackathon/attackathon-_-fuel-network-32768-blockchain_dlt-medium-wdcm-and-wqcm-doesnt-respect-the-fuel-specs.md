# Attackathon \_ Fuel Network 32768 - \[Blockchain\_DLT - Medium] WDCM and WQCM doesnt respect the fuel-s

Submitted on Tue Jul 02 2024 01:29:30 GMT-0400 (Atlantic Standard Time) by @jasonxiale for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32768

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/FuelLabs/fuel-vm/tree/0e46d324da460f2db8bcef51920fb9246ac2143b>

Impacts:

* A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

According to the fuel-specs, both [WDCM](https://github.com/FuelLabs/fuel-specs/blob/master/src/fuel-vm/instruction-set.md#wdcm-128-bit-integer-comparison) and [WQCM](https://github.com/FuelLabs/fuel-specs/blob/master/src/fuel-vm/instruction-set.md#wqcm-256-bit-integer-comparison) should clears $of and $err registers, but those two instruction don't clear these regs.

## Vulnerability Details

I will take WDCM as example: In [WDCM](https://github.com/FuelLabs/fuel-vm/blob/0e46d324da460f2db8bcef51920fb9246ac2143b/fuel-vm/src/interpreter/executors/instruction.rs#L196-L202), self.alu\_wideint\_cmp\_u256 will be called, and self.alu\_wideint\_cmp\_u256 is defined as a [macro](https://github.com/FuelLabs/fuel-vm/blob/0e46d324da460f2db8bcef51920fb9246ac2143b/fuel-vm/src/interpreter/alu/wideint.rs#L71-L95)

```rust
 73                 pub(crate) fn [<alu_wideint_cmp_ $t:lower>](
 74                     &mut self,
 75                     ra: RegisterId,
 76                     b: Word,
 77                     c: Word,
 78                     args: CompareArgs,
 79                 ) -> SimpleResult<()> {
 80                     let (SystemRegisters { pc, .. }, mut w) = split_registers(&mut self.registers);
 81                     let dest: &mut Word = &mut w[ra.try_into()?];
 82 
 83                     // LHS argument is always indirect, load it
 84                     let lhs: $t = $t::from_be_bytes(self.memory.as_ref().read_bytes(b)?);
 85 
 86                     // RHS is only indirect if the flag is set
 87                     let rhs: $t = if args.indirect_rhs {
 88                         $t::from_be_bytes(self.memory.as_ref().read_bytes(c)?)
 89                     } else {
 90                         c.into()
 91                     };
 92 
 93                     *dest = [<cmp_ $t:lower>](lhs, rhs, args.mode);
 94 
 95                     inc_pc(pc)?;
 96                     Ok(())
 97                 }
```

As above code shows, only $pc is increased, both $err and $of are not cleared.

## Impact Details

Quoting from [reg spec](https://github.com/FuelLabs/fuel-specs/blob/master/src/fuel-vm/index.md#semantics)

> $err is used to store `Error codes for particular operations.`

If there are instructions after WDCM that check if $err is zero, and if the $err is not zero, the code flow will end early, the tx logic will be incorrect. For example, the pseudocode like:

1. $err is set by instructions like div, with `UNSAFEMATH`
2. `WDCM` is executed
3. $err is checked to see if its value is ZERO, if not, revert/return the tx logic

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

please add the following code in `fuel-vm/src/tests/wideint.rs` and run `cargo test tests::wideint::cmp_u128_one -- --nocapture`

```bash
root@hw_ether:/opt/fuel/vm-fuel/fuel-vm# cargo test tests::wideint::cmp_u128_one -- --nocapture
...

running 1 test
CompareMode::EQ: 0
CompareMode::NE: 1
CompareMode::LT: 2
CompareMode::GT: 3
CompareMode::LTE: 4
CompareMode::GTE: 5
err: 1
test tests::wideint::cmp_u128_one ... ok
```

As we can see from above, err is **1**, which means $err reg isn't cleared after `WDCM`

```diff
diff --git a/fuel-vm/src/tests/wideint.rs b/fuel-vm/src/tests/wideint.rs
index 459425f6..d85c1510 100644
--- a/fuel-vm/src/tests/wideint.rs
+++ b/fuel-vm/src/tests/wideint.rs
@@ -96,6 +96,56 @@ fn cmp_u128(
     }
 }
 
+#[test]
+fn cmp_u128_one() {
+    let a: u128 = 1;
+    let b: u128 = 1;
+    let mode: CompareMode = CompareMode::EQ;
+    let mut ops = Vec::new();
+    ops.extend(make_u128(0x20, a));
+    ops.extend(make_u128(0x21, b));
+    // the following 3 instruction is used to change err reg
+    ops.push(op::movi(0x10, Flags::UNSAFEMATH.bits().try_into().unwrap()));
+    ops.push(op::flag(0x10));
+    ops.push(op::divi(0x10, RegId::ONE, 0));
+    ops.push(op::wdcm_args(
+        0x22,
+        0x20,
+        0x21,
+        CompareArgs {
+            indirect_rhs: true,
+            mode,
+        },
+    ));
+    ops.push(op::log(0x22, 0x08, RegId::ZERO, RegId::ZERO));
+    ops.push(op::ret(RegId::ONE));
+
+    let receipts = run_script(ops);
+
+    if let Receipt::Log { ra, rb, .. } = receipts.first().unwrap() {
+        let expected = match mode {
+            CompareMode::EQ => (a == b) as u64,
+            CompareMode::NE => (a != b) as u64,
+            CompareMode::LT => (a < b) as u64,
+            CompareMode::GT => (a > b) as u64,
+            CompareMode::LTE => (a <= b) as u64,
+            CompareMode::GTE => (a >= b) as u64,
+            CompareMode::LZC => a.leading_zeros() as u64,
+        };
+        assert_eq!(*ra, expected);
+        println!("CompareMode::EQ: {}", CompareMode::EQ as u8);
+        println!("CompareMode::NE: {}", CompareMode::NE as u8);
+        println!("CompareMode::LT: {}", CompareMode::LT as u8);
+        println!("CompareMode::GT: {}", CompareMode::GT as u8);
+        println!("CompareMode::LTE: {}", CompareMode::LTE as u8);
+        println!("CompareMode::GTE: {}", CompareMode::GTE as u8);
+        println!("err: {}", *rb);
+    } else {
+        panic!("Expected log receipt");
+    }
+}
+
+
```
