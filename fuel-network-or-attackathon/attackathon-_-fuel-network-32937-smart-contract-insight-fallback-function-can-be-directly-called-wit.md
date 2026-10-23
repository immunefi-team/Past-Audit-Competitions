# Attackathon \_ Fuel Network 32937 - \[Smart Contract - Insight] Fallback function can be directly call

Submitted on Sun Jul 07 2024 22:14:10 GMT-0400 (Atlantic Standard Time) by @rbz for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32937

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`fallback` function can be executed directly with provided arguments, circumventing the usual requirement of being called only if the contract selection process fails to match any existing ABI methods.

## Vulnerability Details

The documentation suggests that fallback is exclusively triggered when a transaction's data payload fails to match any existing function signature as defined by "contract selection", or potentially when no data is sent. It doesn't mention the possibility of directly calling this function. It's unusual to find that the default fallback function can be directly invoked using its name.

Additionally, the compiler doesn't stop you from compiling the following version of the contract, which results in producing questionable ABIs.Eventhough, this doesn't seem to directly affect how the contract works, this could lead to confusion because the ABI doesn't accurately reflect the contract's behavior. Other contracts or users might accidentally call the fallback function, thinking it's a regular one, which could cause unexpected problems.

```
// ... ommited
// <- lines 12-13 compile with no errors resulting incorrect abi's

abi RunExternalTest {
    #[fallback(arg1)] // the compiler doesn't prevent this invalid syntax                   
    fn fallback(foo: u64) -> u64;
    
    fn double_value(foo: u64) -> u64;
    fn large_value() -> b256;
}

// <- lines 19-22 compile with no errors resulting incorrect abi's
impl RunExternalTest for Contract {
    #[fallback(arg1)] // the compiler doesn't prevent this invalid syntax                  
    fn fallback(foo: u64) -> u64 {
        foo * 10
    }
    // ...omitted

// ANCHOR: fallback
#[fallback, storage(read, write)] // fallback(arg1) will also be compiled
fn fallback() -> u64 {
    use std::call_frames::*;   
    let foo = called_args::<u64>(); 
    storage.simple_value.write(foo);
    foo
// ANCHOR_END: fallback    

`functions` from `run_external_target-abi.json`:

  "functions": [
    {
      ... // omitted
      "name": "fallback", // should never be present in ABI
      "output": {
        "name": "",
        "type": 1,
        "typeArguments": null
      },
      "attributes": [
        {
          "name": "fallback",
          "arguments": [  
            "arg1"  // should never has arguments defined
          ]
        }
      ]
    },
    ... // omitted
    
  ],
```

## Impact Details

* Allowing direct calls to the fallback function muddies the waters between regular functions and the fallback, potentially causing confusion and misuse.
* The fallback function is often designed to be lightweight due to gas limitations. Direct calls might encourage overuse, leading to higher gas costs.
* It might make it harder to distinguish between intentional fallback calls and erroneous function calls.

## References

<https://docs.fuel.network/docs/sway/blockchain-development/calling\\_contracts/#fallback>

## Proof of concept

## Proof of Concept

<https://gist.github.com/0xZRA/873b12ab2c66b21a013574a3e6f738b1>
