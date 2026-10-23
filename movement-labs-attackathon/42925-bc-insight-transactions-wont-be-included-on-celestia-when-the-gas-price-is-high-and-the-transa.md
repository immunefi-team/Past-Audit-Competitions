# #42925 \[BC-Insight] Transactions won't be included on Celestia when the gas price is high, and the transactions on Movement will be forgotten

**Submitted on Mar 29th 2025 at 09:26:29 UTC by @Franfran for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42925
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/providers/celestia>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The blob submission transaction only sets the minimum gas price, which will potentially prevent the transaction from being published by proposers, censoring all transaction on Movement in case of heavy load on Celestia.

## Vulnerability Details

The transaction gas fees are only set to the minimum [when the transaction is constructed](https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/celestia/src/da/mod.rs#L44), meaning that the gas price will be set to the node-defined `minGasPrice` as explained in the [Default trait implementation](broken://pages/AhVeFfnIhd491tgxKT5i) of the `TxConfig` struct. By default, nodes have a `minGasPrice` parameter set to `0.002`, meaning that the blob could be here sitting in the mempool of the Celestia node for a some time before being evicted, since the gas price might be higher and a Celestia proposer won't accept the blob with this few gas price paid, because it will give priority to higher gas bids as they fit in the block space. Indeed, the [docs](https://docs.celestia.org/how-to-guides/submit-data#submitting-multiple-transactions-in-one-block-from-the-same-account) state that a transaction that hasn't been included in 5 blocks is dropped. That means that if the gas market price is over `minGasPrice` for more than 5 blocks, all transactions sent in the Movement namespace will be censored.

## Impact Details

Under network load, all transactions from Movement will be censored.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/celestia/src/da/mod.rs#L44\\>
<https://docs.celestia.org/how-to-guides/submit-data#submitting-multiple-transactions-in-one-block-from-the-same-account>

## Proof of Concept

## Proof of Concept

The last 24 hours average gas price can be visualized on the [Celenium explorer](https://celenium.io/gas), and as you can see having an average gas price that is over the `minGasPrice` is not trivial, especially in some heavy load cases. This screenshot was taken on 29/03.![Image](https://github-production-user-asset-6210df.s3.amazonaws.com/51274081/428262126-456e5c63-29c6-4977-ac4f-255632b47871.png?X-Amz-Algorithm=AWS4-HMAC-SHA256\&X-Amz-Credential=AKIAVCODYLSA53PQK4ZA%2F20250329%2Fus-east-1%2Fs3%2Faws4_request\&X-Amz-Date=20250329T043113Z\&X-Amz-Expires=300\&X-Amz-Signature=59595c2cdc9b640982e42c9a903999e49990f047835b54713258e6794390659f\&X-Amz-SignedHeaders=host)\
Let's look at few recent transactions, and calculate their gas price by doing the following calculation:$$gas\_{price} = \frac{fee\_{$TIA} \times 10^{6}}{gas\_{used}}$$

### 1. [c80e25a226a771ac763ae861f86852350ea09e0a0fffb81dee3179e6d1a60ee7](https://celenium.io/tx/c80e25a226a771ac763ae861f86852350ea09e0a0fffb81dee3179e6d1a60ee7?page=1)

$$\frac{0.020406 \times 10^6}{9258357} = 0.0022$$

### 2. [96b8fabe9461eb9c030465a1c3c6a34fd6271e512f6f2ba7c504df9c1db64f0d](https://celenium.io/tx/96b8fabe9461eb9c030465a1c3c6a34fd6271e512f6f2ba7c504df9c1db64f0d?tab=messages)

$$\frac{0.026958 \times 10^6}{12236156} = 0.0022$$

### 3. [d10fa57d435f5864081d271a924e97b8d343a74cf814ea7371c2a5beda9d8eb4](https://celenium.io/tx/d10fa57d435f5864081d271a924e97b8d343a74cf814ea7371c2a5beda9d8eb4?tab=messages)

$$\frac{0.036655 \times 10^6}{16644159} = 0.0022$$

### 4. [d76c0581151956acb6e4da651e75fce65b0bb86b811b3a9173e1a3f206c433db](https://celenium.io/tx/d76c0581151956acb6e4da651e75fce65b0bb86b811b3a9173e1a3f206c433db?tab=messages)

$$\frac{0.00016 \times 10^6}{64981} = 0.0025$$

### 5. [791a7a9dedc2f99851668433af4fb3e5741e48c57796ce4463992128c3ed02b1](https://celenium.io/tx/791a7a9dedc2f99851668433af4fb3e5741e48c57796ce4463992128c3ed02b1?tab=messages)

$$\frac{0.00016 \times 10^6}{64956} = 0.0025$$

### 6. [c46d1ac4892da31cf2ae6dd83e3351218ef70ed510c8c7be2654436201ee171e](https://celenium.io/tx/c46d1ac4892da31cf2ae6dd83e3351218ef70ed510c8c7be2654436201ee171e?tab=messages)

$$\frac{0.001224 \times 10^6}{428637} = 0.0029$$\
It is clear that none of them use the `minGasPrice` as Movement does.
