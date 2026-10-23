# 49671 sc insight wrong emission in stake

Submitted on Jul 18th 2025 at 03:42:01 UTC by @holydevoti0n for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #49671
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* Impacts:

## Description

### Summary

Wrong information when emitting `Staked` events.

### Vulnerability Details

The `Staked` event is defined as follows, where the last argument is supposed to be the `pendingRewards` used for the staking.

```solidity
/**
 * @notice Emitted when tokens are staked in the contract
 * @param staker Address of the staker
 * @param validatorId ID of the validator receiving the stake
 * @param amount Amount of tokens staked
 * @param fromCooled Amount of tokens used from cooled tokens
 * @param fromParked Amount of tokens used from parked (withdrawn) tokens
 * @param pendingRewards Amount of tokens staked from pending rewards
 */
event Staked(
    address indexed staker,
    uint16 indexed validatorId,
    uint256 amount,
    uint256 fromCooled,
    uint256 fromParked,
    uint256 pendingRewards
);
```

However, when emitting the events throughout `StakingFacet.sol`, the last argument does not always represent `pendingRewards`. For example:

Example 1:

```solidity
function stake(
    uint16 validatorId
) external payable returns (uint256) {
    uint256 stakeAmount = msg.value;

    // Perform all common staking setup
    bool isNewStake = _performStakeSetup(msg.sender, validatorId, stakeAmount);

    // Emit stake event
    emit Staked(msg.sender, validatorId, stakeAmount, 0, 0, stakeAmount);

    return stakeAmount;
}
```

Example 2:

```solidity
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    if (staker == address(0)) {
        revert ZeroRecipientAddress();
    }

    uint256 stakeAmount = msg.value;

    // Perform all common staking setup for the beneficiary
    bool isNewStake = _performStakeSetup(staker, validatorId, stakeAmount);

    // Emit events
    emit Staked(staker, validatorId, stakeAmount, 0, 0, stakeAmount);
    emit StakedOnBehalf(msg.sender, staker, validatorId, stakeAmount);

    return stakeAmount;
}
```

In the examples above, the staking didn't actually use any pending rewards — the amount comes from the transaction value (`msg.value`) — but the event is emitted with `pendingRewards` equal to the staked amount.

### Impact

Off-chain consumers reading the `Staked` event may believe the stake used pending rewards, while it was actually funded via the transaction value. This can cause incorrect reporting in dashboards and trackers.

## Recommendation

Pass the correct `pendingRewards` value when emitting `Staked` in functions that do not consume pending rewards. For the two examples above, replace the last argument with `0`:

Suggested change for `stake`:

```diff
function stake(
    uint16 validatorId
) external payable returns (uint256) {
    uint256 stakeAmount = msg.value;

    // Perform all common staking setup
    bool isNewStake = _performStakeSetup(msg.sender, validatorId, stakeAmount);

    // Emit stake event
-   emit Staked(msg.sender, validatorId, stakeAmount, 0, 0, stakeAmount);
+   emit Staked(msg.sender, validatorId, stakeAmount, 0, 0, 0);

    return stakeAmount;
}
```

Suggested change for `stakeOnBehalf`:

```diff
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    if (staker == address(0)) {
        revert ZeroRecipientAddress();
    }

    uint256 stakeAmount = msg.value;

    // Perform all common staking setup for the beneficiary
    bool isNewStake = _performStakeSetup(staker, validatorId, stakeAmount);

    // Emit events
-   emit Staked(staker, validatorId, stakeAmount, 0, 0, stakeAmount);
+   emit Staked(staker, validatorId, stakeAmount, 0, 0, 0);
    emit StakedOnBehalf(msg.sender, staker, validatorId, stakeAmount);

    return stakeAmount;
}
```

## Proof of Concept

1. User stakes 1 ETH:
   * contract.stake{value: 1}(validatorId);
2. `stake` processes the stake and emits:
   * emit Staked(staker, validatorId, stakeAmount, 0, 0, stakeAmount);
3. Off-chain listeners will report that pending rewards were used for staking, while the stake was funded by the transaction value.
