# #37695 \[BC-Insight] Executing transaction that has a wrong nonce might triggered a chain split due to mismatch stateroot

**Submitted on Dec 12th 2024 at 17:49:55 UTC by @Omik for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37695
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/NethermindEth/nethermind>
* **Impacts:**
  * Unintended permanent chain split affecting greater than or equal to 25% of the network, requiring hard fork (network partition requiring hard fork)

## Description

## Brief/Intro

Hey There, I am doing statetest using goevmlab and found out that executing a transaction that has a wrong nonce leads to a Nethermind client diverging from GETH and the spec, which can be seen by their stateroot after the transaction was executed, this means that Nethermind is not consistent with GETH when executing those transaction.

## Vulnerability Details

When submitting a transaction, nonce is one of the args that's getting submitted to the blockchain, and it will increment for every executed transaction.

I found out that Nethermind execution client is not consistent with GETH and EELS, when handling a transaction that has a wrong nonce. Below is the statetest that I used to make the POC (taken from the attached gist).

```
{
  "testing-create": {
    "env": {
      "currentCoinbase": "b94f5374fce5edbc8e2a8697c15331677e6ebf0b",
      "currentDifficulty": "0x200000",
      "currentRandom": "0x0000000000000000000000000000000000000000000000000000000000200000",
      "currentGasLimit": "0x26e1f476fe1e22",
      "currentNumber": "0x1",
      "currentTimestamp": "0x3e8",
      "previousHash": "0x044852b2a670ade5407e78fb2863c51de9fcb96542a07186fe3aeda6bb8a116d",
      "currentBaseFee": "0x10"
    },
    "pre": {
      "0x00000000000000000000000000000000000000cc": {
        "code": "0x",
        "storage": {},
        "balance": "0x0",
        "nonce": "0x0"
      },
      "0xa94f5374fce5edbc8e2a8697c15331677e6ebf0b": {
        "code": "0x",
        "storage": {},
        "balance": "0xffffffffff",
        "nonce": "0x1" //current nonce
      }
    },
    "transaction": {
      "gasPrice": "0x16",
      "nonce": "0x2", //the correct nonce is 1, but we set this to 2 intentionally for the POC
      "to": "0x00000000000000000000000000000000000000cc",
      "data": [
        "0x"
      ],
      "gasLimit": [
        "0x7a1200"
      ],
      "value": [
        "0x1"
      ],
      "sender": "0xa94f5374fce5edbc8e2a8697c15331677e6ebf0b",
      "secretKey": "0x45a915e4d060149eb4365960e6a7a45f334393093061116b197e3240065ff2d8"
    },
    "out": "0x",
    "post": {
      "Cancun": [
        {
          "hash": "0x0000000000000000000000000000000000000000000000000000000000000000", 
          "logs": "0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347",
          "indexes": {
            "data": 0,
            "gas": 0,
            "value": 0
          }
        }
      ]
    }
  }
}
```

This is the output from these 3 client (Nethermind, GETH, EELS)

```
root@209381238e3f:/traces# cat nonce_test.geth.stderr.txt 
{"stateRoot": "0xe6bd082c31ce6f0f7168666281dab0641f4908f5c64b20f11ed7b83ced0ffaa1"}
root@209381238e3f:/traces# cat nonce_test.nethermind.stderr.txt 
{"gasUsed":"0xffffffffffffadf8","time":14.0386}
{"stateRoot":"0x480a9df66e5c98579344d6586b46c88c9632578d85eecf2628cac941eba658dd"}
root@209381238e3f:/traces# cat nonce_test.eels.stderr.txt 
# WARNING:T8N:Transaction 0 failed: InvalidBlock()
{"stateRoot": "0xe6bd082c31ce6f0f7168666281dab0641f4908f5c64b20f11ed7b83ced0ffaa1"}
```

As we can see Nethermind and GETH have a different stateroot, and for GETH we can dump the state after running the statetest. below is the output of the statetest dump.

```
[
  {
    "name": "testing-create",
    "pass": false,
    "stateRoot": "0xe6bd082c31ce6f0f7168666281dab0641f4908f5c64b20f11ed7b83ced0ffaa1",
    "fork": "Cancun",
    "error": "unexpected error: nonce too high: address 0xa94f5374Fce5edBC8E2a8697C15331677e6EbF0B, tx: 2 state: 1",
    "state": {
      "root": "e6bd082c31ce6f0f7168666281dab0641f4908f5c64b20f11ed7b83ced0ffaa1",
      "accounts": {
        "0x00000000000000000000000000000000000000cc": {
          "balance": "0",
          "nonce": 0,
          "root": "0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
          "codeHash": "0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470",
          "address": "0x00000000000000000000000000000000000000cc",
          "key": "0xaa8d9c947771632a645c67655595b61da72837bfa97f30f417dbaed82f2f11c3"
        },
        "0xa94f5374Fce5edBC8E2a8697C15331677e6EbF0B": {
          "balance": "1099511627775",
          "nonce": 1,
          "root": "0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
          "codeHash": "0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470",
          "address": "0xa94f5374fce5edbc8e2a8697c15331677e6ebf0b",
          "key": "0x03601462093b5945d1676df093446790fd31b20e7b12a2e8e5e09d068109616b"
        }
      }
    }
  }
]
```

We can see that the nonce of the account is still 1 meaning the transaction is not getting executed, Since Nethermind has a different stateroot, this implies that Nethermind does some state changes that differ from GETH.

## Impact Details

Since the stateroot for the Nethermind is different than the GETH and EELS, this might lead to chain split between Nethermind and GETH, which is 2 of the largest execution client in Ethereum based on <https://clientdiversity.org/?utm\\_source=immunefi> and <https://ethernodes.org/?utm\\_source=immunefi>

## References

<https://gist.github.com/GibranAkbaromiL/3562c8bbfc7bb25e075b84ba908ee601>

## Link to Proof of Concept

<https://gist.github.com/GibranAkbaromiL/3562c8bbfc7bb25e075b84ba908ee601>

## Proof of Concept

## Proof of Concept

To reproduce the behavior:

1. clone the goevmlab repo <https://github.com/holiman/goevmlab>
2. build the docker image, this is to prepare the binary for each evms. (I'm getting errors for building a lot of evms, that's why I only test the most common evm client which are GETH and Nethermind, and the eels to know the intended spec)
3. get the state test from this gist link <https://gist.github.com/GibranAkbaromiL/3562c8bbfc7bb25e075b84ba908ee601>
4. run the docker image
5. run these commands

```
###/gethvm statetest --trace.format json --nomemory --noreturndata --trace --dump /shared/nonce_test.json 2>/traces/nonce_test.geth.stderr.txt 1>/traces/nonce_test.geth.stdout.txt 
###/ethereum-spec-evm statetest --json --nomemory --noreturndata /shared/nonce_test.json 2>/traces/nonce_test.eels.stderr.txt 1>/traces/nonce_test.eels.stdout.txt
###/neth/nethtest --memory --trace --input /shared/nonce_test.json 2>/traces/nonce_test.nethermind.stderr.txt 1>/traces/nonce_test.nethermind.stdout.txt
```
