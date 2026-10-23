# 34364 - \[BC - Insight] pp deserialization denial of service issue

Submitted on Aug 10th 2024 at 16:46:07 UTC by @gln for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34364

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer
* Unintended chain split (network partition)

## Description

## Brief/Intro

When parsing incoming p2p messages, the code does not have any upper bounds check on sizes of incoming arrays.

Malformed p2p requests could trigger out of memory issue in nodejs and crash the server.

## Vulnerability Details

Let's look at the code <https://github.com/shardeum/shardus-core/blob/dev/src/types/GetTrieHashesReq.ts#L25>

```
export function deserializeGetTrieHashesReq(stream: VectorBufferStream): GetTrieHashesRequest {
  const version = stream.readUInt8()
  if (version > cGetTrieHashesReqVersion) {
    throw new Error('Unsupported version in deserializeGetTrieHashesReq')
  }
  const length = stream.readUInt32()
  const radixList = []
  for (let i = 0; i < length; i++) {
    radixList.push(stream.readString())
  }
  return { radixList }
}

```

Another similar example from <https://github.com/shardeum/shardus-core/blob/dev/src/types/RequestStateForTxReq.ts#L26>

```
export function deserializeRequestStateForTxReq(stream: VectorBufferStream): RequestStateForTxReq {
  const version = stream.readUInt8()
  if (version !== cRequestStateForTxReqVersion) {
    throw new Error('Unsupported version')
  }
  const txid = stream.readString()
  const timestamp = parseInt(stream.readString())
  const keysLength = stream.readUInt32()
  const keys = new Array<string>(keysLength)
  for (let i = 0; i < keysLength; i++) {
    keys[i] = stream.readString()
  }
  return { txid, timestamp, keys }
}
```

Here there is no upper-bound limit on keysLength variable.

Attacker could send the large number of keys and trigger out of memory error in nodejs.

## Impact Details

Remote denial of service issues (remote crash, requires restart).

## Proof of Concept

How to reproduce:

1. get proof of concept by using provided gist link
2. set memory limit :

```
$ ulimit -Sv 2000000
`` 

3) run poc:

```

$ node test1.js

... FATAL ERROR: Scavenger: semi-space copy Allocation failed - JavaScript heap out of memory ----- Native stack trace -----

1: 0xb84bd6 node::OOMErrorHandler(char const\*, v8::OOMDetails const&) \[node] 1: 0xb84bd6 node::OOMErrorHandler(char const\*, v8::OOMDetails const&) \[node] 2: 0xefec30 v8::Utils::ReportOOMFailure(v8::internal::Isolate\*, char const\*, v8::OOMDetails const&) \[node] 1: 0xb84bd6 node::OOMErrorHandler(char const\*, v8::OOMDetails const&) \[node] 2: 0xefec30 v8::Utils::ReportOOMFailure(v8::internal::Isolate\*, char const\*, v8::OOMDetails const&) \[node] 2: 0xefec30 v8::Utils::ReportOOMFailure(v8::internal::Isolate\*, char const\*, v8::OOMDetails const&) \[node] 3: 0xefef17 v8::internal::V8::FatalProcessOutOfMemory(v8::internal::Isolate\*, char const\*, v8::OOMDetails const&) \[node] 4: 0x1110925 \[node] 5: 0x11a2566 heap::base::SlotCallbackResult v8::internal::Scavenger::EvacuateShortcutCandidate[v8::internal::FullHeapObjectSlot](v8::internal::FullHeapObjectSlot)(v8::internal::Map, v8::internal::FullHeapObjectSlot, v8::internal::ConsString, int) \[node] 6: 0x11a71e9 heap::base::SlotCallbackResult v8::internal::Scavenger::ScavengeObject[v8::internal::FullHeapObjectSlot](v8::internal::FullHeapObjectSlot)(v8::internal::FullHeapObjectSlot, v8::internal::HeapObject) \[node] ... ...

```
```
