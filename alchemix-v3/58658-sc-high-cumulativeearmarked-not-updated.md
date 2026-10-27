# 58658 sc high cumulativeearmarked not updated

**Submitted on Nov 3rd 2025 at 21:05:19 UTC by @emmac002 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58658
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \_forceRepay function does not update the global cumulativeEarmarked when repaying debt, unlike the standard repay function which correctly decrements both the individual and global earmarked balances. This discrepancy can lead to an inflated global cumulativeEarmarked and cause miscalculations in liveUnearmarked (totalDebt - cumulativeEarmarked).

## Vulnerability Details

In the repay function, it reduces the earmarkToRemove from the individual (account.earmarked) and global variables (cumulativeEarmarked).

```
    // Repay debt from earmarked amount of debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
    cumulativeEarmarked -= earmarkPaidGlobal;
```

But in the \_forceRepay function, it only reduces the earmarkToRemove from the individual (account.earmarked) and fails to adjust the global running total:

```
    // Repay debt from earmarked amount of debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;
```

## Impact Details

Since cumulativeEarmarked is used in critical functions like \_earmark and \_calculateUnrealizedDebt to track system-wide earmarked debt and determine unearmarked debt for yield allocation and redemption, this discrepancy can inflate the global earmarked value and makes liveUnearmarked smaller. \_earmark() can’t earmark as much as it should. \_calculateUnrealizedDebt() shows each token less debt than it actually has.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L522-L526> <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L761-L762>

## Proof of Concept

## POC

```
    function testRepayBeforeAfter() external {
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(100_000e18, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, 8500e18, address(0xbeef));
        alchemist.mint(tokenIdFor0xBeef, 1000e18, address(0xaaaa));
        alchemist.mint(tokenIdFor0xBeef, 500e18, address(0xbbbb));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), type(uint256).max);
        transmuterLogic.createRedemption(3500e18);
        vm.stopPrank();

        vm.startPrank(address(0xaaaa));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), type(uint256).max);
        transmuterLogic.createRedemption(1000e18);
        vm.stopPrank();

        vm.startPrank(address(0xbbbb));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), type(uint256).max);
        transmuterLogic.createRedemption(500e18);
        vm.stopPrank();

        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(100_000e18, address(0xdad), 0);
        uint256 tokenIdFor0xdad = AlchemistNFTHelper.getFirstTokenId(address(0xdad), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xdad, 100e18, address(0xdad));
        vm.stopPrank();

        vm.roll(block.number + 5_256_000 * 2 / 5);

        alchemist.poke(tokenIdFor0xdad);
        alchemist.poke(tokenIdFor0xBeef);

        (uint256 collateral, uint256 debt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xdad);
        (uint256 collateralBeef, uint256 debtBeef, uint256 earmarkedBeef) = alchemist.getCDP(tokenIdFor0xBeef);

        // The first redemption
        vm.startPrank(address(0xaaaa));
        transmuterLogic.claimRedemption(2);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000 / 10);

        // The second redemption
        vm.startPrank(address(0xbbbb));
        transmuterLogic.claimRedemption(3);
        vm.stopPrank();

        alchemist.poke(tokenIdFor0xdad);
        alchemist.poke(tokenIdFor0xBeef);

        (collateral, debt, earmarked) = alchemist.getCDP(tokenIdFor0xdad);
        (collateralBeef, debtBeef, earmarkedBeef) = alchemist.getCDP(tokenIdFor0xBeef);

        assertApproxEqAbs(earmarked + earmarkedBeef, alchemist.cumulativeEarmarked(), 2);
        assertApproxEqAbs(debt + debtBeef, alchemist.totalDebt(), 2);
        console.log("alchemist.cumulativeEarmarked() before:", alchemist.cumulativeEarmarked());
        console.log("alchemist.totalDebt() before:", alchemist.totalDebt());

        vm.startPrank(address(0xdad));
        uint256 repayAmount = 100e18/2;
        console.log(">>>> Repay <<<<");
        alchemist.repay(repayAmount,tokenIdFor0xdad);
        // vm.assertEq(earmarked + earmarkedBeef - repayAmount, alchemist.cumulativeEarmarked());
        console.log("alchemist.cumulativeEarmarked() after:", alchemist.cumulativeEarmarked());
        console.log("alchemist.totalDebt() after:", alchemist.totalDebt());
    }
```

Output:

```
alchemist.cumulativeEarmarked() before: 1750000000000000000000
alchemist.totalDebt() before: 9450000000000000000000
alchemist.cumulativeEarmarked() after: 1732673267326732673266
alchemist.totalDebt() after: 9400000000000000000000
```

Change the forcePay function to external:

```
    function testForceRepayBeforeAfter() external {
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(100_000e18, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, 8500e18, address(0xbeef));
        alchemist.mint(tokenIdFor0xBeef, 1000e18, address(0xaaaa));
        alchemist.mint(tokenIdFor0xBeef, 500e18, address(0xbbbb));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), type(uint256).max);
        transmuterLogic.createRedemption(3500e18);
        vm.stopPrank();

        vm.startPrank(address(0xaaaa));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), type(uint256).max);
        transmuterLogic.createRedemption(1000e18);
        vm.stopPrank();

        vm.startPrank(address(0xbbbb));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), type(uint256).max);
        transmuterLogic.createRedemption(500e18);
        vm.stopPrank();

        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(100_000e18, address(0xdad), 0);
        uint256 tokenIdFor0xdad = AlchemistNFTHelper.getFirstTokenId(address(0xdad), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xdad, 100e18, address(0xdad));
        vm.stopPrank();

        vm.roll(block.number + 5_256_000 * 2 / 5);

        alchemist.poke(tokenIdFor0xdad);
        alchemist.poke(tokenIdFor0xBeef);

        (uint256 collateral, uint256 debt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xdad);
        (uint256 collateralBeef, uint256 debtBeef, uint256 earmarkedBeef) = alchemist.getCDP(tokenIdFor0xBeef);

        // The first redemption
        vm.startPrank(address(0xaaaa));
        transmuterLogic.claimRedemption(2);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000 / 10);

        // The second redemption
        vm.startPrank(address(0xbbbb));
        transmuterLogic.claimRedemption(3);
        vm.stopPrank();

        alchemist.poke(tokenIdFor0xdad);
        alchemist.poke(tokenIdFor0xBeef);

        (collateral, debt, earmarked) = alchemist.getCDP(tokenIdFor0xdad);
        (collateralBeef, debtBeef, earmarkedBeef) = alchemist.getCDP(tokenIdFor0xBeef);

        assertApproxEqAbs(earmarked + earmarkedBeef, alchemist.cumulativeEarmarked(), 2);
        assertApproxEqAbs(debt + debtBeef, alchemist.totalDebt(), 2);
        console.log("alchemist.cumulativeEarmarked() before:", alchemist.cumulativeEarmarked());
        console.log("alchemist.totalDebt() before:", alchemist.totalDebt());

        vm.startPrank(address(0xdad));
        uint256 repayAmount = 100e18/2;
        console.log(">>>> Force Repay <<<<");
        alchemist.forceRepay(tokenIdFor0xdad,repayAmount);
        // vm.assertEq(earmarked + earmarkedBeef - repayAmount, alchemist.cumulativeEarmarked());
        console.log("alchemist.cumulativeEarmarked() after:", alchemist.cumulativeEarmarked());
        console.log("alchemist.totalDebt() after:", alchemist.totalDebt());
    }
```

Output:

```
alchemist.cumulativeEarmarked() before: 1750000000000000000000
alchemist.totalDebt() before: 9450000000000000000000
alchemist.cumulativeEarmarked() after: 1750000000000000000000
alchemist.totalDebt() after: 9400000000000000000000
```
