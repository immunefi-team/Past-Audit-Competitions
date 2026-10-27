# Boost \_ Folks Finance 33852 - \[Smart Contract - Insight] Small positions will not get liquidated

Submitted on Wed Jul 31 2024 07:43:42 GMT-0400 (Atlantic Standard Time) by @OxAnmol for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33852

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140>

Impacts:

* Permanent freezing of unclaimed yield
* Theft of unclaimed yield

## Description

## Brief/Intro

There is no minimum amount of borrow amount set for the user, which means a user can open a dust position and will never get liquidated because of the larger gas cost.

## Vulnerability Details

In general, lending protocols like Aave and Compound impose a minimum borrowable amount to address the issue of small dust positions. However, in this protocol, users can open small positions without any restrictions.

This can lead to a situation where liquidators may not liquidate the debt because the gas cost is higher than the liquidation bonus.

Here are some resources regarding this common issue

[Rareskills:DangerOfSmallLoans](https://www.rareskills.io/post/defi-liquidations-collateral)

[Example finding](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/issues/1096)

## Impact Details

As we can see in the console output for liquidating a $6 borrow position, the liquidator receives $0.30, which is below the average gas cost of the Avalanche blockchain. Therefore, the liquidator will never liquidate this type of position, which will eventually lead to bad debt and lender loss. With this, I think this issue should qualify as high severity.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/LoanManager.sol#L163>

## Proof of concept

Paste this test in `hub/test` by creating a new folder.

```js
import { expect } from "chai";
import { ethers } from "hardhat";
import { PANIC_CODES } from "@nomicfoundation/hardhat-chai-matchers/panic";
import { loadFixture, reset, time } from "@nomicfoundation/hardhat-toolbox/network-helpers";
import {
  LiquidationLogic__factory,
  LoanManagerLogic__factory,
  LoanManager__factory,
  LoanPoolLogic__factory,
  MockHubPool__factory,
  MockOracleManager__factory,
  RewardLogic__factory,
  UserLoanLogic__factory,
} from "../../typechain-types";
import { BYTES32_LENGTH, convertStringToBytes, getAccountIdBytes, getEmptyBytes, getRandomBytes } from "../utils/bytes";
import { SECONDS_IN_DAY, SECONDS_IN_HOUR, getLatestBlockTimestamp, getRandomInt } from "../utils/time";
import { UserLoanBorrow, UserLoanCollateral } from "./libraries/assets/loanData";
import { getNodeOutputData } from "./libraries/assets/oracleData";
import {
  calcAverageStableRate,
  calcBorrowBalance,
  calcBorrowInterestIndex,
  calcReserveCol,
  calcStableInterestRate,
  convToCollateralFAmount,
  convToRepayBorrowAmount,
  convToSeizedCollateralAmount,
  toFAmount,
  toUnderlingAmount,
} from "./utils/formulae";

describe("Liquidation", () => {
  const DEFAULT_ADMIN_ROLE = getEmptyBytes(BYTES32_LENGTH);
  const LISTING_ROLE = ethers.keccak256(convertStringToBytes("LISTING"));
  const ORACLE_ROLE = ethers.keccak256(convertStringToBytes("ORACLE"));
  const HUB_ROLE = ethers.keccak256(convertStringToBytes("HUB"));

  async function deployLoanManagerFixture() {
    const [admin, hub, user, ...unusedUsers] = await ethers.getSigners();

    // libraries
    const userLoanLogic = await new UserLoanLogic__factory(user).deploy();
    const userLoanLogicAddress = await userLoanLogic.getAddress();
    const loanPoolLogic = await new LoanPoolLogic__factory(user).deploy();
    const loanPoolLogicAddress = await loanPoolLogic.getAddress();
    const liquidationLogic = await new LiquidationLogic__factory(
      {
        ["contracts/hub/logic/UserLoanLogic.sol:UserLoanLogic"]: userLoanLogicAddress,
      },
      user
    ).deploy();
    const liquidationLogicAddress = await liquidationLogic.getAddress();
    const loanManagerLogic = await new LoanManagerLogic__factory(
      {
        ["contracts/hub/logic/UserLoanLogic.sol:UserLoanLogic"]: userLoanLogicAddress,
        ["contracts/hub/logic/LoanPoolLogic.sol:LoanPoolLogic"]: loanPoolLogicAddress,
        ["contracts/hub/logic/LiquidationLogic.sol:LiquidationLogic"]: liquidationLogicAddress,
      },
      user
    ).deploy();
    const loanManagerLogicAddress = await loanManagerLogic.getAddress();
    const rewardLogic = await new RewardLogic__factory(user).deploy();
    const rewardLogicAddress = await rewardLogic.getAddress();

    const libraries = {
      userLoanLogic,
      loanPoolLogic,
      liquidationLogic,
      loanManagerLogic,
      rewardLogic,
    };

    // deploy contract
    const oracleManager = await new MockOracleManager__factory(user).deploy();
    const loanManager = await new LoanManager__factory(
      {
        ["contracts/hub/logic/LoanManagerLogic.sol:LoanManagerLogic"]: loanManagerLogicAddress,
        ["contracts/hub/logic/RewardLogic.sol:RewardLogic"]: rewardLogicAddress,
      },
      user
    ).deploy(admin, oracleManager);

    // set hub role
    await loanManager.connect(admin).grantRole(HUB_ROLE, hub);

    // common
    const loanManagerAddress = await loanManager.getAddress();

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
    };
  }

  async function createLoanTypeFixture() {
    const { admin, hub, user, unusedUsers, loanManager, loanManagerAddress, oracleManager, libraries } =
      await loadFixture(deployLoanManagerFixture);

    // create loan type
    const loanTypeId = 1;
    const loanTargetHealth = BigInt(1.05e4); //
    await loanManager.connect(admin).createLoanType(loanTypeId, loanTargetHealth);

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      loanTypeId,
      loanTargetHealth,
    };
  }

  async function addPoolsFixture() {
    const { admin, hub, user, unusedUsers, loanManager, loanManagerAddress, oracleManager, libraries, loanTypeId } =
      await loadFixture(createLoanTypeFixture);

    // prepare pools
    const usdcPoolId = 1;
    const usdcPool = await new MockHubPool__factory(user).deploy("Folks USD Coin", "fUSDC", usdcPoolId);
    const ethPoolId = 2;
    const ethPool = await new MockHubPool__factory(user).deploy("Folks Ethereum", "fETH", ethPoolId);

    // add pools
    await loanManager.connect(admin).addPool(usdcPool);
    await loanManager.connect(admin).addPool(ethPool);

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      loanTypeId,
      usdcPoolId,
      usdcPool,
      ethPoolId,
      ethPool,
    };
  }
  async function addPoolToLoanTypeFixture() {
    const {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      loanTypeId,
      usdcPoolId,
      usdcPool,
      ethPoolId,
      ethPool,
    } = await loadFixture(addPoolsFixture);

    // add pools to loan type
    const rewardCollateralSpeed = BigInt(0);
    const rewardBorrowSpeed = BigInt(0);
    const rewardMinimumAmount = BigInt(1e18);
    const collateralCap = BigInt(20e6);
    const borrowCap = BigInt(10e6);

    const usdcCollateralFactor = BigInt(0.8e4);
    const usdcBorrowFactor = BigInt(1e4);
    const usdcLiquidationBonus = BigInt(0.04e4);
    const usdcLiquidationFee = BigInt(0.1e4);

    const ethCollateralFactor = BigInt(0.7e4);
    const ethBorrowFactor = BigInt(1e4);
    const ethLiquidationBonus = BigInt(0.06e4);
    const ethLiquidationFee = BigInt(0.1e4);

    const pools = {
      USDC: {
        poolId: usdcPoolId,
        pool: usdcPool,
        rewardCollateralSpeed,
        rewardBorrowSpeed,
        rewardMinimumAmount,
        collateralCap,
        borrowCap,
        collateralFactor: usdcCollateralFactor,
        borrowFactor: usdcBorrowFactor,
        liquidationBonus: usdcLiquidationBonus,
        liquidationFee: usdcLiquidationFee,
        tokenDecimals: BigInt(6),
      },
      ETH: {
        poolId: ethPoolId,
        pool: ethPool,
        rewardCollateralSpeed,
        rewardBorrowSpeed,
        rewardMinimumAmount,
        collateralCap,
        borrowCap,
        collateralFactor: ethCollateralFactor,
        borrowFactor: ethBorrowFactor,
        liquidationBonus: ethLiquidationBonus,
        liquidationFee: ethLiquidationFee,
        tokenDecimals: BigInt(18),
      },
    };

    await loanManager
      .connect(admin)
      .addPoolToLoanType(
        loanTypeId,
        usdcPoolId,
        usdcCollateralFactor,
        collateralCap,
        usdcBorrowFactor,
        borrowCap,
        usdcLiquidationBonus,
        usdcLiquidationFee,
        rewardCollateralSpeed,
        rewardBorrowSpeed,
        rewardMinimumAmount
      );
    await loanManager
      .connect(admin)
      .addPoolToLoanType(
        loanTypeId,
        ethPoolId,
        ethCollateralFactor,
        collateralCap,
        ethBorrowFactor,
        borrowCap,
        ethLiquidationBonus,
        ethLiquidationFee,
        rewardCollateralSpeed,
        rewardBorrowSpeed,
        rewardMinimumAmount
      );

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      loanTypeId,
      pools,
    };
  }
  /* ---------------------------- CREATE USER LOAN ---------------------------- */
  async function createUserLoanFixture() {
    const {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      loanTypeId,
      pools,
    } = await loadFixture(addPoolToLoanTypeFixture);

    // create user loan
    const loanId = getRandomBytes(BYTES32_LENGTH);
    const accountId = getAccountIdBytes("ACCOUNT_ID");
    const loanName = getRandomBytes(BYTES32_LENGTH);
    const createUserLoan = await loanManager.connect(hub).createUserLoan(loanId, accountId, loanTypeId, loanName);

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      createUserLoan,
      loanTypeId,
      pools,
      loanId,
      accountId,
      loanName,
    };
  }

  async function depositEtherFixture() {
    const {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      loanTypeId,
      pools,
      loanId,
      accountId,
    } = await loadFixture(createUserLoanFixture);

    // prepare deposit
    const depositAmount = BigInt(3333333000000000); // 10$ worth ether
    const depositFAmount = depositAmount;
    const depositInterestIndex = BigInt(1e18);
    const ethPrice = BigInt(3000e18);
    await pools.ETH.pool.setDepositPoolParams({
      fAmount: depositFAmount,
      depositInterestIndex,
      priceFeed: { price: ethPrice, decimals: pools.ETH.tokenDecimals },
    });

    // deposit into eth pool
    const deposit = await loanManager.connect(hub).deposit(loanId, accountId, pools.ETH.poolId, depositAmount);

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      deposit,
      loanTypeId,
      pools,
      loanId,
      accountId,
      depositAmount,
      depositFAmount,
    };
  }

  after(async () => {
    await reset();
  });

  async function depositEtherAndVariableBorrowUSDCFixture() {
    const {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      loanTypeId,
      pools,
      loanId,
      accountId,
      depositAmount,
      depositFAmount,
    } = await loadFixture(depositEtherFixture);

    // set prices
    const usdcNodeOutputData = getNodeOutputData(BigInt(1e18));
    await oracleManager.setNodeOutput(pools.USDC.poolId, pools.USDC.tokenDecimals, usdcNodeOutputData);

    const ethNodeOutputData = getNodeOutputData(BigInt(3000e18));
    await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

    // prepare borrow
    const variableInterestIndex = BigInt(1.05e18);
    const stableInterestRate = BigInt(0.1e18);
    await pools.USDC.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
    await pools.USDC.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);

    // borrow from USDC pool
    const borrowAmount = BigInt(6e6); // 7 USDC
    const borrow = await loanManager.connect(hub).borrow(loanId, accountId, pools.USDC.poolId, borrowAmount, BigInt(0));

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      borrow,
      loanTypeId,
      pools,
      loanId,
      accountId,
      depositAmount,
      depositFAmount,
      borrowAmount,
      usdcVariableInterestIndex: variableInterestIndex,
      usdcStableInterestRate: stableInterestRate,
    };
  }

  describe("Correctly Liquidate", () => {
    it.only("should correctly liquidate", async () => {
      const {
        admin,
        hub,
        user,
        unusedUsers,
        loanManager,
        loanManagerAddress,
        oracleManager,
        libraries,
        loanTypeId,
        pools,
        loanId,
        accountId,
        depositAmount,
        depositFAmount,
        borrowAmount,
        usdcVariableInterestIndex,
        usdcStableInterestRate,
      } = await loadFixture(depositEtherAndVariableBorrowUSDCFixture);

      // With 2000$ price the borrow is 1400$, so health is 170% which is at boarder line

      // ETH price drops to 1900$

      const ethNodeOutputData = getNodeOutputData(BigInt(1900e18));
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // Liquidate
      //   const liquidateAmount = BigInt(1000e6); // 1000 USDC

      // create liquidator loan
      const liquidatorLoanId = getRandomBytes(BYTES32_LENGTH);
      const liquidatorAccountId = getAccountIdBytes("LIQUIDATOR_ACCOUNT_ID");
      const liquidatorLoanName = getRandomBytes(BYTES32_LENGTH);
      await loanManager
        .connect(hub)
        .createUserLoan(liquidatorLoanId, liquidatorAccountId, loanTypeId, liquidatorLoanName);

      // deposit
      const liquidatorDepositAmount = BigInt(1e18); // 1 ETH
      const liquidatorDepositFAmount = liquidatorDepositAmount;
      const liquidatorDepositInterestIndex = BigInt(1e18);
      await pools.ETH.pool.setDepositPoolParams({
        fAmount: liquidatorDepositFAmount,
        depositInterestIndex: liquidatorDepositInterestIndex,
        priceFeed: { price: ethNodeOutputData.price, decimals: pools.ETH.tokenDecimals },
      });
      await loanManager
        .connect(hub)
        .deposit(liquidatorLoanId, liquidatorAccountId, pools.ETH.poolId, liquidatorDepositAmount);

      // calculate interest
      const violatorLoanBefore = await loanManager.getUserLoan(loanId);
      const violatorOldBorrow = violatorLoanBefore[5][0];
      const timestamp = (await getLatestBlockTimestamp()) + getRandomInt(SECONDS_IN_HOUR);
      await time.setNextBlockTimestamp(timestamp);
      const newInterestIndex = calcBorrowInterestIndex(
        violatorOldBorrow.stableInterestRate,
        violatorOldBorrow.lastInterestIndex,
        BigInt(timestamp) - violatorOldBorrow.lastStableUpdateTimestamp,
        true
      );
      const borrowBalance = calcBorrowBalance(
        violatorOldBorrow.balance,
        newInterestIndex,
        violatorOldBorrow.lastInterestIndex
      );

      //   console.log("borrowBalance", borrowBalance.toString()); // 1400e6 USDC

      const repayAmount = BigInt(6e6); // 100 USDC
      const usdcPrice = BigInt(1e18);
      const collateralFAmount = convToCollateralFAmount(
        repayAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        BigInt(1e18)
      ); // value of 100 USDC in ETH

      //   console.log("collateralFAmount", ethers.formatEther(collateralFAmount), " ETH"); // 0.05 ETH

      const seizeCollateralAmount = convToSeizedCollateralAmount(
        repayAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        pools.USDC.liquidationBonus
      );
      // This include liquidation bonus
      //   console.log("seizeCollateralAmount", ethers.formatEther(seizeCollateralAmount), " ETH"); // 0.054736842105263157
      // 4% of 0.052631578947368421 is 0.002105263157894737 which is bonus
      const liquidationBonusCollateral = seizeCollateralAmount - collateralFAmount;
      //   console.log("Liquidation Bonus collatera", ethers.formatEther(liquidationBonusCollateral), " ETH");

      const seizeCollateralFAmount = toFAmount(seizeCollateralAmount, BigInt(1e18));
      const reserveCollateralFAmount = calcReserveCol(
        seizeCollateralFAmount,
        collateralFAmount,
        pools.ETH.liquidationFee
      );
      // 10 % of 0.002105263157894737 is 0.000210526315789474 = protocol fee
      // at last liquidatator will get 0.002105263157894737 - 0.000210526315789474 = 0.001894736842105263
      //   console.log("Protocol fee collateral", ethers.formatEther(reserveCollateralFAmount), " ETH"); // 0.002736842105263158

      console.log(
        "Liquidator profit collateral",
        ethers.formatEther(liquidationBonusCollateral - reserveCollateralFAmount),
        " ETH"
      ); // 0.052631578947368421

      const liquidatorCollateralFAmount = seizeCollateralFAmount - reserveCollateralFAmount;
      // How much collateral liquidator get. it will be worth 103.60 USDC as per my calculation, means 3.60 USDC profit - gas fee
      //   console.log("liquidatorCollateralFAmount", ethers.formatEther(liquidatorCollateralFAmount), " ETH"); // 0.052631578947368421
      // liquidate
      const minSeizedAmount = BigInt(0);
      const liquidate = await loanManager
        .connect(hub)
        .liquidate(
          loanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          repayAmount,
          minSeizedAmount
        );
      //TODO:repay, See if this leaves any bad debt to protocol

      // Check violator loan and see how much is remaining
      const violatorLoan = await loanManager.getUserLoan(loanId);
      //   console.log("Violator collateral", violatorLoan[4][0].balance);
      //   console.log("Violator Loan", violatorLoan[5]);

      const liquidatorDebt = await loanManager.getUserLoan(liquidatorLoanId);
      //   console.log("Liquidator collateral", liquidatorDebt[4][0].balance);
      //   console.log("Liquidator Loan", liquidatorDebt[5][0]);

      // Check liquidator loan and see how much is remaining
    });
  });
});
```

## console output

Here `0.000113684210526316` is a net profit liquidator is getting for 6$(not adjusted to target heath) which in terms of USD is like 0.3$.

```shell
Liquidation
    Correctly Liquidate
Liquidator profit collateral 0.000113684210526316  ETH
      ✔ should correctly liquidate (1385ms)


  1 passing (1s)
```
