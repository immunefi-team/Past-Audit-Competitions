# #38146 \[BC-Medium] nimbus-eth2 remote crash

**Submitted on Dec 26th 2024 at 08:23:45 UTC by @gln for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38146
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/status-im/nimbus-eth2>
* **Impacts:**
  * Direct loss of funds
  * Shutdown of greater than or equal to 10% or equal to but less than 33% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

Nimbus-eth2 libp2p incorrectly parses protobuf messages. As a result it will lead to denial of service issue.

## Vulnerability Details

First we need to see how Nim converts uint64 to int type.

Consider the following simple nim program:

```
var bsize:uint64 = 0x80000000_00000000'u64
echo "casting..."
var offset:int  = int(bsize)
echo "offset = ", offset
```

If you compile and run it, you will receive the exception:

```
casting...
../nim-2.2.0/lib/system/fatal.nim(53) sysFatal
Error: unhandled exception: value out of range [RangeDefect]
Error: execution of an external program failed:
```

So, If the value of uint64 is larger than 0x7fffffff\_ffffffff, fatal RangeDefect exception will be thrown and program will stop.

In gossipsub protocol RPC messages are encoded by using protobuf.

In case of nimbus-eth2 it is handled by custom protobuf library - miniprotobuf.nim

Let's look at the code <https://github.com/vacp2p/nim-libp2p/blob/8855bce0854ecf4adad7a0556bb2b2d2f98e0e20/libp2p/varint.nim#L106>

```
proc getUVarint*[T: PB | LP](
    vtype: typedesc[T],
    pbytes: openArray[byte],
    outlen: var int,
    outval: var SomeUVarint,
): VarintResult[void] =
  outlen = 0
  outval = type(outval)(0)

  let parsed = type(outval).fromBytes(pbytes, Leb128)

  if parsed.len == 0:
    return err(VarintError.Incomplete)
  if parsed.len < 0:
    return err(VarintError.Overflow)

  when vtype is LP and sizeof(outval) == 8:
    if parsed.val >= 0x8000_0000_0000_0000'u64:
      return err(VarintError.Overflow)

  if vsizeof(parsed.val) != parsed.len:
    return err(VarintError.Overlong)

  (outval, outlen) = parsed

  ok()
```

1. If vtype is PB, there are no checks for parsed.val, it can be arbitrary large value

Now we need to see how protobuf parser is being used <https://github.com/vacp2p/nim-libp2p/blob/8855bce0854ecf4adad7a0556bb2b2d2f98e0e20/libp2p/protocols/pubsub/rpc/protobuf.nim#L331>

```
proc decodeRpcMsg*(msg: seq[byte]): ProtoResult[RPCMsg] {.inline.} =
  trace "decodeRpcMsg: decoding message", payload = msg.shortLog()
1) var pb = initProtoBuffer(msg, maxSize = uint.high)
  var rpcMsg = RPCMsg()
  assign(rpcMsg.messages, ?pb.decodeMessages())
  assign(rpcMsg.subscriptions, ?pb.decodeSubscriptions())
  assign(rpcMsg.control, ?pb.decodeControl())
  discard ?pb.getField(60, rpcMsg.ping)
  discard ?pb.getField(61, rpcMsg.pong)
  ok(rpcMsg)
```

Let's look at the actual parser <https://github.com/vacp2p/nim-libp2p/blob/8855bce0854ecf4adad7a0556bb2b2d2f98e0e20/libp2p/protobuf/minprotobuf.nim#L344>

```
proc skipValue(data: var ProtoBuffer, header: ProtoHeader): ProtoResult[void] =
  case header.wire
  ...
  of ProtoFieldKind.Length:
    var length = 0
    var bsize = 0'u64
2)  if PB.getUVarint(data.toOpenArray(), length, bsize).isOk():
      data.offset += length
3)    if bsize <= uint64(data.maxSize):
4)      if data.isEnough(int(bsize)):
          data.offset += int(bsize)
          ok()
        else:
          err(ProtoError.MessageIncomplete)
      else:
        err(ProtoError.MessageTooBig)
    else:
      err(ProtoError.VarintDecode)
```

1. Note that maxSize is equal to uint.high
2. Varint is fetched from incoming stream
3. Even if bsize is larger than 0x7fffffff\_ffffffff, the check will pass because data.maxSize is equal to 0xffffffff\_ffffffff
4. Nim throws fatal exception when trying to convert bsize to 'int' type

## Impact Details

Basically, attacker will be able to crash nimbus-eth2 nodes remotely with a single packet.

## Link to Proof of Concept

<https://gist.github.com/gln7/e41de97351999a048e30436d05593dbd>

## Proof of Concept

## Proof of Concept

How to reproduce:

1. get nimbus-eth2 source code

```
$ git rev-parse stable
4e440277cf8a3fed72f32eb2f01fc5e910ad6768

```

2. apply patch to nim-libp2p (see gist link)
3. run localnet:

```
$ make VALIDATORS=50 NUM_NODES=6 USER_NODES=0 local-testnet-minimal
```

4. after some time, you should see exception in local-testnet-minimal/logs/nimbus\_beacon\_node.1.jsonl

```
nimbus-eth2/beacon_chain/nimbus_beacon_node.nim(2132) _ZN18nimbus_beacon_n
ode3runE3refIN11beacon_node26BeaconNodecolonObjectType_EE
nimbus-eth2/vendor/nim-chronos/chronos/internal/asyncengine.nim(150) _ZN11
asyncengine4pollE
nimbus-eth2/vendor/nim-chronos/chronos/internal/asyncfutures.nim(371) _ZN1
2asyncfutures14futureContinueE3refIN7futures26FutureBasecolonObjectType_EE
nimbus-eth2/vendor/nim-libp2p/libp2p/protocols/pubsub/pubsubpeer.nim(223) 
_ZN6handle71handle
nimbus-eth2/vendor/nimbus-build-system/vendor/Nim/lib/system/stacktraces.n
im(62) _ZN11stacktraces30auxWriteStackTraceWithOverrideE3varI3seqIN6system15StackTraceEntryEEE
]]
Error: unhandled exception: value out of range [RangeDefect]

```
