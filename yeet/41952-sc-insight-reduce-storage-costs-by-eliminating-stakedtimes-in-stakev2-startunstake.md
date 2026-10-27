# #41952 \[SC-Insight] Reduce storage costs by eliminating stakedTimes in StakeV2::startUnstake

**Submitted on Mar 19th 2025 at 15:17:42 UTC by @Ragnarok for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41952
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**

## Description

## Description

In the `StakeV2::startUnstake` function, `stakedTimes[msg.sender]` is used to limit the number of times a user can initiate an unstake. However, instead of maintaining a separate counter, we can use `vestings[msg.sender].length` to enforce this restriction.

By doing so, we eliminate an unnecessary `SSTORE` operation (used to update `stakedTimes[msg.sender]`), which can help reduce gas costs.

`StakeV2::startUnstake` function:

```solidity
function startUnstake(uint256 unStakeAmount) external {
    require(unStakeAmount > 0, "Amount must be greater than 0");
=>  require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); 
    ...
    vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
=>  stakedTimes[msg.sender]++;
    emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
}
```

## Recommendation

Modify the `StakeV2::startUnstake` function:

```solidity
function startUnstake(uint256 unStakeAmount) external {
    require(unStakeAmount > 0, "Amount must be greater than 0");
=>  require(vestings[msg.sender].length < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); 
    ...
    vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
=>  // Delete this line
    emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
}
```

Modify the `StakeV2::_unstake` function:

```solidity
function _unstake(uint256 index) private {
    Vesting memory vesting = vestings[msg.sender][index];
    (uint256 unlockedAmount, uint256 lockedAmount) = calculateVesting(vesting);
    require(unlockedAmount != 0, "No unlocked amount");

    _remove(msg.sender, index);
    if (lockedAmount > 0) {
        emit RageQuit(msg.sender, unlockedAmount, lockedAmount, index);
    } else {
        emit Unstake(msg.sender, unlockedAmount, index);
    }
=>  stakedTimes[msg.sender]--; // Delete this line
}
```

## Proof of Concept

## Proof of Concept

Modify the `StakeV2::startUnstake` function:

```solidity
function startUnstake(uint256 unStakeAmount) external {
    require(unStakeAmount > 0, "Amount must be greater than 0");
=>  require(vestings[msg.sender].length < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); 
    ...
    vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
=>  // Delete this line
    emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
}
```

Modify the `StakeV2::_unstake` function:

```solidity
function _unstake(uint256 index) private {
    Vesting memory vesting = vestings[msg.sender][index];
    (uint256 unlockedAmount, uint256 lockedAmount) = calculateVesting(vesting);
    require(unlockedAmount != 0, "No unlocked amount");

    _remove(msg.sender, index);
    if (lockedAmount > 0) {
        emit RageQuit(msg.sender, unlockedAmount, lockedAmount, index);
    } else {
        emit Unstake(msg.sender, unlockedAmount, index);
    }
=>  stakedTimes[msg.sender]--; // Delete this line
}
```
