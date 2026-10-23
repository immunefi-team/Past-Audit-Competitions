# #35537 \[W\&A-Insight] json rpc server websocket remote crash

**Submitted on Sep 26th 2024 at 21:59:50 UTC by @gln for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35537
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/json-rpc-server/tree/dev>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

JSON RPC server stores subscription information in internal structure, which basically a map.

There is no limit on max number of elements in this map.

## Vulnerability Details

Let's look at the code <https://github.com/shardeum/json-rpc-server/blob/dev/src/websocket/index.ts#L94>

\`\`\` if (method\_name === 'eth\_subscribe') { if (!CONFIG.websocket.enabled || !CONFIG.websocket.serveSubscriptions) { socket.send(JSON.stringify(constructRPCErrorRes('Subscription serving disabled', -1, request.id))) return } try { nestedCountersInstance.countEvent('websocket', 'eth\_subscribe') if ( typeof request.params\[1] === 'object' && ('address' in request.params\[1] || 'topics' in request.params\[1]) ) { let subscription\_id = crypto.randomBytes(32).toString('hex') subscription\_id = '0x' + crypto.createHash('sha256').update(subscription\_id).digest().toString('hex') subscription\_id = subscription\_id.substring(0, 46) request.params\[10] = subscription\_id const address = request.params\[1].address const topics = request.params\[1].topics ... ... if (request.params\[1].topics) { request.params\[1].topics = request.params\[1].topics.map((topic: string | undefined) => { return topic?.toLowerCase() }) } const subscriptionDetails: SubscriptionDetails = { address: request.params\[1].address as string\[], topics: request.params\[1].topics as string\[], }

```
      logSubscriptionList.set(subscription_id, socket, subscriptionDetails, request.id)
    }
```

\`\`\`

As we can see, logSubscriptionList map is updated with subscription details and there is no upper bounds on size of this map.

As a result a remote attacker could trigger out of memory issue and crash the server.

## Impact Details

By sending a lot of subscription requests via eth\_subscribe method, a remote attacker could cause denial of service issue.

## Link to Proof of Concept

<https://gist.github.com/gln7/ae95443ac53ed74d127f7637ae4ffb4a>

## Proof of Concept

## Proof of Concept

How to reproduce:

1. set memory limit then enable websocket.serveSubscriptions options and start json rpc server. Also you need to run log\_server and enable it in options.ts

\`\`\` $ ulimit -Sv 4000000 $ npm run start \`\`\`

2. get proof of concept (via gist link) and run it:

\`\`\` $./t1.py \<host> \`\`\`

3. after a few requests, server will crash with a message: \`\`\` <--- Last few GCs --->

\[76910:0x7514c80] 407371 ms: Scavenge 2023.3 (2063.0) -> 2023.0 (2074.0) MB, 7.6 / 0.0 ms (average mu = 0.546, current mu = 0.474) allocation failure; \[76910:0x7514c80] 407385 ms: Scavenge 2029.8 (2074.0) -> 2029.7 (2075.0) MB, 8.9 / 0.0 ms (average mu = 0.546, current mu = 0.474) allocation failure; \[76910:0x7514c80] 407759 ms: Scavenge 2030.7 (2075.0) -> 2030.0 (2098.0) MB, 373.0 / 0.0 ms (average mu = 0.546, current mu = 0.474) allocation failure;

<--- JS stacktrace --->

FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory 1: 0xb7a940 node::Abort() \[node] 2: 0xa8e823 \[node] 3: 0xd5c990 v8::Utils::ReportOOMFailure(v8::internal::Isolate\*, char const\*, bool) \[node] 4: 0xd5cd37 v8::internal::V8::FatalProcessOutOfMemory(v8::internal::Isolate\*, char const\*, bool) \[node] 5: 0xf3a435 \[node] 6: 0xf4c91d v8::internal::Heap::CollectGarbage(v8::internal::AllocationSpace, v8::internal::GarbageCollectionReason, v8::GCCallbackFlags) \[node]

\`\`\`
