# 58337 sc high incorrect handling of cumulativeearmarked in forcerepay leads to inflated survival accumulator&#x20;

## #58337 \[SC-High] Incorrect Handling of cumulativeEarmarked in \_forceRepay leads to inflated survival accumulator.

**Submitted on Nov 1st 2025 at 11:22:49 UTC by @Bizarro for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58337
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

### Description

### Brief/Intro

In the `_forceRepay` function, when a liquidator liquidates a user, the user's individial earmarked amount is reduced but the global cumulativeEarmarked variable is not reduced. This leads to an inflated cumulativeEarmarked value, when redemptions occur, which in turn causes the `_survivalAccumulator` to be incorrectly calculated.

### Vulnerability Details

When a liquidator liquidates a user's position, the liquidate function is called which internally calls the `_liquidate` fucntion, In the liquidate function if the user's collateraleralizaionRation is less than the `collateralizationLowerBound`, the function calls the `_forceRepay` to repay the earmarked position of the user. Inside the `_forceRepay` function user's collateral is reduced by the `earmarked amount + fee amount` and the user's earmarked amount is set to 0, but here the `cumulativeEarmaked` which globally tracks the earmarked amount is not updated which leads to inflated `cumulativeEarmaked` amount. The main problem lies in the redeem function because at the time of redemption, the cumulativeEarmarked is used to calculate the new `_survivalAccumulator` which signifies the fraction of earmarked debt that has survived redemption. and as the cumulativeEarmarked is wrong as well as the `_survivalAccumulator`.

```solidity
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
@1>        account.earmarked -= earmarkToRemove;

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

function redeem(uint256 amount) external onlyTransmuter {
        _earmark();

@2>        uint256 liveEarmarked = cumulativeEarmarked;
        if (amount > liveEarmarked) amount = liveEarmarked;

        ---

        uint256 redeemedDebtTotal = amount + coverToApplyDebt;

       // Apply redemption weights/decay to the full amount that left the earmarked bucket
        if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
@3>            uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;
@4>            _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
            _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
        }

        // earmarks are reduced by the full redeemed amount (net + cover)
        cumulativeEarmarked -= redeemedDebtTotal;

        // global borrower debt falls by the full redeemed amount
        totalDebt -= redeemedDebtTotal;

        ---

        emit Redemption(redeemedDebtTotal);
    }

```

* @1 -> here user's earmarked amount is redeuced but the cumulativeEarmark amount is not.
* @2 -> here liveEarmark is cumulativeEarmark
* @3 -> here survival is calculated as `(liveEarmarked - redeemedDebtTotal) / liveEarmarked`
* @4 -> wrong `_survivalAccumulator` is set because of inflated liveEarmarked.

### Impact Details

1. since the cumulativeEarmarked is too high, the survival ratio decreases more than it should. All remaining user share are reduced unfairly.
2. Protocol ends up transferring more mytTokens to the transmuter than it should because of increased `cumulativeEarmarked` causing protocol to transfer unearmarked amount to the transmuter.

```solidity
        // if the cumulativeEarmarked is inflated(100e18) but the actual earmarkedAmount is 50e18 and user redeems amount(80e18), the protocol ends up paying 30e18 tokens from unearmarked amount from the users.
        uint256 liveEarmarked = cumulativeEarmarked;
        if (amount > liveEarmarked) amount = liveEarmarked;
```

## Attack Path

1. System starts with 1e6 tokens and 9e5 earmarked.
2. user liqudiator for 1e5 eamarked, but cumulativeEarmarked stays at 9e5(should be 8e5).
3. On redemption, the system uses 9e5 in the denominator, so everyone's claim shrinks by more than 1/9th instead of 1/8th.
4. After several rounds, the sum of all user claims + protocol liabilities exceeds the vault's real assets.
5. Users cannot withdraw all the valut they are owed, even though no explicit hack or loss happened.

### References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L592>

### Proof of Concept

paste this test in `test/AlchemistV3.t.sol` and run `forge test --mt test_cumulativeEarmarked_notReduced -vvv`

```solidity
function test_cumulativeEarmarked_notReduced() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // ------- User 1 Deposits depositAmount 
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        uint256 anotherUserToken = AlchemistNFTHelper.getFirstTokenId(address(yetAnotherExternalUser), address(alchemistNFT));
        vm.stopPrank();

        // ------- User 2 Deposits depositAmount and borrow
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization(), address(0xbeef));
        address debtToken = alchemist.debtToken();

        Transmuter muter = Transmuter(alchemist.transmuter());
        IERC20(debtToken).approve(address(muter), 1000e18);
        muter.createRedemption(1000e18);
        vm.stopPrank();


        vm.roll(block.number + 100);

        alchemist.poke(anotherUserToken);

        uint256 earmarkedBefore = alchemist.cumulativeEarmarked();
        console.log(earmarkedBefore);

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 900 bps or 9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 750 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        uint256 liquidatorBalanceBefore =  IERC20(address(vault)).balanceOf(address(externalUser));
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);

        uint256 earmarkedAfter = alchemist.cumulativeEarmarked();
        console.log(earmarkedAfter);
        vm.stopPrank();

        // -------- After liquidation the cumulativeEarmark does not change even though the user's earmark amount is reduced.
        assert(earmarkedAfter == earmarkedBefore);
    }
```

```sh
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_cumulativeEarmarked_notReduced() (gas: 3527162)
Logs:
  19025875190258752
  19025875190258752

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 29.25ms (8.66ms CPU time)

Ran 1 test suite in 264.36ms (29.25ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
