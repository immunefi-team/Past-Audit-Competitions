# #47020 \[SC-High] A malicious agent can extract funds from the collateral pool by diluting the value of existing collateral providers' shares.

**Submitted on Jun 8th 2025 at 03:26:24 UTC by @a090325 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47020
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

To reduce the risk of liquidations due to low collateral in the pool, it can be topped up at a discounted rate when the collateral ratio (CR) falls below the top-up threshold. This involves minting new PCT shares at a lower price, which can lead to a slight decrease in the PCT price (in comparison to FLR). However, top-ups are rare and usually only triggered during significant market volatility (leading to a partial liquidation event), so the impact on PCT price is typically minimal.

The problem here is an attacker (malicious agent) can trigger many "artificial top-ups" whenever they choose, allowing them to buy a lot of PCT at a reduced price. This dilutes the value of existing collateral providers' shares, enabling the attacker to extract more value from the collateral pool than they originally deposited.

More information about pool collateral thresholds here (<https://dev.flare.network/fassets/collateral#agent-thresholds>). Please be noted that in the docs (Top-up definition section):

> "the pool can be topped up at a reduced price when the CR is above the top-up CR."

That's incorrect. It should be: "...when the CR is **below** the top-up CR". The reason is that after a partial liquidation, the pool CR is higher than minimum CR but could be still quite low to make the pool become healthy. Top-up mechanism provides incentives for collateral providers to help increase pool CR to reach a healthier value (Top-up CR). Top-up CR serves as the upper limit. I'll explain it in more detail (code analysis) in Vulnerability Details part.

## Vulnerability Details

* The core issue is that the Minting Pool CR can be set lower than the Top-up CR.
* To meet the basic requirement for minting, a pool's collateral ratio (CR) only needs to exceed the minimum pool CR (Condition A). However, to actually mint FXRP, the pool CR must be above the Safety CR threshold of 1.6. <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Agents.sol#L42>

```
 require(_mintingPoolCollateralRatioBIPS >= collateral.minCollateralRatioBIPS,
            "collateral ratio too small");
        _agent.mintingPoolCollateralRatioBIPS = _mintingPoolCollateralRatioBIPS.toUint32();
```

* Top-up CR is required to be lower than Exit CR, not Minting CR or Safety CR (Condition B). <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L132>

```
require(_topupCollateralRatioBIPS < exitCollateralRatioBIPS, "value too high");
        require(_topupCollateralRatioBIPS > 0, "must be nonzero");
        topupCollateralRatioBIPS = _topupCollateralRatioBIPS.toUint32();
```

* **From condition A and B, we can conclude that Minting pool CR can be set to be lower than Top-up CR. When Minting pool CR < Top-up CR, an attacker can mint FXRP to reduce pool CR to be lower than Top-up CR, triggering an "artificial Top-up" opportunity.** This allows the attacker to purchase PCT at a discounted price whenever they choose, diluting the value of existing PCT holders' shares.
* The collateralAtTopupPrice is calculated as: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L499>

```
uint256 collateralAtTopupPrice = collateralForTopupPricing.mulDiv(
            SafePct.MAX_BIPS, topupTokenPriceFactorBIPS);
```

* collateralForTopupPricing depends on the gap between pool Top-ups CR and pool CR. So by setting Minting pool CR to be significant lower than Top-ups CR and minting FXRP to reduce pool CR to Minting pool CR, attacker can expand that gap and buy more CPT at a reduced price in a Top-up. <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L494>

```
 uint256 _aux = (_assetData.assetPriceMul * _assetData.agentBackedFAsset).mulBips(topupCollateralRatioBIPS);
        uint256 natRequiredToTopup = _aux > _assetData.poolNatBalance * _assetData.assetPriceDiv ?
            _aux / _assetData.assetPriceDiv - _assetData.poolNatBalance : 0;
        uint256 collateralForTopupPricing = Math.min(_collateral, natRequiredToTopup);
```

Top-up happens only when *\_aux > \_assetData.poolNatBalance*. And

```
_aux = (_assetData.assetPriceMul * _assetData.agentBackedFAsset).mulBips(topupCollateralRatioBIPS)
```

**So Top-up happens when pool CR falls below Top-up CR (contrary to the docs)**

* Attacker can trigger "artificial Top-ups" many times with just limited initial funds (more details in below PoC).
* Additionally, pool fee share can be set to 0%, so the only cost of an exploit is gas fee. <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Agents.sol#L63>

```
require(_poolFeeShareBIPS <= SafePct.MAX_BIPS, "value too high");
        _agent.poolFeeShareBIPS = _poolFeeShareBIPS.toUint16();
```

## Impact Details

A malicious agent could exploit this vulnerability to steal funds from collateral providers. Based on live data from collateral pools on Songbird (<https://fasset.oracle-daemon.com/sgb/pools>), approximately $150,000 to $900,000—about 10-50% of the total value across all pools—is currently at risk.

## References

* <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L499>
* <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Agents.sol#L42>
* <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L132>
* <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Agents.sol#L63>
* <https://dev.flare.network/fassets/collateral#agent-thresholds>
* <https://dev.flare.network/fassets/collateral#top-up>

## Proof of Concept

## Proof of Concept

1. Attacker (Alice) creates an agent, depositing 1M FLR into collateral pool, receiving 1M PCT.
2. Victim (Bob) deposits 1M FLR into collateral pool, receiving 1M PCT.
3. Alice updates agent's settings so that:

* Exit CR = 3,01
* Top-up CR = 3 (< Exit CR)
* Minting CR = 2 (> Safety CR = 1,6)
* Pool fee share = 0%

4. Alice mints enough FXRP, reducing pool CR to 2 (< Top-up CR)
5. Alice deposits another 1M FLR, receiving 1M + 5025 PCT (due to Top-up effect). Total FLR in the pool increases to 3M.
6. Alice self-redeems all FXRP (backing by her pool).
7. Alice redeems 1M + 5025 PCT against the pool, receiving 1M + 3344 FLR reducing total FLR in the pool by 3344 FLR. After this step, Alice still owns 1M PCT (50% of the pool), but has already drained 3344 FLR from the pool (1672 FLR from Bob)
8. Alice repeats step 4-7 for 1000 times. Each times the total FLR in the pool is reduced by 0,1672% (5025/(3M+5025) = 0,001672 = 0,1672%). After 1000 iterations, total FLR in the pool: 2M\*(1-0,1672%)^1000 = 0,38M FLR. Alice has drained 1,62 M FLR from the pool and she still owns 50% of remaining funds.
9. Alice redeems all her 1M PCT, receiving 0,19M FLR. Since the gas fee is minimal, Alice gains: (1,62+0,19)-1 = 0,81M FLR as a profit. The required funds for her attack is around 3M FLR (2M FLR for collateral/top-up funds and an amount of XRP worthes 1M FLR for minting).
