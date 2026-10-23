# #46677 \[SC-Insight] Wrong comment in \_getFAssetRequiredToNotSpoilCR

**Submitted on Jun 3rd 2025 at 08:56:15 UTC by @Paludo0x for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46677
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**

## Description

## Vulnerability Details

In CollateralPool.\_getFAssetRequiredToNotSpoilCR, the else‐branch carries this comment:

```
// f-asset that preserves pool CR (assume poolNatBalance >= natShare > 0)
// solve (N - n) / (F - f) = N / F get n = N f / F
return _assetData.agentBackedFAsset.mulDiv(_natShare, _assetData.poolNatBalance);
```

Here “N” refers to poolNatBalance, “F” to agentBackedFAsset, and “n” to the NAT withdrawn. But the code inside this branch computes

```
f = F * n / N 
```

This should be a copy/paste error from function `_getNatRequiredToNotSpoilCR` which is correct.

## Proof of Concept

## Proof of Concept

Snippets from relevant code

```
    function _getFAssetRequiredToNotSpoilCR(
        AssetData memory _assetData,
        uint256 _natShare
    )
        internal pure
        returns (uint256)
    {
        // calculate f-assets required for CR to stay above min(exitCR, poolCR) when taking out _natShare
        // if pool is below exitCR, we shouldn't require it be increased above exitCR, only preserved
        // if pool is above exitCR, we require only for it to stay that way (like in the normal exit)
        if (_staysAboveCR(_assetData, 0, _assetData.exitCR)) {
            // f-asset required for CR to stay above exitCR (might not be needed)
            // solve (N - n) / (p / q (F - f)) >= cr get f = max(0, F - q (N - n) / (p cr))
            return MathUtils.subOrZero(_assetData.agentBackedFAsset, _assetData.assetPriceDiv *
                (_assetData.poolNatBalance - _natShare) * SafePct.MAX_BIPS /
                (_assetData.assetPriceMul * _assetData.exitCR)
            ); // _assetPriceMul > 0, exitCR > 1
        } else {
            // f-asset that preserves pool CR (assume poolNatBalance >= natShare > 0)
            // solve (N - n) / (F - f) = N / F get n = N f / F
            return _assetData.agentBackedFAsset.mulDiv(_natShare, _assetData.poolNatBalance);
        }
```
