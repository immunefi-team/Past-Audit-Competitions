# #38319 \[BC-Insight] Edge case difference for GETH and NETHERMIND when calculating memory expansion gas

**Submitted on Dec 30th 2024 at 22:58:53 UTC by @Omik for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38319
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/NethermindEth/nethermind>
* **Impacts:**
  * Unintended permanent chain split affecting greater than or equal to 25% of the network, requiring hard fork (network partition requiring hard fork)

## Description

## Brief/Intro

Hey There, Merry christmas and happy new year!!!

I found out that there is a discrepancy in the way nethermind handles gas calculation for memory expansion, instead of limiting word size to uint32 max like GETH EL, its limiting the word size to int32 max, which is way lower than uint32 max.

This attack is an edge case when the gas limit is higher than 30M. Fortunately, the current Ethereum gas limit is set to 30M, Therefore the possibility of this attack to occur is small.

Based on the things that you concerned about, this attack falls under this statement

```
Edge case behavior that deviates from "true" Ethereum can also cause issues, including:
- Opcode behavior at extremes (e.g., blockhash at 255/256 depth)
- Precompiles decoding their arguments differently
```

## Vulnerability Details

In EVM every opcode is executed with the same flow pattern, roughly the steps for each opcode that interact with memory were:

1. calculate the memory size
2. based on the memory size, calculate the gas
3. pay the gas
4. resize the memory based on its size from step 1
5. execute the opcode

In GETH there is a hardcoded value that is being used while calculating the gas, this value is "0x1FFFFFFFE0" (<https://github.com/ethereum/go-ethereum/blob/master/core/vm/gas\\_table.go#L39>), and it's being used to make sure that the word size never goes beyond "0xFFFFFFFF" or uint32 max, because if the word size goes beyond uint32 max the gas calculation will overflow. (<https://github.com/ethereum/go-ethereum/blob/master/core/vm/gas\\_table.go#L34-L38>)

This limitation is also applied in Nethermind, However, instead of limiting the word size to uint32 max like GETH, Nethermind limits the word size only to (0x7FFFFFFF) int32 max (<https://github.com/NethermindEth/nethermind/blob/master/src/Nethermind/Nethermind.Evm/EvmPooledMemory.cs#L303>).

Therefore, if an opcode that interacts with a memory tries to expand the memory between 0x80000000 - 0xFFFFFFFF, Nethermind will marked the opcode execution as out of gas (<https://github.com/NethermindEth/nethermind/blob/master/src/Nethermind/Nethermind.Evm/EvmPooledMemory.cs#L303-L307>), and GETH will continue the execution and not marked it as out of gas.

## Impact Details

Since Nethermind will fail the transaction as out of gas, and GETH will continue the execution and not fail the transaction, both of these clients will have a different state root, and this could trigger Nethermind and GETH clients to split.

GETH and Nethermind are 2 of the largest execution clients in Ethereum based on <https://clientdiversity.org/?utm\\_source=immunefi> and <https://ethernodes.org/?utm\\_source=immunefi>

## Link to Proof of Concept

<https://gist.github.com/GibranAkbaromiL/2f6246d5ba3c35031ae68169376a743a>

## Proof of Concept

## Proof of Concept

To reproduce the behavior:

1. clone the goevmlab repo <https://github.com/holiman/goevmlab>
2. build the docker image, this is to prepare the binary for each evms. (I'm getting errors for building a lot of evms, that's why I only test the most common evm client which are GETH and Nethermind, and the eels to know the intended spec)
3. get the state test from this gist link <https://gist.github.com/GibranAkbaromiL/2f6246d5ba3c35031ae68169376a743a>
4. run the docker image
5. run these commands

```
###/gethvm statetest --trace.format json --noreturndata --trace --dump /shared/gas_test.json 2>/traces/gas_test.geth.stderr.txt 1>/traces/gas_test.geth.stdout.txt 
###/ethereum-spec-evm statetest --json --noreturndata /shared/gas_test.json 2>/traces/gas_test.eels.stderr.txt 1>/traces/gas_test.eels.stdout.txt
###/neth/nethtest --trace --input /shared/gas_test.json 2>/traces/gas_test.nethermind.stderr.txt 1>/traces/gas_test.nethermind.stdout.txt
```

Due to hardware limitations on my end GETH and EELS state tests output "killed", this is because of OOM (Out of memory). This means that it already passed the gas calculation steps, and fails when trying to actually expand the memory in my/your machine. However, for Nethermind the execution goes out of gas immediately, without encountering an OOM error, Because the opcodes execution ends in the gas calculation steps, and didn't even try to expand the memory on my/your machine.

And if you try to access the traces for GETH you notice that the execution didn't give the full traces, and stops at MLOAD, this is because of the OOM that you encountered due to hardware limitation. However, Nethermind will still give you the full traces of the transaction, because it didn't encounter the OOM.
