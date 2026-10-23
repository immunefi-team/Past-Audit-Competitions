# 39893 \[W\&A-Critical] malicious validator can modify txid in global transactions

## #39893 \[W\&A-Critical] Malicious Validator Can Modify \`txId\` in Global Transactions

**Submitted on Feb 9th 2025 at 23:31:53 UTC by @Blockian for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39893
* **Report Type:** Websites and Applications
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as:
* Modifying transaction arguments or parameters
* Substituting contract addresses
* Submitting malicious transactions

### Description

## **Shardeum Ancillaries**

### **Malicious Validator Can Modify `txId` in Global Transactions**

#### **Description**

When submitting a Global Transaction (Global Tx) Receipt, the transaction ID (`txId`) is not included in the signed object that validators verify.

As a result, a malicious validator can alter the `txId` field without invalidating the receipt, allowing them to:

* Censor transactions by preventing specific `txId`s from being processed.

#### **Root Cause**

During signature verification in `verifyAppliedReceiptSignatures`, the `txId` is **not** part of the signed object for Global Transactions.

**Relevant Code Snippet:**

```ts
    const appliedReceipt = receipt.signedReceipt as P2PTypes.GlobalAccountsTypes.GlobalTxReceipt
    // Refer to https://github.com/shardeum/shardus-core/blob/7d8877b7e1a5b18140f898a64b932182d8a35298/src/p2p/GlobalAccounts.ts#L294

    const { signs, tx } = appliedReceipt

    // ... Irrelevant code omitted

    const acceptableSigners = new Set<P2PTypes.P2PTypes.Signature>()

    // ... Irrelevant code omitted

    // Using a map to store the good signatures to avoid duplicates
    const goodSignatures = new Map()
    for (const sign of acceptableSigners) {
      if (Crypto.verify({ ...tx, sign: sign })) {
```

Here, `tx` comes from `receipt.signedReceipt` and is used for validation. However, unlike `receipt.beforeStates` and `receipt.afterStates`—which are properly validated in `verifyGlobalTxAccountChange`—there is **no validation** ensuring that `receipt.tx` remains unchanged.

This oversight allows an attacker to modify `receipt.tx` while keeping the signature valid.

#### **Impact**

The primary risk is **transaction censorship**:

* An attacker can modify `txId`s for transactions not yet processed by the archiver.
* When the legitimate transaction is eventually submitted, it will be rejected due to the `txId` being used already.

#### **How to Censor**

The Attacker needs to change the `receipt.tx.txId` to the txId of the transaction they wish to censor and `receipt.tx.timestamp` to the timestamp of the transaction they wish to censor, thus when the real transaction will be submitted it will skip it and wont save it.

#### **Proposed Fix**

To prevent this exploit, add a validation step to ensure `receipt.tx` matches the values in `receipt.signedReceipt`.

### Proof of Concept

## **Proof of Concept**

Since we only wish to demonstrate that the Global Transaction will be processed as normal, the following minimal POC should be enough.

The first transaction the Archiver recives after starting a network is a Global Transaction, so all we need to do is Apply the following diff on the Archiver:

```diff
diff --git a/src/Data/Data.ts b/src/Data/Data.ts
index 7e92c59..77a4991 100644
--- a/src/Data/Data.ts
+++ b/src/Data/Data.ts
@@ -276,6 +276,9 @@ export function initSocketClient(node: NodeList.ConsensusNodeInfo): void {
               sender.nodeInfo.port,
               newData.responses.RECEIPT.length
             )
+          
+          if (newData.responses.RECEIPT && newData.responses.RECEIPT.length > 0) newData.responses.RECEIPT[0].tx.txId = "1000000000000000000000000000000000000000000000000000000000000011"
+          if (newData.responses.RECEIPT && newData.responses.RECEIPT.length > 0) newData.responses.RECEIPT[0].tx.timestamp = 2739055681599
           storeReceiptData(
             newData.responses.RECEIPT,
             sender.nodeInfo.ip + ':' + sender.nodeInfo.port,

```

Run the network as normal `LOAD_JSON_CONFIGS=debug-10-nodes.config.json shardus start 10` and see the Archiver processes the transaction as expected. For further information you can also check the sqlite3 db after the transaction is processed and look for `txId = "1000000000000000000000000000000000000000000000000000000000000011"`
