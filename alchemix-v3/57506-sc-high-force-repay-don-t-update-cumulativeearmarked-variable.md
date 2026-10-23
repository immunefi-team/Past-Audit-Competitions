# 57506 sc high force repay don t update cumulativeearmarked variable

**Submitted on Oct 26th 2025 at 20:09:20 UTC by @OxPhantom for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57506
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Force repay during liquidation decrements an account’s `earmarked` debt but does not decrement the global `cumulativeEarmarked`. This desynchronizes global earmark accounting, leading to an underestimated unearmarked bucket and potentially throttling future earmarks and mis-weighting decay.

## Vulnerability Details

In `repay()`, the contract removes earmark at both the account and global levels:

```solidity
// Repay debt from earmarked amount of debt first
uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
account.earmarked -= earmarkToRemove;

uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
cumulativeEarmarked -= earmarkPaidGlobal;
```

But in `_forceRepay()` (used by liquidation pre-step), only the account’s earmark is decremented; the global `cumulativeEarmarked` is not updated:

```solidity
// Repay debt from earmarked amount of debt first
uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
account.earmarked -= earmarkToRemove;
```

Effects:

* `cumulativeEarmarked` stays too high, so `liveUnearmarked = totalDebt - cumulativeEarmarked` is too low.
* Future `_earmark()` calls may clamp `amount` unnecessarily: `if (amount > liveUnearmarked) amount = liveUnearmarked;`.
* Earmark/redemption weights and survival math use `cumulativeEarmarked` (e.g., in `_doLiquidation` and decay weighting), potentially skewing per-user attribution after force-repay events.

## Impact Details

* Operationally reduces earmark throughput post-liquidation (less debt considered unearmarked than reality), delaying redemptions.
* Skews global decay/weight accounting (e.g., `_redemptionWeight` is parameterized by `cumulativeEarmarked`).

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L758-L763>

## Proof of Concept

## Proof of Concept

You can copy paste this code in AlchemistV3.t.sol and run `forge test --mt testLiquidate_POC -vv`

This POC demonstrated that the cumulativeEarmarked is not changed by force repay.

```solidity
    function testLiquidate_POC() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;

        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Need to start a transmutator deposit, to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        
        // skip to a future block. Lets say 60% of the way through the transmutation period (5_256_000 blocks)
        vm.roll(block.number + (5_256_000 * 60 / 100));

        // Earmarked debt should be 60% of the total debt
        (, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        require(earmarked == prevDebt * 60 / 100, "Earmarked debt should be 60% of the total debt");

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);
        alchemist.poke(tokenIdFor0xBeef);
        uint256 cumulativeEarmarkedBefore= alchemist.cumulativeEarmarked();
        // let another user mint a debt and liquidate the previous user position
        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);

        alchemist.deposit(depositAmount, address(externalUser), 0);
         uint256 tokenIdForExternalUser= AlchemistNFTHelper.getFirstTokenId(address(externalUser), address(alchemistNFT));

        alchemist.mint(tokenIdForExternalUser, 36000000000000000003600, address(externalUser)   );
        alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();
        uint256 cumulativeEarmarkedAfter= alchemist.cumulativeEarmarked();

        console.log("cumulativeEarmarkedAfter", cumulativeEarmarkedAfter);
        console.log("cumulativeEarmarkedBefore", cumulativeEarmarkedBefore);
        assertEq(cumulativeEarmarkedAfter,cumulativeEarmarkedBefore, "cumulativeEarmarked didn't changed !");
    }
```
