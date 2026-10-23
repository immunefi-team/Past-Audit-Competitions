# #49061 \[SC-Insight] The Function \`get\_current\_capacity()\` returns wrong value when bucket is infinite

**Submitted on Jul 11th 2025 at 06:51:31 UTC by @c3phas for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49061
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/RateLimiter.py>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When we call the function `get_current_capacity()` with the id of a bucket that has an infinite capacity, ie zero duration, the function returns the capacity as the value set when creating the bucket. ie if we set limit as 100, the capacity would be 100.

## Vulnerability Details

According to the docs, a bucket with a duration of zero is considered an infinite bucket, meaning infinite capacity. For this reason when we check if the bucket has capacity by calling `has_capacity()` it always returns `True`. Function such as `consume_amount` do not modify such a bucket as we should always be able to consume from it.

The problem is for any one relying on the function `get_current_capacity()` maybe on their frontend to determine how much capacity is left, the function would be returning a finite value which can be misleading if they build their logic around this function.

## Impact Details

Any front end relying on the result of `get_current_capacity()` would be reading the wrong value.

## References

<https://github.com/Folks-Finance/algorand-smart-contract-library/blob/7673a43fa5183af736b65f17d1a297fdea672059/contracts/library/RateLimiter.py#L45-L58>

## Proof of Concept

## Proof of Concept

1. Create a bucket with a duration of zero(infinite bucket) passing limit as 5
2. The current capacity of the bucket would in turn be set to 5.
3. If we call `get_current_capacity()` it would return 5.
4. For someone monitoring this value, they might think we only have five requests left for this bucket and could center their logic around 5, reducing the amount for every amount consumed.
