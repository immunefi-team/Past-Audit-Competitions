# #47121 \[SC-Insight] Incorrect documentation on pool Top-up feature

**Submitted on Jun 9th 2025 at 03:30:28 UTC by @a090325 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47121
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**

## Description

## Brief/Intro

* The docs (listed in "references) said

> the pool can be topped up at a reduced price when the CR is above the top-up CR

* That’s inconsistent with the actual implementation of this feature, which is correctly implemented. The documentation should be updated to:

> the pool can be topped up at a reduced price when the CR is below the top-up CR

## Vulnerability Details

* The collateralAtTopupPrice is calculated as:

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L499>

```
uint256 collateralAtTopupPrice = collateralForTopupPricing.mulDiv(
            SafePct.MAX_BIPS, topupTokenPriceFactorBIPS);
```

collateralForTopupPricing depends on the gap between pool Top-ups CR and pool CR. So by setting Minting pool CR to be significant lower than Top-ups CR and minting FXRP to reduce pool CR to Minting pool CR, attacker can expand that gap and buy more CPT at a reduced price in a Top-up. <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L494>

```
 uint256 _aux = (_assetData.assetPriceMul * _assetData.agentBackedFAsset).mulBips(topupCollateralRatioBIPS);
        uint256 natRequiredToTopup = _aux > _assetData.poolNatBalance * _assetData.assetPriceDiv ?
            _aux / _assetData.assetPriceDiv - _assetData.poolNatBalance : 0;
        uint256 collateralForTopupPricing = Math.min(_collateral, natRequiredToTopup);
```

Top-up happens only when \_aux > \_assetData.poolNatBalance \* \_assetData.assetPriceDiv ? . And

```
_aux = (_assetData.assetPriceMul * _assetData.agentBackedFAsset).mulBips(topupCollateralRatioBIPS)
```

So Top-up happens when pool CR falls below Top-up CR (contrary to the docs)

## Impact Details

This issue could cause confusion to users and auditors as well.

## References

-<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L494-499>

* <https://dev.flare.network/fassets/collateral#top-up>
* <https://dev.flare.network/fassets/collateral#agent-thresholds>

## Proof of Concept

## Proof of Concept

This could be simulated through Fasset testing suit:

1. Create an agent
2. Mint some FXRP
3. Reduce collateral in the pool so that pool CR < Top-up CR
4. Top-up the pool. Depositor who tops-up the pool will receive more PCT than usual.
5. Deposit more into the pool. Depositor cannot get discount now.
