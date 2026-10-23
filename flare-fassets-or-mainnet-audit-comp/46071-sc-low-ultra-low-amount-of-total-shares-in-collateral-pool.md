# #46071 \[SC-Low] Ultra-low amount of total shares in collateral pool

**Submitted on May 24th 2025 at 13:29:59 UTC by @Audittens for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46071
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Using lack of check in the `_selfCloseExitTo` function it is posssible to reach state of ultra-low (but non-zero) amount of shares in collateral pool, leading to attacks involving incorrect calculations due to rounding.

## Vulnerability Details

In the `CollaterPool` contract `assetData.poolTokenSupply` and `assetData.poolNatBalance` represent shares of the pool and balance of the native token, respectively. The following invariant holds across different part of the code: both `poolTokenSupply` and `poolNatBalance` should be big enough (greater than `MIN_TOKEN_SUPPLY_AFTER_EXIT`=`MIN_NAT_BALANCE_AFTER_EXIT`=`1 ether`) or be equal to zero. In the [`_selfCloseExitTo` function](https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/CollateralPool.sol#L313) this invariant maintained as well -- it is [checked](https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/CollateralPool.sol#L324-L333) for the expected updated values:

```solidity=
AssetData memory assetData = _getAssetData();
require(assetData.poolTokenSupply == _tokenShare ||
    assetData.poolTokenSupply - _tokenShare >= MIN_TOKEN_SUPPLY_AFTER_EXIT,
    "token supply left after exit is too low and non-zero");
uint256 natShare = assetData.poolNatBalance.mulDiv(
    _tokenShare, assetData.poolTokenSupply); // poolTokenSupply >= _tokenShare > 0
require(natShare > 0, "amount of sent tokens is too small");
require(assetData.poolNatBalance == natShare ||
    assetData.poolNatBalance - natShare >= MIN_NAT_BALANCE_AFTER_EXIT,
    "collateral left after exit is too low and non-zero");
```

Later, in this function there is a special case for the situation when `maxRedemptionFromAgent` is not big enough to cover `requiredFAssets`:

```solidity=
uint256 maxAgentRedemption = assetManager.maxRedemptionFromAgent(agentVault);
uint256 requiredFAssets = _getFAssetRequiredToNotSpoilCR(assetData, natShare);
// rare case: if agent has too many low-valued open tickets they can't redeem the requiredFAssets
// in one transaction. In that case we lower/correct the amount of spent tokens and nat share.
if (maxAgentRedemption < requiredFAssets) {
    // natShare and _tokenShare decrease!
    requiredFAssets = maxAgentRedemption;
    natShare = _getNatRequiredToNotSpoilCR(assetData, requiredFAssets);
    require(natShare > 0, "amount of sent tokens is too small after agent max redemption correction");
    require(assetData.poolNatBalance == natShare ||
        assetData.poolNatBalance - natShare >= MIN_NAT_BALANCE_AFTER_EXIT,
        "collateral left after exit is too low and non-zero");
    // poolNatBalance >= previous natShare > 0
    _tokenShare = assetData.poolTokenSupply.mulDiv(natShare, assetData.poolNatBalance);
    emit IncompleteSelfCloseExit(_tokenShare, requiredFAssets);
}
```

As mentioned in the [comment](https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/CollateralPool.sol#L339) it is very important to recheck the updated values of `natShare` and `_tokenShare`, as this values are substracted from `poolNatBalance` and `poolTokenSupply`, respectively. The check for `natShare` is present, but the check for `_tokenShare` is missing.

## Impact Details

Because of the mentioned scenario it is possible to reach the state of ultra-low token share. This leads to variety of attacks, e. g., [donation attack](https://www.cyfrin.io/blog/solodit-checklist-explained-3-donation-attacks), errors in roundings during calculations of needed collateral and slashes.

## Proof of Concept

## Proof of Concept

To reach mentioned situation the attacker needs to fill the redemption queue in a way, such that because of the [`maxRedeemedTickets = Globals.getSettings().maxRedeemedTickets` limit](https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/Redemptions.sol#L140-L156) on the number of redemption tickets, the `natShare` value will be equal to `poolNatBalance - 1 ether`, while the `_tokenShare = assetData.poolTokenSupply.mulDiv(natShare, assetData.poolNatBalance);` value will decrease significantly.

To make the scenario more concrete:

* before the call of the `_selfCloseExitTo` function `poolTokenSupply` was equal to `1 ether` and the `poolNatBalance` was equal `100 ether`;
* after the mentioned correction of the `natShare` and `_tokenShare` values: `natShare` equals `99 ether`;
* `_tokenShare` will be corrected to `99/100*poolTokenSupply`;
* in the result the `poolTokenSupply` divided by a factor of one hundred.

Attacker can peform donation (therefore, increase the `poolNatBalance` value) and repeat the mentioned sequence of calls. To pass [checks in lines 325-327 and 331-333](https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/CollateralPool.sol#L325-L333), attacker will make calls in a way, such that at the start of the function, `_tokenShare` equals to `poolTokenSupply`.

By repeating mentioned attack the `poolTokenSupply` will decrease as much as it is required.
