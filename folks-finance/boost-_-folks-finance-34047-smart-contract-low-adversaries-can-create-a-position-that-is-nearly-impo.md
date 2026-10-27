# Boost \_ Folks Finance 34047 - \[Smart Contract - Low] Adversaries can create a position that is nearl

Submitted on Sun Aug 04 2024 16:20:03 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34047

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xf8E94c5Da5f5F23b39399F6679b2eAb29FE3071e>

Impacts:

* Protocol insolvency

## Description

## Description

Users' loan position is stored in a mapping of a struct `UserLoan`

```
struct UserLoan {
    bool isActive;
    bytes32 accountId;
    uint16 loanTypeId;
    uint8[] colPools;
    uint8[] borPools;
    mapping(uint8 poolId => UserLoanCollateral) collaterals;
    mapping(uint8 poolId => UserLoanBorrow) borrows;
}
mapping(bytes32 loanId => UserLoan) internal _userLoans;
```

The `borPools` member keeps a list of `poolId` values representing the pools that users have positions in.\
This is similar to how other lending platforms track the markets a user has entered.

Below is the code snippet responsible for adding a new `poolId` to user's loan.

```
function increaseBorrow(
    LoanManagerState.UserLoan storage loan,
    DataTypes.UpdateUserLoanBorrowParams memory params,
    bool isStable
) external {
    LoanManagerState.UserLoanBorrow storage loanBorrow = loan.borrows[params.poolId];

    if (loanBorrow.balance == 0) {
        initLoanBorrowInterests(loanBorrow, params, isStable);
        loanBorrow.amount = params.amount;
        loanBorrow.balance = params.amount;
        loan.borPools.push(params.poolId); @<--
    } else {
        if (isStable != loanBorrow.stableInterestRate > 0) revert BorrowTypeMismatch();
        updateLoanBorrowInterests(loanBorrow, params);

        // update amount
        loanBorrow.amount += params.amount;
        loanBorrow.balance += params.amount;
    }
}
```

If the current borrow balance for the `poolId` is zero, the system adds the `poolId` to `borPools`. However, this approach has a flaw: it's possible to borrow an amount of 0. If the borrow function is called with an amount of 0 multiple times, the `increaseBorrow` function will repeatedly add the same `poolId` to `borPools`.

`borPools` is used in `getLoanLiquidity` to calculate the effective value of the loan.

```
function getLoanLiquidity(
    LoanManagerState.UserLoan storage loan,
    mapping(uint8 => IHubPool) storage pools,
    mapping(uint8 => LoanManagerState.LoanPool) storage loanPools,
    IOracleManager oracleManager
) internal view returns (DataTypes.LoanLiquidityParams memory loanLiquidity)
    ...
    ... snipped
    ...
    poolsLength = loan.borPools.length;
    for (uint8 i = 0; i < poolsLength; i++) { @<-- maximum iteration = 255
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
    ...
    ... snipped
```

We can see that it iterates over the entire length of the array. Each iteration incurs a significant gas cost because it needs to retrieve the price from the oracle during every iteration. An adversary could exploit this by adding multiple `poolId` values to the array (up to a maximum of 255). This would increase the gas cost significantly, potentially even reaching the block gas limit, making it impossible to liquidate their position as decribe in the following attack scenario.

### Attack Scenario

1. Attacker deposits AVAX as collateral
2. Attacker borrows USDC out

Attacker's borPools = \[USDC]

3. Attacker starts repeatedly borrowing AVAX with amount=0

Attacker's borPool = \[USDC, AVAX, AVAX, AVAX, ...]

Each iteration increases the gas consumption to liquidate attacker's position because it increases the number of iterations in `UserLoanLogic#getLoanLiquidity` which is called in liquidation execution flow.

4. Attacker can keep repeating *Step 3* to the maximum of 255 iterations.

In cases where a complex oracle is used (such as nodes with many parents), the gas consumption for liquidation could reach the block gas limit, making it impossible to liquidate the attacker's position.

While the attacker does have to cover the gas fees to create this position, the cost is minimal. Even in the final iteration, where the gas usage approaches the block limit, the expense remains under $20.

## Impact

* This could result in bad debt for the protocol, as the attacker's position cannot be liquidated.

## Recommended Mitigations

* Consider setting a maximum limit on the borrow and collateral pools for a single loan.

## Proof of concept

## Proof-of-Concept

A test in the secret gist demonstrates the attack scenario described. It shows that an attacker with a borrow pool length of 255 could potentially incur a cost of 13 million gas units to execute the liquidation.

### Steps

1. Run `forge init --no-commit --no-git --vscode`.
2. Create a new test file, `FolksNotLiquidate.t.sol` in `test` directory.
3. Put the test from secret gist in the file: <https://gist.github.com/nnez/6b8c828a6d38b202525bf3c98d532a11>
4. Run `forge t --match-contract FolksNotAbleToLiquidateTest -vv`
5. Observe the gas used to perform liquidation on ALICE's position.
