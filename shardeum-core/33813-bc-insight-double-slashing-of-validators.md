# 33813 - \[BC - Insight] Double slashing of validators

Submitted on Jul 30th 2024 at 03:08:19 UTC by @Lastc0de for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33813

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Direct loss of funds

## Description

## Brief/Intro

Slashing mechanism for POS blockchains is a critical feature. Its implementation must have a strong test suite.

Shardeum has the concept of a penalty, to slash nodes if they have malicious behaviour. For example if a node goes offline while it is in active mode, and stops processing transactions, then other nodes confirm it is a lost node and apply a configurable penalty to that nodes staked tokens. But there is a bug in the shardeum repository which causes the network to apply a penalty 2x of configured value. So when penalty of leaving network early is configured to be 20% of staked value, if a validator leaves it loses 40% of its stake.

I will explain the cause of the bug here and provide a POC after.

## Vulnerability Details

One way of getting a penalty is to leave network early, in the `shardeum/shardus-core` repository if a node removed because it lefts network early, a `node-left-early` event would be triggered

*src/p2p/NodeList.ts*

```typescript
if (isNodeLeftNetworkEarly(node)) {
    const emitParams: Omit<ShardusEvent, 'type'> = {
    nodeId: node.id,
    reason: 'Node left early',
    time: cycle.start,
    publicKey: node.publicKey,
    cycleNumber: cycle.counter,
    }
    emitter.emit('node-left-early', emitParams)
}
```

this event is handled in `src/shardus/index.ts` file of same repository

*src/shardus/index.ts*

```typescript
Self.emitter.on('node-left-early', ({ ...params }) => {
    try {
    if (!this.stateManager.currentCycleShardData) throw new Error('No current cycle data')
    if (params.publicKey == null) throw new Error('No node publicKey provided for node-left-early event')
    const consensusNodes = this.getConsenusGroupForAccount(params.publicKey)
    for (let node of consensusNodes) {
        if (node.id === Self.id) {
        this.app.eventNotify?.({ type: 'node-left-early', ...params })
        }
    }
    } catch (e) {
    this.mainLogger.error(`Error: while processing node-left-early event stack: ${e.stack}`)
    }
})
```

and it calls `eventNotify` function of app which is a method that defined in another repository `shardeum/shardeum`

*src/index.ts*

```typescript
else if (
eventType === 'node-left-early' &&
ShardeumFlags.enableNodeSlashing === true &&
ShardeumFlags.enableLeftNetworkEarlySlashing
) {
    let nodeLostCycle
    let nodeDroppedCycle
    for (let i = 0; i < latestCycles.length; i++) {
        const cycle = latestCycles[i]
        if (cycle == null) continue
        if (cycle.apoptosized.includes(data.nodeId)) {
        nodeDroppedCycle = cycle.counter
        } else if (cycle.lost.includes(data.nodeId)) {
        nodeLostCycle = cycle.counter
        }
    }
    if (nodeLostCycle && nodeDroppedCycle && nodeLostCycle < nodeDroppedCycle) {
        const violationData: LeftNetworkEarlyViolationData = {
        nodeLostCycle,
        nodeDroppedCycle,
        nodeDroppedTime: data.time,
        }
        nestedCountersInstance.countEvent('shardeum-staking', `node-left-early: injectPenaltyTx`)

        await PenaltyTx.injectPenaltyTX(shardus, data, violationData)
    } else {
        nestedCountersInstance.countEvent('shardeum-staking', `node-left-early: event skipped`)
        /* prettier-ignore */ if (logFlags.dapp_verbose) console.log(`Shardeum node-left-early event skipped`, data, nodeLostCycle, nodeDroppedCycle)
    }
}
```

which calls the `injectPenaltyTX` function. `injectPenaltyTX` function creates an *internal penalty transactions* and puts into shardus by calling `shardus.put` function

*src/tx/penalty/transaction.ts*

```typescript
export async function injectPenaltyTX(
  shardus: Shardus,
  eventData: ShardusTypes.ShardusEvent,
  violationData: LeftNetworkEarlyViolationData | NodeRefutedViolationData | SyncingTimeoutViolationData
): Promise<{
  success: boolean
  reason: string
  status: number
}> {
  let violationType: ViolationType
  if (eventData.type === 'node-left-early') violationType = ViolationType.LeftNetworkEarly
  else if (eventData.type === 'node-refuted') violationType = ViolationType.NodeRefuted
  else if (eventData.type === 'node-sync-timeout') violationType = ViolationType.SyncingTooLong
  const unsignedTx = {
    reportedNodeId: eventData.nodeId,
    reportedNodePublickKey: eventData.publicKey,
    operatorEVMAddress: '',
    timestamp: shardeumGetTime(),
    violationType,
    violationData,
    isInternalTx: true,
    internalTXType: InternalTXType.Penalty,
  }

  const wrapeedNodeAccount: ShardusTypes.WrappedDataFromQueue = await shardus.getLocalOrRemoteAccount(
    unsignedTx.reportedNodePublickKey
  )

  if (!wrapeedNodeAccount) {
    return {
      success: false,
      reason: 'Penalty Node Account not found',
      status: 404,
    }
  }

  if (wrapeedNodeAccount && isNodeAccount2(wrapeedNodeAccount.data)) {
    unsignedTx.operatorEVMAddress = wrapeedNodeAccount.data.nominator
  } else {
    return {
      success: false,
      reason: 'Operator address could not be found for penalty node',
      status: 404,
    }
  }

  // to make sure that differnt nodes all submit an equivalent unsignedTx that is counted as the same unsignedTx,
  // we need to make sure that we have a determinstic timestamp
  const cycleEndTime = eventData.time
  let futureTimestamp = cycleEndTime * 1000
  while (futureTimestamp < shardeumGetTime()) {
    futureTimestamp += 30 * 1000
  }
  unsignedTx.timestamp = futureTimestamp

  const signedTx = shardus.signAsNode(unsignedTx) as PenaltyTX
  const txId = generateTxId(unsignedTx)
  // store the unsignedTx to local map for later use
  recordPenaltyTX(txId, signedTx)

  // Limit the nodes that send this to the <ShardeumFlags.numberOfNodesToInjectPenaltyTx> closest to the node address ( publicKey )
  const closestNodes = shardus.getClosestNodes(
    eventData.publicKey,
    ShardeumFlags.numberOfNodesToInjectPenaltyTx
  )
  const ourId = shardus.getNodeId()
  const isLuckyNode = closestNodes.some((nodeId) => nodeId === ourId)
  if (!isLuckyNode) {
    if (ShardeumFlags.VerboseLogs)
      console.log(`injectPenaltyTX: not lucky node, skipping injection`, signedTx)
    return
  }
  const waitTime = futureTimestamp - shardeumGetTime()
  // since we have to pick a future timestamp, we need to wait until it is time to submit the signedTx
  await sleep(waitTime)

  if (ShardeumFlags.VerboseLogs) {
    console.log(`injectPenaltyTX: tx.timestamp: ${signedTx.timestamp} txid: ${txId}`, signedTx)
  }

  const result = await shardus.put(signedTx)
  /* prettier-ignore */ if (logFlags.dapp_verbose) console.log('INJECTED_PENALTY_TX', result)
  return result
}
```

`shardus.put` is defined in the `shardeum/shardus-core` repository and it validates and sends transactions to queue for execution. After the queue, transactions are passed to the `apply` function in the `shardeum/shardeum` repository for execution. `apply` function would pass the created *internal* transaction to the `applyInternalTx` function.

*src/index.ts*

```typescript
async apply(timestampedTx: ShardusTypes.OpaqueTransaction, wrappedStates, originalAppData) {
      //@ts-ignore
      const { tx } = timestampedTx
      const txTimestamp = getInjectedOrGeneratedTimestamp(timestampedTx)
      const appData = fixBigIntLiteralsToBigInt(originalAppData)
      // Validate the tx
      const { result, reason } = this.validateTransaction(tx)
      if (result !== 'pass') {
        throw new Error(`invalid transaction, reason: ${reason}. tx: ${Utils.safeStringify(tx)}`)
      }

      if (isInternalTx(tx)) {
        return applyInternalTx(tx, wrappedStates, txTimestamp)
      }

```

and `applyInternalTx` function would pass it to `applyPenaltyTx` function

*src/index.ts*

```typescript
async function applyInternalTx(
  tx: InternalTx,
  wrappedStates: WrappedStates,
  txTimestamp: number
): Promise<ShardusTypes.ApplyResponse> {
...
  if (internalTx.internalTXType === InternalTXType.Penalty) {
    const penaltyTx = internalTx as PenaltyTX
    applyPenaltyTX(shardus, penaltyTx, wrappedStates, txId, txTimestamp, applyResponse)
  }
  return applyResponse
}
```

this `applyPenaltyTx` function calculated amount of penalty, and calls `applyPenalty` function to apply *penaltyAmount* to validator account.

*src/tx/penalty/transaction.ts*

```typescript
export async function applyPenaltyTX(
  shardus,
  tx: PenaltyTX,
  wrappedStates: WrappedStates,
  txId: string,
  txTimestamp: number,
  applyResponse: ShardusTypes.ApplyResponse
): Promise<void> {
...
  //TODO should we check if it was already penalized?
  const penaltyAmount = getPenaltyForViolation(tx, nodeAccount.stakeLock)
  applyPenalty(nodeAccount, operatorAccount, penaltyAmount)
  nodeAccount.nodeAccountStats.penaltyHistory.push({
    type: tx.violationType,
    amount: penaltyAmount,
    timestamp: eventTime,
  })
...
```

`applyPenalty` function does some calculations and some modifications

*src/tx/penalty/penaltyFunctions.ts*

```typescript
export function applyPenalty(
  nodeAccount: NodeAccount2,
  operatorEOA: WrappedEVMAccount,
  penalty: bigint
): boolean {
  /* prettier-ignore */ if (logFlags.dapp_verbose) console.log(`\nApplying Penalty on Node: ${nodeAccount.id} of ${penalty.toString()} SHM`)

  // convert hex value to BN
  operatorEOA.operatorAccountInfo.stake = _base16BNParser(operatorEOA.operatorAccountInfo.stake)
  operatorEOA.operatorAccountInfo.operatorStats.totalNodePenalty = _base16BNParser(
    operatorEOA.operatorAccountInfo.operatorStats.totalNodePenalty
  )
  nodeAccount.stakeLock = _base16BNParser(nodeAccount.stakeLock)
  nodeAccount.penalty = _base16BNParser(nodeAccount.penalty)
  nodeAccount.nodeAccountStats.totalPenalty = _base16BNParser(nodeAccount.nodeAccountStats.totalPenalty)

  if (penalty > nodeAccount.stakeLock) penalty = nodeAccount.stakeLock

  // update operator account
  operatorEOA.operatorAccountInfo.stake -= penalty
  operatorEOA.operatorAccountInfo.operatorStats.totalNodePenalty += penalty

  // update node account
  nodeAccount.stakeLock -= penalty
  nodeAccount.penalty += penalty
  nodeAccount.nodeAccountStats.totalPenalty += penalty
  return true
}
```

root cause of this bug is because *penalty* is reduced from stake of validator account here

```typescript
operatorEOA.operatorAccountInfo.stake -= penalty
...
nodeAccount.stakeLock -= penalty
```

so if other parts of application wants to do some calculation based on original stake, it is so easy to mistake this *stake* field by original stake of validator, **which is happened in unstake transaction**.

Again in `apply` function of `shardeum/shardeum` repository, we have below implementation for unstake transaction

*src/index.ts*

```typescript
async apply(timestampedTx: ShardusTypes.OpaqueTransaction, wrappedStates, originalAppData) {
...
    if (appData.internalTx && appData.internalTXType === InternalTXType.Unstake) {
        nestedCountersInstance.countEvent('shardeum-unstaking', 'applying unstake transaction')
        if (ShardeumFlags.VerboseLogs) console.log('applying unstake tx', wrappedStates, appData)

        // get unstake tx from appData.internalTx
        const unstakeCoinsTX: UnstakeCoinsTX = appData.internalTx

        // todo: validate tx timestamp, compare timestamp against account's timestamp

        // set stake value, nominee, cert in OperatorAcc (if not set yet)
        const operatorShardusAddress = toShardusAddress(unstakeCoinsTX.nominator, AccountType.Account)
        const nomineeNodeAccount2Address = unstakeCoinsTX.nominee
        // eslint-disable-next-line security/detect-object-injection
        const operatorEVMAccount: WrappedEVMAccount = wrappedStates[operatorShardusAddress]
            .data as WrappedEVMAccount
        operatorEVMAccount.timestamp = txTimestamp

        if (operatorEVMAccount.operatorAccountInfo == null) {
            nestedCountersInstance.countEvent(
            'shardeum-unstaking',
            'unable to apply unstake tx, operator account info does not exist'
            )
            throw new Error(
            `Unable to apply Unstake tx because operator account info does not exist for ${unstakeCoinsTX.nominator}`
            )
        } else {
            operatorEVMAccount.operatorAccountInfo = fixBigIntLiteralsToBigInt(
            operatorEVMAccount.operatorAccountInfo
            )
        }
        fixDeserializedWrappedEVMAccount(operatorEVMAccount)

        if (
            operatorEVMAccount.operatorAccountInfo.certExp > txTimestamp &&
            ShardeumFlags.unstakeCertCheckFix
        ) {
            throw new Error(
            `Unable to apply Unstake tx because stake cert has not yet expired. Expiry timestamp ${operatorEVMAccount.operatorAccountInfo.certExp}`
            )
        }

        // eslint-disable-next-line security/detect-object-injection
        const nodeAccount2: NodeAccount2 = wrappedStates[nomineeNodeAccount2Address].data as NodeAccount2

        const currentBalance = operatorEVMAccount.account.balance
        const stake = BigInt(operatorEVMAccount.operatorAccountInfo.stake)
        let reward = BigInt(nodeAccount2.reward)
        const penalty = BigInt(nodeAccount2.penalty)
        const txFeeUsd = BigInt(ShardeumFlags.constantTxFeeUsd)
        const txFee = scaleByStabilityFactor(txFeeUsd, AccountsStorage.cachedNetworkAccount)
        /* prettier-ignore */ if (logFlags.dapp_verbose) console.log('calculating new balance after unstake', currentBalance, stake, reward, penalty, txFee)
        if (nodeAccount2.rewardEndTime === 0 && nodeAccount2.rewardStartTime > 0) {
            // This block will only be reached if the node is inactive and the force unstake flag has been set
            reward = BigInt(0)

            /* prettier-ignore */ if (logFlags.dapp_verbose) console.log('discarding staking rewards due to zero rewardEndTime')
        }
        const newBalance = currentBalance + stake + reward - penalty - txFee
        operatorEVMAccount.account.balance = newBalance
        operatorEVMAccount.account.nonce = operatorEVMAccount.account.nonce + BigInt(1)
...
    }
...
}
```

It sets *stake* variable to current stake value, *penalty* variable to current penalty value, and when it calculates *newBalance* it reduces penalty from stake, **AGAIN**.

## Impact Details

This bug could affect direct loss for any validator

## References

Add any relevant links to documentation or code

## Proof of Concept

To see how the validator penalty applies twice we have to run a network. Almost all of what we do below is from README of different repositories of shardeum. If you have a running network just make sure step 3 and 4 is configured and skip to step 16.

1. clone repositories

```bash
git clone --depth=1 git@github.com:shardeum/shardeum.git
git clone --depth=1 git@github.com:shardeum/json-rpc-server.git rpc
git clone --depth=1 https://github.com/shardeum/validator-dashboard.git validator01
git clone --depth=1 https://github.com/shardeum/validator-dashboard.git validator02
```

2. install metamask on your browser and create two accounts validator01, validator02. Copy their address and open `shardeum/src/config/genesis.json` and append them to the list of genesis accounts.
3. open `shardeum/src/config/index.ts` and change below values

```js
    cycleDuration: 30,
    baselineNodes: 10, // this is to allow network to process transactions with 10 not 300 node, thanks to Schnilch at discord
    minNodes: 10, // this is to allow network to process transactions with 10 not 300 node, thanks to Schnilch at discord
    forceBogonFilteringOn: false,
    mode: "debug",
```

4. open `shardeum/src/shardeum/shardeumFlags.ts` and change below values

```js
    penaltyPercent: 0.2, // it means 20% penalty to stake
    blockProductionRate: 3,
    enableLeftNetworkEarlySlashing: true,
    enableNodeRefutedSlashing: true,
```

5. go to the `shardeum` folder and execute `npm install && npm prepare`
6. install shardus command line tool with `npm install -g shardus && npm update @shardus/archiver`
7. start a local network by `shardus start 10`. it spins up a network with 10 nodes and an archiver. run `shardus pm2 logs` and wait until all 10 nodes are in *active* mode.
8. Now we have to run our validator and stake some SHM. go to `validator01` folder and change *docker-compose.yaml* to

```yaml
version: '3.4'

services:
  shardeum-dashboard01:
    container_name: shardeum-dashboard01
    # network_mode: 'host'
    image: ghcr.io/shardeum/server:latest
    working_dir: /home/node/app
    entrypoint: /home/node/app/entrypoint.sh
    volumes:
      - './:/home/node/app:Z'
      - '../validator:/usr/src/app:Z'
    network_mode: host
    env_file: .env
```

9. create a hashed password with `echo password | openssl dgst -sha256 -r` (which is what installer.sh does)
10. add a `.env` file with below content (replace 192.168.1.100 with your computer ip, and replace DASHPASS value with previous step output)

```env
EXT_IP=auto
INT_IP=auto
EXISTING_ARCHIVERS=[{"ip":"192.168.1.100","port":4000,"publicKey":"758b1c119412298802cd28dbfa394cdfeecc4074492d60844cc192d632d84de3"}]
APP_MONITOR=192.168.1.100
DASHPASS=a9d2d1355c90e24d1676e27bfb4b7a37ab8a4041c44f26bfac538a453c46355c
DASHPORT=9030
SERVERIP=192.168.1.100
LOCALLANIP=192.168.1.100
SHMEXT=19001
SHMINT=11001
RPC_SERVER_URL=http://json-rpc-server:18080
RUNDASHBOARD=y
```

11. replace entrypoint.sh with below text

```
#!/usr/bin/env bash
ln -s /usr/src/app /home/node/app/validator

echo "Install PM2"

npm i -g pm2

# Pull latest versions of the CLI and GUI

git clone https://github.com/shardeum/validator-cli.git cli

echo "Install the CLI"
cd cli
npm i --silent
npm link
cd ..

git clone https://github.com/shardeum/validator-gui.git gui

echo "Install the GUI"
cd gui
npm i --silent
npm run build
#openssl req -x509 -nodes -days 99999 -newkey rsa:2048 -keyout ./selfsigned.key -out selfsigned.crt -subj "/C=US/ST=Texas/L=Dallas/O=Shardeum/OU=Shardeum/CN=shardeum.org"

# if CA.cnf does not exist, create it
if [ ! -f "CA.cnf" ]; then
    echo "[ req ]
prompt = no
distinguished_name = req_distinguished_name

[ req_distinguished_name ]
C = XX
ST = Localzone
L = localhost
O = Certificate Authority Local Validator Node
OU = Develop
CN = mynode-sphinx.sharedum.local
emailAddress = community@.sharedum.local" > CA.cnf
fi

# if CA.key does not exist, create it
if [ ! -f "CA_key.pem" ]; then
    openssl req -nodes -new -x509 -keyout CA_key.pem -out CA_cert.pem -days 1825 -config CA.cnf
fi

# if selfsigned.cnf does not exist, create it
if [ ! -f "selfsigned.cnf" ]; then
    echo "[ req ]
default_bits  = 4096
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
countryName = XX
stateOrProvinceName = Localzone
localityName = Localhost
organizationName = Shardeum Sphinx 1.x Validator Cert.
commonName = localhost

[req_ext]
subjectAltName = @alt_names

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = $SERVERIP
IP.2 = $LOCALLANIP
DNS.1 = localhost" > selfsigned.cnf
fi

# if csr file does not exist, create it
if [ ! -f "selfsigned.csr" ]; then
    openssl req -sha256 -nodes -newkey rsa:4096 -keyout selfsigned.key -out selfsigned.csr -config selfsigned.cnf
fi

# if selfsigned.crt does not exist, create it
if [ ! -f "selfsigned_node.crt" ]; then
    openssl x509 -req -days 398 -in selfsigned.csr -CA CA_cert.pem -CAkey CA_key.pem -CAcreateserial -out selfsigned_node.crt -extensions req_ext -extfile selfsigned.cnf
fi
# if selfsigned.crt does not exist, create it
if [ ! -f "selfsigned.crt" ]; then
  cat selfsigned_node.crt CA_cert.pem > selfsigned.crt
fi
cd ../..

# Start GUI if configured to in env file
echo $RUNDASHBOARD
if [ "$RUNDASHBOARD" == "y" ]
then
echo "Starting operator gui"
# Call the CLI command to set the GUI password
operator-cli gui set password -h $DASHPASS
# Call the CLI command to set the GUI port
operator-cli gui set port $DASHPORT
# Call the CLI command to start the GUI
operator-cli gui start
fi

# Deprecated
# operator-cli set external_port $SHMEXT
# operator-cli set internal_port $SHMINT

echo "done";

# Keep container running
tail -f /dev/null

```

12. replace Dockerfile content with below text

```
FROM ghcr.io/shardeum/server:latest

ARG RUNDASHBOARD=y
ENV RUNDASHBOARD=${RUNDASHBOARD}

# Copy cli src files as regular user
WORKDIR /home/node/app
COPY --chown=node:node . .

# RUN ln -s /usr/src/app /home/node/app/validator

# Start entrypoint script as regular user
CMD ["./entrypoint.sh"]
```

13. apply step 8 to 12 in `validator02` folder, only in `docker-compose.yml` file instead of *validator01* use *validator02*
14. Now we have to run a json-rpc-server. To do so go to the `rpc` folder and open `archiverConfig.json` and change *ip* to `192.168.1.100` (or your computer ip address) and run `docker compose up`. It should run and connect to the archive node without problem.
15. Now go to the `validator01` folder and start validator by `docker compose up`. After a while you would see *done* output and can open the dashboard in your browser at `https://localhost:9030`. Then login with your password and go to the Maintenance tab.
16. In the validator01 dashboard, click on Start button to start validator node. After a few seconds in the terminal that you executed `shardus pm2 logs` there must be a *standby* node which is this *validator01* that we started.
17. Open Metamask and add a new network

| Field                         | Details                          |
| ----------------------------- | -------------------------------- |
| Network Name                  | Shardeum Atomium                 |
| New RPC URL                   | <http://localhost:8080>          |
| Chain ID                      | 8082                             |
| Currency Symbol               | SHM                              |
| Block Explorer URL (optional) | none or <http://localhost:6001/> |

18. Now change your metamask network to *Shardeum Atomium* and you must be able to see values that we set for *validator01* and *validator02* accounts in the `genesis.json` as balances of these accounts.
19. In the Maintenance tab of validator01 dashboard, click add stake. It opens Metamask, connects *validator01* account then set for example 100 as stake value, click on add stake again and confirm transaction in Metamask, when the transaction confirmed node goes to *sync* mode and after a while, it goes to *active*!. You can see in Metamask that the account now has a send transaction of 100 and its balance is reduced by \~101 tokens, that 1 is because of the fee. In `shardus pm2 logs` we can see after a few seconds, the network kicks an old node and resizes its active size to be 10 which is cool. And the removed node would go to *standby* mode.
20. stop validator01 docker container to simulate an early left node. After a few seconds in `shardus pm2 logs` you should see active nodes become 9 again.
21. Now go to the `validator02` folder and start it like validator01, go to its dashboard, stake some token and wait for the node to go to *active* mode. We add this seconds validator so network can process our unstake transaction. network size must be at least 10 to process application transactions. Although there is a standby node which started by `shardus start 10` but it does not go to *active* mode again, i think because it does not stake anything. Anyway, now that we have 10 active node again, we can send an unstake transaction to the network.
22. go to `validator01` folder and execute this command

```bash
RPC_SERVER_URL=http://localhost:8080 node ./cli/build/src/index.js unstake
```

it asks for a private key, go to Metamask and open validator01 account and copy it's private key, and paste it here.

23. After transaction confirmed you can see in Metamask that instead of 80, 60 tokens are returned to your account. Which is 40% penalty.
