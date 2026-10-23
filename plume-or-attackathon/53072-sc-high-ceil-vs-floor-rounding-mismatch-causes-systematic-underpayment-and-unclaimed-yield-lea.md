# 53072 sc high ceil vs floor rounding mismatch causes systematic underpayment and unclaimed yield leakage

**Submitted on Aug 14th 2025 at 18:52:35 UTC by @r1ver for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53072
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

In `plume/src/lib/PlumeRewardLogic.sol`, per-user commission uses ceiling division while validator commission accrues with floor division, so user deductions can exceed validator accrual; the excess isn’t credited to anyone, causing systematic underpayment and unclaimed-yield leakage.

### Vulnerability Details

In `plume/src/lib/PlumeRewardLogic.sol`, `updateRewardPerTokenForValidator` calculates validator commission accrual using floor division:

{% code title="PlumeRewardLogic.sol (validator floor example)" %}

```solidity
// Use regular division (floor) for validator's accrued commission
uint256 commissionDeltaForValidator = (
    grossRewardForValidatorThisSegment * commissionRateForSegment
) / PlumeStakingStorage.REWARD_PRECISION;
```

{% endcode %}

But per-user commission deduction in `_calculateRewardsCore` uses ceiling division:

{% code title="PlumeRewardLogic.sol (\_calculateRewardsCore) (user ceil example)" %}

```
```

{% endcode %}

```solidity
// Use ceiling division for commission charged to user to ensure rounding up
uint256 commissionForThisSegment =
    _ceilDiv(grossRewardForSegment * effectiveCommissionRate, PlumeStakingStorage.REWARD_PRECISION);

if (grossRewardForSegment >= commissionForThisSegment) {
    totalUserRewardDelta += (grossRewardForSegment - commissionForThisSegment);
} // else, net reward is 0 for this segment for the user.
// Commission is still generated for the validator based on gross.
// This was previously missing, commission should always be based on gross.
totalCommissionAmountDelta += commissionForThisSegment;
```

Because Solidity lacks decimals and any non-integer division rounds, applying ceil per user per time-segment makes the sum of user-side commissions often exceed the validator’s floor-accrued commission. The difference is neither credited to validators nor returned to users, creating unclaimed-yield leakage and systematic underpayment that grows with more users, more segments, and more rate changes.

### Impact Details

Users are systematically underpaid because per-user ceil commission can exceed validator floor-accrued commission, and the excess is not credited to anyone, causing accumulating unclaimed-yield leakage and breaking the accounting invariant (user\_net + validator\_commission < gross), which undermines auditability.

## Proof of Concept

A test demonstrating the mismatch:

{% code title="plume/test/RoundingMismatchPOC.t.sol" %}

```
```

{% endcode %}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";

contract RoundingMismatchPOC is Test {
    uint256 constant PREC = 1e18;

    // Demonstrates user-side ceil > validator-side floor on the same commission fraction
    function testCeilVsFloorCommissionLeak() public {
        // Example: 1 token staked, rewardPerTokenIncrease = 101 → gross = 101
        uint256 gross = 101;
        uint256 rate = 0.1e18; // 10%

        // Mirrors user-side ceil:
        uint256 userCeil = ceilDiv(gross * rate, PREC);
        // Mirrors validator-side floor:
        uint256 validatorFloor = (gross * rate) / PREC;

        assertEq(userCeil, 11);
        assertEq(validatorFloor, 10);
        assertEq(userCeil - validatorFloor, 1); // leakage
    }

    function ceilDiv(uint256 a, uint256 b) internal pure returns (uint256) {
        return a == 0 ? 0 : (a + b - 1) / b;
    }
}
```

Run it with: `forge test RoundingMismatchPOC -vvv`

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol#L184-L187>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol#L346-L355>

## Notes / Remediation Suggestions (observational)

{% hint style="info" %}

* Ensure the same rounding convention is used for both the per-user deduction and the validator accrual (preferably use floor consistently or accumulate the rounding remainders and attribute them deterministically).
* Alternatively, compute commission at the validator-level from the summed gross amounts before applying rounding, or aggregate per-segment remainder adjustments into a dedicated accounting bucket that is credited to validators or distributed back to users to preserve the invariant.
  {% endhint %}
