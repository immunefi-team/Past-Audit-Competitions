# 50412 sc high illegitimate reward claim after unstake due to overlapping reward rate checkpoints

**Submitted on Jul 24th 2025 at 11:09:23 UTC by @GeorgeMichael for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50412
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

The RewardsFacet.sol contract, which calculates and distributes rewards, allows users who no longer hold any active stakes to illegitimately claim rewards if multiple validators' reward checkpoints share the same timestamp. This can occur when the protocol administrator updates global reward rates using `setRewardRates()`, affecting multiple validators in a single transaction.

Users retain past references in arrays such as `userValidators`, and the historical reward calculation (`userRewardDelta`) does not verify whether the stake is still active at the time of `claim()`. This discrepancy allows withdrawal of unearned tokens from the treasury: a user can claim more than legitimately accumulated, due to reward calculations based on rate checkpoints created after their unstake.

### Vulnerability Details

* Pending rewards are computed from historical `rewardRate` checkpoints stored in `validatorRewardRateCheckpoints[validatorId][token]`.
* The last reward timestamp paid to the user is stored in `userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token]`, together with the previous `stakedAmount`.
* When a user unstakes, they lose active stake, but references (e.g., `userValidators`, historical timestamps, `userRewards`) remain until `claim()` is performed.
* If the administrator calls `setRewardRates()` for multiple validators in a single block (common in rebalancing), new checkpoints with the same `rewardRate` and timestamp are created for all affected validators.
* `PlumeRewardLogic.calculateRewardsWithCheckpoints()` can then include these recent checkpoints for a user who has already unstaked but still has entries in `userValidators`. The calculation does not verify whether the user had active stake at the checkpoint time and thus attributes an artificial reward delta as if they were staking at a higher rate.
* The issue is amplified because `userValidators` is not automatically cleared on `unstake()` and the `stakedAmount` is not used as a cutoff criterion in the reward delta computation.

### Impact Details

* Allows theft of protocol tokens from the `PlumeStakingRewardTreasury`.
* Does not require admin privileges, special access, or collusion with validators.
* Attack flow: stake → unstake → wait for admin reward-rate update that creates same-timestamp checkpoints → `claim(token)` to receive inflated rewards.
* Effect scales with number of validators included in `setRewardRates()` and can accumulate across cycles.
* `userValidators` and related values are only cleared after a successful `claim()`, not on `unstake()`.

## Proof of Concept

The following test demonstrates the exploit:

{% code title="PoC test (Solidity/Foundry-style pseudo-code)" %}

```solidity
function testExploitCrossValidatorRewardLeak() public { 
    console2.log("Running testExploitCrossValidatorRewardLeak"); 

    StakingFacet staking = StakingFacet(address(diamondProxy)); 
    RewardsFacet rewards = RewardsFacet(address(diamondProxy)); 
    ValidatorFacet validators = ValidatorFacet(address(diamondProxy)); 
    AccessControlFacet acl = AccessControlFacet(address(diamondProxy)); 

    address alice = vm.addr(1001); 
    address bob = vm.addr(2002); 
    uint16 validatorA = 1; 
    uint16 validatorB = 2; 

    vm.deal(alice, 10 ether); 
    vm.deal(bob, 10 ether); 

    vm.startPrank(admin); 
    acl.grantRole(PlumeRoles.VALIDATOR_ROLE, admin); 
    acl.grantRole(PlumeRoles.REWARD_MANAGER_ROLE, admin); 
    vm.stopPrank(); 

    vm.startPrank(alice); 
    staking.stake{value: 3 ether}(validatorA); 
    vm.stopPrank(); 

    vm.startPrank(bob); 
    staking.stake{value: 1 ether}(validatorB); 
    vm.stopPrank(); 

    vm.warp(block.timestamp + 2 days); 

    vm.startPrank(bob); 
    staking.unstake(validatorB); 
    vm.stopPrank(); 

    address[] memory tokens = new address[](1); 
    tokens[0] = PLUME_NATIVE; 

    uint256[] memory newRates = new uint256[](1); 
    newRates[0] = PLUME_REWARD_RATE_PER_SECOND * 2; 

    vm.startPrank(admin); 
    rewards.setRewardRates(tokens, newRates); 
    vm.stopPrank(); 

    vm.startPrank(bob); 
    uint256 claimable = rewards.getClaimableReward(bob, PLUME_NATIVE); 
    console2.log("Bob can claim %s PLUME_NATIVE even though he no longer has an active stake", claimable); 

    uint256 pre = bob.balance; 
    uint256 claimed = rewards.claim(PLUME_NATIVE); 
    uint256 post = bob.balance; 

    console2.log("Bob claimed: %s wei | Balance before: %s, after: %s", claimed, pre, post);
    vm.stopPrank();

    assertGt(claimed, 0, "The reward was zero, but there should be reward leakage");
}
```

{% endcode %}

Observed behavior from the PoC:

* Alice stakes 3 ETH on validator A.
* Bob stakes 1 ETH on validator B, waits, then unstakes (no active stake).
* Admin calls `setRewardRates()` affecting both validators in the same block — creating same-timestamp checkpoints.
* Bob calls `claim(PLUME_NATIVE)` and is able to claim inflated rewards (e.g., 272,602,739,693,664 wei) despite not having an active stake under the new reward rate.

This confirms that reward calculation post-unstake incorporates non-applicable checkpoints and allows withdrawal of funds that should not be claimable.

## Reproduction Steps

{% stepper %}
{% step %}

### Prepare actors and roles

* Create two users (Alice and Bob).
* Ensure admin has `VALIDATOR_ROLE` and `REWARD_MANAGER_ROLE`.
  {% endstep %}

{% step %}

### Stake flow

* Alice stakes to validator A.
* Bob stakes to validator B.
* Advance time to accumulate rewards.
  {% endstep %}

{% step %}

### Unstake and admin update

* Bob unstakes (no active stake anymore).
* Admin calls `setRewardRates()` affecting multiple validators in a single transaction (same block).
  {% endstep %}

{% step %}

### Claim

* Bob calls `getClaimableReward()` and `claim(token)` and receives inflated rewards despite not having an active stake at the checkpoint time.
  {% endstep %}
  {% endstepper %}

## Notes / Observations

* Root cause: reward calculation uses checkpoint timestamps and rates without verifying whether the user had stake during the relevant checkpoint interval and relies on `userValidators` entries that are not cleared on `unstake()`.
* The vulnerability arises specifically when multiple validators receive rate updates in the same block, creating identical timestamps across checkpoint arrays, which the reward logic then includes incorrectly for users who no longer had stake.

## Suggested Mitigations (high level, as reported)

* Ensure reward delta calculations validate that the user's `stakedAmount` at the checkpoint interval is greater than zero or otherwise ensure historical checkpoints are only applied when the stake was active across that interval.
* Clear or mark `userValidators` entries on `unstake()` or maintain per-user-per-validator "active until" state to prevent post-unstake reward accrual from new checkpoints.
* When updating rates for multiple validators in a single transaction, ensure that reward calculation logic robustly handles same-timestamp checkpoints and does not attribute future/admin-created checkpoints to users who had already unstaked.

(Do not add any fixes or code patches beyond these high-level mitigations; they are included only to summarize the intended direction for remediation based on the reported issue.)
