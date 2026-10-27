# 53051 sc high unconsented stakeonbehalf enables third party gas griefing dos by bloating uservalidators breaking withdraw claimall

**Submitted on Aug 14th 2025 at 18:10:58 UTC by @tansegv for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #53051
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Theft of gas
  * Unbounded gas consumption

## Description

Because anyone can `stakeOnBehalf(staker)` and permanently append distinct validators to a victim’s `userValidators`, and many public flows iterate that array, an attacker can force the victim’s operations (withdrawals, claims, restakes) to consume unbounded gas and revert, freezing access to funds.

`stakeOnBehalf` lets any external account fund a stake for someone else. The first stake with a validator appends that validator ID to the victim’s `$.userValidators`. Core user actions (`withdraw`, `claimAll`, reward calculations, cooldown processing) loop over this entire list. A malicious actor can “spray” the victim across many validators with tiny stakes (≥ `minStake`), inflating loop work until calls OOG/revert, causing a liveness DoS of user funds.

## Vulnerability Details

* Unconsented association & list growth

```solidity
// StakingFacet.sol (L428)
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    ... 
    _performStakeSetup(staker, validatorId, msg.value); // no consent check
}
```

```solidity
// PlumeValidatorLogic.sol (L58-L66)
function addStakerToValidator(..., address staker, uint16 validatorId) internal {
    if (!$.userHasStakedWithValidator[staker][validatorId]) {
        $.userValidators[staker].push(validatorId); // grows victim's list
        $.userHasStakedWithValidator[staker][validatorId] = true;
    }
    ...
}
```

* Hot paths iterate `$.userValidators[user]`
  * Cooldown & withdraw:

```solidity
// StakingFacet.sol (L855)
uint16[] memory userAssociatedValidators = $.userValidators[user];
for (uint256 i = 0; i < userAssociatedValidators.length; i++) { ... }
// called by withdraw() (L395) and restakeRewards() (L451)
```

* Rewards & claims:

```solidity
// RewardsFacet.sol (L537)
uint16[] memory validatorIds = $.userValidators[user];
for (uint256 i = 0; i < validatorIds.length; i++) { ... } // used by claimAll()
```

* Pruning is hard to trigger\
  Removal from `$.userValidators` happens only if no active stake, no active cooldown, and no pending rewards for that validator (`PlumeValidatorLogic.sol` L113–L141). An attacker who keeps tiny stakes outstanding (and never unstakes) effectively pins those entries.
* Asymmetry\
  Attacker cost per validator = `minStakeAmount` (configurable; can be very small). Victim pays ongoing extra gas (O(N)) forever. With enough validators, the loops exceed the block gas limit → revert.

## Impact Details

* Primary impact: Liveness DoS of user funds — the victim cannot successfully:
  * `withdraw()` parked native (after cooldown processing),
  * `claimAll()` rewards,
  * `restakeRewards(...)`,\
    whenever looped processing runs out of gas due to inflated `userValidators`.
* Scope: Per-user targeted; no privileged roles required.
* Persistence: Lasts as long as attacker’s tiny stakes remain ≥ `minStakeAmount`. Victim could try to unwind validator-by-validator, but that’s expensive and time-consuming (cooldowns, per-validator interactions), while the DoS persists during each attempt.
* Economics: If `minStakeAmount` is small, an attacker can cheaply create hundreds of entries. Even with moderate `minStakeAmount`, this is a griefing vector (attacker spends once; victim pays gas tax forever).

## Proof of Concept

{% stepper %}
{% step %}

### Setup

Ensure there are many active validators (as in production). `minStakeAmount` > 0 (as required by code), but keep it small to minimize attacker cost.
{% endstep %}

{% step %}

### Victim pre-state

Alice has normal positions (e.g., some cooled “parked” balance to withdraw or pending rewards to claim).
{% endstep %}

{% step %}

### Attack

Mallory iterates over a large set of validator IDs and calls:

```solidity
stakingFacet.stakeOnBehalf{value: minStakeAmount}(validatorId, alice);
```

This appends each `validatorId` to `$.userValidators[alice]` and leaves a tiny stake outstanding.
{% endstep %}

{% step %}

### Trigger DoS

Alice now tries:

* `stakingFacet.withdraw()` → enters `_processMaturedCooldowns` and loops over all entries in `$.userValidators[alice]`.
* or `rewardsFacet.claimAll()` → `_processAllValidatorRewards` loops through all entries.

With enough spray, gas usage exceeds block limits → revert.
{% endstep %}

{% step %}

### Persistence

Because Mallory’s tiny stakes are still > 0, the cleanup path does not prune entries. Alice’s calls continue to revert until she individually unwinds those stakes across many validators (expensive and slow), or the protocol patches the logic.
{% endstep %}
{% endstepper %}
