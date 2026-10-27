# 52424 sc high there is a retroactive commission miscalculation in plumerewardlogic

Submitted on Aug 10th 2025 at 14:57:27 UTC by @XDZIBECX for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #52424
* Report Type: Smart Contract
* Report severity: High
* Target: attackathon-plume-network/plume/src/lib /PlumeRewardLogic.sol
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

In the library, if a validator commission is updated at time T without any earlier commission checkpoint, the historical reward calculation can apply the new commission to the past time. This happens because the commission lookup falls back to the validator’s current commission when no prior checkpoint exists. As a result, historical segments can be charged the new commission retroactively, causing mispayments: users receive incorrect net rewards and validators accrue incorrect commission amounts. No function is immediately at risk of being reentrancy-exploited; the issue is incorrect accounting.

## Vulnerability Details

{% hint style="danger" %}
When reconstructing history, the commission lookup falls back to the validator's current commission if no checkpoint exists at or before the queried timestamp. After a commission update, this “current” value is already the new commission, so it is applied retroactively to prior time ranges.
{% endhint %}

Relevant code that demonstrates the problematic fallback:

{% code title="PlumeRewardLogic.sol — getEffectiveCommissionRateAt" %}

```
```

{% endcode %}

\`\`\`solidity function getEffectiveCommissionRateAt( PlumeStakingStorage.Layout storage $, uint16 validatorId, uint256 timestamp ) internal view returns (uint256) { PlumeStakingStorage.RateCheckpoint\[] storage checkpoints = $.validatorCommissionCheckpoints\[validatorId]; uint256 chkCount = checkpoints.length;

```
if (chkCount > 0) {
    uint256 idx = findCommissionCheckpointIndexAtOrBefore($, validatorId, timestamp);
    if (idx < chkCount && checkpoints[idx].timestamp <= timestamp) {
        return checkpoints[idx].rate;
    }
}
// Fallback to the current commission rate stored directly in ValidatorInfo
uint256 fallbackComm = $.validators[validatorId].commission;
return fallbackComm;
```

}

````
This fallback is consumed by the segment loop, which uses the commission at each segment’s start:

```solidity
// The Commission rate effective at the START of this segment
uint256 effectiveCommissionRate = getEffectiveCommissionRateAt($, validatorId, segmentStartTime);

// Use ceiling division for commission charged to user to ensure rounding up
uint256 commissionForThisSegment =
    _ceilDiv(grossRewardForSegment * effectiveCommissionRate, PlumeStakingStorage.REWARD_PRECISION);
````

The update path sets the new commission first and then creates a checkpoint at the same timestamp:

```solidity
function setValidatorCommission(
    uint16 validatorId,
    uint256 newCommission
) external onlyValidatorAdmin(validatorId) {
    ...
    PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);
    validator.commission = newCommission;
    PlumeRewardLogic.createCommissionRateCheckpoint($, validatorId, newCommission);
    emit ValidatorCommissionSet(validatorId, oldCommission, newCommission);
}
```

Because the validator’s commission field is updated before creating the checkpoint, any historical lookup that finds no earlier checkpoint will fall back to the updated validator.commission and apply the new rate to prior segments.

### Impact Details

* Contract fails to deliver promised returns.
* Consequences:
  * Users can be overcharged (receive less net reward) if a commission increase applies retroactively, or undercharged if commission decreased.
  * Validators’ accrued commission can diverge from intended amounts, causing accounting mismatches between what users were charged and validator accruals.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L608C1-L625C1>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L339C4-L350C18>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L317C1-L353C1>

## Proof of Concept

<details>

<summary>Solidity test cases demonstrating the miscalculation (expand to view)</summary>

```solidity
function test_SameTimestampRewardAndCommissionChange_SimultaneousBoundary() public {
    RewardsFacet rewardsFacet = RewardsFacet(address(diamondProxy));
    ValidatorFacet validatorFacet = ValidatorFacet(address(diamondProxy));
    StakingFacet stakingFacet = StakingFacet(address(diamondProxy));

    uint16 validatorId = DEFAULT_VALIDATOR_ID;
    address token = address(pUSD);

    uint256 rate1 = 2e14;
    uint256 rate2 = 5e14;
    uint256 commission1 = DEFAULT_COMMISSION;   // 5%
    uint256 commission2 = 20e16;                // 20%
    uint256 stakeAmount = 100e18;

    // Set initial rate1 as REWARD_MANAGER (admin)
    vm.startPrank(admin);
    {
        address[] memory tokens = new address[](1);
        tokens[0] = token;
        uint256[] memory rates = new uint256[](1);
        rates[0] = rate1;
        rewardsFacet.setRewardRates(tokens, rates);
    }
    vm.stopPrank();
    
    // User stakes
    vm.prank(user1);
    stakingFacet.stake{ value: stakeAmount }(validatorId);

    // Accrue at (rate1, commission1)
    uint256 t0 = block.timestamp;
    vm.warp(t0 + 100);

    // Same-timestamp changes: set reward rate (admin) then commission (validatorAdmin)
    vm.startPrank(admin);
    {
        address[] memory tokens2 = new address[](1);
        tokens2[0] = token;
        uint256[] memory rates2 = new uint256[](1);
        rates2[0] = rate2;
        rewardsFacet.setRewardRates(tokens2, rates2);
    }
    vm.stopPrank();
    
    vm.prank(validatorAdmin);
    validatorFacet.setValidatorCommission(validatorId, commission2);

    // Accrue at (rate2, commission2)
    uint256 tChange = block.timestamp;
    vm.warp(tChange + 200);

    // Expected
    uint256 gross1 = (100 * rate1 * stakeAmount) / 1e18; // 2e18
    uint256 gross2 = (200 * rate2 * stakeAmount) / 1e18; // 10e18
    uint256 comm1 = (gross1 * commission1 + 1e18 - 1) / 1e18; // ceilDiv
    uint256 comm2 = (gross2 * commission2 + 1e18 - 1) / 1e18; // ceilDiv
    uint256 expectedNet = (gross1 - comm1) + (gross2 - comm2); // 9.9e18

    uint256 actual = rewardsFacet.earned(user1, token);
    assertApproxEqAbs(actual, expectedNet, 1, "Same-timestamp boundary must apply both new rates from T");
}

function test_SameTimestampRewardAndCommissionChange_OrderIrrelevant() public {
    RewardsFacet rewardsFacet = RewardsFacet(address(diamondProxy));
    ValidatorFacet validatorFacet = ValidatorFacet(address(diamondProxy));
    StakingFacet stakingFacet = StakingFacet(address(diamondProxy));

    uint16 validatorId = DEFAULT_VALIDATOR_ID;
    address token = address(pUSD);

    uint256 rate1 = 2e14;
    uint256 rate2 = 5e14;
    uint256 commission1 = DEFAULT_COMMISSION;   // 5%
    uint256 commission2 = 20e16;                // 20%
    uint256 stakeAmount = 100e18;

    vm.startPrank(admin);
    {
        address[] memory tokens = new address[](1);
        tokens[0] = token;
        uint256[] memory rates = new uint256[](1);
        rates[0] = rate1;
        rewardsFacet.setRewardRates(tokens, rates);
    }
    vm.stopPrank();
    
    vm.prank(user1);
    stakingFacet.stake{ value: stakeAmount }(validatorId);

    uint256 t0 = block.timestamp;
    vm.warp(t0 + 100);

    // Flip order but keep same timestamp
    vm.prank(validatorAdmin);
    validatorFacet.setValidatorCommission(validatorId, commission2);

    vm.startPrank(admin);
    {
        address[] memory tokens2 = new address[](1);
        tokens2[0] = token;
        uint256[] memory rates2 = new uint256[](1);
        rates2[0] = rate2;
        rewardsFacet.setRewardRates(tokens2, rates2);
    }
    vm.stopPrank();
    
    uint256 tChange = block.timestamp;
    vm.warp(tChange + 200);

    uint256 gross1 = (100 * rate1 * stakeAmount) / 1e18; // 2e18
    uint256 gross2 = (200 * rate2 * stakeAmount) / 1e18; // 10e18
    uint256 comm1 = (gross1 * commission1 + 1e18 - 1) / 1e18;
    uint256 comm2 = (gross2 * commission2 + 1e18 - 1) / 1e18;
    uint256 expectedNet = (gross1 - comm1) + (gross2 - comm2); // 9.9e18

    uint256 actual = rewardsFacet.earned(user1, token);
    assertApproxEqAbs(actual, expectedNet, 1, "Order within same timestamp should not change accrual");
} 
```

</details>

## Notes for Remediation (non-exhaustive / hints)

* Ensure that historical lookups never fall back to the current validator.commission for timestamps earlier than the time that current value became effective.
* Possible fixes:
  * When creating a new commission checkpoint, create it before updating validator.commission, or
  * Modify getEffectiveCommissionRateAt to treat the absence of an earlier checkpoint as "no commission available" (e.g., revert or use an explicit genesis/default checkpoint), or
  * During commission updates, ensure the chronological ordering of state changes and checkpoint writes guarantees that historical lookups for earlier timestamps will not observe the updated commission.
* Tests should cover same-timestamp changes (both orders) and verify accruals match expected segment-by-segment accounting.
