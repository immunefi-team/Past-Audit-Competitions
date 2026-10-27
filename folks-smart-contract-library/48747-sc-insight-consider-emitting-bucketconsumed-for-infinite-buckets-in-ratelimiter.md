# #48747 \[SC-Insight] Consider emitting BucketConsumed for infinite buckets in RateLimiter

**Submitted on Jul 7th 2025 at 15:05:33 UTC by @Blobism for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #48747
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/RateLimiter.py>
* **Impacts:**

## Description

## Brief/Intro

The RateLimiter method `_consume_amount` currently does not emit a `BucketConsumed` event. An off-chain system monitoring these events would never see consumption from infinite buckets, which could be misleading.

## Vulnerability Details

The `_consume_amount` method has an early return for infinite buckets (buckets with zero duration), shown below. It would be ideal to either emit a `BucketConsumed` event at the early return, or clearly specify in the documentation that these events will not be emitted for infinite buckets.

```python
def _consume_amount(self, bucket_id: Bytes32, amount: UInt256) -> None:
    """Consumes an amount inside a bucket.

    Args:
        bucket_id: The bucket to consume from.
        amount: The amount to consume.

    Raises:
        AssertionError: If the bucket is unknown.
        AssertionError: If there is insufficient capacity.
    """
    # fails if bucket is unknown
    self._update_capacity(bucket_id)

    # ignore if duration is zero
    rate_limit_bucket = self._get_bucket(bucket_id)
    if not rate_limit_bucket.duration.native:
        #                      <-------------- consider emitting BucketConsumed here
        return

    # ensure there is enough capacity
    assert amount <= rate_limit_bucket.current_capacity, "Insufficient capacity to consume"

    # consume amount
    new_capacity = rate_limit_bucket.current_capacity.native - amount.native
    self.rate_limit_buckets[bucket_id].current_capacity = ARC4UInt256(new_capacity)
    emit(BucketConsumed(bucket_id, amount))
```

## Impact Details

This is an insight for a code/documentation improvement, but does not have any security implications aside from impacting off-chain monitoring systems.

## References

See `contracts/library/RateLimiter.py`

## Proof of Concept

## Proof of Concept

Observing the early return above should be sufficient to see that the event is not emitted for infinite buckets.
