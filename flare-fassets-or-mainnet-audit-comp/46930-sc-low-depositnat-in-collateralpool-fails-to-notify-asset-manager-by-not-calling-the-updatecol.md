# #46930 \[SC-Low] \`depositNat()\` in \`CollateralPool\` Fails to Notify Asset Manager, By not calling the \`updateCollateral\`

**Submitted on Jun 6th 2025 at 11:47:31 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46930
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/MintingFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

The CollateralPool contract includes a `depositNat()` function that allows the asset manager to deposit native currency (`msg.value`). However, this function does not call `updateCollateral()`, which is essential to notifying the agent collateral system that a deposit has occurred. As a result, although the internal `totalCollateral` value in the pool is updated, the broader system remains unaware of the collateral change.

This breaks critical collateral synchronization between `CollateralPool` and the asset manager, potentially allowing incorrect liquidation behavior and desynchronization of agent status.

```solidity
function depositNat()
    external payable
    onlyAssetManager
{
    @>> _depositWNat();
}

function _depositWNat()
    internal
{
    // msg.value is always > 0 in this contract
    if (msg.value > 0) {
        totalCollateral += msg.value;
       @>> wNat.deposit{value: msg.value}();
    }
}
```

unlike the `agentVault` the `depositNat` calls the `updateCollateral()` as intended:

```solidity
 // used by asset manager to transfer collateral reservation fee to the agent vault
    function depositNat(IWNat _wNat)
        external payable override
        onlyAssetManager
    {
        _wNat.deposit{value: msg.value}();
        assetManager.updateCollateral(address(this), _wNat);
        _tokenUsed(_wNat, TOKEN_DEPOSIT);
    }
```

## Impact Details

This can lead to Incorrect Liquidation Status, since The agent remains in liquidation despite meeting collateral requirements.

## Recommendation

Ensure that every deposit function that affects an agent's collateral position calls `assetManager.updateCollateral()` immediately after the deposit is complete. This guarantees consistency and ensures that liquidation and recovery flows function as intended.

## Proof of Concept

## Proof of Concept

Consider the following scenario:

1. An agent’s collateral ratio drops below the minimum, triggering liquidation.
2. The asset manager calls the `CollateralReservations.distributeCollateralReservationFee()`expects a deposit which can help to rescue the agent (through `depositNat()`).
3. The deposit goes through to the collateralPool via `CollateralPool.depositNat()`, which updates `totalCollateral`, but does not update the asset manager.

As a result, the system continues to treat the agent as undercollateralized unfairly. This can result in the liquidation process continuing despite the agent having sufficient funds.
