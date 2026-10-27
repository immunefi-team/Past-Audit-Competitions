# Boost \_ Folks Finance 34085 - \[Smart Contract - Low] partial repayment with collaterals will revert

Submitted on Mon Aug 05 2024 11:59:32 GMT-0400 (Atlantic Standard Time) by @A2Security for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34085

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* partial repayment reverts

## Description

## Title: repayWithCollateral will revert because of underflow

## Impact

Repayments are always a healthy action on the protocol, and shouldn't revert. Due to this bug, repayment with collateral will revert, preventing users from repaying their debt in certain conditions and improving their loan health.

## Description

`contracts/hub/logic/HubPoolLogic.sol`

```solidity
    function updateWithRepayWithCollateral(HubPoolState.PoolData storage pool, uint256 principalPaid, uint256 interestPaid, uint256 loanStableRate)
        external
        returns (DataTypes.RepayWithCollateralPoolParams memory repayWithCollateralPoolParams)
    {
        if (loanStableRate > 0) {
            pool.stableBorrowData.averageInterestRate =
                MathUtils.calcDecreasingAverageStableBorrowInterestRate(principalPaid, loanStableRate, pool.stableBorrowData.totalAmount, pool.stableBorrowData.averageInterestRate);
            pool.stableBorrowData.totalAmount -= principalPaid;
        } else {
            pool.variableBorrowData.totalAmount -= principalPaid;
        }
        // @audit-issue : underflow when principalPaid < interestPaid
>>        pool.depositData.totalAmount -= principalPaid - interestPaid; // totalAmount - principal + interest
        repayWithCollateralPoolParams.fAmount = (principalPaid + interestPaid).toFAmount(pool.depositData.interestIndex);

        pool.updateInterestRates();
    }
```

The bugs simply concerns the line mentioned above, if principalPaid is less than interestPaid the transaction will revert due to underflow. Please also notice when reducing loan balance of a userLoan, it is the intended design to repay the interest off first before reducing the collateral

## Recomendation

To fix this, and avoid the underflow, a possible fix would be like this

```diff
--    pool.depositData.totalAmount -= principalPaid - interestPaid; 
++    pool.depositData.totalAmount += interestPaid; 
++    pool.depositData.totalAmount -= principalPaid; 

```

## Proof of concept

## Proof of Concept

To showcase the problem we simply made a simple reimplementation of the vulenrable code to showcase the underflow: In this example:

* initial depositData.totalAmount = 1000
* interestPaid = 150
* principalPaid = 100

\=> this will revert with an underflow **Result**:

```log
└─$ forge test --mt test_poc_03 -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/pocs/test_poc.sol:Pocs_3
[FAIL. Reason: panic: arithmetic underflow or overflow (0x11)] test_poc_03() (gas: 407)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 706.78ms (143.02µs CPU time)

Ran 1 test suite in 711.03ms (706.78ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/pocs/test_poc.sol:Pocs_3
[FAIL. Reason: panic: arithmetic underflow or overflow (0x11)] test_poc_03() (gas: 407)

Encountered a total of 1 failing tests, 0 tests succeeded
```

```solidity
contract Pocs_3 is Test {
    // Mock pool struct
    struct Pool {
        uint256 totalAmount;
    }
    function updatePool(Pool memory _pool, uint256 _principalPaid, uint256 _interestPaid) internal pure {
        _pool.totalAmount -= _principalPaid - _interestPaid;
    }
    function test_poc_03() public {
        // Setup
        uint256 initialTotalAmount = 1000;
        uint256 principalPaid = 100;
        uint256 interestPaid = 150;

        Pool memory pool = Pool(initialTotalAmount);

        // Test
 //       vm.expectRevert(stdError.arithmeticError);
        updatePool(pool, principalPaid, interestPaid);
    }
}
```
