# #36025 \[W\&A-Critical] A malicious validator can overwrite the account data of any archive server con

**Submitted on Oct 16th 2024 at 05:55:23 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #36025
* **Report Type:** Websites and Applications
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/archive-server/tree/dev>
* **Impacts:**
  * Direct theft of user funds

## Description

## Brief/Intro

When any active archive server is connected to a malicious validator to receive data updates, the attacker can overwrite any account data on this archiver

## Vulnerability Details

The archive server places too much trust in the data it receives from the source validator. It only performs data format validation, but lacks data verification and account ownership checks. As a result, if an active archive server connects to a malicious validator, the validator can overwrite any account data on it. If data of type RECEIPT is received, the storeReceiptData function will be called, which performs basic format validation by calling validateArchiverReceipt function. \`\`\` if (newData.responses && newData.responses.RECEIPT) { if (config.VERBOSE) Logger.mainLogger.debug( 'RECEIPT', sender.nodeInfo.publicKey, sender.nodeInfo.ip, sender.nodeInfo.port, newData.responses.RECEIPT.length ) storeReceiptData( newData.responses.RECEIPT, sender.nodeInfo.ip + ':' + sender.nodeInfo.port, true, config.saveOnlyGossipData ) } \`\`\` It is important to set the globalModification flag to true, as this will allow the remaining validation to be skipped.: \`\`\` if (receipt.globalModification) return true \`\`\` After the validation is complete, the account provided in the receipt will be updated with the data included in the same receipt: \`\`\` for (const account of afterStates) { const accObj: Account.AccountsCopy = { accountId: account.accountId, data: account.data, timestamp: account.timestamp, hash: account.hash, cycleNumber: cycle, isGlobal: account.isGlobal || false, } if (account.timestamp !== account.data\['timestamp']) Logger.mainLogger.error('Mismatched account timestamp', txId, account.accountId) if (account.hash !== account.data\['hash']) Logger.mainLogger.error('Mismatched account hash', txId, account.accountId)

```
  const accountExist &#x3D; await Account.queryAccountByAccountId(account.accountId)
  if (accountExist) {
    if (accObj.timestamp &gt; accountExist.timestamp) await Account.updateAccount(accObj)
  } else {
    // await Account.insertAccount(accObj)
    combineAccounts.push(accObj)
  }

  //check global network account updates
  if (accObj.accountId &#x3D;&#x3D;&#x3D; config.globalNetworkAccount) {
    setGlobalNetworkAccount(accObj)
  }
```

...... } \`\`\` There is no check for votes; an attacker can even overwrite the global network account, which contains the entire network configuration.

## Impact Details

Any account data on the archive server can be changed if it is connected to a malicious validator via a socket.io connection. If combined with my report 36021, the attacker can change account data on all archivers.

## References

<https://github.com/shardeum/archive-server/blob/0337daa477b3a30f8fb65b87c23b021a261441bd/src/Data/Data.ts#L211> <https://github.com/shardeum/archive-server/blob/0337daa477b3a30f8fb65b87c23b021a261441bd/src/Data/Collector.ts#L765>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/c9f1b1c10eed42e07d3e4980f1199205>

## Proof of Concept

1. **Enable the VERBOSE flag** in \`src/Config.ts\` of the archive-server repository.
2. **Apply \`validator.patch\`** to all non-attacker validators in the \`shardus-core\` repository. This will reduce the wait time (for POC) until the archive server connects to the malicious validator. Otherwise, you will need rotation enabled.
3. **Start the local network.**
4. **Apply \`attacker.patch\`** to the malicious code repository (\`shardus-core\`).
5. **Start the malicious node** and wait until it becomes active.
6. **Check \`main.log\`** of the archive-server. When it connects to the malicious validator, there will be a message like: \`\`\` RECEIPT \<publicKey> \<ip> \<port> 0 \`\`\`
7. **Run the attack** by executing the following command: \`\`\`shell curl '<http://ATTACKER\\_IP:ATTACKER\\_PORT/attack?id=1234>' \`\`\` The value of the \`id\` parameter can be any random string.
8. **Verify that the account was updated** using the SQLite client for \`accountId\` \`1000000000000000000000000000000000000000000000000000000000000001\`.
