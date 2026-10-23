# 52165 sc high user can t claim reward erc20 tokens since rewards transfer will revert

Submitted on Aug 8th 2025 at 12:24:06 UTC by @WinSec for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #52165
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* Impacts:
  * Theft of unclaimed yield

## Description

### Brief/Intro

When users call `claim` it calls `_finalizeRewardClaim` which further calls `_transferRewardFromTreasury` which finally calls `distributeReward` on the `PlumeStakingRewardTreasury` contract. That call can revert due to this check in the treasury:

```solidity
if (!_isRewardToken[token]) {
    revert TokenNotRegistered(token);
}
```

### Vulnerability Details

Reward tokens are added/removed in the RewardFacet contract via `addRewardToken` / removal functions and that facet maintains a mapping:

```solidity
$.isRewardToken[token] = true;
```

When tokens are removed the mapping is set to `false`. However, the treasury contract uses a different mapping (`_isRewardToken`) and enforces the check above before distributing ERC20 rewards. If a token was removed in the RewardFacet but not registered in the treasury's `_isRewardToken` mapping (or vice versa), attempting to claim rewards will revert.

Relevant code paths:

* RewardFacet ultimately calls:

```solidity
function _transferRewardFromTreasury(address token, uint256 amount, address recipient) internal {
    address treasury = getTreasuryAddress();
    if (treasury == address(0)) {
        revert TreasuryNotSet();
    }

    // Make the treasury send the rewards directly to the user
    IPlumeStakingRewardTreasury(treasury).distributeReward(token, amount, recipient);
}
```

* PlumeStakingRewardTreasury's ERC20 distribution branch:

```solidity
} else {
    // ERC20 token distribution
    if (!_isRewardToken[token]) {
        revert TokenNotRegistered(token);
    }

    uint256 balance = IERC20(token).balanceOf(address(this));
    if (balance < amount) {
        revert InsufficientBalance(token, balance, amount);
    }

    // Use SafeERC20 to safely transfer tokens
    SafeERC20.safeTransfer(IERC20(token), recipient, amount);
}
```

Because the treasury checks its own `_isRewardToken` mapping, tokens added/removed only in RewardFacet may not be reflected in the treasury, causing a revert during claim.

The intended design appears to add reward tokens via the RewardFacet; therefore the treasury-side token existence check is unnecessary for ERC20 distribution (this is supported by the fact the native token branch doesn't perform an equivalent check).

### Impact Details

Users won't be able to claim ERC20 reward tokens if the treasury's `_isRewardToken` mapping does not mark the token as registered — resulting in denial of reward claims and potential loss/theft of unclaimed yield.

### References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/PlumeStakingRewardTreasury.sol#L185-L187>

## Proof of Concept

{% stepper %}
{% step %}

### Add reward token in RewardFacet

RewardToken is added in the RewardFacet:

```solidity
$.rewardTokens.push(token);
$.isRewardToken[token] = true;
```

{% endstep %}

{% step %}

### User accrues rewards

A user accrues some rewards over time for that token.
{% endstep %}

{% step %}

### RewardToken removed in RewardFacet

The token is removed in the RewardFacet:

```solidity
$.rewardTokens.pop();
// Update the mapping
$.isRewardToken[token] = false;
```

{% endstep %}

{% step %}

### User calls claim()

User calls `claim()`:

```solidity
if (reward > 0) {
    _finalizeRewardClaim(token, reward, msg.sender);
}
```

This calls:

```solidity
_transferRewardFromTreasury(token, totalAmount, recipient);
```

which calls the treasury:

```solidity
IPlumeStakingRewardTreasury(treasury).distributeReward(token, amount, recipient);
```

The treasury will revert because the token isn't registered in its `_isRewardToken` mapping:

```solidity
if (!_isRewardToken[token]) {
    revert TokenNotRegistered(token);
}
```

{% endstep %}
{% endstepper %}
