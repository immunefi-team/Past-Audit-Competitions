# #37427 \[SC-Critical] Delegation is not updated on slash and unlock

**Submitted on Dec 4th 2024 at 14:44:39 UTC by @shadowHunter for** [**Audit Comp | Celo**](https://immunefi.com/audit-competition/audit-comp-celo)

* **Report ID:** #37427
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/celo-org/celo-monorepo/blob/release/core-contracts/12/packages/protocol/contracts/governance/LockedGold.sol>
* **Impacts:**
  * Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

It was observed that delegation is not updated on slashing for reporter and caller in unlock operation. This causes incorrect balance accounting

## Vulnerability Details

1. Observe the 'slash' function and 'unlock\` function
2. Both of this function calls `_incrementNonvotingAccountBalance` and `_decrementNonvotingAccountBalance` which changes the non voting balance and thus `AccountTotalLockedGold`

```
function getAccountTotalLockedGold(address account) public view returns (uint256) {
    uint256 total = balances[account].nonvoting;
    return total.add(getElection().getTotalVotesByAccount(account));
  }
```

3. Lets say User A is unlocking or calling slash on another User B then it is always required to call `_updateDelegatedAmount` on both Users so that delegated balance gets updated correctly as per new total locked
4. But the same is missing in both functions

## Impact Details

Delegated balance power will be incorrect

## References

In `unlock` call `_updateDelegatedAmount(msg.sender);` In `slash` call `_updateDelegatedAmount(reporter);`

## Proof of Concept

## Proof of Concept

```
function test_ShouldReduceAccountsLockedGoldBalance_WhenAccountIsSlashedForAllOfItsLockedGoldAndIsDelegating()
    public
  {
    uint256 penalty = value;
    uint256 reward = value / 2;
    whenVoteSigner_LockedGoldDelegateGovernanceVotes();
    vm.prank(caller);
    lockedGold.delegateGovernanceVotes(delegatee1, FixidityLib.newFixedFraction(30, 100).unwrap());
    assertEq(lockedGold.getAccountNonvotingLockedGold(caller), 1000);
    assertEq(lockedGold.getAccountTotalLockedGold(caller), 1000);
    assertEq(lockedGold.getAccountTotalGovernanceVotingPower(delegatee1), 300);
    helper_WhenAccountIsSlashedForAllOfItsLockedGold(penalty, reward, delegator);
    assertEq(lockedGold.getAccountNonvotingLockedGold(delegator), 0);
    assertEq(lockedGold.getAccountTotalLockedGold(delegator), 0);
    assertNotEq(lockedGold.getAccountTotalGovernanceVotingPower(delegatee1), 300);
  }
```
