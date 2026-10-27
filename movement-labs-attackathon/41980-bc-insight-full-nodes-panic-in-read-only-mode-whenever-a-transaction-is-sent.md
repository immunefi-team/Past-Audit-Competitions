# #41980 \[BC-Insight] Full nodes panic in read-only mode whenever a transaction is sent

**Submitted on Mar 19th 2025 at 18:50:44 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41980
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

## Brief/Intro

When a full node is run in read-only mode and a SubmitTransaction request is sent to the full node, the full node will panic and shut down.

## Vulnerability Details

To understand how this happens, we need to take a look at the implementation inside of `read_only.rs`.

If the next request of the `mempool_client_receiver` is a SubmitTransaction request, the node panics. As a result, the node is shut down (reference (1)).\
A node is run in read-only mode whenever the `maptos_config.chain.maptos_read_only` flag is set to true. This can be seen in reference (2).

## Impact Details

An attacker can send a SubmitTransaction request to the full node API to shut down the node.

The solution is to return an error upon invalid API calls but not to shut down the node.

## References

(1): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/read\\_only.rs#L23>

(2): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/executor/initialization.rs#L208-L209>

## Proof of Concept

## Proof of Concept

1. A full node is run in read-only mode.
2. The next request of the `mempool_client_receiver` is a SubmitTransaction request. This causes the node to panic. Again, this can be verified by taking a look at reference (1).
