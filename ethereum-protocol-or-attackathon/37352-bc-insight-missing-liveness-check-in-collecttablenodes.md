# #37352 \[BC-Insight] Missing Liveness Check in \`collectTableNodes()\`

**Submitted on Dec 2nd 2024 at 17:42:35 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37352
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

An issue that the node misses liveness check will be added to the node table with `collectTableNodes()` was identified in the Ethereum client Erigon ( <https://github.com/erigontech/erigon> ).

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/tree/v2.61.0-beta1>

The function `collectTableNodes()` is intended to collect all the nodes for the FindNode result given a specified distance:

<https://github.com/erigontech/erigon/blob/v2.60.10/accounts/abi/type.go#L158>

```
func (t *UDPv5) collectTableNodes(rip net.IP, distances []uint, limit int) []*enode.Node {
	nodes := make([]*enode.Node, 0, len(distances))
	var processed = make(map[uint]struct{})
	for _, dist := range distances {
		// Reject duplicate / invalid distances.
		_, seen := processed[dist]
		if seen || dist > 256 {
			continue
		}


		// Get the nodes.
		var bn []*enode.Node
		if dist == 0 {
			bn = []*enode.Node{t.Self()}
		} else if dist <= 256 {
			t.tab.mutex.Lock()
			bn = unwrapNodes(t.tab.bucketAtDistance(int(dist)).entries)
			t.tab.mutex.Unlock()
		}
		processed[dist] = struct{}{}


		// Apply some pre-checks to avoid sending invalid nodes.
		for _, n := range bn {
			// TODO livenessChecks > 1
			if netutil.CheckRelayIP(rip, n.IP()) != nil {
				continue
			}
			nodes = append(nodes, n)
			if len(nodes) >= limit {
				return nodes
			}
		}
	}
	return nodes
}
```

However, it misses the liveness chen when collecting the nodes into the table, which is also mentioned as the TODO: (<https://github.com/erigontech/erigon/blob/v2.61.0-beta1/p2p/discover/v5\\_udp.go#L839> ）:

In this case, the node that has not been checked liveness will also be included in the table.

It is worth noted a similar issue has been fixed in go-ethereum: <https://github.com/ethereum/go-ethereum/pull/28686>

## Impact Details

.Nodes with no liveness check will be included in the node table.

## References

* <https://github.com/erigontech/erigon/tree/v2.61.0-beta1>
* <https://github.com/ethereum/go-ethereum/pull/28686>

## Proof of Concept

## Proof of Concept

Here we provide the following test case to show that nodes with no liveness check will be collected.

The default livenessChecks in function `wrapNode()` is zero, so there is no liveness check, which is used to mimic the nodes without liveness checks.

```
package discover


import (
   "bytes"
   "context"
   "crypto/ecdsa"
   "encoding/binary"
   "errors"
   "fmt"
   "net"
   "reflect"
   "runtime"
   "testing"
   "time"


   "github.com/ledgerwatch/erigon/turbo/testlog"
   "github.com/ledgerwatch/log/v3"


   "github.com/ledgerwatch/erigon/p2p/discover/v5wire"
   "github.com/ledgerwatch/erigon/p2p/enode"
   "github.com/ledgerwatch/erigon/p2p/enr"
   "github.com/ledgerwatch/erigon/rlp"
)



func wrapNode(n *enode.Node) *node {
   return &node{Node: *n}
}


func wrapNodes(ns []*enode.Node) []*node {
   result := make([]*node, len(ns))
   for i, n := range ns {
      result[i] = wrapNode(n)
   }
   return result
}

////////////unit test//////////

func TestCollectTableNodes(t *testing.T) {
   logger := log.New()
   test := newUDPV5Test(t, logger)
   t.Cleanup(test.close)

   nodes253 := nodesAtDistance(test.table.self().ID(), 253, 5)
   fillTable(test.table, wrapNodes(nodes253))

   rip := new(net.IP)
   distances := []uint{253}
   limit := 256

   nodes := test.udp.collectTableNodes(*rip, distances, limit)

   fmt.Printf("The collected nodes are: %v\n", nodes)
   fmt.Printf("Number of the collected nodes is: %d\n", len(nodes))

}
```

As the test result shows, all nodes without liveness check are also collected into the table:

```
=== RUN   TestCollectTableNodes
The collected nodes are: [enr:-DyAgIJpZIRudWxsgmlwhAEAAgGIbnVsbGFkZHKgN_Nh_UDMNfBEwoyLNAHxti9lG9dB0WZUoYuTrh3Fikg enr:-DyAgIJpZIRudWxsgmlwhAIAAgKIbnVsbGFkZHKgN8qNUV_UmPwblUu29QgouNp_lYSdQkXJ7WowYEHpX_0 enr:-DyAgIJpZIRudWxsgmlwhAMAAgOIbnVsbGFkZHKgN6JjlG7eFq-HMOlKgjkhSh0dzSAeuFeUfcTQSwF6qrQ enr:-DyAgIJpZIRudWxsgmlwhAQAAgSIbnVsbGFkZHKgN3xzWvF5XH-wIVucytScYrWLImpix0rdD5oFVO2Ytrc enr:-DyAgIJpZIRudWxsgmlwhAUAAgWIbnVsbGFkZHKgN5LIK4BmeON56kQ4ocADZHbe9pKkBfoOLctBpfgBpWM]
Number of the collected nodes is: 5
--- PASS: TestCollectTableNodes (0.04s)
PASS
```
