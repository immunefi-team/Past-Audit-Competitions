# Boost \_ Folks Finance 33311 - \[Smart Contract - Critical] Infinite Interest rate bug

Submitted on Wed Jul 17 2024 12:14:00 GMT-0400 (Atlantic Standard Time) by @kankodu for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33311

Report type: Smart Contract

Report severity: Critical

Target: <https://testnet.snowtrace.io/address/0x96e957bF63B5361C5A2F45C97C46B8090f2745C2>

Impacts:

* Protocol insolvency
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Here's the equation for utilisation that is being used currently `U=TotalVariableBorrowAmount+TotalStableBorrowAmountTotalDeposits/TotalDeposits`

When totalDeposits is lower than the totalBorrowed amount, the utilization can be much greater than 100%, which in turn makes borrowRates and depositRates very high.

## Vulnerability Details

* When a new token with CF > 0 is added, the attacker deposits 1e5 wei of its token, making TotalDeposits = 1e5.
* Let's say the decimals for this newly added token are 1e18. The attacker then donates 1e18 wei of tokens directly to the HubPool and borrows.
  * This is allowed as there is no check for it in the borrow method. It makes the totalBorrows = 1e18.
  * Utilization is 1e13 in this case, which makes the interest rate \~4e31. This translates to 4 trillion percent per second.
* After just a block, the attacker's original 1e5 deposits would have turned into a very large amount (in billions) due to the interest rate being an outrageous trillion percent per second.
* The attacker goes ahead and borrows all the tokens against this deposit as CF for this token is non-zero.

## Impact Details

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Protocol insolvency

## References

* <https://medium.com/certora/silo-finance-post-mortem-3b690fffeb08>

## Proof of concept

## Proof of Concept

Add below testcase in `test/hub/HubPool.test.ts` that shows the the interest rate is a very large amount when totalDeposits is way smaller than the totalBorrowedAmount

```
  it.only("infinite interest rate", async () => {
      const { loanManager, hubPool } = await loadFixture(deployHubPoolFixture);

      // set pool data with deposit total amount
      const depositTotalAmount = BigInt(1e5);
      const poolData = getInitialPoolData();
      poolData.depositData.totalAmount = depositTotalAmount;
      await hubPool.setPoolData(poolData);

      // update pool with borrow
      const amount = BigInt(1e18);
      const isStable = false;
      const updatePoolWithBorrow = await hubPool.connect(loanManager).updatePoolWithBorrow(amount, isStable);
      expect((await hubPool.getVariableBorrowData())[3]).to.equal(poolData.variableBorrowData.totalAmount + amount);
      await expect(updatePoolWithBorrow).to.emit(hubPool, "InterestRatesUpdated");

      expect((await hubPool.getVariableBorrowData())[4]).to.be.greaterThan(BigInt(1e31))
    });
```
