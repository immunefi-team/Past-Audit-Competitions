# 51966 sc low totalamountclaimable reverts instead of returning the claimable reward for historical tokens

Submitted on Aug 6th 2025 at 21:32:59 UTC by @holydevoti0n for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #51966
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `totalAmountClaimable()` function reverts when called for historical reward tokens, even if users still have claimable balances.

### Vulnerability Details

The function `StakingFacet.totalAmountClaimable` checks for `$.isRewardToken`:

```solidity
    /**
     * @notice Get the total amount of a specific token claimable across all users.
     * @param token Address of the token to check.
     * @return amount Total amount of the token claimable.
     */
    function totalAmountClaimable(
        address token
    ) external view returns (uint256 amount) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
            
             // @audit - revert when token is historical reward token
@>        require($.isRewardToken[token], "Token is not a reward token");
        // Return the total claimable amount
        return $.totalClaimableByToken[token];
    }
```

But historical reward tokens were once reward tokens that accrued rewards; they may also have amounts to be claimable.

### Impact Details

It is not possible to see the amount claimable of a specific token across all users when the token to be consulted is a historical reward token.

### Recommended Change

Check against `isHistoricalRewardToken` instead of the current `isRewardToken` mapping to allow access to all tokens that have ever been valid rewards.

Proposed diff:

```diff
    function totalAmountClaimable(
        address token
    ) external view returns (uint256 amount) {
        ...
        // Check if token is a reward token using the mapping
-        require($.isRewardToken[token], "Token is not a reward token");
+        require($.isHistoricalRewardToken[token], "Token is not a reward token");
          ...
    }
```

## Proof of Concept

**Context**

* An active reward token accrues rewards over time. Several users can claim their rewards.
* Admin removes the reward token, but it remains active as a historical reward token, so users can claim it.

{% stepper %}
{% step %}

### Reproduce - step

User calls `totalAmountClaimable` passing the recently removed token as a parameter, expecting to see the total amount to be claimed by all users.
{% endstep %}

{% step %}

### Reproduce - step

`totalAmountClaimable` reverts as it checks for `isRewardToken` instead of `isHistoricalRewardToken`.
{% endstep %}
{% endstepper %}

Result: The function works partially, but reverts when tokens are historical reward tokens.
