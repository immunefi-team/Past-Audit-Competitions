# Boost \_ Folks Finance 33817 - \[Smart Contract - High] Incorrect calculation of effective borrow valu

Submitted on Tue Jul 30 2024 04:21:26 GMT-0400 (Atlantic Standard Time) by @zarkk for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33817

Report type: Smart Contract

Report severity: High

Target: <https://testnet.snowtrace.io/address/0xf8E94c5Da5f5F23b39399F6679b2eAb29FE3071e>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

Effective stable borrow value is calculated incorrectly with stable borrow balance to be decreasing instead of increasing, leading to protocol insolvency due to under-collateralized loan acceptance and wrong liquidations.

## Vulnerability Details

When a withdrawal or a liquidation is about to happen in Folks Finance, the collateralization of the `UserLoan` is checked through the `getLoanLiquidity` function which returns the `effective borrow value` and the `effective collateral value` of a `UserLoan`. We can see the implementation here :

```solidity
function getLoanLiquidity(
        LoanManagerState.UserLoan storage loan,
        mapping(uint8 => IHubPool) storage pools,
        mapping(uint8 => LoanManagerState.LoanPool) storage loanPools,
        IOracleManager oracleManager
    ) internal view returns (DataTypes.LoanLiquidityParams memory loanLiquidity) {
        // declare common variables
        uint256 effectiveValue;
        uint256 balance;
        uint8 poolId;
        uint256 poolsLength;
        DataTypes.PriceFeed memory priceFeed;

        // calc effective collateral value
        poolsLength = loan.colPools.length;
        for (uint8 i = 0; i < poolsLength; i++) {
            poolId = loan.colPools[i];

            balance = loan.collaterals[poolId].balance.toUnderlingAmount(
                pools[poolId].getUpdatedDepositInterestIndex()
            );
            priceFeed = oracleManager.processPriceFeed(poolId);
            effectiveValue += MathUtils.calcCollateralAssetLoanValue(
                balance,
                priceFeed.price,
                priceFeed.decimals,
                loanPools[poolId].collateralFactor
            );
        }
        loanLiquidity.effectiveCollateralValue = effectiveValue;

        // calc effective borrow value
        effectiveValue = 0;
        poolsLength = loan.borPools.length;
        for (uint8 i = 0; i < poolsLength; i++) {
            poolId = loan.borPools[i];

            LoanManagerState.UserLoanBorrow memory loanBorrow = loan.borrows[poolId];
            balance = loanBorrow.lastStableUpdateTimestamp > 0
                ? calcStableBorrowBalance(
                    loanBorrow.balance,
                    loanBorrow.lastInterestIndex,
                    loanBorrow.stableInterestRate,
                    block.timestamp - loanBorrow.lastStableUpdateTimestamp
                )
                : calcVariableBorrowBalance(
                    loanBorrow.balance,
                    loanBorrow.lastInterestIndex,
                    pools[poolId].getUpdatedVariableBorrowInterestIndex()
                );
            priceFeed = oracleManager.processPriceFeed(poolId);
            effectiveValue += MathUtils.calcBorrowAssetLoanValue(
                balance,
                priceFeed.price,
                priceFeed.decimals,
                loanPools[poolId].borrowFactor
            );
        }
        loanLiquidity.effectiveBorrowValue = effectiveValue;
    }
```

However, if we deep on the calculations of stable borrow balance, we will see that they are done incorrectly. In order to calculate the stable borrow balance, `calcStableBorrow` function is called passing the current debt (`loanBorrow.balance`), the current interest index(`loanBorrow.lastInterestIndex`), the current interest rate (`loanBorrow.stableInterestRate`) and the time passed since the last update. This happens in order to update the interest index and then the debt to be updated also, so to reflect the present. Let's see how `calcStableBorrowBalance` is implemented :

```solidity
function calcStableBorrowBalance(
        uint256 balance,
        uint256 loanInterestIndex,
        uint256 loanInterestRate,
        uint256 stableBorrowChangeDelta
    ) private pure returns (uint256) {
        uint256 stableBorrowInterestIndex = MathUtils.calcBorrowInterestIndex(
            loanInterestRate,
            loanInterestIndex,
            stableBorrowChangeDelta
        );
        return balance.calcBorrowBalance(loanInterestIndex, stableBorrowInterestIndex);
    }
```

The `stableborrowInterestIndex` is the new updated interest index which is supposed to inflate the `balance`. However, as we see in the `calcBorrowBalance`, due to wrong order of parameters, the `balance` is decreased instead of increased. Here, is the `calcBorrowBalance` of `MathUtils` which expects as second parameter the new interest index and as third parameter the old interest index.

```solidity
function calcBorrowBalance(
        uint256 borrowBalanceAtTn_1,
        uint256 borrowInterestIndexAtT,
        uint256 borrowInterestIndexAtTn_1
    ) internal pure returns (uint256) {
        return
            borrowBalanceAtTn_1.mulDiv(
                borrowInterestIndexAtT.mulDiv(ONE_18_DP, borrowInterestIndexAtTn_1, Math.Rounding.Ceil),
                ONE_18_DP,
                Math.Rounding.Ceil
            );
    }
```

However, the parameters are passed in the opposite order and a deflated balance (debt) is returned. As a result, the `UserLoan` seems to have less debt than what really has.

## Impact Details

This vulnerability leads, eventually and rapidly, to the insolvency of Folks Finance. Firstly, as demonstrated in the Proof of Concept (PoC), users are able to withdraw collateral from their loans, leaving them under-collateralized. This can have catastrophic effects for the protocol as it accrues bad debt. Secondly, it prevents legitimate liquidations from happening since the debt of the violator seems smaller than it actually is. The combined effect of these issues can lead to a cascading failure of the lending system, where the protocol cannot cover the borrowed amounts with the available collateral, ultimately leading to insolvency and significant financial losses for both the protocol and its users.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L387>

## Proof of concept

## Proof of Concept

To understand better this critical vulnerability, add this test under the `"Withdraw" test suite of` LoanManager.test.ts`and run`npm run test\`\`\` :

```javascript
it.only("Should successfuly withdraw and leave user loan under-collateralised", async () => {
      const { hub, loanManager, loanManagerAddress, pools, loanId, accountId } = await loadFixture(
        depositEtherAndStableBorrowUSDCFixture
      );

      // Deposited 1 ETH = $3,000
      // Borrowed stable 1,000 USDC = $1,000
      // stableInterestRate = BigInt(0.1e18) => 10%

      const { pool, poolId } = pools.ETH;

      time.increase(31536000); // Advance 1 year.

      const depositInterestIndex = BigInt(1.05e18);
      await pool.setUpdatedDepositInterestIndex(depositInterestIndex);


      // Now, with depositInterestIndex to equal 1.05, our 1 ETH deposit -> 1.05 ETH = $3,150 -> 70% CF = $2,100
      // Now, with stableInterestRate = 0.1 -> 1,000 USDC = $1,100 debt.
      // That means that if we try to withdraw 0.6 ETH = $1,260, we will leave in the protocol $2,100 - $1,260 = $840 and the loan will be under-collateralised.
      // So, it should revert with under-collateralised error, but it is not.

      // Withdraw 0.6 ETH = $1,260 and leave the loan under-collateralised.
      let withdrawAmount = BigInt(0.6e18);
      let withdrawFAmount = toFAmount(withdrawAmount, depositInterestIndex);
      await pool.setWithdrawPoolParams({ underlingAmount: withdrawAmount, fAmount: withdrawFAmount });
      const withdraw = loanManager.connect(hub).withdraw(loanId, accountId, poolId, withdrawAmount, false);
    });
```
