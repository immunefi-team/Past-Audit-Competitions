# 57460 sc high protocol fails to subtract fee from total locked when burning and repaying

**Submitted on Oct 26th 2025 at 12:16:31 UTC by @securehash1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57460
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

In AlchemistV3.sol in line 482 total locked not updated when subtracting fee, The issue is that `_subDebt calculates the new _totalLocked value based on the user's remaining debt`, but it is unaware that some collateral was just removed to pay the fee. The \_totalLocked variable represents the amount of collateral that is locked due to debt obligations. Since the fee payment reduces the user's available collateral, \_totalLocked should also be reduced by the fee amount to keep the system's accounting consistent.

## Vulnerability Details

The vulnerability lies within the repay and burn functions of the AlchemistV3.sol contract. Both of these functions, which allow users to reduce their debt, involve charging a protocol fee. This fee is correctly deducted from the user's collateral balance (account.collateralBalance), but not deducted from total-locked which goes to protocol receiver.

```solidity
        uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
        uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        // For cases when someone above minimum LTV gets liquidated.
        if (toFree > _totalLocked) {
            toFree = _totalLocked;
        }

// @audit :missing fee
        account.debt -= amount;
        totalDebt -= amount;
        _totalLocked -= toFree;
```

However, when this collateral is taken for the fee, the corresponding update to the global \_totalLocked state variable is missed. \_totalLocked is meant to represent the total amount of collateral that is encumbered by debt across all positions.

Later in the execution flow of both functions, a call to \_subDebt is made. The \_subDebt function recalculates the locked collateral for the user's position based on their new, lower debt amount. It then adjusts \_totalLocked based on this recalculation. The critical flaw is that this recalculation is unaware that some collateral has already been removed to pay the protocol fee. As a result, the reduction in \_totalLocked is insufficient, causing it to become progressively inflated over time as more fees are collected.

## Impact Details

Incorrect solvency check on total locked which would accumulate over time

## Recommended Mitigation

```soldity
        // Debt is subject to protocol fee similar to redemptions
        _accounts[recipientId].collateralBalance -= convertDebtTokensToYield(credit) * protocolFee / BPS;
@>        _totalLocked -= convertDebtTokensToYield(credit) * protocolFee / BPS;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, convertDebtTokensToYield(credit) * protocolFee / BPS);
        _mytSharesDeposited -= convertDebtTokensToYield(credit) * protocolFee / BPS;

```

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi>

## Proof of Concept

## Proof of Concept

1. I added a get locked amount function Copy and paste to Alchemist.t.sol

```solidity
     function testTotalLockedNotUpdatedOnRepayFee() external {
        vm.prank(alOwner);
        // 10% protocol fee
        alchemist.setProtocolFee(1000);
        vm.stopPrank();

        uint256 depositAmount = 200_000e18;
        uint256 mintAmount = 100_000e18;

        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, externalUser, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
        alchemist.mint(tokenId, mintAmount, externalUser);
        vm.stopPrank();

        uint256 initialTotalLocked = alchemist.getTotalLocked();
        uint256 expectedToLock = alchemist.convertDebtTokensToYield(mintAmount) * alchemist.minimumCollateralization() / FIXED_POINT_SCALAR;
        assertEq(initialTotalLocked, expectedToLock);

        vm.roll(block.number + 1);

        uint256 repayAmount = alchemist.convertDebtTokensToYield(mintAmount / 2);

        deal(address(vault), externalUser, repayAmount);

        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), repayAmount);
        alchemist.repay(repayAmount, tokenId);
        vm.stopPrank();

        uint256 finalTotalLocked = alchemist.getTotalLocked();

        uint256 expectedToFree = alchemist.convertDebtTokensToYield(mintAmount / 2) * alchemist.minimumCollateralization() / FIXED_POINT_SCALAR;
        uint256 feeAmount = repayAmount * alchemist.protocolFee() / BPS;

        uint256 expectedFinalTotalLocked = initialTotalLocked - expectedToFree - feeAmount;

        assertEq(finalTotalLocked, expectedFinalTotalLocked, "Total locked should be reduced by fee amount");
    }
```

2. forge test --match-path src/test/AlchemistV3.t.sol Logs:

```solidity
[PASS] testSetV3PositionNFTAlreadySetRevert() (gas: 25637)
@> [PASS] testTotalLockedNotUpdatedOnRepayFee() (gas: 1285174)
[PASS] testUnauthorizedAlchmistV3PositionNFTMint() (gas: 15749)
```
