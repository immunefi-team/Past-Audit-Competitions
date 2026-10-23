# #46218 \[SC-Insight] Documentation-Implementation Discrepancy in Agent Vault Access Control

**Submitted on May 26th 2025 at 20:30:49 UTC by @Victor\_TheOracle for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46218
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentVault.sol>
* **Impacts:**

## Description

## Brief/Intro

The `depositCollateral` and `updateCollateral` functions in the Agent Vault contract are documented to be callable by anyone but are implemented with `onlyOwner` modifiers, restricting access to vault owners only. This prevents third-party funding mechanisms and flexible collateral management as described in the documentation.

## Vulnerability Details

The documentation explicitly states that both `depositCollateral` and `updateCollateral` should be callable by anybody:

**Documentation Claims:**

* `depositCollateral`: "NOTE: anybody can call this method, to allow the owner to deposit from any wallet."
* `updateCollateral`: "NOTE: anybody can call this method, to allow the owner to deposit from any source."

**Actual Implementation:**

```solidity
function depositCollateral(IERC20 _token, uint256 _amount)
    external override
    onlyOwner  //@audit Contradicts documentation
{
    _token.safeTransferFrom(msg.sender, address(this), _amount);
    assetManager.updateCollateral(address(this), _token);
    _tokenUsed(_token, TOKEN_DEPOSIT);
}

function updateCollateral(IERC20 _token)
    external override
    onlyOwner  //@audit Contradicts documentation
{
    assetManager.updateCollateral(address(this), _token);
    _tokenUsed(_token, TOKEN_DEPOSIT);
}
```

This implementation shows that it has deviated from what was documented.

## Impact Details

The primary consequence of this access control discrepancy is **loss of flexibility** as third parties cannot assist with collateral management despite this functionality being explicitly documented.

## References

1. AgentVault documentation for the `depositCollateral`, and `updateCollateral` functions: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/docs/IAgentVault.md?plain=1#L1-L5>
2. AgentVault implementation: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/AgentVault.sol#L93-L109>

## Proof of Concept

1. The documentation clearly states that anybody can call the `depositCollateral` and `updateCollateral` methods, to allow the owner to deposit from any wallet or source.

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/docs/IAgentVault.md?plain=1#L1-L5>

```
**depositCollateral** - Deposit vault collateral. Parameter `_token` is explicit to allow depositing before collateral switch.
NOTE: owner must call `token.approve(vault, amount)` before calling this method. NOTE: anybody can call this method, to allow the owner to deposit from any wallet.

**updateCollateral** - Update collateral after `transfer(vault, some amount)` was called (alternative to depositCollateral). Parameter `_token` is explicit to allow depositing before collateral switch.
NOTE: anybody can call this method, to allow the owner to deposit from any source.
```

2. Owner tries to deposit or update collateral from another wallet, but the transaction fails with the "Ownable: caller is not the owner" error.
3. This does not align with the documentation and it should be corrected.
