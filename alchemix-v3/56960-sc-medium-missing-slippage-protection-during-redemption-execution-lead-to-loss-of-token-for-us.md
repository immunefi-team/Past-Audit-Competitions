# 56960 sc medium missing slippage protection during redemption execution lead to loss of token for user&#x20;

**Submitted on Oct 22nd 2025 at 08:18:13 UTC by @aua\_oo7 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56960
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary:

The `claimRedemption()` function lacks slippage protection when applying the `badDebtRatio`, exposing users to unexpected redemption value losses if the protocol’s collateral ratio worsens before redemption execution.

## Description:

When users call `createRedemption(uint256 syntheticDepositAmount)`, the `syntheticDepositAmount` is amount that user want to change to `myt` token(yield), user redemption is scheduled based on system parameters at that time `block.number + timeToTransmute`. However, the actual redemption value is determined later in `claimRedemption()` function call when user call it. inside the `claimRedemption()` the `amountTransmuted` is the amount that will be transmute and send as `myt` token to user, the function also computes a dynamic `badDebtRatio` using the following formula:

```javascript

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
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;

    @>    uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

    @>    uint256 scaledTransmuted = amountTransmuted;
    @>    if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }

```

if `badDebtRatio` get bigger than 1e18(1:1 ratio of synthetic to Myt token break, mean more synthetic token exist in alchemist then underlying token) then the `amountTransumted` should divide by `badDebtRation` which lead to decrease the actual transmute amount user set and want. now after calculating badDebt scaling a portion from user transmuted amount, this portion is reduced from user `amountTransmuted` and `scaledTransmuted` which is `transmuteAmount - portion` will transfer to user, mean problem is that the `claimRedemption` function burn the `amountTransmuted` which is the hole amount user request for:

```javascript
        ...
        uint256 debtValue = alchemist.convertYieldTokensToDebt(yieldTokenBalance);
        uint256 amountToRedeem = scaledTransmuted > debtValue ? scaledTransmuted - debtValue : 0;
 
        if (amountToRedeem > 0) alchemist.redeem(amountToRedeem);

        uint256 totalYield = alchemist.convertDebtTokensToYield(scaledTransmuted);

        // Cap to what we actually hold now (handles redeem() rounding shortfalls).
        uint256 balAfterRedeem = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        uint256 distributable = totalYield <= balAfterRedeem ? totalYield : balAfterRedeem;

        uint256 feeYield = distributable * transmutationFee / BPS;
        uint256 claimYield = distributable - feeYield;

        uint256 syntheticFee = amountNottransmuted * exitFee / BPS;
        uint256 syntheticReturned = amountNottransmuted - syntheticFee;
 
        if (blocksLeft > 0) _updateStakingGraph(-position.amount.toInt256() * BLOCK_SCALING_FACTOR / transmutationTime.toInt256(), blocksLeft);

        TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield);
        TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield);

        TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
        TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);

        // Burn remaining synths that were not returned
@>      TokenUtils.safeBurn(syntheticToken, amountTransmuted);
        alchemist.reduceSyntheticsIssued(amountTransmuted);
        alchemist.setTransmuterTokenBalance(TokenUtils.safeBalanceOf(alchemist.myt(), address(this)));

        totalLocked -= position.amount;

        emit PositionClaimed(msg.sender, claimYield, syntheticReturned);

        delete _positions[id];
    }
```

at the end we see the `amountTransmuted` is burn for user instead of `scaledTransmuted`. user will loss that scale reduce amount forever and user will face with loss. becouse user will receive `scaledTransmuted` amount as `myt` token instead of `amountTransumted` amount, but burn is vice versa in value. If between creation and redemption the `badDebtRatio` rises (due to system bad debt or synthetic inflation) which is not fault of user who request claim, the user’s redemption value can be drastically reduced without prior notice. There is no mechanism allowing users to define acceptable slippage bounds or to abort under unfavorable conditions.

## Impact:

Users face unexpected synthetic token loss and burn forever and unfair redemption outcomes, as their expected transmutation output can significantly drop due to system-level ratio changes unrelated to their position.

## Mitigation:

Implement a minimum acceptable output (slippage protection) mechanism during redemption execution. Example approach:

```javascript

-- function claimRedemption(uint256 id) external {
++ function claimRedemption(uint256 id, uint256 minTransmutedOut) external {
    
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
        
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        uint256 scaledTransmuted = amountTransmuted;
        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;       
    ++      require(scaledTransmuted >= minTransmutedOut, "Slippage exceeded"); 
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

        TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield);
        TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield);

        TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
        TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);

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

## Proof of Concept

add the following function into Transmuter.t.sol test file and run the test.

```javascript
  function testClaimRedemptionBadDebtLossDueToNoSlippageProtection() public {
        deal(address(collateralToken), address(transmuter), 200e18);
        alchemist.setSyntheticsIssued(1200e18);
        vm.prank(address(0xbeef));
        transmuter.createRedemption(100e18);

        vm.roll(block.number + 5_256_000);

        assertEq(collateralToken.balanceOf(address(0xbeef)), 0);
        assertEq(alETH.balanceOf(address(transmuter)), 100e18);

        alchemist.setUnderlyingValue(200e18);

        vm.prank(address(0xbeef));
        transmuter.claimRedemption(1);

        assertEq(collateralToken.balanceOf(address(0xbeef)), alchemist.convertUnderlyingTokensToYield(100e18) / 2);
        assertEq(alETH.balanceOf(address(transmuter)), 0);

        // console.log("User deposit synthetic token amount to transmute to yield token", 100e18);
        console.log("User MYT token amount after claimRedeemption", collateralToken.balanceOf(address(0xbeef)));
        console.log("75% loss for user!!!");
        console.log("User didn't know about bad debt but he face with half of loss of his synthetic token");
        console.log("Bad Debt scenario can happen after user redemption is create which is not user fault but user face with loss");
        console.log("No minimum loss check/slippage protection exist to prevent from this loss!");
        console.log("Even this function is only callable by redemption request owner but during calling he doesn't know is it badDebt exist or not.");
        console.log("Bad debt scenario loss can be happen during waiting tx exection in mempool also!!");
    }

```
