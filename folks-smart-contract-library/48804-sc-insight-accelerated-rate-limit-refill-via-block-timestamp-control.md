# #48804 \[SC-Insight] Accelerated Rate Limit Refill via Block Timestamp Control

**Submitted on Jul 8th 2025 at 03:59:28 UTC by @a090325 for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #48804
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/RateLimiter.py>
* **Impacts:**
  * Bypass of the rate limit beyond set parameters

## Description

## Brief/Intro

Capacity can be re-filled to the maximum value (limit) if (Global.latest\_timestamp - rate\_limit\_bucket.last\_updated.native) is greater than or equal to (rate\_limit.duration). So an attacker can exhaust capacity in block n-1. Then when being chosen as block proposer, he can set timestamp of block n further into the future so that in block n+1: (Global.latest\_timestamp - rate\_limit\_bucket.last\_updated.native) is greater than or equal to (rate\_limit.duration) to refill capacity to the maximum value (limit) if (rate\_limit.duration) is small enough. Please be noted that Global.lastest\_timestamp is the timestamp of block n (controlled by attacker).

On Algorand, a block’s timestamp can be at most 25 seconds further into the future than the previous block timestamp. So any rate limit with duration less than 25 seconds can be refilled to maximum value just after two blocks (\~ 7 seconds) by exploiting this vulnerability. Generally, any rate limit can be refilled faster than intended (since the refilled amount is proportional to the difference between Global-lastest\_timestamp and rate\_limit\_bucket.last\_updated.native) but the effect is less significant if the rate\_limit.duration.native is large.

## Vulnerability Details

* In \_update\_capacity () function, capacity can be refilled to maximum capacity (limit) if time\_delta >= rate\_limit\_bucket.duration.native:

<https://github.com/Folks-Finance/algorand-smart-contract-library/blob/7673a43fa5183af736b65f17d1a297fdea672059/contracts/library/RateLimiter.py#L248>

```
 # increase capacity by fill rate of <limit> per <duration> without exceeding limit
        time_delta = Global.latest_timestamp - rate_limit_bucket.last_updated.native
        new_capacity_without_max = rate_limit_bucket.current_capacity.native + (
                (rate_limit_bucket.limit.native * time_delta) // rate_limit_bucket.duration.native
        )

        # update capacity and last updated timestamp
        self.rate_limit_buckets[bucket_id].current_capacity = rate_limit_bucket.limit \
            if new_capacity_without_max > rate_limit_bucket.limit else ARC4UInt256(new_capacity_without_max)
        self.rate_limit_buckets[bucket_id].last_updated = ARC4UInt64(Global.latest_timestamp)
```

* The refilled amount is:

```
(rate_limit_bucket.limit.native * time_delta) // rate_limit_bucket.duration.native
        )
```

* So if time\_delta is greater than or equal to rate\_limit\_bucket.duration.native, capacity can be refilled to maximum value (rate\_limit\_bucket.limit.native) in this line:

```
self.rate_limit_buckets[bucket_id].current_capacity = rate_limit_bucket.limit \
            if new_capacity_without_max > rate_limit_bucket.limit else ARC4UInt256(new_capacity_without_max)
```

## Impact Details

* Rate limits with a duration shorter than 25 seconds can be fully refilled just two blocks later (approximately 7 seconds in real time), due to allowable timestamp manipulation. Even longer-duration rate limits can be partially refilled faster than intended under the same conditions.
* Since only block proposers can influence timestamps, an attacker must be consistently selected and well-funded to carry out repeated exploits.
* If the rate limit protects critical operations (e.g., fund withdrawals), this vulnerability could have serious consequences, especially given its repeatability.

## References

* <https://github.com/algorandfoundation/specs/blob/master/dev/ledger.md>
* <https://developer.algorand.org/articles/developer-improvements-in-go-algorand-316/?from\\_query=25%20seconds>

## Proof of Concept

## Proof of Concept

1. Set up a rate limit bucket using the library with the following parameters:

* Limit: 1000
* Duration: 21 seconds (approximately 6 blocks)

2. At block n-1, Alice (an attacker) fully consumes the bucket's capacity.

* Under normal conditions, the bucket would be refilled at t + 21s (real time) or block n + 6 (block time).

3. At block n, Alice is selected as the block proposer.

* She sets the block’s timestamp 25 seconds ahead of real time (within the allowed t\_delta limit).
* Because block n's timestamp is artificially ahead, the contract sees the 21-second duration as already elapsed.

4. At block n + 1 (just \~3.5 seconds later in real time), Alice is able to consume the bucket again, bypassing the intended rate limit.
