# #37483 \[BC-Insight] There is a trace discrepancy for Nethermind when handling EOF from PUSH opcode

**Submitted on Dec 5th 2024 at 23:46:14 UTC by @Omik for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37483
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/NethermindEth/nethermind>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

Hey There, I was testing Nethermind, GETH, and EELS evms, using goevmlab and I found out that Nethermind treated push opcode EOF differently.

## Vulnerability Details

Traces specification is introduced in the eip 3155 <https://eips.ethereum.org/EIPS/eip-3155>, and it will be emitted for every instruction in the lifetime of the transaction, for example, Push4 (0x63) meaning it is pushing the next 4 bytes to the stack, so 0x63deadbeef will push "deadbeef" to the stack, and it will emit the trace for this push4 instruction, this also means that during the execution of this transaction, push4 was executed.

The Nethermind diverges its traces implementation when handling push opcode that has an eof shorter than it is supposed to be, taking from the example above, instead of pushing 4 bytes "deadbeef" to the stack, we will try to push 4 bytes of "deadbe", as you can see it only contains 3 bytes instead of 4, in Nethermind the execution will have one additional opcode that is getting executed after the push 4 instruction executed which is a push 0 opcode. However, in GETH and EELS, both of these EVMs will directly go to the stop instruction, without executing the push 0 opcode.

## Impact Details

Although these discrepancies don't change the stateroot for now, this might make any off-chain actor diverge in consensus when handling traces of transaction execution.

## Additional information

From testing this case, I also found out that Nethermind never emit the STOP instruction.

## Link to Proof of Concept

<https://gist.github.com/GibranAkbaromiL/9e827cd6b37f31ca42e7435a88b14edc>

## Proof of Concept

To reproduce the behavior:

1. clone the goevmlab repo <https://github.com/holiman/goevmlab>
2. build the docker image, this is to prepare the binary for each evms. (I'm getting errors for building a lot of evms, that's why I only test the most common evm client which are GETH and Nethermind, and the eels to know the intended spec)
3. get the state test from this gist link <https://gist.github.com/GibranAkbaromiL/9e827cd6b37f31ca42e7435a88b14edc>
4. run the docker image
5. run these commands

```
###/gethvm statetest --trace.format json --nomemory --noreturndata --trace /shared/push_test.json 2>/traces/push_test.geth.stderr.txt 1>/traces/push_test.geth.stdout.txt 
###/ethereum-spec-evm statetest --json --nomemory --noreturndata /shared/push_test.json 2>/traces/push_test.eels.stderr.txt 1>/traces/push_test.eels.stdout.txt
###/neth/nethtest --memory --trace --input /shared/push_test.json 2>/traces/push_test.nethermind.stderr.txt 1>/traces/push_test.nethermind.stdout.txt
```
