# Boost \_ Folks Finance 34074 - \[Smart Contract - Critical] Hub missing check for available liquidity

Submitted on Mon Aug 05 2024 08:09:42 GMT-0400 (Atlantic Standard Time) by @A2Security for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34074

Report type: Smart Contract

Report severity: Critical

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Temporary freezing of funds of at least 24h
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Smart contract unable to operate due to lack of token funds
* Permanent freezing of funds

## Description

## Impact

The Hub doesn't check for available liquidity, leading to borrow/withdraw operation on the hub succeeding eventhough the spokeToken doesn't have enough liquidity. In the case of race conditions, e.g user A borrows 5000 ETH, and Whale B borrows/withdraws remaining liquidity from the ethHubPool. The borrow operation for user A will succeed, but the second part of the operation will revert. => This will lead to:

* Utilization ratio surpassing 100% -> interest Rates becomes unbelievable high
* User A wouldn't be able to withdraw his borrowed funds (locked) and will be forced to pay really high interest
* User A wouldn't be able to reverse the action from spoke, because all operations from Hub -> spoke Chain can't be reversed

The higher the borrow amount, the bigger the chance that the user funds will be stuck (**possibly forever**, if the liquidity in the spoketoken, never reaches the wanted amount to borrow)\
Please note that, such a scenario is highly likeable considering that Folks Finance is cross chain by design, and operations like borrow/withdraw from spoke chains != Avalanche , will require 2 cross-chain actions that requires bridging through wormhole/ccip (**which will take considerable time to be executed, and make predicting available liquidity not an easy task for protocol users**)

## Description

In `HubPool.getSendTokenMessage()` which is used to build the message to send through the bridgeRouter to the spoke contracts (mostly cross-chains)

```solidity
    function getSendTokenMessage(IBridgeRouter bridgeRouter, uint16 adapterId, uint256 gasLimit, bytes32 accountId, uint16 chainId, uint256 amount, bytes32 recipient)
        external
        override
        onlyRole(HUB_ROLE)
        nonReentrant
        returns (Messages.MessageToSend memory)
    {
        // check chain is compatible
        bytes32 spokeAddress = getChainSpoke(chainId);

        // prepare message
>>        Messages.MessageParams memory params = Messages.MessageParams({adapterId: adapterId, returnAdapterId: 0, receiverValue: 0, gasLimit: gasLimit, returnGasLimit: 0});
```

As we can see in the message returnAdapterId is set to 0 meaning messages can't be reversed

## Recomendation

An easy fix for this is to add a check for available liquidity to each a user actions that removes liquidity from a hubpool

```solidity
if (pool.depositData.totalAmout < pool.variableData.totalAmount + pool.stableData.totalAmount + amount_to_remove){
    revert NotEnoughLiquidityAvailable;
}
```

(a side note we know that totaldebt here only includes principal and can in certain condition not be accurate, so we would also recommend adding a margin of safety (e.g 5%))

## Proof of concept

## Proof Of Concept

**Test Result:**

```log
Ran 1 test for test/pocs/test_poc.sol:Pocs
[PASS] test_poc06() (gas: 1632608)
Logs:
  variable Interest Rate 1 %
  stable Interest Rate   8 %
  depositData.totalAmount avax 248 avax
  total debt hubpool avax 17 avax
  Alice borrows 258 avax
  Avax Alice Recieved 0
  variable Interest Rate 271 %
  stable Interest Rate   272 %
  depositData.totalAmount avax 248 avax
  total debt hubpool avax 276 avax

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 684.53ms (9.27ms CPU time)

Ran 1 test suite in 687.47ms (684.53ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

To run the test please add `test/pocs/base_test.sol`

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

please add the poc to `test/pocs/test_poc.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.19;
import "@openzeppelin/contracts/utils/math/Math.sol";

import "./base_test.sol";
import "@forge-std/console.sol";
import "contracts/bridge/libraries/Messages.sol";

contract Pocs is baseTest {
    using Math for uint256;

    uint256 private constant SECONDS_IN_YEAR = 365 days;
    uint256 internal constant ONE_4_DP = 1e4;
    uint256 internal constant ONE_6_DP = 1e6;
    uint256 internal constant ONE_10_DP = 1e10;
    uint256 internal constant ONE_12_DP = 1e12;
    uint256 internal constant ONE_14_DP = 1e14;
    uint256 internal constant ONE_18_DP = 1e18;

    function _getStableRateUsdc() internal returns (uint256) {
        HubPoolState.StableBorrowData memory stableData = hubPoolUsdc.getStableBorrowData();
        return stableData.interestRate;
    }
    function _getVariableRateUsdc() internal returns (uint256) {
        HubPoolState.VariableBorrowData memory stableData = hubPoolUsdc.getVariableBorrowData();
        return stableData.interestRate;
    }
    function _getDepositTotalUsdc() internal returns (uint256) {
        HubPoolState.DepositData memory stableData = hubPoolUsdc.getDepositData();
        return stableData.totalAmount;
    }
    function _getTotalDebtUsdc() internal returns (uint256) {
        HubPoolState.VariableBorrowData memory variableData = hubPoolUsdc.getVariableBorrowData();
        HubPoolState.StableBorrowData memory stableData = hubPoolUsdc.getStableBorrowData();
        return stableData.totalAmount + variableData.totalAmount;
    }

    function _getStableRateAvax() internal returns (uint256) {
        HubPoolState.StableBorrowData memory stableData = hubPoolAvax.getStableBorrowData();
        return stableData.interestRate;
    }
    function _getVariableRateAvax() internal returns (uint256) {
        HubPoolState.VariableBorrowData memory stableData = hubPoolAvax.getVariableBorrowData();
        return stableData.interestRate;
    }
    function _getDepositTotalAvax() internal returns (uint256) {
        HubPoolState.DepositData memory stableData = hubPoolAvax.getDepositData();
        return stableData.totalAmount;
    }
    function _getTotalDebtAvax() internal returns (uint256) {
        HubPoolState.VariableBorrowData memory variableData = hubPoolAvax.getVariableBorrowData();
        HubPoolState.StableBorrowData memory stableData = hubPoolAvax.getStableBorrowData();
        return stableData.totalAmount + variableData.totalAmount;
    }
    function test_poc06() public {
        // initialize caps
        vm.startPrank(LISTING_ROLE);
        loanManager.updateLoanPoolCaps(1, hubPoolAvax.getPoolId(), 1e12, 1e12);
        hubPoolAvax.updateCapsData(HubPoolState.CapsData(type(uint64).max, type(uint64).max, 1e18));
        loanManager.updateLoanPoolCaps(2, hubPoolAvax.getPoolId(), 1e12, 1e12);
        vm.stopPrank();

        // Deposit USDC as coll for alice
        uint256 largeUsdcDeposit = 1e11; //
        _approveUsdc(alice, address(spokeUsdc), largeUsdcDeposit);
        _deposit(alice, aliceAccountId, aliceLoanIds[0], largeUsdcDeposit, spokeUsdc);
        console.log("variable Interest Rate", _getVariableRateAvax() / 1e16, "%");
        console.log("stable Interest Rate  ", _getStableRateAvax() / 1e16, "%");
        console.log("depositData.totalAmount avax", _getDepositTotalAvax() / 1e18, "avax");
        console.log("total debt hubpool avax", _getTotalDebtAvax() / 1e18, "avax");

        uint256 borrowAmount = _getDepositTotalAvax() + 10e18; // 10 AVAX

        uint256 alice_balance_before = alice.balance;
        console.log("Alice borrows", borrowAmount / 1e18, "avax");
        _borrowVariable(alice, aliceAccountId, aliceLoanIds[0], hubPoolAvax.getPoolId(), borrowAmount);
        console.log("Avax Alice Recieved", alice_balance_before - alice.balance);

        console.log("variable Interest Rate", _getVariableRateAvax() / 1e16, "%");
        console.log("stable Interest Rate  ", _getStableRateAvax() / 1e16, "%");
        console.log("depositData.totalAmount avax", _getDepositTotalAvax() / 1e18, "avax");
        console.log("total debt hubpool avax", _getTotalDebtAvax() / 1e18, "avax");
    }
}
```
