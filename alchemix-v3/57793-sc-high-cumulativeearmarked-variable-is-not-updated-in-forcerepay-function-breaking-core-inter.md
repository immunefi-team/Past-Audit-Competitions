# 57793 sc high cumulativeearmarked variable is not updated in forcerepay function breaking core internal logic and leading to user funds being stuck&#x20;

**Submitted on Oct 28th 2025 at 22:28:02 UTC by @Tadev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57793
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `cumulativeEarmarked` variable is used to store the system debt currently earmarked for redemption. This variable is updated as follows:

* decremented in the `repay` function and in the `redeem` function
* incremented in the `_earmark` function.

The problem arises because the `_forceRepay` function used for liquidation doesn't update `cumulativeEarmarked` while it should.

This means that with the current design, every time a liquidation for a position with earmarked debt (triggering a force repay) occurs, `cumulativeEarmarked` won't be decremented and will be inflated. This has severe consequences for the protocol as it results in many other variables being wrongly computed.

## Vulnerability Details

The `_forceRepay` function is defined as follows:

```
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        if (amount == 0) {
            return 0;
        }
        _checkForValidAccountId(accountId);
        Account storage account = _accounts[accountId];

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before deciding how much is available to be repaid
        _sync(accountId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 credit = amount > debt ? debt : amount;
        uint256 creditToYield = convertDebtTokensToYield(credit);
        _subDebt(accountId, credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield;

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
        return creditToYield;
    }
```

Contrary to `repay` function which updates `cumulativeEarmarked` after updating `account.earmarked`, `_forceRepay` doesn't update this variable. This is a severe vulnerability which breaks core internal logic.

Indeed, inflated value for `cumulativeEarmarked` will induce multiple consequences.

1. `_earmark` function will be impacted: the line:

```
uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
```

will underestimate the real unearmarked debt. This will lead to `_survivalAccumulator` and `_earmarkWeight` being also wrong.

In an extreme scenario, this line can systematically revert if `cumulativeEarmarked > totalDebt`. This means the whole protocol will be DOS.

2. `_sync` function will also be impacted. Because `_survivalAccumulator` and `_earmarkWeight` values are wrong, many other variables in this function will have a wrong value. In the end, the user raw collateral, debt and earmarked debt will be incorrect.

This is very serious as it may lead to the user unable to withdraw their tokens while they should be able to do so. Also, excess debt could be counted for every user.

## Impact Details

This vulnerability has serious impacts as it breaks the core mechanism of the protocol. `cumulativeEarmarked` being inflated, many other components of the protocol will be affected, leading to potential DOS of the `_earmarked` function which means that all funds in the protocol would be stuck. This vulnerability will also lead to wrong accounting for debt, collateral and earmarking for every user.

## Proof of Concept

## Proof of Concept

Please copy paste the following test in AlchemistV3.t.sol file:

```
    function testCumulativeEarmarkedNotUpdated() external {
        vm.prank(alOwner);
        alchemist.setProtocolFee(protocolFee);

        vm.startPrank(address(0xbeef));
        uint256 amount = 200_000e18;
        SafeERC20.safeApprove(address(vault), address(alchemist), amount );
        // deposit MYT tokens in the Alchemist
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        // mint debt token in the Alchemist
        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // create a redemption to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        // cumulativeEarmarked should be zero at this point
        uint256 cumulativeEarmarked = alchemist.cumulativeEarmarked();
        assert(cumulativeEarmarked == 0);

        // skip to a future block. Lets say 60% of the way through the transmutation period (5_256_000 blocks)
        vm.roll(block.number + (5_256_000 * 60 / 100));

        // earmarked debt should be 60% of the total debt
        (uint256 prevCollateral, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        require(earmarked == prevDebt * 60 / 100, "Earmarked debt should be 60% of the total debt");

        console.log("position collateral:", prevCollateral);
        console.log("position debt:", prevDebt);
        console.log("earmarked debt at 60% of transmutation period:", earmarked);

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yield token supply by 59 bps or 5.9%  while keeping the underlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // poke to update state
        alchemist.poke(tokenIdFor0xBeef);

        console.log("cumulativeEarmarked after poke:", alchemist.cumulativeEarmarked());

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();

        // earmaked debt of the user is 0 after liquidation
        (uint256 depositedCollateral, uint256 debt, uint256 newEarmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        console.log("depositedCollateral after liquidation:", depositedCollateral);
        console.log("debt after liquidation:", debt);
        console.log("earmarked after liquidation:", newEarmarked);

        // cumulativeEarmarked remains unchanged after liquidation
        uint256 newCumulativeEarmarked = alchemist.cumulativeEarmarked();
        console.log("cumulativeEarmarked after liquidation:", newCumulativeEarmarked);
    }
```

This tests highlights the fact that when someone liquidates a user with earmarked debt, the user earmarked debt is repaid but the `cumulativeEarmarked` is not correctly updated.

The output is as follows:

```
[PASS] testCumulativeEarmarkedNotUpdated() (gas: 3373682)
Logs:
  position collateral: 200000000000000000000000
  position debt: 180000000000000000018000
  earmarked debt at 60% of transmutation period: 108000000000000000010800
  cumulativeEarmarked after poke: 108000000000000000010800
  depositedCollateral after liquidation: 83340559999999999876575
  debt after liquidation: 72000000000000000007200
  earmarked after liquidation: 0
  cumulativeEarmarked after liquidation: 72000000000000000007200
```

This means liquidating positions with earmarked debt will always incorrectly leave `cumulativeEarmarked` unchanged, increasing its value over time.
