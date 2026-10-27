# #46266 \[SC-Insight] Cannot use a pool token suffix of MAX\_SUFFIX\_LEN

**Submitted on May 27th 2025 at 14:47:51 UTC by @Oxgritty for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46266
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/AgentsCreateDestroy.sol>
* **Impacts:**

## Description

## Brief/Intro

* When agent owner wants to create a vault, along with other parameters he also has to provide `poolTokenSuffix`, now the intended behaviour is that, its maximum length can be of `MAX_SUFFIX_LEN` but due to a logical error, agent owner can only provide a `poolTokenSuffix` of max length `MAX_SUFFIX_LEN - 1`.

## Vulnerability Details

* For creating an agent `createAgentVault` function is called along with certain parameters of type `AgentSettings.Data calldata _settings`, inside `_settings` there is one parameter called `_settings.poolTokenSuffix`, now its max length can be of `MAX_SUFFIX_LEN`, but due the use of `<` instead of `<=`, its maximum allowed length is `MAX_SUFFIX_LEN - 1`.

## Impact Details

* The maximum length allowed for `poolTokenSuffix` will be `MAX_SUFFIX_LEN - 1`, which is 1 less than the intended length.

## References

* <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentsCreateDestroy.sol#L249>

## Proof of Concept

## Proof of Concept

1. Agent owner calls `createAgentVault` with `_settings` and length of `_settings.poolTokenSuffix` is equal to `MAX_SUFFIX_LEN`.
2. The code will revert in `_reserveAndValidatePoolTokenSuffix` due to following check:

```solidity
        require(len < MAX_SUFFIX_LEN, "suffix too long");
```
