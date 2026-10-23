# 50167 sc high retroactive reward drain via incomplete reward debt reset

**Submitted on Jul 22nd 2025 at 08:24:50 UTC by @BeastBoy for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50167
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

In the full-unstake path, the contract correctly advances every token’s “paid” pointer by iterating over all historical tokens:

```solidity
address[] storage historicalTokens = $.historicalRewardTokens;
for (…) {
    updateRewardsForValidatorAndToken($, user, validatorId, historicalTokens[i]);
}
```

This ensures that when a user fully withdraws, their `userValidatorRewardPerTokenPaid` for *every* token moves to the current cumulative index. By contrast, the re-stake initialization only touches the *active* tokens:

```solidity
address[] memory rewardTokens = $.rewardTokens;
for (…) {
    updateRewardPerTokenForValidator($, token, validatorId);
    $.userValidatorRewardPerTokenPaid[user][validatorId][token] =
        $.validatorRewardPerTokenCumulative[validatorId][token];
    $.userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token] = block.timestamp;
}
```

Any token that was later removed never has its paid pointer updated on re-entry. Its global cumulative index continued to climb (until removal), so when the user finally calls `claim`, the delta between the stale pointer and the frozen cumulative index is paid out—even though the user held zero stake during that period.

## Impact

Users can siphon unearned rewards for removed tokens, draining the reward pool and causing financial loss to other stakers.

## Recommendation

Change the restake initialization to reset pointers over `$.historicalRewardTokens` rather than `$.rewardTokens`.

## Proof of Concept

{% stepper %}
{% step %}

### Setup

Deploy the PlumeStaking diamond including the RewardsFacet and RewardLogic library. Register PLUME as a reward token and add a validator (ID 42). Ensure `$.historicalRewardTokens` and `$.rewardTokens` both contain PLUME.
{% endstep %}

{% step %}

### Initial Stake & Unstake

At time T₀ Alice stakes 1 000 PLUME on validator 42. At time T₁ she fully unstakes. The call path enters:

```solidity
updateRewardsForValidator($, Alice, 42)   // uses $.historicalRewardTokens
```

which advances **every** token’s `userValidatorRewardPerTokenPaid` to `validatorRewardPerTokenCumulative[42][token]` at T₁.
{% endstep %}

{% step %}

### Accrue Past Unstake

Between T₁ and T₂ the protocol continues issuing PLUME rewards to other stakers. The validator’s cumulative index for PLUME climbs from C₁ at T₁ to C₂ at T₂.
{% endstep %}

{% step %}

### Token Removal

At T₂ the admin calls `removeRewardToken(PLUME)`, which emits a final checkpoint at cumulative index C₂ and sets the global rate to zero. Alice’s pointer remains at C₁, because her paid pointer was last updated on unstake.
{% endstep %}

{% step %}

### Re-stake (Pointer Omission)

At T₃ Alice stakes again (any amount). The facet runs `_initializeRewardStateForNewStake`, which loops over `$.rewardTokens` (now excluding PLUME) and resets only active tokens’ pointers. PLUME is omitted, so Alice’s PLUME pointer stays at C₁.
{% endstep %}

{% step %}

### Claim & Drain

Alice invokes `claim(PLUME, 42)`. The code computes:

```solidity
pending = (validatorRewardPerTokenCumulative[42][PLUME] – userValidatorRewardPerTokenPaid[Alice][42][PLUME]) × stake / PRECISION
        = (C₂ – C₁) × 1_000 / 1e18
```

and transfers that amount, even though Alice held zero stake during \[T₁, T₂]. She thereby withdraws unearned PLUME from the pool.
{% endstep %}
{% endstepper %}
