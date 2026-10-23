# #45813 \[SC-Insight] Missing \`setAutoClaiming\` Function

**Submitted on May 20th 2025 at 21:27:19 UTC by @Victor\_TheOracle for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45813
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `AgentVault` documentation specifies a `setAutoClaiming` function that allows the vault owner to authorize specific executors and recipients to automatically claim delegation rewards and airdrops. However, this function is entirely absent from the contract implementation.

## Vulnerability Details

According to the official documentation, the `setAutoClaiming` function is designed to let the vault owner define which addresses are allowed to automatically claim both rewards and airdrops on their behalf.

However, in the current implementation of the `AgentVault` contract, **no such function is present**. The absence of `setAutoClaiming` results in the following issues:

* **Automated claiming cannot be configured**, despite the documentation implying otherwise.
* **Any systems or frontends that integrate with the contract expecting this feature will break**.

This omission represents a significant deviation from the specified functionality.

## Impact Details

The primary consequence of this missing function is **loss of automation** as vault owners cannot assign trusted claim executors despite this functionality being explicitly documented.

## References

* AgentVault Documentation for `setAutoClaiming`– <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/docs/IAgentVault.md?plain=1#L40-L41>
* AgentVault.sol (source): function not implemented - <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentVault.sol>

## Proof of Concept

1. `setAutoClaiming` is described as a function callable only by the vault owner to set executors and recipients for auto-claiming.

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/docs/IAgentVault.md?plain=1#L40-L41>

```
**setAutoClaiming** - Set executors and recipients that can then automatically claim rewards and airdrop.
NOTE: only the owner of the agent vault may call this method.
```

2. The `AgentVault.sol` contract does not define `setAutoClaiming` anywhere in its implementation.
3. Callers attempting to use the function via ABI or frontend integration will get an error that tells them that the *function does not exist*.
