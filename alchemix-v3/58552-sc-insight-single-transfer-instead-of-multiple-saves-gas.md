# 58552 sc insight single transfer instead of multiple saves gas

**Submitted on Nov 3rd 2025 at 07:34:28 UTC by @SAAJ for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58552
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**

## Description

## Summary

`createRedemption` function have multiple transfers to same addresses which consumes more gas.

## Vulnerability Details

`createRedemption` function in `Transmuter` contract carries out transfer of yield and fees to `msg.sender` and `protocolFeeReceiver` respectively.

```
        TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield);
        TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield);

        TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
        TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);
```

However the transfer of amount carried out twice instead of a single transfer which leads to more gas consumption.

## Code Reference

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L250C68-L254C84>

## Recommendation

The recommendation is made to transfer the amount on a single transfer call instead of multiple to reduce gas consumption.

```diff
    /// @inheritdoc ITransmuter
    function claimRedemption(uint256 id) external {
        StakingPosition storage position = _positions[id];

        if (position.maturationBlock == 0) {
            revert PositionNotFound();
        }

        if (position.startBlock == block.number) {
            revert PrematureClaim();
        }

        uint256 transmutationTime = position.maturationBlock - position.startBlock;
        uint256 blocksLeft = position.maturationBlock > block.number ? position.maturationBlock - block.number : 0;
        uint256 rounded = position.amount * blocksLeft / transmutationTime + (position.amount * blocksLeft % transmutationTime == 0 ? 0 : 1);
        uint256 amountNottransmuted = blocksLeft > 0 ? rounded : 0;
        uint256 amountTransmuted = position.amount - amountNottransmuted;

        if (_requireOwned(id) != msg.sender) {
            revert CallerNotOwner();
        }

        // Burn position NFT
        _burn(id);

        // Ratio of total synthetics issued by the alchemist / underlingying value of collateral stored in the alchemist
        // If the system experiences bad debt we use this ratio to scale back the value of yield tokens that are transmuted
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0
            ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance)
            : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10 ** TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        uint256 scaledTransmuted = amountTransmuted;

        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }

        // If the contract has a balance of yield tokens from alchemist repayments then we only need to redeem partial or none from Alchemist earmarked
        uint256 debtValue = alchemist.convertYieldTokensToDebt(yieldTokenBalance);
        uint256 amountToRedeem = scaledTransmuted > debtValue ? scaledTransmuted - debtValue : 0;

        if (amountToRedeem > 0) alchemist.redeem(amountToRedeem);

        uint256 totalYield = alchemist.convertDebtTokensToYield(scaledTransmuted);

        // Cap to what we actually hold now (handles redeem() rounding shortfalls).
        uint256 balAfterRedeem = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        uint256 distributable = totalYield <= balAfterRedeem ? totalYield : balAfterRedeem;

        // Split distributable amount. Round fee down; claimant gets the remainder.
        uint256 feeYield = distributable * transmutationFee / BPS;
        uint256 claimYield = distributable - feeYield;

        uint256 syntheticFee = amountNottransmuted * exitFee / BPS;
        uint256 syntheticReturned = amountNottransmuted - syntheticFee;

        // Remove untransmuted amount from the staking graph
        if (blocksLeft > 0) _updateStakingGraph(-position.amount.toInt256() * BLOCK_SCALING_FACTOR / transmutationTime.toInt256(), blocksLeft);

-        TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield);
-        TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield);

-        TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
-        TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);

        
+        TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield + syntheticReturned);
+        TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield + syntheticFee);

        // Burn remaining synths that were not returned
        TokenUtils.safeBurn(syntheticToken, amountTransmuted);
        alchemist.reduceSyntheticsIssued(amountTransmuted);
        alchemist.setTransmuterTokenBalance(TokenUtils.safeBalanceOf(alchemist.myt(), address(this)));

        totalLocked -= position.amount;

        emit PositionClaimed(msg.sender, claimYield, syntheticReturned);

        delete _positions[id];
    }

```

## Proof of Concept

## POC

Here is a Test demonstrating the higher amount of gas consumed when transfer of amounts are carried out multiple times.

```
    // forge t --mc TransmuterTest --mt testClaimRedemptionFromAlchemist -vvv
    function testClaimRedemptionFromAlchemist() public {
        vm.prank(address(0xbeef));
        transmuter.createRedemption(100e18);

        vm.roll(block.number + 5_256_000);

        assertEq(collateralToken.balanceOf(address(0xbeef)), 0);
        assertEq(alETH.balanceOf(address(transmuter)), 100e18);

        uint256 startingBalance = collateralToken.balanceOf(address(alchemist));

        vm.prank(address(0xbeef));
        transmuter.claimRedemption(1);

        assertEq(collateralToken.balanceOf(address(0xbeef)), alchemist.convertUnderlyingTokensToYield(100e18));
        assertEq(alETH.balanceOf(address(transmuter)), 0);

        assertEq(collateralToken.balanceOf(address(alchemist)), startingBalance - alchemist.convertUnderlyingTokensToYield(100e18));
    }

```

The result of above test shows the gas consumption for `testClaimRedemptionFromAlchemist` function have `1406546` when transfer is called multiple times.

```
Ran 1 test for src/test/Transmuter.t.sol:TransmuterTest
[PASS] testClaimRedemptionFromAlchemist() (gas: 1406546)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.84ms (2.55ms CPU time)

Ran 1 test suite in 24.13ms (6.84ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

However, when we called the `testClaimRedemptionFromAlchemist` function when amount transferred is carried out on a single call it reduces the gas consumption to `1388993`.

```
     Ran 1 test for src/test/Transmuter.t.sol:TransmuterTest
[PASS] testClaimRedemptionFromAlchemist() (gas: 1388993)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.14ms (2.26ms CPU time)

Ran 1 test suite in 23.78ms (6.14ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
