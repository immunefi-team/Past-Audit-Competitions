# #41949 \[SC-Insight] Optimize StakeV2::startUnstake with \`unchecked\` block to reduce gas costs

**Submitted on Mar 19th 2025 at 14:50:55 UTC by @Ragnarok for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41949
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**

## Description

## Description

In the `StakeV2::startUnstake` function, the `unStakeAmount` parameter is already validated to ensure it is less than or equal to `balanceOf[msg.sender]`. Therefore, when updating `balanceOf[msg.sender]` and `totalSupply`, we can use an `unchecked` block to save gas.

Additionally, since `block.timestamp` is extremely small compared to `type(uint256).max`, we can also use an `unchecked` block when calculating the `end` time for vesting.

`StakeV2::startUnstake` function:

```solidity
function startUnstake(uint256 unStakeAmount) external {
    ...
    uint256 amount = balanceOf[msg.sender];
=>  require(amount >= unStakeAmount, "Insufficient balance");
    
    balanceOf[msg.sender] -= unStakeAmount;
    totalSupply -= unStakeAmount;

    uint256 start = block.timestamp;
    uint256 end = start + VESTING_PERIOD;
    ...    
}
```

## Recommendation

Modify the `StakeV2::startUnstake` function:

```solidity
function startUnstake(uint256 unStakeAmount) external {
    ...
    uint256 amount = balanceOf[msg.sender];
    require(amount >= unStakeAmount, "Insufficient balance");
    
    unchecked {
        balanceOf[msg.sender] -= unStakeAmount;
        totalSupply -= unStakeAmount;
        uint256 start = block.timestamp;
        uint256 end = start + VESTING_PERIOD;
    }
    ...    
}
```

## Proof of Concept

## Proof of Concept

Modify the `StakeV2::startUnstake` function:

```solidity
function startUnstake(uint256 unStakeAmount) external {
    ...
    uint256 amount = balanceOf[msg.sender];
    require(amount >= unStakeAmount, "Insufficient balance");
    
    unchecked {
        balanceOf[msg.sender] -= unStakeAmount;
        totalSupply -= unStakeAmount;
        uint256 start = block.timestamp;
        uint256 end = start + VESTING_PERIOD;
    }
    ...    
}
```
