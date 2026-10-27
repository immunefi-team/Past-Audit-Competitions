# #46442 \[SC-Low] Agent collateral pool is vulnerable to inflation attack

**Submitted on May 30th 2025 at 15:10:59 UTC by @a090325 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46442
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Collateral providers can deposit native tokens like FLR, SGB (for the sake of simplicity, I will only mention FLR from now on) in return for pool collateral token (PCT):

```
Amount of received PCT = (amount of deposited FLR) x 
                   (amount of PCT in pool) / (amount of FLR in pool)
```

Normally, 1 FLR in --> \~1 PCT out.

Before a victim (collateral provider) deposit in the pool, attacker can repeatedly create fake minting requests to accumulate a large amount of collateral reservation fee (paid in FLR by minters, usually \~ 0.5%) to inflate the amount of FLR in the pool. When the ratio (amount of PCT in pool) / (amount of FLR in pool) is small enough, for example 1/1000, collateral providers receive 0 PCT if they deposit less than 1000 FLR in one transaction.

See more about inflation attack here: <https://blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks>

## Vulnerability Details

It's impossible for attacker to directly transfer FLR to agent collateral pool but the pool's FLR balance still can increase significantly through Collateral Reservation process for minting.

Portion of Collateral Reservation Fee is sent to collateral pool in below function, increasing the amount of FLR (poolNatBalance) in the pool: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CollateralReservations.sol#L213>

```
 function distributeCollateralReservationFee(
        Agent.State storage _agent,
        uint256 _fee
    )
        internal
    {
        ...
        _agent.collateralPool.depositNat{value: poolFeeShare}();
        ...
    }
```

Then \_assetData.poolNatBalance is used to calculate the amount of PCT (poolToken) a collateral will receive for his deposit: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L485>

```
 function _collateralToTokenShare(
        AssetData memory _assetData,
        uint256 _collateral
    )
        internal view
        returns (uint256)
    {
        bool poolConsideredEmpty = _assetData.poolNatBalance == 0 || _assetData.poolTokenSupply == 0;
        ........... // some helpers
        **uint256 tokenShareAtTopupPrice = poolConsideredEmpty ?
            collateralAtTopupPrice : _assetData.poolTokenSupply.mulDiv(
                collateralAtTopupPrice, _assetData.poolNatBalance);**
       ** uint256 tokenShareAtStandardPrice = poolConsideredEmpty && tokenShareAtTopupPrice == 0 ?
            collateralAtStandardPrice : (_assetData.poolTokenSupply + tokenShareAtTopupPrice).mulDiv(
                collateralAtStandardPrice, _assetData.poolNatBalance + collateralForTopupPricing);**
        return tokenShareAtTopupPrice + tokenShareAtStandardPrice;
    }
```

If \_assetData.poolNatBalance (FLR amount in pool) is much larger than \_assetData.poolTokenSupply (PCT amount) and collateral amount is not large enough, (tokenShareAtTopupPrice + tokenShareAtStandardPrice) could be rounded to 0.

## Impact Details

Attacker can steal funds from collateral providers.

## References

<https://blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks>

## Proof of Concept

## Proof of Concept

It's a highly simplified PoC. Attacker (Alice) can inflate FLR/PCT ratio into higher value. She also can set attractive fee sharing and provide additional incentives to attract more pool collateral providers to maximize her profit.

1. Attacker (Alice) registers a new agent. Minting pool collateral ratio is set to be low \~1.2, pool fee share fee is set to 99.99% to maximize the FLR inflation in pool. Alice also needs to fund other accounts like agent collateral vault (with enough collateral token for step 3)
2. Alice deposits 1200 FLR --> gets 1200 PCT (pool collateral token)
3. Alice (uses a different account for minting only) reserves collateral for minting (\~1000 PCT, 1000 FLR and the equivalent collateral token amount in agent collateral vault). 5 FLR is on-hold as collateral reservation fee (CRF)
4. Alice doesn't really mint anything. And because pool fee share is set to 99.99%, all CRF (5FLR) is tranfered to collateral pool. Now the pool has 1205 FLR for 1200 PCT. Collateral is released in the end and Alice can repeats her attack
5. Alice repeats step 3&4 for 250000 cycles. Now in the pool, the amount of FLR is approximately 1000 times the amount of PCT. The gas cost for this attack settings is around 500 USD (0.1 FLR/cycle x 250000 cycles x 0.02 USD/FLR)
6. If Bob deposits 900 FLR into Alice collateral pool, he will receive 0 PCT because: Bob's PCT = 900 x (amount of PCT in pool) / (amount of FLR in pool) = 9/10 \~ 0 (being rounded down)
7. Alice has successfully stolen 900 FLR from Bob since she owns all the PCT in the pool.
