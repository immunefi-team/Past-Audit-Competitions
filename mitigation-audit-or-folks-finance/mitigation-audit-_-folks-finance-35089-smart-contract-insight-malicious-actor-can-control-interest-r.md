# Mitigation Audit \_ Folks Finance 35089 - \[Smart Contract - Insight] Malicious actor can control inte

Submitted on Wed Sep 04 2024 19:02:38 GMT-0400 (Atlantic Standard Time) by @zarkk for [Mitigation Audit | Folks Finance](https://immunefi.com/bounty/mitigation-audit-folksfinance/)

Report ID: #35089

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/pull/45>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

At no cost, a malicious actor can DoS legitimate borrowing actions and, possibly, take advantage of it and manipulate the utilization ratio and, thus, interest rates.

## Vulnerability Details

In [pull request #45](https://github.com/Folks-Finance/folks-finance-xchain-contracts/pull/45/commits/d26f680e53802046dbf62d33ec26330f3fc4fd2b), a new check was introduced in the `HubPoolLogic` that ensures that the borrower can not borrow more than the difference between `totalDeposits` minus `totalBorrows`. We can see this check here :

```solidity
    function prepareForBorrow(
        HubPoolState.PoolData storage pool,
        uint256 amount,
        DataTypes.PriceFeed memory priceFeed,
        uint256 maxStableRate
    ) external returns (DataTypes.BorrowPoolParams memory borrowPoolParams) {
       // ...
        uint256 totalDebt = pool.variableBorrowData.totalAmount + pool.stableBorrowData.totalAmount;
@>        if (amount > MathUtils.calcAvailableLiquidity(totalDebt, pool.depositData.totalAmount)) revert InsufficientLiquidity();

        if (isStable && !pool.isStableBorrowSupported()) revert StableBorrowNotSupported();
        if (pool.isBorrowCapReached(priceFeed, amount)) revert BorrowCapReached();
        if (isStable && pool.isStableBorrowCapExceeded(amount)) revert StableBorrowPercentageCapExceeded();
        if (isStable && stableBorrowInterestRate > maxStableRate)
            revert MaxStableRateExceeded(stableBorrowInterestRate, maxStableRate);
       // ...
    }
```

However, this addition opens the door to front-running attacks that DoS the `borrow` action from a legitimate user and instant back-run from the malicious actor. The malicious can `borrow` up to the amount of the `availableLiquidity` or slightly less just to make the legitimate transaction of borrower to revert. After this, he is going to `repay` the loan he just got, at no cost for him. He is incentivized to do this because blocking new borrows may affect negatively for him the interest rates and manipulating the utilization ratio (`totalBorrows / totalDeposits`) can be advantageous for him since it directly influence the interest rates.

## Impact Details

This vulnerability allows a malicious actor to disrupt legitimate borrowing actions by manipulating the `availableLiquidity` calculation, leading to a DoS for genuine users. By front-running a legitimate borrow transaction and then immediately repaying it, the attacker incurs no cost due to the absence of borrowing fees. This behavior can result in fluctuating utilization ratios, which in turn manipulates interest rates and achieve more favorable borrowing conditions. The impact, under normal circumstances, would be medium as the other two DoS related issues found in the original Boost (front running with the same accountId and loanId), however, in this issue we have, also, manipulation of the interest rates at attacker's will. This has direct impact to all other borrowers/deposits and the issue is not isolated DoS to one user.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/ee9b4a85e6b1ef11032f6cf90fadf87d065036ec/contracts/hub/logic/HubPoolLogic.sol#L102>

## Proof of concept

## Proof of Concept

To understand better this vulnerability, add this test under `HubPool.test.ts` test suite and run `npm test` :

```javascript
    it.only("Should allow a malicious actor to DoS borrows and manipulate the interest rates", async () => {
      const { loanManager, hubPool, hubPoolLogicAddress, oracleManager, poolId } =
        await loadFixture(deployHubPoolFixture);
      const ethNodeOutputData = getNodeOutputData(BigInt(1000e18));
      await oracleManager.setNodeOutput(poolId, ethDecimals, ethNodeOutputData);
      
      // The initial state of the pool :
      // Total deposit amount = 1000e18
      // Total borrow amount = 600e18
      // A user is seeing this and tries to borrow 100e18
      const poolData = getInitialPoolData();
      poolData.depositData.totalAmount = BigInt(1000e18);
      poolData.variableBorrowData.totalAmount = BigInt(500e18);
      poolData.stableBorrowData.totalAmount = BigInt(100e18);
      await hubPool.setPoolData(poolData);

      // But.. a malicious actor can DoS him by depositing 399e18
      poolData.depositData.totalAmount = BigInt(1000e18);
      poolData.variableBorrowData.totalAmount = BigInt(500e18) + BigInt(399e18);
      poolData.stableBorrowData.totalAmount = BigInt(100e18);
      await hubPool.setPoolData(poolData);

      // A ligitamate user is trying, now, to borrow.
      let amount = BigInt(100e18);
      const preparePoolForBorrow = hubPool.connect(loanManager).preparePoolForBorrow(amount, 0);
      const hubPoolLogic = await ethers.getContractAt("HubPoolLogic", hubPoolLogicAddress);
      // Her transaction will revert because the malicious borrowed just a little bit less than the borrow amount.
      await expect(preparePoolForBorrow).to.be.revertedWithCustomError(hubPoolLogic, "InsufficientLiquidity");

      // After, he instantly repays the loan, at no cost.
      poolData.depositData.totalAmount = BigInt(1000e18);
      poolData.variableBorrowData.totalAmount = BigInt(500e18);
      poolData.stableBorrowData.totalAmount = BigInt(100e18);
      await hubPool.setPoolData(poolData);
    });
```
