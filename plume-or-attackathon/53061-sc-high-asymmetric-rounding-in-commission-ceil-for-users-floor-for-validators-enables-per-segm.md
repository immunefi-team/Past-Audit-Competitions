# 53061 sc high asymmetric rounding in commission ceil for users floor for validators enables per segment rounding loss validators can amplify via frequent commission checkpoints&#x20;

**Submitted on Aug 14th 2025 at 18:31:09 UTC by @tansegv for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53061
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

Users are charged commission with ceiling rounding per time segment while validators accrue commission with floor rounding on aggregated math. By creating many commission checkpoints (more segments), a validator can increase rounding events so that small segments yield zero user net rewards more often, causing economic leakage from users.

The reward/commission pipeline divides time into segments at reward/commission checkpoints. In each segment, user commission is computed with ceil, validator accrual with floor. Increasing the number of segments (by updating commission frequently) increases the number of ceiling events on users, leading to consistent, compounding rounding loss for stakers.

### Vulnerability Details

* User commission uses ceil (bias up):

```solidity
// In _calculateRewardsCore(...) for the user
uint256 grossRewardForSegment =
    (userStakedAmount * rewardPerTokenDeltaForUserInSegment) / REWARD_PRECISION;

uint256 commissionForThisSegment =
    _ceilDiv(grossRewardForSegment * effectiveCommissionRate, REWARD_PRECISION);

if (grossRewardForSegment >= commissionForThisSegment) {
    totalUserRewardDelta += grossRewardForSegment - commissionForThisSegment;
} else {
    // net reward becomes 0 for this segment
}
```

* Validator commission accrual uses floor (bias down vs exact):

```solidity
// In updateRewardPerTokenForValidator(...) for the validator
uint256 grossRewardForValidatorThisSegment =
    (totalStaked * rewardPerTokenIncrease) / REWARD_PRECISION;

uint256 commissionDeltaForValidator =
    (grossRewardForValidatorThisSegment * commissionRateForSegment) / REWARD_PRECISION;
```

* Segmentation multiplies rounding events:\
  `getDistinctTimestamps(...)` merges reward-rate and commission-rate checkpoint times. Each commission update creates another segment. With many short segments, `grossRewardForSegment` tends to small integers; with any nonzero commission rate, `ceil` frequently rounds the per-segment user commission to `1`, making net `0` for that segment.
* Who can drive it:\
  The **validator admin** (not protocol admin) can call `setValidatorCommission(...)` repeatedly (bounded only by `maxCommissionCheckpoints`, min allowed ≥10). Same-block updates are coalesced, but across blocks they can push many checkpoints.

{% hint style="warning" %}
Impact: Economic leakage from users to validators/protocol due to asymmetric rounding and “zeroed” segments. Especially harmful for small stakers or when reward rates are low.
{% endhint %}

### Impact Details

* Impact: Economic leakage from users to validators / protocol (due to asymmetric rounding and “zeroed” segments). Over many periods and many users, this can be material—especially for small stakers or when reward rates are low.
* Exploitability: No privileged roles required beyond a validator admin of their own validator; they can create many commission checkpoints (until hitting the configured cap).
* Bounded but amplifiable: Loss per user is roughly bounded by O(#segments) × 1 unit per relevant segment while reward is small; increasing segment count amplifies loss. Even if validator accrual floors globally, users are still charged via ceil per segment, so users can be systematically shorted on small segments.

## Proof of Concept

{% stepper %}
{% step %}

### Setup

* One validator V with small nonzero commission (e.g., 10%).
* A staker Alice with a small stake so that per-block reward for Alice is tiny (e.g., \~1 wei per short segment).
  {% endstep %}

{% step %}

### Create many commission checkpoints

Each block, the validator admin calls:

```solidity
setValidatorCommission(V, 10%)
```

(repeating each block records a new checkpoint per block). Repeat until near `maxCommissionCheckpoints`.
{% endstep %}

{% step %}

### Accrue rewards

Let time pass so reward accumulates but remains small per segment.
{% endstep %}

{% step %}

### Observe user rounding

* For each segment, `grossRewardForSegment ≈ 1`.
* User commission per segment = `ceil(1 * 10%) = 1`, so net = 0 for that segment.
* After many segments, Alice’s total net is far lower than if the same period were computed as a single segment.
  {% endstep %}

{% step %}

### Contrast scenario (control)

With no extra commission checkpoints, the same elapsed time would be computed as one segment, producing `gross >> 1` and `commission = ceil(gross * 10%)` once, yielding a larger net to Alice.
{% endstep %}
{% endstepper %}
