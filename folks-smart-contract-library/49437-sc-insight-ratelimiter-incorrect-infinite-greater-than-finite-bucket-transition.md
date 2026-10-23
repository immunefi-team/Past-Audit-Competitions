# #49437 \[SC-Insight] \`RateLimiter\`: incorrect infinite -> finite bucket transition

**Submitted on Jul 15th 2025 at 23:42:49 UTC by @ustas for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49437
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/RateLimiter.py>
* **Impacts:**
  * Temporary denial of service for more than one block
  * Temporary denial of service (smart contract is made unable to operate for one block, functionality is restored in the next block)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

The documentation for the `RateLimiter` contract specifies that a bucket with `duration = 0` is considered an infinite bucket, meaning it should not be subject to capacity limits. For these buckets, core functions like `_update_capacity`, `_consume_amount`, and `_fill_amount` become no-ops, and `has_capacity` always returns `True`, effectively disabling the rate-limiting checks.

However, the transition of a finite bucket to an infinite one will keep the state variables `limit` and `current_capacity` without change, preserving the previous state. When an infinite bucket is then transitioned back to finite, the state variables will be simply unfreezed without the imitation of the rate limit change calculations from the infinite to a normal value.

## Remediation

Add a special case to the `_update_rate_duration` function to fully restore the capacity if the previous `duration` value was 0, as any finite rate limit is always smaller than the infinity.

## Proof of Concept

## Proof of Concept

A visual example:

```
Finite bucket A

In storage
    Limit: 100
    Duration: 100
    Current capacity: 77

Effective
    Limit: 100
    Duration: 100
    Current capacity: 77
```

After the transition to infinity:

```
Infinite bucket A

In storage
    Limit: 100
    Duration: 0
    Current capacity: 77

Effective
    Limit: infinite
    Duration: 0
    Current capacity: infinite
```

After the transition back to finite:

```
Finite bucket A

In storage
    Limit: 100
    Duration: 100
    Current capacity: 77

Effective
    Limit: 100
    Duration: 100
    Current capacity: 77
```

While the correct values would be:

```
Finite bucket A

In storage
    Limit: 100
    Duration: 100
    Current capacity: 100

Effective
    Limit: 100
    Duration: 100
    Current capacity: 100
```

Representing the transition from the infinite limits to the previous limited ones.
