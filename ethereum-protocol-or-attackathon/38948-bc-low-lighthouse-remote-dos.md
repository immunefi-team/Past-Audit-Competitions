# #38948 \[BC-Low] lighthouse remote DoS

**Submitted on Jan 18th 2025 at 17:45:48 UTC by @gln for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38948
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/sigp/lighthouse>
* **Impacts:**
  * Increasing greater than or equal to 25% of network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

The issue is very similar to #38920 which I've sent earlier.

Lighthouse internal rate limiter is vulnerable to integer overflow.

Remote attacker will be able to bypass rate limit checks and trigger potential Denial of Service issue.

## Vulnerability Details

The BlobSidecarsByRange p2p request <https://github.com/ethereum/consensus-specs/blob/dev/specs/deneb/p2p-interface.md?ref=bankless.ghost.io#blobsidecarsbyrange-v1> has two parameters - start\_slot and count.

To prevent potential Denial of service issues, lighhouse has internal rate limiter.

If the 'count' value is too high, this rate limiter returns an error and request will not be processed.

Let's look at the code <https://github.com/sigp/lighthouse/blob/stable/beacon\\_node/lighthouse\\_network/src/rpc/mod.rs#L396>

```
 fn on_connection_handler_event(
        &mut self,
        peer_id: PeerId,
        conn_id: ConnectionId,
        event: <Self::ConnectionHandler as ConnectionHandler>::ToBehaviour,
    ) {
        match event {
            HandlerEvent::Ok(RPCReceived::Request(Request {
                id,
                substream_id,
                r#type,
            })) => {
                if let Some(limiter) = self.limiter.as_mut() {
                    // check if the request is conformant to the quota
1)                    match limiter.allows(&peer_id, &r#type) {
                        Err(RateLimitedErr::TooLarge) => {
                            // we set the batch sizes, so this is a coding/config err for most protocols
                            let protocol = r#type.versioned_protocol().protocol();
                            if matches!(
                                protocol,
                                Protocol::BlocksByRange
                                    | Protocol::BlobsByRange
                                    | Protocol::DataColumnsByRange
                                    | Protocol::BlocksByRoot
                                    | Protocol::BlobsByRoot
                                    | Protocol::DataColumnsByRoot
                            ) {
                                debug!(self.log, "Request too large to process"; "request" => %r#type, "protocol" => %protocol);
                            } else {
                                // Other protocols shouldn't be sending large messages, we should flag the peer kind
                                crit!(self.log, "Request size too large to ever be processed"; "protocol" => %protocol);
                            }
                            // send an error code to the peer.
                            // the handler upon receiving the error code will send it back to the behaviour
                            ...
							return;
                        }
                        Err(RateLimitedErr::TooSoon(wait_time)) => {
                            debug!(self.log, "Request exceeds the rate limit";
                        "request" => %r#type, "peer_id" => %peer_id, "wait_time_ms" => wait_time.as_millis());
                            // send an error code to the peer.
                            // the handler upon receiving the error code will send it back to the behaviour
                            ...
							return;
                        }
                        // No rate limiting, continue.
                        Ok(()) => {}
                    }
                }
		...
}
```

Line #1 - rate limiter is called on this line, let's look at the actual implementation <https://github.com/sigp/lighthouse/blob/stable/beacon\\_node/lighthouse\\_network/src/rpc/rate\\_limiter.rs>

```
    pub fn allows<Item: RateLimiterItem>(
        &mut self,
        peer_id: &PeerId,
        request: &Item,
    ) -> Result<(), RateLimitedErr> {
        let time_since_start = self.init_time.elapsed();
1)        let tokens = request.max_responses().max(1);

        let check =
            |limiter: &mut Limiter<PeerId>| limiter.allows(time_since_start, peer_id, tokens);
        ...
    }


 pub fn allows(
        &mut self,
        time_since_start: Duration,
        key: &Key,
        tokens: u64,
    ) -> Result<(), RateLimitedErr> {
        let time_since_start = time_since_start.as_nanos() as u64;
        let tau = self.tau;
        let t = self.t;
        // how long does it take to replenish these tokens
2)        let additional_time = t * tokens;
3)        if additional_time > tau {
            // the time required to process this amount of tokens is longer than the time that
            // makes the bucket full. So, this batch can _never_ be processed
            return Err(RateLimitedErr::TooLarge);
        }
        // If the key is new, we consider their bucket full (which means, their request will be
        // allowed)
        let tat = self
            .tat_per_key
            .entry(key.clone())
            .or_insert(time_since_start);
        // check how soon could the request be made
        let earliest_time = (*tat + additional_time).saturating_sub(tau);
        // earliest_time is in the future
        if time_since_start < earliest_time {
            Err(RateLimitedErr::TooSoon(Duration::from_nanos(
                /* time they need to wait, i.e. how soon were they */
                earliest_time - time_since_start,
            )))
        } else {
            // calculate the new TAT
            *tat = time_since_start.max(*tat) + additional_time;
            Ok(())
        }
    }


    pub fn max_responses(&self) -> u64 {
        match self {
            RequestType::BlobsByRange(req) => req.max_blobs_requested::<E>(),
        }
    }

impl BlobsByRangeRequest {
    pub fn max_blobs_requested<E: EthSpec>(&self) -> u64 {
        self.count.saturating_mul(E::max_blobs_per_block() as u64)
    }
}

```

1. The value of 'tokens' variable is equal to req.max\_blob\_requested() ,\
   which is equal to 'count' multiplied by 6 (max blobs per block according to spec)
2. If 'tokens' is large enough, integer overflow will occur on this line
3. Basically, 'additional\_time' is time estimate required to process the request - if it is small, this check will not pass

As a result, rate limiter returns ok and request will be processed by lighhouse.

## Impact Details

Potential Denial of service issue, as attacker could request large number of blobs from a lighthouse node.

Estimate for a space occupied by blobs is around 50gb - <https://lighthouse-book.sigmaprime.io/advanced-blobs.html>

## Link to Proof of Concept

<https://gist.github.com/gln7/f318da428a95e72594bcc1d761c4282a>

## Proof of Concept

## Proof of Concept

How to reproduce:

1. get lighthouse source

```
$ git rev-parse stable
0d90135047519f4c2ee586d50e560f7bb2ff9b10

```

2. apply poc.patch (see gist link)
3. run test:

```
$ cd lighthouse/beacon_node/lighthouse_network/src/rpc
$ cargo test -r test_next_peer_request_ready -- --nocapture
```

4. you should see output like this:

```
running 1 test
XXXXKE testing rate limit with count 0x4142...
XXXXKE allows tokens 1603800 maxresp 1603800 proto blob_sidecars_by_range
XXXXKE allows2 t 19531250, tokens 1603800 add_time 31324218750000 vs tau 10000000000
XXXXKE rate limit error - too large!

XXXXKE testing rate limit with count 0x24a67fcd20...
XXXXKE allows tokens 944473296576 maxresp 944473296576 proto blob_sidecars_by_range
XXXXKE allows2 t 19531250, tokens 944473296576 add_time 40448384 vs tau 10000000000
XXXKE rate limit is OK
test rpc::self_limiter::tests::test_next_peer_request_ready ... ok

```

First run we are testing rate limit with a huge count value, rate limit signals error (too large).

Second time we are running with a count which causes integer overflow and rate limits checks are passed.
