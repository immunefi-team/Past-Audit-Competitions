# 57476 sc high forcerepay fails to decrement global cumulativeearmarked

**Submitted on Oct 26th 2025 at 14:45:14 UTC by @Paludo0x for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57476
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

During liquidations, `AlchemistV3::_forceRepay()` correctly reduces the account earmarked value `account.earmarked` but does not reduce the global earmarked `cumulativeEarmarked`.

## Vulnerability Details

The buggy snippet is the following:

```
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
      ....
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

While in `repay()` call global earmarked is reduced:

```
function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
....

      uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
        cumulativeEarmarked -= earmarkPaidGlobal;

....
```

This breaks the core invariant that the global earmark tracks the aggregate of user earmarks.

Because `cumulativeEarmarked` is used both as the denominator for redemption weighting and to compute `liveUnearmarked = totalDebt - cumulativeEarmarked` for future earmarks, the error propagates into worngly priced redemptions and persistent accounting drift.

These are the relevant snippet where `cumulativeEarmarked` is used:

```
    function _earmark() internal {

...
        uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
        if (amount > liveUnearmarked) amount = liveUnearmarked;

        if (amount > 0 && liveUnearmarked != 0) {
            // Previous earmark survival
            uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
            if (previousSurvival == 0) previousSurvival = ONE_Q128;

            // Fraction of unearmarked debt being earmarked now in UQ128.128
            uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

            _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
            _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

            cumulativeEarmarked += amount;
        }
...
 }
```

```
  function _calculateUnrealizedDebt(uint256 tokenId)
        internal
        view
        returns (uint256, uint256, uint256)
    {

....
       if (block.number > lastEarmarkBlock) {
....
            uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
            if (amount > liveUnearmarked) amount = liveUnearmarked;

            if (amount > 0 && liveUnearmarked != 0) {
                // Previous earmark survival
                uint256 previousSurvival = PositionDecay.SurvivalFromWeight(earmarkWeightCopy);
                if (previousSurvival == 0) previousSurvival = ONE_Q128;

                // Fraction of unearmarked debt being earmarked now in UQ128.128
                uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

                survivalAccumulatorCopy += _mulQ128(previousSurvival, earmarkedFraction);
                earmarkWeightCopy += PositionDecay.WeightIncrement(amount, liveUnearmarked);
            }
        }
...
    }
```

## Impact Details

These are the main impacts:

* Users debts are reduced less than they should during redemptions
* Protocol global accounting corruption: divergence between burned synthetics and aggregate debt reduction
* Protocol risk over time: repeated liquidations via `_forceRepay()` can degrade solvency

Given the above impacts the severity is CRITICAL.

## Proof of Concept

## Proof of Concept

The PoC forces calling of \_forceRepay() during a liquidation and demonstrates that global cumulativeEarmarked doesn't decrease after a liquidation.

This is the log:

```
[PASS] test_Bug_ForceRepay_DoesNotDecrementCumulativeEarmarked() (gas: 1729373)
Logs:
  === before _forceRepay ===
  account earmarked 600000000000000000000
  cumulativeEarmarked 600000000000000000000
  === after _forceRepay ===
  account earmarked 0
  cumulativeEarmarked 600000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 55.10ms (34.43ms CPU time)
```

```
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {Test} from "forge-std/Test.sol";
import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";
import {console} from "forge-std/console.sol";
import "./AlchemistV3.t.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {AlchemistV3} from "../AlchemistV3.sol";


contract Bug_ForceRepay_CumulativeEarmark_PoC is AlchemistV3Test {
    using stdStorage for StdStorage;

    uint256 private constant ACCOUNTS_SLOT = 31;

    function setAccountCollateral(address alchemistAddr, uint256 accountId, uint256 amount) internal {
        bytes32 accountSlot = keccak256(abi.encode(accountId, ACCOUNTS_SLOT));
        vm.store(alchemistAddr, accountSlot, bytes32(amount));
    }

    function setAccountDebt(address alchemistAddr, uint256 accountId, uint256 amount) internal {
        bytes32 accountSlot = keccak256(abi.encode(accountId, ACCOUNTS_SLOT));
        bytes32 debtSlot = bytes32(uint256(accountSlot) + 1);
        vm.store(alchemistAddr, debtSlot, bytes32(amount));
    }

    function setAccountEarmarked(address alchemistAddr, uint256 accountId, uint256 amount) internal {
        bytes32 accountSlot = keccak256(abi.encode(accountId, ACCOUNTS_SLOT));
        bytes32 earmarkedSlot = bytes32(uint256(accountSlot) + 2);
        vm.store(alchemistAddr, earmarkedSlot, bytes32(amount));
    }

    function test_Bug_ForceRepay_DoesNotDecrementCumulativeEarmarked() public {
        address user = address(0xbeef);
        address liquidator = address(0xcafe);
        uint256 collateralAmount = 10_000e18;
        uint256 debtAmount = 600e18;
        uint256 totalDebtGlobal = debtAmount * 2;
        uint256 earmarkedAmount = debtAmount;

        deal(address(vault), user, collateralAmount);

        vm.startPrank(user);
        IERC20(address(vault)).approve(address(alchemist), collateralAmount);
        alchemist.deposit(collateralAmount, user, 0);
        vm.stopPrank();

        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));

        // Force the account into a state with earmarked debt and corresponding global totals.
        setAccountCollateral(address(alchemist), tokenId, collateralAmount);
        setAccountDebt(address(alchemist), tokenId, debtAmount);
        setAccountEarmarked(address(alchemist), tokenId, earmarkedAmount);

        stdstore.target(address(alchemist)).sig("totalDebt()").checked_write(totalDebtGlobal);
        stdstore.target(address(alchemist)).sig("cumulativeEarmarked()").checked_write(earmarkedAmount);
        stdstore.target(address(alchemist)).sig("collateralizationLowerBound()").checked_write(type(uint256).max);
        stdstore.target(address(alchemist)).sig("lastEarmarkBlock()").checked_write(block.number);

        uint256 cumulativeBefore = alchemist.cumulativeEarmarked();
        (,, uint256 earmarkedBefore) = alchemist.getCDP(tokenId);
        assertEq(earmarkedBefore, earmarkedAmount, "setup failed");
        console.log("=== before _forceRepay ===");
        console.log("account earmarked", earmarkedBefore);
        console.log("cumulativeEarmarked", cumulativeBefore);

        // Force the branch in _liquidate that executes `_forceRepay()`; since earmarked == debt,
        // the subsequent `account.debt == 0` early return prevents `_doLiquidation()` from running.
        vm.prank(liquidator);
        (uint256 yieldAmount,,) = alchemist.liquidate(tokenId);
        assertGt(yieldAmount, 0, "liquidation path must execute _forceRepay()");

        (,, uint256 earmarkedAfter) = alchemist.getCDP(tokenId);
        uint256 cumulativeAfter = alchemist.cumulativeEarmarked();
        console.log("=== after _forceRepay ===");
        console.log("account earmarked", earmarkedAfter);
        console.log("cumulativeEarmarked", cumulativeAfter);

        assertEq(earmarkedAfter, 0, "_forceRepay() burned the user earmark");
        assertEq(cumulativeAfter, cumulativeBefore, "BUG: global cumulativeEarmarked stays inflated");
    }
}

```
