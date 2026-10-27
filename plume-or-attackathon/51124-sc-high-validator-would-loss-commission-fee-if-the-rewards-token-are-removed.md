# 51124 sc high validator would loss commission fee if the rewards token are removed

**Submitted on Jul 31st 2025 at 10:45:48 UTC by @pks271 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51124
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

### Bug Description

When `stake/unstake/claim` including `RewardsFacet#removeRewardToken` actions are called, the `$.validatorAccruedCommission[validatorId][token] += commissionDeltaForValidator` inside `PlumeRewardLogic#updateRewardPerTokenForValidator()` function will be updated. This parameter tracks the history of validators' commission fees and validators can call `ValidatorFacet#requestCommissionClaim/finalizeCommissionClaim` to claim their commission fee.

When `REWARD_MANAGER_ROLE` calls `RewardsFacet#removeRewardToken` to remove the reward token, the `$.validatorAccruedCommission[validatorId][token]` is updated first, then `$.isRewardToken[token]` is set to `false`. The problem is that `requestCommissionClaim` will revert because it uses `_validateIsToken` to check whether the reward token exists; if not, it reverts.

Relevant code excerpts:

```solidity
function removeRewardToken(
        address token
) external onlyRole(PlumeRoles.REWARD_MANAGER_ROLE) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    if (!$.isRewardToken[token]) {
        revert TokenDoesNotExist(token);
    }

    // Find the index of the token in the array
    uint256 tokenIndex = _getTokenIndex(token);

    // Store removal timestamp to prevent future accrual
    uint256 removalTimestamp = block.timestamp;
    $.tokenRemovalTimestamps[token] = removalTimestamp;

    // Update validators (bounded by number of validators, not users)
    // @audit - update the reward per token for all validators
    for (uint256 i = 0; i < $.validatorIds.length; i++) {
        uint16 validatorId = $.validatorIds[i];

        // Final update to current time to settle all rewards up to this point
        PlumeRewardLogic.updateRewardPerTokenForValidator($, token, validatorId);

        // Create a final checkpoint with a rate of 0 to stop further accrual definitively.
        PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, 0);
    }

    // Set rate to 0 to prevent future accrual. This is now redundant but harmless.
    $.rewardRates[token] = 0;
    // DO NOT delete global checkpoints. Historical data is needed for claims.
    // delete $.rewardRateCheckpoints[token];

    // Update the array
    $.rewardTokens[tokenIndex] = $.rewardTokens[$.rewardTokens.length - 1];
    $.rewardTokens.pop();
    // Update the mapping
    // @audit - set `$.isRewardToken[token]` to false
    $.isRewardToken[token] = false;

    delete $.maxRewardRates[token];
    emit RewardTokenRemoved(token);
}
```

The `_validateIsToken` modifier currently reverts when `isRewardToken[token]` is false:

```solidity
modifier _validateIsToken(
        address token
) {
    if (!PlumeStakingStorage.layout().isRewardToken[token]) {
        revert TokenDoesNotExist(token);
    }
    _;
}
```

`requestCommissionClaim` uses this modifier, which causes it to revert for tokens that were valid reward tokens but were later removed after commission accrual:

```solidity
function requestCommissionClaim(
        uint16 validatorId,
        address token
    )
    external
    onlyValidatorAdmin(validatorId)
    nonReentrant
    _validateValidatorExists(validatorId)
    // @audit - revert here
    _validateIsToken(token)
    {
        // skip
    }
```

## Impact

* Validators who accrued commission for a token that is later removed will be unable to request/finalize claims for that accrued commission because `_validateIsToken` will revert if the token has been removed.
* This can temporarily freeze validator funds (commission) for at least 24 hours (or until the token is re-added or the contract is changed).

## Recommendation

Consider adding a historical commission check to the token validation so that tokens that are no longer active but have accrued commission can still be claimed. For example:

```solidity
modifier _validateIsToken(address token, uint16 validatorId) {
    if (!$.isRewardToken[token]) {
        bool hasAccruedCommission = false;
        if ($.validatorAccruedCommission[validatorId][token] > 0) {
            hasAccruedCommission = true;
        }
        if (!hasAccruedCommission) {
            revert TokenNotRegistered(token);
        }
    }
}
```

This approach allows validators to claim previously accrued commission for tokens that have been removed while still preventing operations for tokens that were never registered.

## Proof of Concept

Insert the following test case into `PlumeStakingDiamond.t.sol`:

```solidity
function testCommissionLostAfterRewardTokenRemoval() public {
    console2.log("--- Test: testCommissionLostAfterRewardTokenRemoval START ---");
    
    // Setup: Add reward token and set reward rate
    uint256 rewardRate = 1e18; // 1 PUSD per second
    vm.startPrank(admin);
    address[] memory tokens = new address[](1);
    tokens[0] = address(pUSD);
    uint256[] memory rates = new uint256[](1);
    rates[0] = rewardRate;
    RewardsFacet(address(diamondProxy)).setRewardRates(tokens, rates);
    RewardsFacet(address(diamondProxy)).setTreasury(address(treasury));
    
    // Ensure treasury has enough PUSD
    uint256 treasuryAmount = 1000 ether;
    pUSD.transfer(address(treasury), treasuryAmount);
    vm.stopPrank();

    // Setup validator with commission
    vm.startPrank(validatorAdmin);
    uint256 commissionRate = 10e16; // 10%
    ValidatorFacet(address(diamondProxy)).setValidatorCommission(
        DEFAULT_VALIDATOR_ID,
        commissionRate
    );
    vm.stopPrank();

    // User stakes to generate rewards and commission
    uint256 stakeAmount = 10 ether;
    vm.startPrank(user1);
    StakingFacet(address(diamondProxy)).stake{value: stakeAmount}(
        DEFAULT_VALIDATOR_ID
    );
    vm.stopPrank();

    // Move time forward to accrue rewards and commission
    vm.roll(block.number + 100);
    vm.warp(block.timestamp + 100);

    // Trigger reward updates to ensure commission is accrued
    vm.startPrank(user1);
    StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, 1); // Unstake 1 wei to trigger update
    vm.stopPrank();

    // Verify commission has accrued
    uint256 accruedCommission = ValidatorFacet(address(diamondProxy))
        .getAccruedCommission(DEFAULT_VALIDATOR_ID, address(pUSD));
    assertEq(accruedCommission, 100*1e18, "Commission should have accrued");

    // Now remove the reward token
    vm.startPrank(admin);
    RewardsFacet(address(diamondProxy)).removeRewardToken(address(pUSD));
    vm.stopPrank();

    // Verify token is no longer a reward token
    bool isRewardToken = RewardsFacet(address(diamondProxy)).isRewardToken(address(pUSD));
    assertEq(isRewardToken, false, "Token should no longer be a reward token");

    // Try to request commission claim - this should fail due to _validateIsToken check
    vm.startPrank(validatorAdmin);
    
    // This should revert because _validateIsToken will check isRewardToken[token] which is now false
    vm.expectRevert(); // Expect revert due to TokenNotRegistered error
    ValidatorFacet(address(diamondProxy)).requestCommissionClaim(
        DEFAULT_VALIDATOR_ID,
        address(pUSD)
    );
    vm.stopPrank();
}
```
