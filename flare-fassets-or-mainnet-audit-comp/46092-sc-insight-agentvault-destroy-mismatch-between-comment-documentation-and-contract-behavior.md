# #46092 \[SC-Insight] AgentVault::destroy mismatch between comment documentation and contract behavior

**Submitted on May 24th 2025 at 18:37:18 UTC by @hunter0xweb3 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46092
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentVault.sol>
* **Impacts:**

## Description

## Brief/Intro

The documentation comments in AgentVault::destroy states that all funds (tokens and native balance) are transfer out to AgentVaultOwner\
However destroy functionality behaves different

## Vulnerability Details

The documentation comments in AgentVault::destroy states that all funds (tokens and native balance) are transfered out to AgentVaultOwner\
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/AgentVault.sol#L189-L190>:

```js
    // Used by asset manager when destroying agent.
    // Completely erases agent vault and transfers all funds to the owner.
```

However destroy functionality behaves different because it allows sending the funds to an arbitrary address (defined on call by assetManager).\
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/AgentVault.sol#L189-L218>

```js
@>    function destroy(address payable _recipient)
        external override
        onlyAssetManager
        nonReentrant
    {
        uint256 length = usedTokens.length;
        for (uint256 i = 0; i < length; i++) {
            IERC20 token = usedTokens[i];
            uint256 useFlags = tokenUseFlags[token];
            //... snippet; undelegation logic...
            if ((useFlags & TOKEN_DEPOSIT) != 0) {
                uint256 balance = token.balanceOf(address(this));
                if (balance > 0) {
@>                    token.safeTransfer(_recipient, balance);
                }
            }
        }
        // transfer native balance, if any (used to be done by selfdestruct)
@>      Transfers.transferNAT(_recipient, address(this).balance);
    }
```

This leads to a mismatch between documentation and contract behavior leading to potentially loss of funds and mismatch behavior between documentation and implementation described in detail in impact section

A better approach is use assetManager.getAgentVaultOwner(address(this)) to retrieve the owner address like used in AgentVault::transferExternalToken

```js
    function transferExternalToken(IERC20 _token, uint256 _amount)
        external override onlyOwner nonReentrant {
        //...
@>        address ownerManagementAddress = assetManager.getAgentVaultOwner(address(this));
@>        _token.safeTransfer(ownerManagementAddress, _amount);
    }
```

## Impact Details

* Extra parameter defined in AgentVault::destroy leading to potentially loss of funds if non owner controlled address is provided by assetManager
* Mismatch between documentation and contract behavior

## References

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/AgentVault.sol#L189-L190\\>
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/AgentVault.sol#L191-L218\\>
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/AgentVault.sol#L124-L132>

## Proof of Concept

## Proof of Concept

AgentVault::destroy is called with \_recipient parameter address non controlled by AgentVault's owner\
Loss of funds occurs because of mismatch between documentation and contract behavior
