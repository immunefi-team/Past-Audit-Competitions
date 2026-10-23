# #49413 \[SC-Insight] discrepancy between document and codebase

**Submitted on Jul 15th 2025 at 16:27:45 UTC by @danial for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49413
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/RateLimiter.py>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

discrepancy in code base and document

## Vulnerability Details

the document says this about `_fill_amount` in `RateLimiter.py`

> \_fill\_amount(byte\[32] bucket\_id, uint64 amount) returns void. Fills an amount inside a bucket. Will not exceed the bucket’s limit. It emits the BucketFilled event.

says this functioni will get `uint64` as amount input however in the codebase it takes `uint256`:

```python
 def _fill_amount(self, bucket_id: Bytes32, amount: UInt256) -> None:
```

***

also about `get_rate_duration` document says it should return `uint256`

> get\_rate\_duration(byte\[32] bucket\_id) returns uint256. Returns the rate duration of the given bucket.

but codebase returns `uint64`:

```python
get_rate_duration(byte[32] bucket_id) returns uint256. Returns the rate duration of the given bucket.

```

## Impact Details

although this doesnt make direct risk but its a discrepancy that should be fixed hence reported `Low` severity that might cause issue

## References

(<https://github.com/Folks-Finance/algorand-smart-contract-library/blob/7673a43fa5183af736b65f17d1a297fdea672059/contracts/library/RateLimiter.py#L96-L97>)

(<https://github.com/Folks-Finance/algorand-smart-contract-library/blob/7673a43fa5183af736b65f17d1a297fdea672059/contracts/library/RateLimiter.py#L222-L223>)

## Proof of Concept

## Proof of Concept

discrepancy in code base and document
