# #38292 \[SC-Medium] Incorrect Sqrt Calculation Result

**Submitted on Dec 30th 2024 at 12:38:02 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38292
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior
  * (Compiler) Unexpected behavior

## Description

## Brief/Intro

Vyper `sqrt` builtin uses the babylonian method to calculate square roots of decimals. Unfortunately, improper handling of the oscillating final states may lead to `sqrt` incorrectly returning rounded up results.

## Vulnerability Details

Vyper injects the following code to handle calculation of decimal `sqrt`. `x` is the input provided by user.

```
assert x >= 0.0
z: decimal = 0.0

if x == 0.0:
    z = 0.0
else:
    z = x / 2.0 + 0.5
    y: decimal = x

    for i: uint256 in range(256):
        if z == y:
            break
        y = z
        z = (x / z + z) / 2.0
```

Notably, the terminal condition of the algorithm is either `z_cur == z_prev` or the algorithm runs for 256 rounds. The expectation here is that the babylonian method should converge within 256 iterations.

However, if we look carefully at the babylonian algorithm, there's a problem with our usage. It does not guarantee to converge to a single value. For certain inputs, the `z` might actually oscillate between `N` and `N + epsilon`, where `N ** 2 <= x < (N + epsilon) ** 2`. To explain this better, let's look at the following analysis.

When we have `z_cur = N`, the possible `z_next` are

```
z_next = (x / z_cur + z_cur) / 2
       = (x / N + N) / 2
       -> ((((N ** 2) / N) + N) / 2) <= z_next < ((((N + epsilon) ** 2) / N) + N) / 2
       -> N <= z_next < N + epsilon + (epsilon ** 2) / (2 * N)
       -> possible candidates for z_next are N, N + epsilon
```

When we have `z_cur = N + epsilon`, the range of the next possible `z_next` are

```
z_next = (x / z_cur + z_cur) / 2
       = (x / (N + epsilon) + (N + epsilon)) / 2
       -> ((((N ** 2) / (N + epsilon)) + (N + epsilon)) / 2) <= z_next < ((((N + epsilon) ** 2) / (N + epsilon)) + (N + epsilon)) / 2
       -> N + (epsilon ** 2) / (2 * (N + epsilon)) <= z_next < N + epsilon
       -> possible candidates for z_next are N
```

Now we've shown that it is theoretically possible to have oscillating outputs. To further support this analysis, we provide a concrete example where such oscillation happens.

The snippet here returns `0.9999999999`, the **rounded up** result for `sqrt(0.9999999998)`. This is due to the oscillation ending in `N + epsilon` instead of the correct `N`.

```
@external
def test():
    d: decimal = 0.9999999998
    r: decimal = sqrt(d)    #this will be 0.9999999999
```

The most intuitive fix is to compare `y` and `z` after breaking out of the loop, and selecting the smaller of the two. This is the approach used in `isqrt`. It's probably also useful to formally prove convergence (at least up to the oscillating state) within 256 iterations if that's not yet done.

## Impact Details

Precision mishandling and rounding in incorrect directions have been a notorious bug class, and also the root cause of several high profile attacks. These kind of issues have been especially devastating to AMMs that relies on precise tracking of values boundaries (e.g. tick boundaries in uni-v3). So while the `sqrt` miscalculation only happens in a limited set of input, we still believe it is a serious issue due to its potential to introduce subtle bugs in critical business logic.

## References

* `https://github.com/vyperlang/vyper/blob/e20c36376e8566184b63b7ed340e4587bfb3735b/vyper/builtins/functions.py#L2133`

## Proof of Concept

## Proof of Concept

Already shown in Vulnerability Details section.
