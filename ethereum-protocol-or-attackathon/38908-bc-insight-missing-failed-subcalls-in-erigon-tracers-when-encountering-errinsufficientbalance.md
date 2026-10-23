# #38908 \[BC-Insight] Missing Failed Subcalls in Erigon Tracers When Encountering \`ErrInsufficientBalance\` Error

**Submitted on Jan 17th 2025 at 16:23:21 UTC by @a3yip6 for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38908
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

Given any transactions with a failed subcalls due to insufficient balance, the tracer module in `Erigon` does not work properly. More specifically, the corresponding subcall is missing. This bug can be exploited in production.

## Vulnerability Details

#### Expected behaviour

For a transaction with a failed subcall due to `ErrInsufficientBalance`, the tracer should return the full trace, including all subcalls, even if some fail. The expected output is:

```
{
  "txHash": "0xd0d52b784a239d1a7e765f6cd9daf0ffdee29f6c942f517b42b42388b8a7ff87",
  "result": {
    "from": "0xfdce42116f541fc8f7b0776e2b30832bd5621c85",
    "gas": "0x14e9b",
    "gasUsed": "0xf240",
    "to": "0x3dbacbf3da19ff6dedea74ed5c8722107672cb83",
    "input": "0x3bfc026c",
    "calls": [
      {
        "from": "0x3dbacbf3da19ff6dedea74ed5c8722107672cb83",
        "gas": "0xcc05",
        "gasUsed": "0x56f0",
        "to": "0xb7e811662fa10ac068aee115ac2e682821630535",
        "input": "0x",
        "value": "0xde0b6b3a7640000",
        "type": "CALL"
      },
      {
        "from": "0x3dbacbf3da19ff6dedea74ed5c8722107672cb83",
        "gas": "0x5b64",
        "gasUsed": "0x0",
        "to": "0xb7e811662fa10ac068aee115ac2e682821630535",
        "input": "0x",
        "error": "insufficient balance for transfer",
        "value": "0xde0b6b3a7640000",
        "type": "CALL"
      }
    ],
    "value": "0xde0b6b3a7640000",
    "type": "CALL"
  }
}
```

This behavior matches the output from `geth` and `reth`.

#### Actual behaviour

In `erigon`, the tracer does not include the second subcall, resulting in the following truncated output:

```
{
  "txHash": "0xd0d52b784a239d1a7e765f6cd9daf0ffdee29f6c942f517b42b42388b8a7ff87",
  "result": {
    "from": "0xfdce42116f541fc8f7b0776e2b30832bd5621c85",
    "gas": "0x14e9b",
    "gasUsed": "0xf240",
    "to": "0x3dbacbf3da19ff6dedea74ed5c8722107672cb83",
    "input": "0x3bfc026c",
    "calls": [
      {
        "from": "0x3dbacbf3da19ff6dedea74ed5c8722107672cb83",
        "gas": "0xcc05",
        "gasUsed": "0x56f0",
        "to": "0xb7e811662fa10ac068aee115ac2e682821630535",
        "input": "0x",
        "value": "0xde0b6b3a7640000",
        "type": "CALL"
      }
    ],
    "value": "0xde0b6b3a7640000",
    "type": "CALL"
  }
}
```

The second subcall with the `ErrInsufficientBalance` error is missing, which differs from the behavior of `geth` and `reth`.

## Impact Details

Some platform (e.g., Etherscan) might use `Erigon`'s tracer to calculate some front-end data. A buggy implementation would result in incorrect UI data.

## References

I believe the bug is in here:\
<https://github.com/erigontech/erigon/blob/ab8c054a7179072bb12fa30c94dbb28f008c28d3/core/vm/evm.go#L181-L202>

## Proof of Concept

## Proof of Concept

Option 1: Testnet with Custom Nodes

1. Set up a testnet using `geth` and `erigon` as nodes. Use [kurtosis](https://github.com/ethpandaops/ethereum-package) for automation.
2. Deploy the PoC contracts to the testnet.
3. Send a transaction that includes a failed subcall due to `ErrInsufficientBalance`. One can directly use the [PoC.zip](https://github.com/user-attachments/files/18267133/PoC.zip) and run:

```
forge script ./script/DeployAndInteract.s.sol:DeployAndInteract --rpc-url $TESTNET_RPC_URL --broadcast
```

4. Inspect the transaction trace via the RPC of `geth` and `erigon`:

* `debug_traceTransaction` on both nodes.
* Compare the outputs to observe the discrepancy in the erigon trace.

Option 2: Mainnet Analysis

1. Identify a mainnet transaction with a failed subcall caused by `ErrInsufficientBalance`.
2. Trace the transaction using the `debug_traceTransaction` RPC method:

* Use geth RPC.
* Use erigon RPC.

3. Compare the outputs from both nodes.
