# #37148 \[BC-Insight] \`wantedPeerDials()\` branch will never be executed

**Submitted on Nov 26th 2024 at 19:59:03 UTC by @br0nz3p1ck4x3 for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37148
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/prysmaticlabs/prysm>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

Hey Triagers, Ethereum Foundation, Prysm!

We encountered the following during our research and we think it can add value to the Prysm team.

## Description

Inside `prysm/beacon-chain/p2p/discovery.go::listenForNodes()` we can find the following:

```go
func (s *Service) listenForNewNodes() {
//..
			if s.isPeerAtLimit(false /* inbound */) {
				// Pause the main loop for a period to stop looking
				// for new peers.
				log.Trace("Not looking for peers, at peer limit")
				time.Sleep(pollingPeriod)
				continue
			}
			wantedCount := s.wantedPeerDials()
			if wantedCount == 0 {
				log.Trace("Not looking for peers, at peer limit")
				time.Sleep(pollingPeriod)
				continue
			}
//..
}
```

There is a problem here though in the logic.

To showcase this, we will first explain it with arbitrary values. After reading this, please run the Proof of Concept attached to the report.

Alright.

If we look at `isPeerAtLimit()`, we see the following:

```go
func (s *Service) isPeerAtLimit(inbound bool) bool {
	numOfConns := len(s.host.Network().Peers())
	maxPeers := int(s.cfg.MaxPeers)
	// If we are measuring the limit for inbound peers
	// we apply the high watermark buffer.
	if inbound {
		maxPeers += highWatermarkBuffer
		maxInbound := s.peers.InboundLimit() + highWatermarkBuffer
		currInbound := len(s.peers.InboundConnected())
		// Exit early if we are at the inbound limit.
		if currInbound >= maxInbound {
			return true
		}
	}
	activePeers := len(s.Peers().Active())
	return activePeers >= maxPeers || numOfConns >= maxPeers
}
```

Let's take the following arbitrary values that maximize the numbers such that the `return` statement will return `true`:

* `numOfConns = 9`
* `maxPeers = 10`
* `activePeers = 9`

These values will evaluate to `true` in this `return statement`:

* `activePeers >= maxPeers || numOfConns >= maxPeers`

Now, we go back to `listenForNewNodes()`:

```go
func (s *Service) listenForNewNodes() {
//..
			if s.isPeerAtLimit(false /* inbound */) {
				// Pause the main loop for a period to stop looking
				// for new peers.
				log.Trace("Not looking for peers, at peer limit")
				time.Sleep(pollingPeriod)
				continue
			}
			wantedCount := s.wantedPeerDials()
			if wantedCount == 0 {
				log.Trace("Not looking for peers, at peer limit")
				time.Sleep(pollingPeriod)
				continue
			}
			//..
}
```

Now, `wantedPeerDials()` is called. If we look at the implementation:

```go
func (s *Service) wantedPeerDials() int {
	maxPeers := int(s.cfg.MaxPeers)

	activePeers := len(s.Peers().Active())
	wantedCount := 0
	if maxPeers > activePeers {
		wantedCount = maxPeers - activePeers
	}
	return wantedCount
}
```

The problem here is the following `if-statement`:

* `if maxPeers > activePeers {`

As we saw in `isPeerAtLimit()`, which was called before `wantedPeerDials()`, `maxPeers` will **always** be bigger than `activePeers`. If it was not bigger than `activePeers`, it would enter the `if-statement` and `continue`:

```go
func (s *Service) listenForNewNodes() {
//..
			if s.isPeerAtLimit(false /* inbound */) {
				// Pause the main loop for a period to stop looking
				// for new peers.
->				log.Trace("Not looking for peers, at peer limit")
->				time.Sleep(pollingPeriod)
->				continue
			}
			wantedCount := s.wantedPeerDials()
			if wantedCount == 0 {
				log.Trace("Not looking for peers, at peer limit")
				time.Sleep(pollingPeriod)
				continue
			}
			//..
}
```

Thus, `wantedPeerDials()` will **always** return a `non-nil` value and thus, this `if-statement` inside `listenForNewNodes()` will never be executed:

```go
func (s *Service) listenForNewNodes() {
//..
			if s.isPeerAtLimit(false /* inbound */) {
				// Pause the main loop for a period to stop looking
				// for new peers.
				log.Trace("Not looking for peers, at peer limit")
				time.Sleep(pollingPeriod)
				continue
			}
			wantedCount := s.wantedPeerDials()
->			if wantedCount == 0 {
				log.Trace("Not looking for peers, at peer limit")
				time.Sleep(pollingPeriod)
				continue
			}
			//..
}
```

## Proof of Concept

## Proof of Concept

* `cd prysm/beacon-chain/p2p`
* `vim discovery_test.go`\
  Paste the following test in this file:

```go
func TestPoc(t *testing.T) {
	// - `numOfConns = 9`
	// - `maxPeers = 10`
	// - `activePeers = 9`
	fakePeer := testp2p.NewTestP2P(t)
	s := &Service{
		cfg:       &Config{MaxPeers: 10},
		ipLimiter: leakybucket.NewCollector(ipLimit, ipBurst, 1*time.Second, false),
		peers: peers.NewStatus(context.Background(), &peers.StatusConfig{
			PeerLimit:    10,
			ScorerParams: &scorers.Config{},
		}),
		host: fakePeer.BHost,
	}

	for i := 0; i < 9; i++ {
		_ = addPeer(t, s.peers, peerdata.PeerConnectionState(ethpb.ConnectionState_CONNECTED), false)
	}

	// We will make two cases, one case where there are 9 peers with a limit of 10 maxPeers
	fmt.Print("Current setup: 9 peers with a limit of 10 peers\n")
	isAtLimit := s.isPeerAtLimit(false)
	fmt.Print("Has the limit been hit? ", isAtLimit, "\n")

	wantedPeers := s.wantedPeerDials()
	fmt.Print("Wanted Peers:", wantedPeers, "\n")

	// Add one more peer so we hit the limit
	_ = addPeer(t, s.peers, peerdata.PeerConnectionState(ethpb.ConnectionState_CONNECTED), false)
	fmt.Print("Current setup: 10 peers with a limit of 10 peers\n")
	isAtLimit = s.isPeerAtLimit(false)
	fmt.Print("Has the limit been hit? ", isAtLimit, "\n")

	fmt.Print("This will never execute because the previous `isPeerAtLimit()` call will lead to `continue` and replaying the loop\n")
	wantedPeers = s.wantedPeerDials()
	fmt.Print("Wanted Peers:", wantedPeers, "\n")
}
```

* Run using:
  * `go test -timeout 30s -run ^TestPoc`

The output will be:

```zsh
Current setup: 9 peers with a limit of 10 peers
Has the limit been hit? false
Wanted Peers:1
Current setup: 10 peers with a limit of 10 peers
Has the limit been hit? true
This will never execute because the previous `isPeerAtLimit()` call will lead to `continue` and replaying the loop
Wanted Peers:0
PASS
ok  	github.com/prysmaticlabs/prysm/v5/beacon-chain/p2p	0.627s
```

As we can see, the only way to get `0`wanted peers is for `activePeers == maxPeers`, but as shown, this can never happen because the `for-loop` inside `listenForNewNodes()` will return early because of:

```go
			if s.isPeerAtLimit(false /* inbound */) {
				// Pause the main loop for a period to stop looking
				// for new peers.
				log.Trace("Not looking for peers, at peer limit")
				time.Sleep(pollingPeriod)
				continue
			}
```

## Recommended Patch

Consider removing the `wantedPeerDials()` usage since it can not be invoked right now.
