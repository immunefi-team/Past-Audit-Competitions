# 58723 sc high cumulativeearmarked is not updated at forcerepay&#x20;

**Submitted on Nov 4th 2025 at 09:31:01 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58723
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

in a healthy state, `cumulativeEarmarked` should equal the sum of *all* `account.earmarked` balances. but in the `forceRepay` function, while the `account.earmarked` is deducted by `earmarkToRemove` it is not the case for global `cumulativeEarmarked` . a frequent liquidation where `forceRepay` would actually inflates this state, leading to incorrect global state used by the protocol.

## Vulnerability Details

```solidity
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
...
        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;
```

on the snippet above, the `_forceRepay` is deducting from `account.earmarked`. the amount is based on how much the account can repay their debt where it would prioritize the earmarked amount first.

this is intended but there are oversight on how it should also reduce the global state of `cumulativeEarmarked` but it is not. this issue would make the whole protocol use wrong global state.

## Impact Details

given how `cumulativeEarmarked` is now inflated, various issue would arise. example:

1. in redeem function, the amount is capped by `cumulativeEarmarked` that can be inflated, resulting in more amount than intended can be redeemed unfairly.
2. `_earmark` and `_calculateUnrealizedDebt` relies on `liveUnearmarked = totalDebt - cumulativeEarmarked` that now would be understated resulting in inaccuracy for the rest of the function

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L761-L762>

## Proof of Concept

## Proof of Concept

add this to `src/test/AlchemistV3.t.sol`:

```solidity
    function testForceRepayDoesNotUpdateCumulativeEarmark() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        uint256 sharesBalance = IERC20(address(vault)).balanceOf(address(yetAnotherExternalUser));
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));

        // create redemption and go half maturation
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.roll(block.number + 5_256_000 / 2);
        vm.stopPrank();

        // interact first so cumulativeEarmarked would be populated here
        // can be anything that trigger _earmark()
        alchemist.poke(tokenIdFor0xBeef);

        // modify yield token price via modifying underlying token supply
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        uint256 earmarkBeforeForceRepay = alchemist.cumulativeEarmarked();
        alchemist.liquidate(tokenIdFor0xBeef);
        uint256 earmarkAfterForceRepay = alchemist.cumulativeEarmarked();
        vm.stopPrank();

        // assert the cumualtive earmarked before and after force repay happen
        console.log("earmark before force repay: ", earmarkBeforeForceRepay);
        console.log("earmark after force repay: ", earmarkAfterForceRepay);
        assertEq(earmarkBeforeForceRepay, earmarkAfterForceRepay);
    }

```

the test would pass because the earmark before and after force repay indeed does not change.
