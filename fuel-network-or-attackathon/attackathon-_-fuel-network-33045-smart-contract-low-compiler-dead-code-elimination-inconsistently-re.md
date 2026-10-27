# Attackathon \_ Fuel Network 33045 - \[Smart Contract - Low] Compiler Dead Code Elimination inconsisten

Submitted on Wed Jul 10 2024 03:34:53 GMT-0400 (Atlantic Standard Time) by @LonelySloth for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33045

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Permanent freezing of funds
* Missing arithmetic checks in release bytecode.
* Inconsistent test/release behavior leading to untestable code.

## Description

## Brief/Intro

The DCE module of the compiler/optimizer is responsible for eliminating code that can't be reached or whose resulting computed values are never used -- thus saving bytecode size and gas costs. However the DCE incorrectly identifies arithmetic operations as not producing side-effects, while in fact they can produce reverts. This leads to inconsistent behavior between test/release bytecode, with certain implicit assertions not being included in release. Consequently, contracts will have unexpected behavior that can plausibly lead to loss of funds.

## Vulnerability Details

By default Sway and the underlying FuelVM perform overflow (and division by zero) checks on all arithmetic operations, resulting in a panic in case the operation overflows or results in division by zero.

That means in practice, the statement

```
let x = a - b;
```

Should be equivalent to:

```
assert(b <= a);
let x = a - b;
```

While this is indeed *always* the case for the types `u8`, `u16`, `u32` (which are implemented in Sway using `assert`), **it's not always the case for the native type `u64`**.

The reason is the optimizer's DCE (Dead Code Eliminator) treatment of arithmetic operations. The source file `instruction.rs` in the compiler's `sway-ir` module defines the operations that are deemed to have side-effects:

```
pub fn may_have_side_effect(&self) -> bool {
        match self {
            InstOp::AsmBlock(_, _)
            | InstOp::Call(..)
            | InstOp::ContractCall { .. }
            | InstOp::FuelVm(FuelVmInstruction::Log { .. })
            | InstOp::FuelVm(FuelVmInstruction::Smo { .. })
            | InstOp::FuelVm(FuelVmInstruction::StateClear { .. })
            | InstOp::FuelVm(FuelVmInstruction::StateLoadQuadWord { .. })
            | InstOp::FuelVm(FuelVmInstruction::StateStoreQuadWord { .. })
            | InstOp::FuelVm(FuelVmInstruction::StateStoreWord { .. })
            | InstOp::FuelVm(FuelVmInstruction::Revert(..))
            | InstOp::FuelVm(FuelVmInstruction::JmpMem)
            | InstOp::FuelVm(FuelVmInstruction::Retd { .. })
            | InstOp::MemCopyBytes { .. }
            | InstOp::MemCopyVal { .. }
            | InstOp::Store { .. }
            | InstOp::Ret(..)
            | InstOp::FuelVm(FuelVmInstruction::WideUnaryOp { .. })
            | InstOp::FuelVm(FuelVmInstruction::WideBinaryOp { .. })
            | InstOp::FuelVm(FuelVmInstruction::WideCmpOp { .. })
            | InstOp::FuelVm(FuelVmInstruction::WideModularOp { .. }) => true,

            InstOp::UnaryOp { .. }
            | InstOp::BinaryOp { .. }
            | InstOp::BitCast(..)
            | InstOp::Branch(_)
            | InstOp::CastPtr { .. }
            | InstOp::Cmp(..)
            | InstOp::ConditionalBranch { .. }
            | InstOp::FuelVm(FuelVmInstruction::Gtf { .. })
            | InstOp::FuelVm(FuelVmInstruction::ReadRegister(_))
            | InstOp::FuelVm(FuelVmInstruction::StateLoadWord(_))
            | InstOp::GetElemPtr { .. }
            | InstOp::GetLocal(_)
            | InstOp::GetConfig(_, _)
            | InstOp::IntToPtr(..)
            | InstOp::Load(_)
            | InstOp::Nop
            | InstOp::PtrToInt(..) => false,
        }
    }
```

**All binary operations are listed as not having side effects -- that includes all arithmetic operations.** The fact is that arithmetic operation **do have side-effects, as they can cause a revert.** Thus the above code is incorrect, which will lead to many inconsistencies.

As the Optimizer understands arithmetic functions to have no side effects, *it freely removes such operations from the code whenever it finds the result of the operation isn't used, or the code can't be reached.*

What exactly counts as *not used or unreachable* depends on the exact parameters used when running the compiler, and various *hard to predict* aspects of the contract -- often leading to changes in one part of the code resulting in changed behavior in a different part.

For example, a call to `testf(..)` in the contract below will revert with no optimization:

```
contract;

use std::flags::{
        disable_panic_on_overflow,
        enable_panic_on_overflow,
    };


abi TestContract {
    
    fn testf(x: u64) -> u64;
}


fn check_n_log_diff(x: u64, y: u64, debug: bool){

    if debug {log((x, y));}

    let diff = x-y; //reverts if x<y;

    if debug { log(diff);}
}

impl TestContract for Contract {
    
    fn testf(x: u64) -> u64 {
        check_n_log_diff(0,1,false);

        0
    }
   
}

#[test]
fn test_arith0() {
     let target = abi(TestContract, CONTRACT_ID);

     target.testf(0);
}
```

Running the test using `forc test` will result in the test failing.

**However, running the test using `forc test --release` will result in the test passing.**

Even worse, introducing another function to the contract, without adding any modification to the original code, can change the behavior. This modified contract reverts **both in test and release mode**.

```
contract;

use std::flags::{
        disable_panic_on_overflow,
        enable_panic_on_overflow,
    };


abi TestContract {
    
    fn testf(x: u64) -> u64;

    fn testf2(x: u64) -> u64;
}


fn check_n_log_diff(x: u64, y: u64, debug: bool){

    if debug {log((x, y));}

    let diff = x-y; //reverts if x<y;

    if debug { log(diff);}
}

impl TestContract for Contract {
    
    fn testf(x: u64) -> u64 {
        check_n_log_diff(0,1,false);

        0
    }

    fn testf2(x: u64) -> u64 {
        check_n_log_diff(0,1,true);

        0
    }
   
}

#[test]
fn test_arith0() {
     let target = abi(TestContract, CONTRACT_ID);

     target.testf(0);
}
```

*That obviously means also that removing one of the functions from the code can make the overflow check in the other being removed from the release bytecode, introducing a bug in code that hasn't been changed.* **That shouldn't be possible.**

The explanation is that when the optimizer can statically check that the function `check_n_log_diff` is only ever called with a `false` last argument, it eliminates the `log` operations as unreachable, and consequently (in the next pass), removes the arithmetic operation as well, as it's result isn't used. However, when there are calls to the function using both `true` and `false` arguments, the optimizer can't tell if the `log` operation will be performed, and thus doesn't eliminate the arithmetic operation and its overflow check -- thus changing the behavior of the function for both cases.

As it is common practice to attempt to optimize code by removing redundant checks -- and the expectation that every arithmetic operation will revert in case of an overflow -- **I believe chances of something like the above happening in a real-world complex code base is extremely likely.**

Besides that, the fact that the issue is likely to be missed in tests (as contracts behave differently when compiled for test, and testing in release usually isn't part of the ), and that the behavior is unexpected, inconsistent between integer types, and in fact unpredictable -- makes it almost impossible for developers to guard against such problems.

## Comparing to other languages

To further demonstrate why the optimizer behavior is dangerous and shouldn't happen (ever) I would like to make a quick comparison with Solidity and Rust -- which are the main sources of inspiration for Sway.

* **Solidity**: In Solidity arithmetic overflow checks are implemented by the compiler by adding conditionals and `revert` instructions. That means the optimizer never eliminates them, as `revert` is an instruction with side effects. The same is true in Sway for overflows in non-native types (u8, u16 etc) -- that's how all types are supposed to behave. **Note that division by zero in Sway can be eliminated by the DCE even in the non-native types.**
* **Rust**: While Rust (typically) does have overflow checks that are removed in release mode, the significance of these assertions are the exact opposite of what we have in Sway -- in Rust the overflows are never supposed to happen in production, and are included in test to help catch bugs early, while in Sway **the panics are an essential part of the functionality and security of production code.**

## Impact Details

The issue reported results in release bytecode that has undocumented, unexpected, and unpredictable behavior under realistic scenarios -- causing the smart contract to fail to revert in situation when it should.

As reverting is the main way contracts prevent unauthorized operations, and redundant checks are typically avoided to save gas -- **deployment of a contract with such unpredictable behavior is likely to result in theft of funds (for example an attacker withdrawing more tokens than it should be able to) or freezing of funds (by setting some state to an invalid value).**

In the PoC area I include a little contract to showcase a scenario of loss of funds, as well as numerous test cases to demonstrate the inconsistency of revert behavior between various pieces of code.

**Note that, as with previous submissions of language bugs, it is also highly likely such a bug would be used to create contracts with secret back doors that can't be detected in normal audits or testing.**

## Recommendation

* Arithmetic operations should be treated as having side effects by the optimizer (as they actually do have side effects).
* A thorough review of all IR operations should be performed to check if any other side effect is missing.

## Proof of concept

## Proof of Concept

For all my tests I'm using the following configuration (`fuelup show`):

```
active toolchain
----------------
latest-x86_64-unknown-linux-gnu (default)
  forc : 0.61.2
    - forc-client
      - forc-deploy : 0.61.2
      - forc-run : 0.61.2
    - forc-crypto : 0.61.2
    - forc-debug : 0.61.2
    - forc-doc : 0.61.2
    - forc-explore : 0.28.1
    - forc-fmt : 0.61.2
    - forc-lsp : 0.61.2
    - forc-tx : 0.61.2
    - forc-wallet : 0.8.1
  fuel-core : 0.31.0
  fuel-core-keygen : 0.31.0

```

### Real-world-like exploitable scenario

The following contract should not allow a user to withdraw more than their "funds" balance. However, running the test with `forc test --release` shows it actually doesn't revert in release mode.

While of course this isn't *an actual real-world contract*, I think the sort of operations seen here are extremely likely to appear in real world as thousands of apps are developed and deployed. *Note that the contract itself doesn't have a vulnerability, and it's only when removing the function that isn't used anymore that it becomes vulnerable due to the optimizer bug.*

```
contract;

use std::flags::{
        disable_panic_on_overflow,
        enable_panic_on_overflow,
    };
use std::hash::*;


storage {
    funds: StorageMap<Identity, u64> = StorageMap {},
}

abi SafuVault {
    
    #[storage(read, write)]
    fn deposit(amount: u64);

    #[storage(read, write)]
    fn withdraw(amount: u64);

    // not used anymore, transfers are enough to track
    // withdrawals
    //#[storage(read, write)]
    //fn withdraw_and_log(amount: u64);    
}


/*
  checks if the amount is valid for the balance and logs
*/
fn check_balance_and_log(who: Identity, old_balance: u64, amount: u64, should_log: bool) {

    let new_balance = old_balance - amount; //reverts if amount is more than balance;

    if should_log { log((who, old_balance, new_balance));}
}

impl SafuVault for Contract {
    
    #[storage(read, write)]
    fn deposit(amount: u64) {
        let who = msg_sender().unwrap();

        // PoC: logic to validate tokens sent is omitted for PoC.

        storage.funds.insert(who, amount);
    }

    /*
        just a version of withdraw_and_log
        that does exactly the same thing but doesn't log
    */
    #[storage(read, write)]
    fn withdraw(amount: u64) {
        let who = msg_sender().unwrap();

        let old_balance = storage.funds.get(who).read(); // reverts if no balance exists

        // reverts if not enough balance
        check_balance_and_log(who, old_balance, amount, false);

        // we don't need to check balance again
        // smart gas savings :)
        let _ = disable_panic_on_overflow();

        storage.funds.insert(who,
                old_balance - amount);

        let _ = enable_panic_on_overflow();

        // PoC: logic to send tokens is omitted for PoC.
    }

    /*
    // removed as we don't use these logs anymore.
    #[storage(read, write)]
    fn withdraw_and_log(amount: u64) {
         let who = msg_sender().unwrap();

        let old_balance = storage.funds.get(who).read(); // reverts if no balance exists

        // reverts if not enough balance
        check_balance_and_log(who, old_balance, amount, true);

        // we don't need to check balance again
        // smart gas savings :)
        let _ = disable_panic_on_overflow();

        storage.funds.insert(who,
                old_balance - amount);

        let _ = enable_panic_on_overflow();

        // PoC: logic to send tokens is omitted for PoC.
    }
    */
   
}

// this test reverts so that means funds are Safu
#[test]
fn test_safu_vault_reverts_withdraw_too_much() {
     let target = abi(SafuVault, CONTRACT_ID);

     target.deposit(100u64);
     
     target.withdraw(1_000_000u64);     
}
```

### Testing different (and surprising) arithmetic operations

Some of the following tests will fail while others will pass. Can you guess which ones? I highly doubt DeFi application developers will be able to.

```

#[test]
fn test_arith0() {
     let _ = 0u8 - 1u8;
}

#[test]
fn test_arith1() {
     let _ = 0xffu8 + 1u8;
}

#[test]
fn test_arith2() {
     let _ = 1u8 / 0;
}

#[test]
fn test_arith3() {
     let _ = 0 - 1;
}

#[test]
fn test_arith4() {
     let x = 0 - 1;

     let mut y = x;

     y += 1;
     y += 2;
}

#[test]
fn test_arith5() {
     let x = 0 - 1;

     log(x);
}

#[test]
fn test_arith6() {
     let x = 0 - 1;

     if x==0 {
        // do nothing
     }
}

#[test]
fn test_arith7() {
     let x = 0 - 1;

     if x==0 {
        log(x);
     }
}

#[test]
fn test_arith8() {
     let x = 0 - 1;

     if false {
        log(x);
     }
}

#[test]
fn test_arith9() {
     let z = 0;
     let x = z - 1;

     if z>0 {
        log(x);
     }
}


```

Actual result `forc test`:

```
     Running 10 tests, filtered 0 tests
      test test_arith0 ... ok (15.857µs, 19 gas)
      test test_arith1 ... FAILED (17.685µs, 34 gas)
      test test_arith2 ... ok (17.855µs, 19 gas)
      test test_arith3 ... ok (17.557µs, 19 gas)
      test test_arith4 ... FAILED (19.547µs, 8 gas)
      test test_arith5 ... FAILED (17.484µs, 8 gas)
      test test_arith6 ... FAILED (3.185µs, 8 gas)
      test test_arith7 ... FAILED (18.292µs, 8 gas)
      test test_arith8 ... FAILED (11.117µs, 8 gas)
      test test_arith9 ... FAILED (3.074µs, 10 gas)


```

Actual result `forc test --release`:

```
        Running 10 tests, filtered 0 tests
      test test_arith0 ... ok (11.537µs, 19 gas)
      test test_arith1 ... FAILED (14.415µs, 25 gas)
      test test_arith2 ... ok (2.956µs, 19 gas)
      test test_arith3 ... ok (2.558µs, 19 gas)
      test test_arith4 ... ok (8.734µs, 19 gas)
      test test_arith5 ... FAILED (15.784µs, 7 gas)
      test test_arith6 ... FAILED (16.427µs, 7 gas)
      test test_arith7 ... FAILED (20.475µs, 7 gas)
      test test_arith8 ... ok (15.388µs, 19 gas)
      test test_arith9 ... ok (15.874µs, 19 gas)

```
