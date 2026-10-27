# #37104 \[BC-Insight] Reth RPC is vulnerable to DNS rebinding attacks

**Submitted on Nov 25th 2024 at 15:53:42 UTC by @alpharush for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37104
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/paradigmxyz/reth>
* **Impacts:**
  * Shutdown of less than 10% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

Reth's RPC does not check the host origin of RPC requests which allows DNS rebinding attacks. Note, this does not require a user configuring CORS and the default configuration is vulnerable as DNS rebinding bypasses same origin policies.

## Vulnerability Details

A user opens a malicious website on the same machine as they are running a Reth node that has not exposed its RPC to the internet. However, by responding to DNS queries with 127.0.0.1 instead of the server's IP, the website is able to send fetch requests to 127.0.0.1:8545, bypassing same origin policy.

This is due to the lack of a middleware that checks the requests' hostname\
(<https://github.com/paradigmxyz/reth/blob/422ab1735407c8e9de8ffa24adb416132d41f351/crates/rpc/rpc-builder/src/lib.rs#L1577-L1613).\\>
By default, it should only accept localhost.

## Impact Details

A website can add and remove peers to eclipse the node or exhaust its resources using debug endpoints that aren't intended to be remotely accessible by unprivileged users.

## References

Go-ethereum mitigates this by adding a virtual host flag and checking HTTP "Host" headers\
<https://github.com/ethereum/go-ethereum/pull/15962\\>
Reth could use something like\
<https://github.com/iamsauravsharma/tower\\_allowed\\_hosts>

## Proof of Concept

## Proof of Concept

Run `reth node --chain dev --http --http.api all`

Run `curl http://localhost:8545 \ -X POST \ -H "Content-Type: application/json" \ -H "Host: https://0xalpharush.github.io/" \ -d '{"jsonrpc":"2.0","id":1,"method":"admin_addTrustedPeer","params":["enode://a979fb575495b8d6db44f750317d0f4622bf4c2aa3365d6af7c284339968eef29b69ad0dce72a4d8db5ebb4968de0e3bec910127f134779fbcb0cb6d3331163c@52.16.188.185:30303"]}'`

The node responds even though the request isn't from 127.0.0.1:

```
{"jsonrpc":"2.0","id":1,"result":true}
```
