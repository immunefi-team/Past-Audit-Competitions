# #41368 \[BC-High] RPC server takedown

**Submitted on Mar 14th 2025 at 12:30:50 UTC by @keizo for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41368
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/docker/compose/movement-full-node/docker-compose.celestia-mainnet.yml>
* **Impacts:**
  * RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

## Brief/Intro

There is no batch query limitation on the Celestia node RPC which allows to send hundreds of large RPC queries in a single request. This leads to excessive use of resources and takedown of the RPC service.

## Vulnerability Details

Ín the local testing environment the RPC method "blockchain" returns \~25,000 bytes of data. This is not a lot, but when this is combined with the fact that there is no batch query limitation in place, will it lead to serious consequences. For example if user enters 100 times this same query as a batch query inside a single request, leads it to response size of 2.5 MB. Normal response delay is under 10 ms since it is local environment so keep this in mind.

Always when the server has to return lot of data and the delay starts to grow, causes it unnecessarily large load to the server.

During the testing the amount of batch queries were increased:

* ***1000x*** batch queries resulted in ***25.5 MB*** response with a response delay of ***1 sec***. Request size ***109,000 bytes***
* ***10,000x*** batch queries resulted in ***255 MB*** response with a response delay of ***11.5 sec***. Request size ***1 MB***
* ***50,000x*** batch queries resulted in i/o timeout error from the rpc server after a delay of ***56 sec***. Request size ***5 MB***. This also caused normal queries to have longer delays as normally (2ms --> 5 sec).

When this was taken little bit further by just sending the request with ***50,000x*** batch queries 3x requests in parallel, caused this so large load to the server that it crashed and did not boot itself back online.

## Impact Details

This could be utilized to cause major availability issues to the RPC and the node. Since it requires only very limited amount of requests to be sent by the attacker to cause the impact described, makes it really hard to be detected.

## Link to Proof of Concept

<https://gist.github.com/patuuh/4035799a8bb8eddb140fb199dab26bba>

## Proof of Concept

## Proof of Concept

1. Send request (add rpc ip to host header):

```
POST / HTTP/1.1
Host: <YOUR-RPC-IP-HERE>
Content-Type: application/json
Content-Length: 197

[{"jsonrpc":"2.0","method":"blockchain","params":{"minHeight":"0","maxHeight":"999999"},"id":"12"},{"jsonrpc":"2.0","method":"blockchain","params":{"minHeight":"0","maxHeight":"999999"},"id":"12"}]
```

2. See that you get normal response with \~50,000 bytes long response
3. Use the python script from the gist (<https://gist.github.com/patuuh/4035799a8bb8eddb140fb199dab26bba>) to generate and send single request with 10,000 RPC queries in a batch.
4. Add your RPC ip to line 152 `<YOUR_RPC_IP_HERE>` and run the code with command `python tester.py -v`
5. Observe 255 MB response and longer delay
6. From the line 162 (`threads = 1`) edit the amount of requests sent in parallel to 3
7. Run the code and observe large load caused to the RPC server
8. Continue increasing the amount of queries in a batch and the amount of parallel requests until impact is achieved.
