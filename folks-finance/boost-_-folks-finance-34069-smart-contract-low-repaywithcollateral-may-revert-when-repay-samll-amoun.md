# Boost \_ Folks Finance 34069 - \[Smart Contract - Low] repayWithCollateral may revert when repay samll

Submitted on Mon Aug 05 2024 06:16:20 GMT-0400 (Atlantic Standard Time) by @twcctop for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34069

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* DOS repay

## Description

## Brief/Intro

`repayWithCollateral` may revert when repay samll amount token, because can not make sure `principalPaid` is larger than `interestPaid`

## Vulnerability Details

In function `updateWithRepayWithCollateral` , when update totalAmount,we have logic:

```solidity

    function updateWithRepayWithCollateral( 
     ...
@>     pool.depositData.totalAmount -= principalPaid - interestPaid;   
     ... 
    
```

which means `pool.depositData.totalAmount = pool.depositData.totalAmount -(principalPaid - interestPaid);` the issue is we can not make sure `principalPaid` larger than `principalPaid`.

`LoanManager`# `repayWithCollateral` will call `updateWithRepayWithCollateral`. Let's dive into function `executeRepay`

```solidity
 function executeRepay( 
...
    (uint256 principalPaid, uint256 interestPaid, uint256 excessPaid, uint256 loanStableRate) = userLoan
@>        .decreaseBorrow(
            DataTypes.UpdateUserLoanBorrowParams({
            poolId: params.poolId,
            amount: params.amount,
            poolVariableInterestIndex: borrowPoolParams.variableInterestIndex,
            poolStableInterestRate: borrowPoolParams.stableInterestRate,
            isStableInterestRateToUpdate: false
    })
    );      
...

  function decreaseBorrow(
       ...
        uint256 balance = loanBorrow.balance;
        uint256 interest = balance - loanBorrow.amount;
        
        excessPaid = params.amount > balance ? params.amount - balance : 0;
  @>      interestPaid = Math.min(params.amount, interest);
  @>      principalPaid = params.amount - interestPaid - excessPaid;



```

from the logic calculate `interestPaid` and `principalPaid` , when repay, the interest is first calculated and can not make sure `interestPaid` < `principalPaid` , in some situation, `principalPaid` equals 0 is possible.

modify:

```diff

-   pool.depositData.totalAmount -= principalPaid - interestPaid; 
+   pool.depositData.totalAmount -= pool.depositData.totalAmount;
+   pool.depositData.totalAmount += pool.depositData.interestPaid;

```

## Impact Details

`repayWithCollateral` may get dos when repay small amount of token, in some situation, user try to repay samll amount of token to get

avoid to be liquidated ,funciton could not work, and cause user funds get liquidated and lost

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L84-L106> <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/LoanManagerLogic.sol#L368>

## Proof of concept

```js
it("Should successfully update pool with repay with collateral of stable borrow", async () => {
 const { admin, user, hubPool } = await loadFixture(deployHubPoolFixture);

 // deploy mock loan manager so can emit event with params
 const loanManager = await new HubPoolLogged__factory(user).deploy(hubPool);
 await hubPool.connect(admin).grantRole(LOAN_MANAGER_ROLE, loanManager);

 // set pool data
 const depositInterestIndex = BigInt(1.839232023893e18);
 const depositTotalAmount = BigInt(10e18);
 const stableBorrowTotalAmount = BigInt(1.43543539e18);
 const stableInterestRate = BigInt(0.1420009e18);
 const stableAverageInterestRate = BigInt(0.19014e18);
 const feeTotalRetainedAmount = BigInt(0.14e18);
 const poolData = getInitialPoolData();
 poolData.depositData.interestIndex = depositInterestIndex;
 poolData.depositData.totalAmount = depositTotalAmount;
 poolData.stableBorrowData.totalAmount = stableBorrowTotalAmount;
 poolData.stableBorrowData.interestRate = stableInterestRate;
 poolData.stableBorrowData.averageInterestRate = stableAverageInterestRate;
 poolData.feeData.totalRetainedAmount = feeTotalRetainedAmount;
 await hubPool.setPoolData(poolData);

 // calculate new stable average interest rate
 //@audit try to mididy  principalPaid is 0 , this will revert() 
 const principalPaid = BigInt(0);
 const loanStableRate = BigInt(0.125e18);
 const newStableAverageInterestRate = calcDecreasingAverageStableBorrowInterestRate(
     principalPaid,
     loanStableRate,
     stableBorrowTotalAmount,
     stableAverageInterestRate
 );

 // update pool with repay with collateral
 const interestPaid = BigInt(0.09811e8);
 const updatePoolWithRepayWithCollateral = await loanManager.updatePoolWithRepayWithCollateral(
     principalPaid,
     interestPaid,
     loanStableRate
 );
 
});




```
