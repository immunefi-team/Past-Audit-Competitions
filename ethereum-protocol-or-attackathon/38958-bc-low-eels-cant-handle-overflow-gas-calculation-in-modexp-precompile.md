# #38958 \[BC-Low] EELS cant handle overflow gas calculation in modexp precompile

**Submitted on Jan 18th 2025 at 22:13:02 UTC by @Omik for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38958
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/ethereum/execution-specs>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

Hi Team, I was testing the gas calculation for modexp precompile for GETH, Nethermind, and comparing it with EELS. And I found out that EELS can't handle a very large base length like GETH, and Nethermind.

## Vulnerability Details

When executing a precompile contract there are some steps that the EL takes before actually executing the precompile contract, those steps were:

1. calculate the gas
2. check if the gas didn't exceed the forwarded gas
3. if the gas check passes then it will execute the precompile based on the input that was given.

The bug was found when I noticed that Nethermind and GETH calculate the gas cost for modexp precompile using different type of integer. GETH calculates the gas using big number, which can't get overflow when the number goes bigger than uint256.max. And, Nethermind calculates the gas using UINT256 type, which will overflow if the number goes bigger than uint256.max.

Gas Calculation:

* GETH: <https://github.com/ethereum/go-ethereum/blob/master/core/vm/contracts.go#L367-L444>
* Nethermind: <https://github.com/NethermindEth/nethermind/blob/master/src/Nethermind/Nethermind.Evm/Precompiles/ModExpPrecompile.cs#L43-L76>

Since Nethermind modexp gas calculation allows us to overflow its calculation, I try to overflow this part of the code:\
<https://github.com/NethermindEth/nethermind/blob/master/src/Nethermind/Nethermind.Evm/Precompiles/ModExpPrecompile.cs#L65>

```
 UInt256 startIndex = 96 + baseLength; 
```

by setting the base length value as `0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffa0` or `115792089237316195423570985008687907853269984665640564039457584007913129639840` which when added to 96 it becomes 0

Thankfully, this didn't cause a consensus issue, because stateroot for both GETH and Nethermind is still the same, and Nethermind threw an exception when trying to cast the UINT256 number to the int data type in steps 3 (referring from executing precompile steps above). this leads to a slight difference between GETH and Nethermind, Geth exits the execution at step 2 with "out of gas", and Nethermind exits the execution at step 3 (<https://github.com/NethermindEth/nethermind/blob/master/src/Nethermind/Nethermind.Evm/Precompiles/ModExpPrecompile.cs#L97>) when trying to cast a large base length as an int. Both executions consume all of the gas and become a failed execution.

```
int baseLength = (int)new UInt256(extendedInput[..32], true);
```

However, when I try to test this case against EELS it just throws an error and the program stops with a "raise OverflowError()", and this fails while trying to do the same thing as Nethermind did, which is adding a large base length with 96. (<https://github.com/ethereum/execution-specs/blob/master/src/ethereum/cancun/vm/precompiled\\_contracts/modexp.py#L36>)

```
exp_start = U256(96) + base_length
```

## Impact Details

Although there is no real issue with the two of the largest Ethereum clients,\
This might cause ambiguous interpretations for any new implementation that follows the Spec EELS

## References

Add any relevant links to documentation or code

## Link to Proof of Concept

<https://gist.github.com/GibranAkbaromiL/edbf9c124b252b12f0a8a7eb6395d32e>

## Proof of Concept

## Proof of Concept

To reproduce the behavior:

1. clone the goevmlab repo <https://github.com/holiman/goevmlab>
2. build the docker image, this is to prepare the binary for each evms. (I'm getting errors for building a lot of evms, that's why I only test the most common evm client which are GETH and Nethermind, and the eels to know the intended spec)
3. get the state test from this gist link <https://gist.github.com/GibranAkbaromiL/edbf9c124b252b12f0a8a7eb6395d32e>
4. run the docker image
5. run these commands

```
###/gethvm statetest --trace.format json --noreturndata --trace --dump /shared/modexp_test.json 2>/traces/modexp_test.geth.stderr.txt 1>/traces/modexp_test.geth.stdout.txt 
###/ethereum-spec-evm statetest --json --noreturndata /shared/modexp_test.json 2>/traces/modexp_test.eels.stderr.txt 1>/traces/modexp_test.eels.stdout.txt
###/neth/nethtest --trace --input /shared/modexp_test.json 2>/traces/modexp_test.nethermind.stderr.txt 1>/traces/modexp_test.nethermind.stdout.txt
```
