# Boost \_ Folks Finance 34124 - \[Smart Contract - Low] Smart contract cannot be accessed during the no

Submitted on Mon Aug 05 2024 17:37:39 GMT-0400 (Atlantic Standard Time) by @ICP for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34124

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xf8E94c5Da5f5F23b39399F6679b2eAb29FE3071e>

Impacts:

* Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

Liquidator will ended with underflow exception while normal liquidation process acquiring the borrow balance of the violator

## Vulnerability Details

Liquidation is normal process in market when loan went to underCollateralized normally liquidator acquire the assets by calling `executeLiquidate()` function in LoanManagerLogic from hub. But inside this this function which calls the `updateLiquidationBorrows()` to transfer the funds from the Violator to liquidator here it calls the `transferBorrowFromViolator()` function in order to repay and decrease the balance of the violator . Below code snippet we can see :-

```solidity
    /// @dev Calc the borrow balance and amount to repay and decrease them from violator borrow
    /// @param loan The user loan to transfer the borrow from
    /// @param poolId The pool ID of the borrow
    /// @param repayBorrowAmount The amount to repay
    /// @return repaidBorrowAmount The borrow amount repaid
    /// @return repaidBorrowBalance The borrow balance repaid
    function transferBorrowFromViolator(
        LoanManagerState.UserLoan storage loan,
        uint8 poolId,
        uint256 repayBorrowAmount
    ) external returns (uint256 repaidBorrowAmount, uint256 repaidBorrowBalance, uint256 loanStableRate) {
        LoanManagerState.UserLoanBorrow storage loanBorrow = loan.borrows[poolId];

        // violator loanBorrow has beed updated in prepareLiquidation

        repaidBorrowBalance = repayBorrowAmount;
        repaidBorrowAmount = Math.min(repaidBorrowBalance, loanBorrow.amount);// @audit check here
        loanStableRate = loanBorrow.stableInterestRate;

        loanBorrow.amount -= repaidBorrowAmount;
        loanBorrow.balance -= repaidBorrowBalance; // @audit check here

        if (loanBorrow.balance == 0) clearBorrow(loan, poolId);
    }
```

In above we see that balance will be decreased by the liquidator parameter `repayAmount`. \[<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/Hub.sol#L118>] Which will directly subtract with violator borrow balance after that if the borrow balance is zero then it will clear the borrow here is main issue without validation the user input it will directly subtracts with loanBorrow.

For Scenario We look below numbers :-

```

 Violator Borrow Balance Before Liquidation 1000000000n

Violator Collateral Balance Before Liquidation 1000000000000000000n

Liquidator Borrow Balance Before Liquidation 0n

Liquidator Collateral Balance Before Liquidation 1000000000n

Repay Amount 965000000n
```

In above we can see the balances of violator and liquidator before liquidation which is fetch from the user defined functions.

Violator have borrow 1000 USDC and underCollateralized Liquidator tries to acquire by paying 965 USDC but it will halt the process by panic code 0x11 underFlow or overflow Vm exception. Liquidator by seeing the violator borrow balance and try to acquire by lesser amount than the borrow balance it will landed in halted in hub chain.

## Impact Details

1 . Temporary blocks the liquidation process and liquidator funds.

2 . Liquidator doesn't allow to acquire the default loans ended with exception which will cause halt of normal process in VM because it not handle with try catch.

3 . Smart contracts which is associated with Liquidation Logic will not be able to access due to the error while acquiring the borrow balance.

The above impact assessed with protocol, if any query please ping me.

## code Snippet

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LoanManagerLogic.sol#L477C1-L483C10> <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/UserLoanLogic.sol#L127C1-L128C51>

## Recommendation

1 . Add the check if `repayBorrowAmount` which is liquidator input is greater than the violator balance cause the panic error if so, then no need to subtract

```solidity
if( loanBorrow.balance > repaidBorrowBalance) {
loanBorrow.balance -= repaidBorrowBalance;
}
```

Then we can clear the balance.

2 . Add mechanism that how much can liquidator can repay to acquire full borrow balance to prevent the panic code error.

## Proof of concept

## Proof of Concept

```solidity
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
  LoanManagerStateExposed__factory
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

describe("LoanManager (unit tests)", () => {
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
    const loanTargetHealth = BigInt(2e4);
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

  async function deprecateLoanTypeFixture() {
    const { admin, hub, user, unusedUsers, loanManager, loanTypeId } = await loadFixture(createLoanTypeFixture);

    // deprecate loan type
    await loanManager.connect(admin).deprecateLoanType(loanTypeId);

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanTypeId,
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
    const depositAmount = BigInt(1e18); // 1 ETH
    const depositFAmount = depositAmount;
    const depositInterestIndex = BigInt(1e18);
    const ethPrice = BigInt(3000e18); // 3000 USDC
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

  async function depositFEtherFixture() {
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

    // prepare deposit f token
    const nodeOutputData = getNodeOutputData(BigInt(3000e18));

    await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, nodeOutputData);
    const depositFAmount = BigInt(1e18);
    const depositInterestIndex = BigInt(1e18);
    await pools.ETH.pool.setUpdatedDepositInterestIndex(depositInterestIndex);

    // deposit into eth pool
    const depositFToken = await loanManager
      .connect(hub)
      .depositFToken(loanId, accountId, pools.ETH.poolId, user.address, depositFAmount);

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      depositFToken,
      loanTypeId,
      pools,
      loanId,
      accountId,
      depositFAmount,
    };
  }

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
    const borrowAmount = BigInt(2000e6); // 3000 USDC
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

  async function depositEtherAndStableBorrowUSDCFixture() {
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
    const borrowAmount = BigInt(1000e6); // 1000 USDC
    const borrow = await loanManager
      .connect(hub)
      .borrow(loanId, accountId, pools.USDC.poolId, borrowAmount, stableInterestRate);

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

  async function depositEtherAndVariableBorrowEtherFixture() {
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
    const ethNodeOutputData = getNodeOutputData(BigInt(3000e18));
    await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

    // prepare borrow
    const variableInterestIndex = BigInt(1.05e18);
    const stableInterestRate = BigInt(0.08e18);
    await pools.ETH.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
    await pools.ETH.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);

    // borrow from ETH pool
    const borrowAmount = BigInt(1e18); // 0.5 ETH
    const borrow = await loanManager.connect(hub).borrow(loanId, accountId, pools.ETH.poolId, borrowAmount, BigInt(0));

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      libraries,
      borrow,
      loanTypeId,
      pools,
      loanId,
      accountId,
      depositAmount,
      depositFAmount,
      borrowAmount,
      ethVariableInterestIndex: variableInterestIndex,
    };
  }

  async function depositEtherAndStableBorrowEtherFixture() {
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
    const ethNodeOutputData = getNodeOutputData(BigInt(3000e18));
    await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

    // prepare borrow
    const variableInterestIndex = BigInt(1.05e18);
    const stableInterestRate = BigInt(0.08e18);
    await pools.ETH.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
    await pools.ETH.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);

    // borrow from ETH pool
    const borrowAmount = BigInt(0.5e18); // 0.5 ETH
    const borrow = await loanManager
      .connect(hub)
      .borrow(loanId, accountId, pools.ETH.poolId, borrowAmount, stableInterestRate);

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      libraries,
      borrow,
      loanTypeId,
      pools,
      loanId,
      accountId,
      depositAmount,
      depositFAmount,
      borrowAmount,
      ethStableInterestRate: stableInterestRate,
    };
  }

  // clear timestamp changes
  after(async () => {
    await reset();
  });



  describe("Deployment", () => {
    it("Should set admin and contracts correctly", async () => {
      const { admin, hub, loanManager, oracleManager } = await loadFixture(deployLoanManagerFixture);

      // check default admin role
      expect(await loanManager.owner()).to.equal(admin.address);
      expect(await loanManager.defaultAdmin()).to.equal(admin.address);
      expect(await loanManager.defaultAdminDelay()).to.equal(SECONDS_IN_DAY);
      expect(await loanManager.getRoleAdmin(DEFAULT_ADMIN_ROLE)).to.equal(DEFAULT_ADMIN_ROLE);
      expect(await loanManager.hasRole(DEFAULT_ADMIN_ROLE, admin.address)).to.be.true;

      // check other roles
      expect(await loanManager.getRoleAdmin(LISTING_ROLE)).to.equal(DEFAULT_ADMIN_ROLE);
      expect(await loanManager.hasRole(LISTING_ROLE, admin.address)).to.be.true;
      expect(await loanManager.getRoleAdmin(ORACLE_ROLE)).to.equal(DEFAULT_ADMIN_ROLE);
      expect(await loanManager.hasRole(ORACLE_ROLE, admin.address)).to.be.true;
      expect(await loanManager.getRoleAdmin(HUB_ROLE)).to.equal(DEFAULT_ADMIN_ROLE);
      expect(await loanManager.hasRole(HUB_ROLE, hub.address)).to.be.true;

      // check state
      expect(await loanManager.getOracleManager()).to.equal(oracleManager);
    });
  });




describe("Liquidate the Just Borrowed loan", () => {
  
    it("Should successfully liquidate stable borrow when seizing new borrow and collateral", async () => {
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
          usdcStableInterestRate,
        } = await loadFixture(depositEtherAndStableBorrowUSDCFixture);
        

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
        const violatorLoanBefore = await loanManager.getUserLoan(violatorLoanId);
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
        const { pool, poolId } = pools.USDC;
        let ViolatorBalBefore = await loanManager.getUserBorrowBalance(violatorLoanId);
        let LiquidatorBalBefore = await loanManager.getUserBorrowBalance(liquidatorLoanId);
        let ViolaterCollBalBefore = await loanManager.getUserColBalance(violatorLoanId);
        let LiquidatorCollBalBefore = await loanManager.getUserColBalance(liquidatorLoanId);
  
        console.log("\n Violator Borrow Balance Before Liquidation",ViolatorBalBefore);
        console.log("\nViolator Collateral Balance Before Liquidation",ViolaterCollBalBefore);
        console.log("\nLiquidator Borrow Balance Before Liquidation",LiquidatorBalBefore);
        console.log("\nLiquidator Collateral Balance Before Liquidation",LiquidatorCollBalBefore);
        //const rebalanceDown = loanManager.rebalanceDown(violatorLoanId, poolId);

        // Violator:
        // Collateral 1 ETH = $1,000
        // Borrow 1,000 USDC = $1,000

        // Liquidator:
        // Collateral 10,000 USDC = $10,000
        // Borrow $0
        // Repay $960
        const repayAmount = BigInt(965e6); // 965USDC // @audit Changed only here.
        console.log("\nRepay Amount",repayAmount);
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

      let ViolatorBalAfter = await loanManager.getUserBorrowBalance(violatorLoanId);
      console.log("\nViolater Borrow Balance After Liquidation",ViolatorBalAfter);
      let ViolaterCollBalAfter = await loanManager.getUserColBalance(violatorLoanId);
      console.log(" \nViolator Collateral Balance After Liquidation",ViolaterCollBalAfter);
      let LiquidatorBalAfter = await loanManager.getUserBorrowBalance(liquidatorLoanId);
      console.log("\nLiquidator Borrow Balance After Liquidation",LiquidatorBalAfter);
      let LiquidatorCollBalAfter = await loanManager.getUserColBalance(liquidatorLoanId);
      console.log(" \nLiquidator Collateral Balance After Liquidation",LiquidatorCollBalAfter);
      });
});

});
```

Output :-

```
  LoanManager (unit tests)
    Deployment
      ✔ Should set admin and contracts correctly (1537ms)
    Liquidate the Just Borrowed loan

 Violator Borrow Balance Before Liquidation 1000000000n

Violator Collateral Balance Before Liquidation 1000000000000000000n

Liquidator Borrow Balance Before Liquidation 0n

Liquidator Collateral Balance Before Liquidation 1000000000n

Repay Amount 965000000n
      1) Should successfully liquidate stable borrow when seizing new borrow and collateral


  1 passing (2s)
  1 failing

  1) LoanManager (unit tests)
       Liquidate the Just Borrowed loan
         Should successfully liquidate stable borrow when seizing new borrow and collateral:
     Error: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block)
    at UserLoanLogic.transferBorrowFromViolator (contracts/hub/logic/UserLoanLogic.sol:128)
    at LiquidationLogic.updateLiquidationBorrows (contracts/hub/logic/LiquidationLogic.sol:49)
    at LoanManagerLogic.executeLiquidate (contracts/hub/logic/LoanManagerLogic.sol:478)
    at LoanManager.liquidate (contracts/hub/LoanManager.sol:248)
    at EdrProviderWrapper.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:427:41)
    at async HardhatEthersSigner.sendTransaction (node_modules/@nomicfoundation/hardhat-ethers/src/signers.ts:125:18)
    at async send (node_modules/ethers/src.ts/contract/contract.ts:313:20)
    at async Proxy.liquidate (node_modules/ethers/src.ts/contract/contract.ts:352:16)
    at async Context.<anonymous> (test/hub/Rebalance.test.ts:780:27)
```

In the above we can see the balances of both violator and liquidator before liquidation when initiate the liquidate function it halt by panic error.

Please search `@audit` keyword in above poc.
