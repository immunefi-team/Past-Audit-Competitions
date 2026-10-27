# #46220 \[SC-Insight] Missing Documented Function in the CollateralPool Contract

**Submitted on May 26th 2025 at 20:57:25 UTC by @Victor\_TheOracle for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46220
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**

## Description

## Brief/Intro

The `setAutoClaiming` function is documented as available in the `CollateralPool` contract to allow agent vault owners to set executors for automatic reward and airdrop claiming, but this function is not implemented in the actual `CollateralPool` contract.

## Vulnerability Details

The documentation explicitly states that `setAutoClaiming` should be available in the `CollateralPool` contract:

```
- `setAutoClaiming`: Set executors that can then automatically claim rewards and airdrop. NOTE: only the owner of the pool's corresponding agent vault may call this method.
```

However, the `CollateralPool` contract does not contain any `setAutoClaiming` function implementation, despite being documented as a core feature for automated reward management.

While other documented owner-restricted functions are properly implemented with agent vault owner restrictions, the documented `setAutoClaiming` function is completely missing from the contract.

## Impact Details

The `setAutoClaiming` function is documented but does not exist in the `CollateralPool` contract implementation. Agent vault owners cannot configure automatic reward claiming because the function is missing entirely.

## References

1. `CollateralPool` documentation for the `setAutoClaiming` function: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/docs/ICollateralPool.md?plain=1#L23-L24>
2. `CollateralPool` contract implementation: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>

## Proof of Concept

1. The documentation states that agent vault owners can call `setAutoClaiming` to configure automatic reward claiming:

```
**setAutoClaiming** - Set executors that can then automatically claim rewards and airdrop.
NOTE: only the owner of the pool's corresponding agent vault may call this method.
```

2. Any attempt to call this non-existent function will fail because although it has been documented, it has not been implemented.
