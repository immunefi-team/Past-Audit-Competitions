# #41570 \[SC-Insight] Code Insights Report

**Submitted on Mar 16th 2025 at 15:51:57 UTC by @pxng0lin for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41570
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol>
* **Impacts:**

## Description

## Table of Contents

1. [Gas Optimization: Duplicate Condition Check in Yeet.sol](broken://pages/AsZyH9cOEpMMDhfr7n14)
2. [Missing Zero Address Check: updateStakingContract in Yeet.sol](broken://pages/AsZyH9cOEpMMDhfr7n14)

***

## 1. Gas Optimization: Duplicate Condition Check in Yeet.sol

**Contract**: `Yeet.sol`\
**Function**: `_yeet()`\
**Severity**: Low (Gas Optimization)

### Description

The function `_yeet()` contains duplicate condition checks for `isBoostrapPhase()`. This function is called with every yeet transaction, so optimizing it can lead to substantial gas savings over time.

### Code Snippet

```solidity
// Current implementation
if (isBoostrapPhase()) { 
    endOfYeetTime = roundStartTime + yeetTimeInSeconds + BOOSTRAP_PHASE_DURATION;
} else {
    endOfYeetTime = timestamp + yeetTimeInSeconds;
}

// Useful for the for stats and history
emit YeetDistribution(msg.value, valueToPot, valueToYeetback, valueToStakers, publicGoods, teamRevenue);
emit Yeet(
    msg.sender,
    timestamp,
    TAX_PER_YEET,
    valueToPot,
    valueToYeetback,
    potToWinner,
    potToYeetback,
    yeetTimeInSeconds,
    _minimumYeetPoint(potToWinner),
    nrOfYeets,
    roundNumber,
    timeLeftOnTimer
);
if (isBoostrapPhase()) {
    uint256 amountOfTickers = msg.value / minimumYeetPoint;
    for (uint256 i = 0; i < amountOfTickers; i++) {
        yeetback.addYeetsInRound(roundNumber, msg.sender);
    }
} else {
    yeetback.addYeetsInRound(roundNumber, msg.sender);
}
```

### Recommendation

Store the result of `isBoostrapPhase()` in a local variable and use it for both condition checks. This reduces the number of external function calls and improves gas efficiency.

```solidity
// Recommended implementation
bool isBootstrap = isBoostrapPhase();
if (isBootstrap) { 
    endOfYeetTime = roundStartTime + yeetTimeInSeconds + BOOSTRAP_PHASE_DURATION;
} else {
    endOfYeetTime = timestamp + yeetTimeInSeconds;
}

// Useful for the for stats and history
emit YeetDistribution(msg.value, valueToPot, valueToYeetback, valueToStakers, publicGoods, teamRevenue);
emit Yeet(
    msg.sender,
    timestamp,
    TAX_PER_YEET,
    valueToPot,
    valueToYeetback,
    potToWinner,
    potToYeetback,
    yeetTimeInSeconds,
    _minimumYeetPoint(potToWinner),
    nrOfYeets,
    roundNumber,
    timeLeftOnTimer
);
if (isBootstrap) {
    uint256 amountOfTickers = msg.value / minimumYeetPoint;
    for (uint256 i = 0; i < amountOfTickers; i++) {
        yeetback.addYeetsInRound(roundNumber, msg.sender);
    }
} else {
    yeetback.addYeetsInRound(roundNumber, msg.sender);
}
```

### Impact

* Reduces gas costs for users interacting with the protocol
* Improves code readability and maintainability
* Makes the code more consistent in its approach to repeated condition checks

***

## 2. Missing Zero Address Check: updateStakingContract in Yeet.sol

**Contract**: `Yeet.sol`\
**Function**: `updateStakingContract()`\
**Severity**: Low (Security Best Practice)

### Description

The function `updateStakingContract()` allows the owner to update the staking contract address but lacks a validation check to prevent setting it to the zero address (address(0)). This is a common security best practice that should be implemented for all functions that update critical addresses.

### Code Snippet

```solidity
/// @notice updateStakingContract allows the owner to update the staking contract, used for new on vaults
function updateStakingContract(StakeV2 _staking) external onlyOwner {
    stakingContract = _staking;
}
```

### Recommendation

Add a zero address check to prevent accidentally setting the staking contract to address(0):

```solidity
/// @notice updateStakingContract allows the owner to update the staking contract, used for new on vaults
function updateStakingContract(StakeV2 _staking) external onlyOwner {
    require(address(_staking) != address(0), "Cannot set to zero address");
    stakingContract = _staking;
}
```

### Impact

* If the owner accidentally sets the staking contract to address(0), contract functionality that depends on the staking contract would fail
* This would require additional transactions to fix, wasting gas

## Proof of Concept

## Proof of Concept

All code provided in the main part of the report - insight only
