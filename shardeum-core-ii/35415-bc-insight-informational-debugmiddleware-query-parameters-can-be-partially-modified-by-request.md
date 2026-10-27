# #35415 \[BC-Insight] \[Informational] debugMiddleware query parameters can be partially modified by re

**Submitted on Sep 21st 2024 at 16:14:27 UTC by @Merkle\_Bonsai for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35415
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Informational

## Description

**I'm also kindly asking Immunefi or Shadreum team to report this bug to Shardus so they will be able to generally fix it, as fix is simple and straitforward. Fix below cannot disclose \`qs\` zero-day and can be done non-orchestrated. I can provide other, mostly theoretical attack scenarios to motivate this change if needed**

**I'm also asking Immunefi on assistance here as this looks like an at least med-severity zero-day in \`qs\` package that is dependency of \`express\`, the most popular nodejs http server. Just DM me on Discord so we can discuss it separately**

## Brief/Intro

Express's parser \`qs\` is very powerful and is able to handle complex data structures, allowing to create GET HTTP request URLs that will trick signature check, which is using \`route\` property to verify all together via \`stripQueryParams()\` function that takes only the value between first and second \`?\` signs.

Speaking simply, the actor who will be submitting signed/multisigned calls to debug endpoints in production mode will be able to partially modify request query.

## Vulnerability Details

\`handleDebugAuth\` and \`handleDebugMultiSigAuth\` in \`shardus-core/src/network/debugMiddleware.ts\` are verifying signatures to confirm the request is sent by authorized person. Yet, payload for signature is generated like this: \`\`\` let payload = { route: stripQueryParams(\_req.originalUrl, \['sig', 'sig\_counter', 'nodePubkeys']), count: \_req.query.sig\_counter, nodes: \_req.query.nodePubkeys, // example 8f35,8b3a,85f1 networkId: latestCycle.networkId, cycleCounter: latestCycle.counter, } \`\`\`

\`stripQueryParams\` will only take the string range between first and second \`?\` signs, e.g. for \`get /test?foo=1?bar=2\` it will be the following: \`\`\` req.query = { foo: "1?", bar: "2" } \`\`\` yet \`payload.route\` will be just \`/test?foo=1\`.

There are multiple ways to abuse it but looks like the most realistic and practical one is probably zero-day in \`qs\` package I discovered while this research.

## \`QS\` probable zero-day

I'm still validating this but kindly asking to not disclose this specific example anywhere yet. This was discovered by me today and I cannot find any evidence it's referenced anywhere in public.

If the key-value pair of URL query is formed like this, it will be handled in unexpected manner: \`\&key=value\[]=&\`, the resulting query structure is expected to be parsed like this: \`{key: "value\[]="}\`. However, due to specific logic of \`\[]\` handling it will be actually transformed into \`{"key=value": ""}\`, allowing to write this value again.

Applicable to Shardeum it means that e.g. for \`/route?data=0x1000?\[]=\&data=0xdead\` the \`data\` value would be \`0xdead\`, while \`stripQueryParams\` will produce \`data=0x1000\`.

## Impact Details

Speaking of Shardeum we can only consider any MITM case (e.g. some third-party server to sign messages), since no multisig is applied for those endpoints, but for Shardus in general this can be worse case, since it introduces single point of failure for multisig scenarios, specifically if it is executed by third party for any reason, or sigs are public.

I'm not fully aware of possible use cases but e.g. if there will be some server to perform those signed calls, this can become point of failure despite signing process will not be indicating any odd behavior.

This means that anyone posessing the valid signature not yet executed on specific node will be able to send modified transaction that will pass, being able to:

* modify any last parameter passed in request query
* transform any previous parameters from string X to array of \[X, Y, Z]

Specifically to Shardeum, the following endpoints are protected by this middleware:

* debug-points (readonly)
* debug-point-spenders (readonly)
* debug-point-spenders-clear (state mutating, but no query params to mess with)
* debug-shardus-dependencies (readonly)
* dumpStorage (readonly)
* dumpAddressMap (readonly)
* dumpShardeumStateMap (readonly)
* debug-shardeum-flags (readonly)
* **debug-set-shardeum-flag** (writable and parametrized)
* **debug-set-service-point** (writable and parametrized)
* debug-appdata/:hash (readonly)
* accounts (readonly)
* system-info (readonly)
* debug-set-event-block-threshold (writable but disabled now)

That means that attacker will be able modify any Shardeum flag value (if value is last non-excluded argument) set with query like this (e.g. team decided to change stakeTargetAddress): \`/debug-set-shardeum-flag?key=stakeTargetAddress\&value=0x4200000000000000000000000000000000010000?\[]=\&value=attacker.eth\` will cause to set \`stakeTargetAddress\` to \`attacker.eth\`, while \`queryString\` will be \`key=stakeTargetAddress\&value=0x4200000000000000000000000000000000010000\` and query will look like this: \`\`\` { "key": "stakeTargetAddress", "value=0x4200000000000000000000000000000000010000?": \[ "" ], "value": "attacker.eth" } \`\`\`

## How to fix

In \`shardus-core/src/network/debugMiddleware.ts#237\`, replace this \`\`\` let \[base, queryString] = url.split('?') \`\`\` with this \`\`\` let \[base, ...tail] = url.split('?') let queryString = tail.join('?') \`\`\`

## Proof of Concept

## Proof of Concept

Since it's general issue, better to demonstrate on synthetic example.

Let's say we have this simple server: \`\`\` require('express')() .get('/', (req, res) => { res.json({ route: stripQueryParams(req.originalUrl, \['sig', 'sig\_counter', 'nodePubkeys']), query: req.query, originalUrl: req.originalUrl }) }) .listen(3030) \`\`\`

For this request: \`<http://localhost:3030/?key=stakeTargetAddress\\&value=0x4200000000000000000000000000000000010000?\\[]=\\&value=attacker.eth\\`> answer will be \`\`\` { "route": "/?key=stakeTargetAddress\&value=0x4200000000000000000000000000000000010000", "query": { "key": "stakeTargetAddress", "value=0x4200000000000000000000000000000000010000?": \[ "" ], "value": "attacker.eth" }, "originalUrl": "/?key=stakeTargetAddress\&value=0x4200000000000000000000000000000000010000?\[]=\&value=attacker.eth" } \`\`\`

that clearly demonstrates that \`stripQueryParams\` (and object to hash in general) will differ to actual \`req.query\` values used in handler execution
