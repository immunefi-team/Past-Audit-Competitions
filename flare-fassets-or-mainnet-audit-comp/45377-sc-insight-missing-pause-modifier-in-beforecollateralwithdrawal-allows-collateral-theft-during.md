# #45377 \[SC-Insight] Missing pause modifier in \`beforeCollateralWithdrawal\` allows collateral theft during a pause

**Submitted on May 13th 2025 at 13:28:25 UTC by @Rhaydden for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45377
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-labs-ltd/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `beforeCollateralWithdrawal` hook in `AgentCollateralFacet` is missing the `notEmergencyPaused` check, so even when the `AssetManager` is emergency-paused, an agent vault can still call `withdraw()`, triggering this hook and draining NAT collateral.

## Vulnerability Details

In `AgentCollateralFacet.sol` the withdrawal entry-point is defined without a pause guard:

```solidity
 /**
     * Called by AgentVault when agent calls `withdraw()`.
     * NOTE: may only be called from an agent vault, not from an EOA address.
     * @param _valueNATWei the withdrawn amount
     */
    function beforeCollateralWithdrawal(
        IERC20 _token,
        uint256 _valueNATWei
    )
        external
    {
        // AgentsExternal.beforeCollateralWithdrawal makes sure that only a registered agent vault can call
        AgentsExternal.beforeCollateralWithdrawal(_token, msg.sender, _valueNATWei);
    }
```

That delegates into `AgentsExternal.beforeCollateralWithdrawal`, which performs status, timing and collateral-ratio checks but never checks `emergencyPausedUntil`. Meanwhile all other dangerous flows (e.g. `transferToCoreVault`, `redeemFromCoreVault`) are properly guarded by:

```solidity
modifier notEmergencyPaused {
    _checkEmergencyPauseNotActive();
    _;
}
```

Because `beforeCollateralWithdrawal` skips this modifier, a vault can still execute a collateral withdrawal even if the contract is paused until `emergencyPausedUntil`.

Here's how the entire collateral withdrawal process works:

* First, an agent announces their intention to withdraw collateral via `announceVaultCollateralWithdrawal` or `announceAgentPoolTokenRedemption`
* They must wait for the specified timelock period (defined by `withdrawalWaitMinSeconds`)
* After the waiting period, the agent calls `withdraw()` on their AgentVault contract
* The AgentVault then calls `beforeCollateralWithdrawal` on the AssetManager before transferring the tokens
* If `beforeCollateralWithdrawal` passes (doesn't revert), the actual token transfer occurs in the AgentVault

But note this is how `beforeCollateralWithdrawal` enables withdrawals:

* It verifies that the withdrawal has been properly announced and the waiting period has passed
* It checks that the withdrawal is happening within the allowed time window
* It confirms the withdrawal amount doesn't exceed what was announced
* It updates the withdrawal announcement state (reducing announced amount or clearing it)
* By returning successfully without reverting, it effectively authorizes the AgentVault to proceed with the actual token transfer

This function is the last protocol-level check before tokens actually leave the protocol. Without this function's approval, AgentVaults can't release collateral tokens.

## Impact Details

Impact falls under Direct theft of any user funds (at-rest or in-motion).

An agent vault owner can withdraw all of their NAT collateral at any time during an emergency pause. If multiple vaults collude, they can drain pooled collateral, rendering the protocol insolvent.

If the `beforeCollateralWithdrawal` function were to include the `notEmergencyPaused` modifier, it woould prevent any collateral withdrawals during an emergency pause period. This would lock all collateral in the protocol during emergencies.

Without this modifier, agents could still withdraw collateral during an emergency pause.

## References

<https://github.com/flare-labs-ltd/fassets//blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/facets/AgentCollateralFacet.sol#L53-L61>

<https://github.com/flare-labs-ltd/fassets//blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentsExternal.sol#L98-L137>

## Proof of Concept

## Proof of Concept

1. Setup

* Deploy AssetManager + AgentVault(s).
* Deposit NAT collateral into an agent vault and call `announceVaultCollateralWithdrawal(...)`.

2. Pause

* Governance or emergency-pause sender calls

```
assetManagerController.callOnManagers(
  assetManagerAddress,
  assetManager.pauseEmergency(...parameters...)
)
```

* `AssetManagerState.emergencyPausedUntil` is set > now.

3. Bypass

* Wait until `withdrawalAllowedAt` timestamp passes.
* Despite the pause, call:

```
agentVault.withdraw(valueWei)  
// internally calls facet.beforeCollateralWithdrawal(...)
```

* Because `beforeCollateralWithdrawal` lacks notEmergencyPaused, it succeeds.
* Vault transfers NAT to caller, draining collateral.

## Fix

```diff

     /**
      * Called by AgentVault when agent calls `withdraw()`.
      */
    function beforeCollateralWithdrawal(

+        // block during emergency pause
         IERC20 _token,
         uint256 _valueNATWei
     )
        external
+        notEmergencyPaused
     {
         AgentsExternal.beforeCollateralWithdrawal(_token, msg.sender, _valueNATWei);
     }
```
