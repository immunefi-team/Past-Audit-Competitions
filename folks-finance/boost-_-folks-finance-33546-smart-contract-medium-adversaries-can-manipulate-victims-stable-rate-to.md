# Boost \_ Folks Finance 33546 - \[Smart Contract - Medium] Adversaries can manipulate victims stable ra

Submitted on Tue Jul 23 2024 05:18:04 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33546

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description

Folks finance has two borrow modes: variable rate and stable rate.\
Variable rate is shared amongst all borrowers who does not specify a maxStable rate.\
Stable rate is recorded and locked in for each loan position which is evidence in `UserLoanBorrow` struct.

```solidity
File: /contracts/hub/LoanManagerState.sol
    struct UserLoanBorrow {
        uint256 amount; // excluding interest
        uint256 balance; // including interest
        uint256 lastInterestIndex;
        uint256 stableInterestRate; // defined if stable borrow
        uint256 lastStableUpdateTimestamp; // defined if stable borrow
        uint256 rewardIndex;
    }
```

This explains the need for `rebalanceUp` and `rebalanceDown` functions.\
Once certain conditions are met, loan position with stable rate is eligible for rebalance to adjust the rate up or down to correspond with current utilization and variable rate of the pool.

Read more abount **Rebalancing**:\
<https://docs.google.com/document/d/19HjdYSmSxoXf7b0RIjiv8cff7jwdGZ1lkFrjqRrogiE/edit#heading=h.kav7enwavq9>

A stable rate loan is eligible for `rebalanceUp` when:

* Utilization ratio ≥ given threshold (e.g., 95%).
* Deposit interest rate ≤ given percentage of max variable rate (e.g., 25%).

In a generic lending/borrowing platform, manipulation of utilization rate and variable rate is achievable with large enough capital but it is not effective because the adversary must also pay the incurred interests.

Manipulation via flashloan is also not effective because adversary would have to repay the debt, withdraw collateral then repay flashloan of which would bring everything back to normal at the end of execution flow.

However, an introduction of `rebalanceUp` also introduces a side-effect. Consider this scenario:

* Adversary manipulates the pool's utilization rate and variable rate using flashloan
* Adversary calls `rebalanceUp` for victim's loan position to bring the stable rate up for that loan
* Adversary restore everything back and return the flashloan
* Victim is left with a very high stable rate loan, until someone calls `rebalanceDown`.\
  Adversary can keep doing this at every start of the block and unless the victim can backrun adversary with `rebalanceDown` at the end of the same block, his/her debt will be accrued in the next block.

## Impact

* Victim's debt increases at a very high rate

## Rationale for severity

**Likelihood**: Medium\
The scenario requires borrowers with large position\
**Impact**: High\
It can cause material loss for borrowers with large position.

## Recommended Mitigations

* Rebalance operations should be restricted after `deposit/borrow/repay` in the same transaction.

## Proof of concept

## Proof-of-Concept

A test file included in attached secret gist demonstrates an attack scenario described in the description:

* Adversary manipulates stable rate via flashloan, then call `rebalanceUp` on victim's position.

**Steps**

1. Run `forge init --no-commit --no-git --vscode`.
2. Create a new test file, `FolksInterest.t.sol` in `test` directory.
3. Put the test from secret gist in the file: <https://gist.github.com/nnez/b965c223c745c9db165bc864517f0df3>
4. Run `forge t --match-contract FolksInterest -vv`
5. Observe that EVE gets everthing back to return flashloan, and debt is accrued in ALICE's position at a very high rate
