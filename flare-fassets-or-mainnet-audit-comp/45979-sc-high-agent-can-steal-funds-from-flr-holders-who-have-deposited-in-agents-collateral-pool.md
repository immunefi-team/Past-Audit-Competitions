# #45979 \[SC-High] Agent can steal funds from FLR holders who have deposited in agent's collateral pool

**Submitted on May 23rd 2025 at 06:02:13 UTC by @a090325 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45979
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An agent (attacker) can update TopupTokenPriceFactorBIPS to be 1 (default: 9950). Right after the update is executed, attacker deposits a small amount of FLR in return for 10000x pool collateral token (PCT), for example: 1 FLR for 10000 PCT (normally 1 FLR \~ 1 PCT). Then attacker can exit the pool with much higher amount of FLR than they deposit (potentially stealing all FLR amount in the pool).

Besides TopupTokenPriceFactorBIPS , attacker also must update Top-up CR to be as close to Exit CR as possible (Top-up CR always < Exit CR). They also must reduce CR to Exit CR and wait for a price fluctuation which makes CR < Top-up CR).

Notes: 24h time-lock between update announcement and execution is a good measure to reduce the risk of an exploit. To avoid being caught in the middle of an attack, attacker can update many parameters at once to conceal malicious intention.

Potential fix: set lower bound for TopupTokenPriceFactorBIPS to be close to default value (9950), for example: 9500, instead of just > 0.

## Vulnerability Details

The root cause is the lower limit for TopupTokenPriceFactorBIPS parameter is just 1, as show in below snippet (<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L142>):

```
 function setTopupTokenPriceFactorBIPS(uint256 _topupTokenPriceFactorBIPS)
        external
        onlyAssetManager
    {
        require(_topupTokenPriceFactorBIPS < SafePct.MAX_BIPS, "value too high");
        require(_topupTokenPriceFactorBIPS > 0, "must be nonzero");
        topupTokenPriceFactorBIPS = _topupTokenPriceFactorBIPS.toUint16();
    }
```

When topupTokenPriceFactorBIPS = 1 is used, the amount of PCT attacker receiving can be 10000x the amount of collateralForTopupPricing, as show in below code snippet (<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L485>):

Note: SafePct.MAX\_BIPS = 10000 --> 10000x inflation.

```
  function _collateralToTokenShare(
        AssetData memory _assetData,
        uint256 _collateral
    )
        internal view
        returns (uint256)
    {
.......
 uint256 collateralAtTopupPrice = collateralForTopupPricing.mulDiv(
            SafePct.MAX_BIPS, topupTokenPriceFactorBIPS);
 uint256 tokenShareAtTopupPrice = poolConsideredEmpty ?
            collateralAtTopupPrice : _assetData.poolTokenSupply.mulDiv(
                collateralAtTopupPrice, _assetData.poolNatBalance);
....... }
```

Example: Before attack, a pool has 10000 FLR, depositors hold 10000 PCT. If attacker can deposit just 10 FLR at Top-up discounted price, they will receive 100000 PCT (10x the amount of honest depositors) --> can withdraw > 90% FLR amount in the pool (>90000 FLR)

## Impact Details

* Attacker: agent
* Victim: FLR holders who have deposited FLR into agent's collateral pool
* Impact: stealing funds (adjusting from Critical --> High because a malicious agent is needed)
* Likelihood: Medium. Just a slight price fluctuation is needed to push CR to be lower than Top-up CR because CR has already reduced to just above Top-up CR by attacker. But 24h time-lock between update announcement and execution can significantly affect the likelihood of a successful attack.

## References

* Default agent settings <https://dev.flare.network/fassets/operational-parameters#default-agent-settings>
* Update agent settings: <https://dev.flare.network/fassets/reference/agent-bot#update-agent-setting>

## Proof of Concept

## Proof of Concept

1. Attacker registers an Agent with default parameters
2. Attract FLR holders to deposit into their collateral pool (can be through an incentive protocol)
3. Mint FXRP (using attacker controlled wallet) and/or withdraw FLR from the pool so that CR \~ Exit CR --> depositor can't exit now. Depositor could try redeeming FXRP to push CR higher to exit the pool but Agent can block those redeeming requests.
4. Announce, wait for time-lock passing and execute the update of TopupTokenPriceFactorBIPS=1, TopupCollateralRatioBIPS so that the Top-up discount rate is significant and the Top-up CR is high enough (just below Exit CR). Attacker can update many other parameters at the same time to conceal their malicious intention.
5. Wait for a price fluctuation which makes CR < Top-up CR. Quickly deposit FLR into collateral pool and receive much larger amount of collateral pool shares (PCT) due to manipulated TopupTokenPriceFactorBIPS value. Anyone who notices the change in step 4 can join the race but attacker has an edge because he controls the update execution timing.
6. Burn FXRP to release collateral so everyone can exit.
7. Burn collateral pool tokens (PCT) to exit the pool. Attacker will receive significantly higher amount of FLR from collateral pool than the amount they've deposited into.
