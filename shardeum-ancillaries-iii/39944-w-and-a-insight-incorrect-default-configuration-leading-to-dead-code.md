# 39944 \[W\&A-Insight] incorrect default configuration leading to dead code

## #39944 \[W\&A-Insight] Incorrect Default Configuration Leading to Dead Code

**Submitted on Feb 11th 2025 at 12:08:13 UTC by @Blockian for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39944
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Direct theft of user funds

### Description

## **Shardeum Ancillaries**

### **Incorrect Default Configuration Leading to Dead Code**

#### **Description**

Discussions with the Shardeum team revealed that certain key features of the Archiver are currently unused. This suggests that the default configuration is incorrect, preventing critical functionality from being executed.

#### **Example**

When receiving receipt data from validators, `storeReceiptData` is triggered. Depending on the Archiver's configuration, several execution flows are possible. The most important one involves calling `Utils.robustQuery` to verify that the receipt matches on at least five nodes within the execution group.

However, in the current default configuration, this flow is **inaccessible** due to the following reasons:

1. **`config.newPOQReceipt === false`**
   * During `verifyReceiptData`, execution enters the `if (config.newPOQReceipt === false)` block, preventing it from reaching `verifyReceiptMajority`.
2. **Even if `config.newPOQReceipt === true`**, execution does reach `verifyReceiptMajority`, but another issue arises:

```ts
  if (!config.useRobustQueryForReceipt) {
    return verifyReceiptOffline(receipt, executionGroupNodes, minConfirmations)
  }
  return verifyReceiptWithValidators(receipt, executionGroupNodes, minConfirmations)
```

* Since `config.useRobustQueryForReceipt` is **undefined**, `!undefined` evaluates to `true`, leading to `verifyReceiptOffline` being executed instead of `verifyReceiptWithValidators`.
* As a result, `robustQuery` is never called, making it effectively **dead code**.

#### **Impact**

By skipping a crucial step in receipt verification, attackers gain more flexibility to bypass validation checks and exploit vulnerabilities. For instance, similar issues were demonstrated in **report #39872**.

#### **Proposed Fix**

Update the default configuration to align with the intended behavior of the Archiver, ensuring `robustQuery` is executed as expected.

### Proof of Concept

## **Proof of Concept**

1. Apply the following `git diff` on the Archiver, we'll check the logs to see that this code isn't reached:

```diff
diff --git a/src/Data/Collector.ts b/src/Data/Collector.ts
index 610e0df..fd81d24 100644
--- a/src/Data/Collector.ts
+++ b/src/Data/Collector.ts
@@ -68,6 +68,7 @@ const verifyReceiptMajority = async (
   executionGroupNodes: ConsensusNodeInfo[],
   minConfirmations: number = config.RECEIPT_CONFIRMATIONS
 ): Promise<{ success: boolean; newReceipt?: Receipt.ArchiverReceipt | Receipt.Receipt }> => {
+  Logger.mainLogger.info("BLOCKIAN - DEAD CODE 2")
   /**
    * Note:
    * Currently, only the non-global receipt flow is implemented in `verifyReceiptMajority`,
@@ -133,6 +134,7 @@ const verifyNonGlobalTxReceiptWithValidators = async (
   executionGroupNodes: ConsensusNodeInfo[],
   minConfirmations: number = config.RECEIPT_CONFIRMATIONS
 ): Promise<{ success: boolean; newReceipt?: Receipt.ArchiverReceipt | Receipt.Receipt }> => {
+  Logger.mainLogger.info("BLOCKIAN - DEAD CODE 3")
   const result = { success: false }
   // Created signedData with full_receipt = false outside of queryReceipt to avoid signing the same data multiple times
   let signedData = Crypto.sign({
@@ -615,6 +617,8 @@ export const verifyReceiptData = async (
     return { success: true, requiredSignatures }
   }
 
+  Logger.mainLogger.info("BLOCKIAN - DEAD CODE 1")
+
   // const { confirmOrChallenge } = appliedReceipt as Receipt.AppliedReceipt2
   // // Check if the appliedVote node is in the execution group
   // if (!cycleShardData.nodeShardDataMap.has(appliedVote.node_id)) {
```

2. Add the following wallet to the `genesis.json` file:

```json
  "0xE0291324263D7EC15fa3494bFDc1e902d8bd5d3d": {
    "wei": "10000001000000000000000000"
  }
```

3. Run the following code to execute a transaction which will yield a receipt

```js
const ethers = require("ethers");

const ATTACKER_WALLET = {
  address: "0xE0291324263D7EC15fa3494bFDc1e902d8bd5d3d",
  privateKey: "0x759418c4f40e463452b15eda4b27478d152f2a2c04e6cd324fb620a9eede6021"
}
const JSON_RPC_URL = "http://127.0.0.1:8080";

const TARGET = "0xcb65445d84d15f703813a2829bd1fd836942c9b7";

const main = async () => {
  const provider = new ethers.JsonRpcProvider(JSON_RPC_URL);
  const wallet = new ethers.Wallet(ATTACKER_WALLET.privateKey, provider);

  const balance = await wallet.provider.getBalance(ATTACKER_WALLET.address);

  console.log(`balance: ${balance}`)

  const [feeData, nonce] = await Promise.all([ wallet.provider.getFeeData(), wallet.provider.getTransactionCount(wallet.address), ]);

  const tx = await wallet.sendTransaction({
          from: wallet.address,
          to: TARGET,
          gasPrice: feeData.gasPrice,
          gasLimit: 30000000,
          value: 2,
          nonce,
  });
  const receipt = await tx.wait();

  return receipt
}

main()
.then((v) => console.log(`receipt: ${v}`))
.catch((e) => console.log(e))
```

4. Inspect the Archiver logs and search for "BLOCKIAN". No logs will be found indicating the code is unreachable in the current state.
