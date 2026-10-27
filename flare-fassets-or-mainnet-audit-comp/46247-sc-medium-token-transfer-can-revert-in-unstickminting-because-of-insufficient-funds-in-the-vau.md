# #46247 \[SC-Medium] Token transfer can revert in unstickMinting because of insufficient funds in the vault.

**Submitted on May 27th 2025 at 08:37:42 UTC by @ni8mare for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46247
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `unstickMinting` function may fail when attempting to transfer vault collateral (typically USDX stablecoin) to the agent's owner address. This occurs when the fAsset price increases between collateral reservation and the unstick operation, causing the calculated transfer amount to exceed the vault's available balance, resulting in a denial of service.

## Vulnerability Details

Agents call `unstickMinting` - (<https://dev.flare.network/fassets/minting#expired-proof>) as a last resort to unlock collateral when minters cannot complete the minting process and agents are unable to call default within 24 hours of collateral reservation.

The vulnerability occurs in the following sequence:

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralReservations.sol#L181>

```solidity
function unstickMinting(
        IConfirmedBlockHeightExists.Proof calldata _proof,
        uint64 _crtId
    ) internal {
        ...SNIP...
        
        // burn reserved collateral at market price
        uint256 amgToTokenWeiPrice = Conversion.currentAmgPriceInTokenWei(
            agent.vaultCollateralIndex
        );
        uint256 reservedCollateral = Conversion.convertAmgToTokenWei(
            crt.valueAMG, //doesn't consider minting fee??
            amgToTokenWeiPrice
        );
->       Agents.burnVaultCollateral(agent, reservedCollateral);
        
       ...SNIP..
    }
    
    
    // Used by asset manager for liquidation and failed redemption.
    // Is nonReentrant to prevent reentrancy in case the token has receive hooks.
    function payout(IERC20 _token, address _recipient, uint256 _amount)
        external override
        onlyAssetManager
        nonReentrant
    {
    //@audit revert here because _amount > balance of vault
    ->   _token.safeTransfer(_recipient, _amount);
    }
```

The `burnVaultCollateral` function calculates the transfer amount based on current market prices without verifying vault balance:

```solidity
function burnVaultCollateral(
    Agent.State storage _agent,
    uint256 _amountVaultCollateralWei
) internal {
    // ... initialization code ...
    
    if (vaultCollateral.token == poolCollateral.token) {
        burnVaultNATCollateral(_agent, _amountVaultCollateralWei);
    } else {
        // ... NAT calculation code ...
        
        // @audit: Transfer may fail if vault balance < _amountVaultCollateralWei
        vault.payout(
            vaultCollateral.token,
            _agent.ownerManagementAddress,
            _amountVaultCollateralWei
        );
        
        // ... remaining transfer logic ...
    }
}
```

The `payout` function performs the transfer without balance validation:

```solidity
function payout(IERC20 _token, address _recipient, uint256 _amount)
    external override
    onlyAssetManager
    nonReentrant
{
    // @audit: Reverts if _amount > token balance of vault
    _token.safeTransfer(_recipient, _amount);
}

```

## Impact Details

This vulnerability causes the `unstickMinting` function to revert in critical situations where agents need emergency exit capabilities. Since this is an emergency function designed for rare but important scenarios, any failure is unacceptable and could leave agents unable to recover their collateral.

The impact represents a **denial of service** where `the smart contract becomes unable to operate due to insufficient token funds`, which should be classified as a **medium severity** vulnerability according to Immunefi's impact classification guidelines.

## References

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralReservations.sol#L181>

## Proof of Concept

## Proof of Concept

1. Agent vault have 120 USDX worth 120$.
2. minter call `reserveCollateral` and want to mint 100 fXRP when 1 fXRP = 1$ so total value he want to mint is 100$.
3. Because vault CR is 120%, all the 120 USDX is reserved for the minting.
4. After 24 hours price of fXRP is 1.5$.
5. agent call `unstickMinting`
6. Because market price of fXRP has increased, so the calculated `reservedCollateral` will be 150$(USDX).
7. In `payout` function, it tries to transfer 150 USDX, but it fails because the balance of the vault is only 120 USDX.
