# #35534 \[W\&A-Insight] json rpc server remote crash

**Submitted on Sep 26th 2024 at 19:56:39 UTC by @gln for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35534
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/json-rpc-server/tree/dev>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

JSON RPC server do not have limit on a number of filters in eth\_newFilter method.

## Vulnerability Details

Let's look at the code:

\`\`\` eth\_newFilter: async function (args: RequestParamsLike, callback: JSONRPCCallbackTypePlain) { const api\_name = 'eth\_newFilter' nestedCountersInstance.countEvent('endpoint', api\_name) if (!ensureArrayArgs(args, callback)) { countFailedResponse(api\_name, 'Invalid params: non-array args') return } ... ... const unsubscribe = (): void => void 0 const internalFilter: Types.InternalFilter = { updates: \[], filter: filterObj, unsubscribe, type: Types.FilterTypes.log, } filtersMap.set(filterId.toString(), internalFilter)

```
callback(null, filterId)
countSuccessResponse(api_name, &#x27;success&#x27;, &#x27;TBD&#x27;)
logEventEmitter.emit(&#x27;fn_end&#x27;, ticket, { success: true }, performance.now())
```

},

\`\`\`

As you can see, there is no limit on size of filtersMap.

As a result, it may become very large and trigger server crash due to out of memory error.

## Impact Details

By sending a lot of eth\_newFilter requests attacker could trigger remote denial of service issue.

## Link to Proof of Concept

<https://gist.github.com/gln7/197964764e3bb57078544a59a12b528e>

## Proof of Concept

## Proof of Concept

How to reproduce:

1. set memory limit and run json rpc server

\`\`\` $ ulimit -Sv 2000000 $ npm run start \`\`\`

2. get proof of concept (see gist link)
3. run it against rpc server:

\`\`\` $./t1.py \<host> \`\`\`

Server will crash with the message:

\`\`\` \[65682:0x6defc80] 562572 ms: Scavenge 1156.6 (1268.7) -> 1147.0 (1268.7) MB, 9.8 / 0.4 ms (average mu = 0.960, current mu = 0.967) task; \[65682:0x6defc80] 562899 ms: Scavenge 1157.0 (1268.7) -> 1149.5 (1268.7) MB, 6.4 / 0.1 ms (average mu = 0.960, current mu = 0.967) task; \[65682:0x6defc80] 563317 ms: Scavenge 1159.9 (1268.7) -> 1152.0 (1268.7) MB, 6.1 / 0.2 ms (average mu = 0.960, current mu = 0.967) task;

<--- JS stacktrace --->

FATAL ERROR: NewSpace::Rebalance Allocation failed - JavaScript heap out of memory 1: 0xb7a940 node::Abort() \[node] 2: 0xa8e823 \[node] 3: 0xd5c990 v8::Utils::ReportOOMFailure(v8::internal::Isolate\*, char const\*, bool) \[node] 4: 0xd5cd37 v8::internal::V8::FatalProcessOutOfMemory(v8::internal::Isolate\*, char const\*, bool) \[node] 5: 0xf3a435 \[node] 6: 0xf96a99 \[node] 7: 0xf96af8 v8::internal::MarkCompactCollector::CollectGarbage() \[node] 8: 0xf4a271 v8::internal::Heap::MarkCompact() \[node] 9: 0xf4bd48 \[node]

\`\`\`
