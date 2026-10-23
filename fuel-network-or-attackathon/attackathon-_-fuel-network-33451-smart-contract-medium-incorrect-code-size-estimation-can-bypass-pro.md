# Attackathon \_ Fuel Network 33451 - \[Smart Contract - Medium] Incorrect code size estimation can bypa

Submitted on Sat Jul 20 2024 18:01:41 GMT-0400 (Atlantic Standard Time) by @Solosync6 for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33451

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `call_frames::code_size()` function in the standard library incorrectly returns a memory address instead of the actual code size.

This vulnerability can bypass security checks in contracts that verify contract code size, potentially resulting in unauthorized access and direct theft of user funds.

## Vulnerability Details

The `code_size()` function is intended to return the size of the code in the current call frame. Current in-line documentation clearly suggests that function is expected to return code size of current call frame in bytes and *NOT* its memory location.

Code snipped below:

````
/// Get the code size in bytes (padded to word alignment) from the current call frame.
///
/// # Additional Information
///
/// More information on data from call frames can be found in the Fuel Specs.
/// https://docs.fuel.network/docs/specs/fuel-vm/#call-frames
///
/// # Returns
///
/// * [u64] - The code size of the current call frame.
///
/// # Examples
///
/// ```sway
/// use std::call_frames::code_size;
///
/// fn foo() {
///     let size = code_size();
///     assert(size != 0);
/// }
/// ```
pub fn code_size() -> u64 {
    asm(size, ptr, offset: 576) {
        add size fp offset;
        size: u64 //@audit this returns the memory location -> not value stored
    }
}
````

I was unsure if this is by design or if its a bug, until I looked at the other values also part of [Call Frames Documentation](https://docs.fuel.network/docs/specs/fuel-vm/#call-frames), namely, first param and second param. Although they are byte\[8], they are read as u64 inside the code and their implementation clearly reads the value at the memory offset location (instead of returning the location itself)

```
pub fn first_param() -> u64 {
    frame_ptr().add::<u64>(FIRST_PARAMETER_OFFSET).read()
}

pub fn second_param() -> u64 {
    frame_ptr().add::<u64>(SECOND_PARAMETER_OFFSET).read()
}
```

This makes me conclude that code\_size is implemented incorrectly by dev team. The current implementation erroneously returns the memory address where the code size is stored, rather than the actual size value. This discrepancy can cause critical security issues in contracts that use this function for access control or other security-critical operations.

## Impact Details

Consider a DEX built on Fuel that implements a security feature to protect against malicious contracts by checking their code size. The DEX assumes that very small contracts ( proxy contracts) or extremely large contracts might be suspicious.

Here is a sample code where I inserted a require condition based on code size of calling contract:

```
contract;

use std::call_frames::code_size;

abi CodeSizeAwareDEX {
    #[payable]
    fn swap(token_in: ContractId, token_out: ContractId, amount: u64);
}

impl CodeSizeAwareDEX for Contract {
    #[payable]
    fn swap(token_in: ContractId, token_out: ContractId, amount: u64) {
        let caller_size = code_size();
        
        // Check if the caller's code size is within "safe" limits
        require(caller_size > 100 && caller_size < 1000000, "Suspicious contract size"); //@audit this check detects suspicious calling contracts

        // Perform the swap
        // ... (swap logic here)
    }
}
```

An attacker could exploit this vulnerability by:

* Creating a malicious contract with a very small code size that would normally be rejected.
* Bypassing the size check because `code_size()` returns a memory address that likely falls within the "safe" range.
* Gaining unauthorized access to perform swaps, potentially draining liquidity pools or performing unauthorized trades.
* Consistently exploiting this vulnerability, as the returned "size" (actually a memory address) is likely to be consistent across calls.

This could result in direct theft of user funds locked in liquidity pools or actively being traded

## References

<https://github.com/FuelLabs/sway/blob/e1b1c2bee73e0ba825e07736cefa6c0abd079595/sway-lib-std/src/call\\_frames.sw#L68>

## Proof of concept

## Proof of Concept

Copy the following in call\_frames.sw and run the following command `forc test --logs --filter-exact test_code_size`

Note:

* I mocked the code\_size function by passing the frame pointer as input. (code\_size\_mock)
* I wrote a corrected function `code_size_mock_corrected` to show correction

```

pub fn code_size_mock(frame_ptr: raw_ptr) -> u64 {
    asm(size, ptr: frame_ptr, offset: 576) {
        add size ptr offset;
        size: u64
    }
}


pub fn code_size_mock_corrected(frame_ptr: raw_ptr) -> u64 {
    let ptr = asm(ptr, fptr: frame_ptr, offset: 576) {
        add ptr fptr offset;
        ptr: raw_ptr
    };
    ptr.read::<u64>()
}


#[test]
pub fn test_code_size() {
    use ::assert::assert;
    use ::alloc::alloc;
    use ::logging::log;
    // Set up the frame pointer to point to a simulated call frame
    let frame_ptr: raw_ptr = alloc::<u8>(1024); // Allocate memory for the frame
    let size: u64 = 288u64;

    // Store the size at the offset expected by code_size()
    let offset_ptr = frame_ptr.add_uint_offset(576);    
    asm(ptr: offset_ptr, val: size) {
        sw ptr val i0;
    }



    // Call the function and assert the value
   let returned_size = code_size_mock(frame_ptr);
    log(size);
    log(returned_size);
    assert(returned_size != 288u64);

    let returned_size_corrected = code_size_mock_corrected(frame_ptr);
    log(returned_size_corrected);
     assert(returned_size_corrected == 288u64);    
} 

```
