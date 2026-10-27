# Boost \_ Folks Finance 34127 - \[Smart Contract - Low] Liquidator gets more debt than usual

Submitted on Mon Aug 05 2024 19:22:47 GMT-0400 (Atlantic Standard Time) by @Nyksx for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34127

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xc1FBF54B25816B60ADF322d8A1eaCA37D9A50317>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

Liquidator gets more borrow balance because of the liquidaton bonus.

## Vulnerability Details

In the liquidation process, the calcLiquidationAmounts function calculates the borrow amount to repay and the collateral amount to seize based on the violator's liquidity.

When the seizeUnderlyingCollateralAmount is greater than the violatorUnderlingCollateralBalance, the function calculates the repayBorrowAmount again with the updated seizeUnderlyingCollateralAmount.

```solidity
if (seizeUnderlyingCollateralAmount > violatorUnderlingCollateralBalance) {
                seizeUnderlyingCollateralAmount = violatorUnderlingCollateralBalance;
                repayBorrowAmount = seizeUnderlyingCollateralAmount.convToRepayBorrowAmount(
                    collPriceFeed.price,
                    collPriceFeed.decimals,
                    borrPriceFeed.price,
                    borrPriceFeed.decimals,
                    borrowLoanPool.liquidationBonus
                ); 
            }
```

The problem is that when recalculating the repayBorrowAmount, the function uses a liquidation bonus. This can cause liquidators to take on more debt than usual, and because of that liquidators may not liquidate loans.

For example, the violator has 1 ETH collateral and 1000 USDC borrow. (Let's assume 1 ETH 500$)

In this example, if the liquidator liquidates the violator, the liquidator gets 1 ETH, which is equal to $500$, and also gets 520 USDC debt, which means the liquidator is at a loss. (liquidatorBonus: 0.04e4 500e6 \* (1e4 + 0.04e4) / 1e4 = 520e6 )

But if the liquidation bonus is not used when the seizeUnderlyingCollateralAmount is greater than the violatorUnderlingCollateralBalance, Liquidators do not lose anything.

## Impact Details

Liquidators won't liquidate loans in these situations, which can lead to more bad debt for the protocol.

## References

## Proof of concept

## Proof of Concept

LoanManager.test.ts

```
it("Test seizeUnderlyingCollateralAmount > violatorUnderlingCollateralBalance ", async () => {
      const {
        hub,
        loanManager,
        loanManagerAddress,
        oracleManager,
        pools,
        loanId: violatorLoanId,
        accountId: violatorAccountId,
        loanTypeId,
        depositAmount,
        depositFAmount,
        borrowAmount,
        usdcVariableInterestIndex: oldVariableInterestIndex,
      } = await loadFixture(depositEtherAndVariableBorrowUSDCFixture);

      // create liquidator loan
      const liquidatorLoanId = getRandomBytes(BYTES32_LENGTH);
      const liquidatorAccountId = getAccountIdBytes("LIQUIDATOR_ACCOUNT_ID");
      const liquidatorLoanName = getRandomBytes(BYTES32_LENGTH);
      await loanManager
        .connect(hub)
        .createUserLoan(liquidatorLoanId, liquidatorAccountId, loanTypeId, liquidatorLoanName);

      // deposit USDC into liquidator loan
      const liquidatorDepositAmount = BigInt(10000e6); // 10,000 USDC
      const liquidatorDepositFAmount = liquidatorDepositAmount;
      const liquidatorDepositInterestIndex = BigInt(1e18);
      const usdcPrice = BigInt(1e18);
      await pools.USDC.pool.setDepositPoolParams({
        fAmount: liquidatorDepositFAmount,
        depositInterestIndex: liquidatorDepositInterestIndex,
        priceFeed: { price: usdcPrice, decimals: pools.USDC.tokenDecimals },
      });
      await loanManager
        .connect(hub)
        .deposit(liquidatorLoanId, liquidatorAccountId, pools.USDC.poolId, liquidatorDepositAmount);

      // prepare liquidation
      const ethNodeOutputData = getNodeOutputData(BigInt(500e18));
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // calculate interest
      const variableInterestIndex = BigInt(1.1e18);
      const stableInterestRate = BigInt(0.1e18);
      await pools.USDC.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
      await pools.USDC.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);
      const borrowBalance = calcBorrowBalance(borrowAmount, variableInterestIndex, oldVariableInterestIndex);

      // Violator:
      // Collateral 1 ETH = $500
      // Borrow 1,000 USDC = $1,000

      // Liquidator:
      // Collateral 10,000 USDC = $10,000
      // Borrow $0
      console.log("violator before liquidation", await loanManager.getUserLoan(violatorLoanId));
      console.log("liquidator before liquidation", await loanManager.getUserLoan(liquidatorLoanId));

      const seizeCollateralAmount = depositAmount;
      const seizeCollateralFAmount = depositFAmount;
      const repayAmount = convToRepayBorrowAmount(
        seizeCollateralAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        pools.USDC.liquidationBonus
      );

      console.log("Liquidators gets 520 USDC borrow ", repayAmount);
      const collateralFAmount = convToCollateralFAmount(
        repayAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        BigInt(1e18)
      );
      //console.log("repayToCollateralFAmount", collateralFAmount);
      const reserveCollateralFAmount = calcReserveCol(
        seizeCollateralFAmount,
        collateralFAmount,
        pools.ETH.liquidationFee
      );
      const liquidatorCollateralFAmount = seizeCollateralFAmount - reserveCollateralFAmount;

      console.log("But gets 1 ETH which equals 500$", liquidatorCollateralFAmount);

      // liquidate
      const attemptedRepayAmount = repayAmount + BigInt(10e6);
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

      console.log("vialator after liquidation", await loanManager.getUserLoan(violatorLoanId));

      // check violator loan
      const violatorLoan = await loanManager.getUserLoan(violatorLoanId);
      const violatorBorrows: UserLoanBorrow[] = [
        {
          amount: borrowAmount - repayAmount,
          balance: borrowBalance - repayAmount,
          lastInterestIndex: variableInterestIndex,
          stableInterestRate: BigInt(0),
          lastStableUpdateTimestamp: BigInt(0),
          rewardIndex: BigInt(0),
        },
      ];
      expect(await loanManager.isUserLoanActive(violatorLoanId)).to.be.true;
      expect(violatorLoan[0]).to.equal(violatorAccountId);
      expect(violatorLoan[1]).to.equal(loanTypeId);
      expect(violatorLoan[2]).to.deep.equal([]);
      expect(violatorLoan[3]).to.deep.equal([pools.USDC.poolId]);
      expect(violatorLoan[4]).to.deep.equal([]);
      expect(violatorLoan[5]).to.deep.equal(violatorBorrows.map((bor) => Object.values(bor)));

      // check liquidator loan
      const liquidatorLoan = await loanManager.getUserLoan(liquidatorLoanId);
      console.log("liquidator after liquidation", liquidatorLoan);

      const liquidatorCollaterals: UserLoanCollateral[] = [
        {
          balance: liquidatorDepositFAmount,
          rewardIndex: BigInt(0),
        },
        {
          balance: liquidatorCollateralFAmount,
          rewardIndex: BigInt(0),
        },
      ];
      const liquidatorBorrows: UserLoanBorrow[] = [
        {
          amount: repayAmount,
          balance: repayAmount,
          lastInterestIndex: variableInterestIndex,
          stableInterestRate: BigInt(0),
          lastStableUpdateTimestamp: BigInt(0),
          rewardIndex: BigInt(0),
        },
      ];
      expect(await loanManager.isUserLoanActive(liquidatorLoanId)).to.be.true;
      expect(liquidatorLoan[0]).to.equal(liquidatorAccountId);
      expect(liquidatorLoan[1]).to.equal(loanTypeId);
      expect(liquidatorLoan[2]).to.deep.equal([pools.USDC.poolId, pools.ETH.poolId]);
      expect(liquidatorLoan[3]).to.deep.equal([pools.USDC.poolId]);
      expect(liquidatorLoan[4]).to.deep.equal(liquidatorCollaterals.map((col) => Object.values(col)));
      expect(liquidatorLoan[5]).to.deep.equal(liquidatorBorrows.map((bor) => Object.values(bor)));
    });
```
