# 53028 sc high there is an asymmetric rounding issue that is can cause a theft of unclaimed yield in reward or commission accounting

**Submitted on Aug 14th 2025 at 17:37:33 UTC by @XDZIBECX for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53028
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:** Theft of unclaimed yield

***

## Description / Brief

There is a mismatch in commission accounting: the contract charges each user’s commission using ceiling rounding, while the validator’s commission is accrued using floor rounding at the aggregate level. The difference between these two roundings is not credited to anyone and becomes an untracked remainder that accumulates in the treasury across accrual segments. Over time, this causes users to be slightly underpaid and validators to receive slightly less than users were charged.

In other words:

* Per-user commission uses ceiling (ceil) rounding when reducing the user net.
* Validator’s aggregate commission for the same segment uses floor rounding.
* The gap between ceil(per-user sums) and floor(aggregate) is not accounted for and accumulates as an unclaimed surplus.

***

## Vulnerability Details

Relevant code excerpts showing asymmetric rounding:

* Per-user commission uses ceiling in \_calculateRewardsCore:

```solidity
uint256 grossRewardForSegment =
    (userStakedAmount * rewardPerTokenDeltaForUserInSegment) / PlumeStakingStorage.REWARD_PRECISION;

uint256 effectiveCommissionRate = getEffectiveCommissionRateAt($, validatorId, segmentStartTime);

// Use ceiling division for commission charged to user to ensure rounding up
uint256 commissionForThisSegment =
    _ceilDiv(grossRewardForSegment * effectiveCommissionRate, PlumeStakingStorage.REWARD_PRECISION);

if (grossRewardForSegment >= commissionForThisSegment) {
    totalUserRewardDelta += (grossRewardForSegment - commissionForThisSegment);
}
totalCommissionAmountDelta += commissionForThisSegment;
```

* Validator commission accrues per-segment using floor in updateRewardPerTokenForValidator:

```solidity
uint256 commissionDeltaForValidator = (
    grossRewardForValidatorThisSegment * commissionRateForSegment
) / PlumeStakingStorage.REWARD_PRECISION;

if (commissionDeltaForValidator > 0) {
    $.validatorAccruedCommission[validatorId][token] += commissionDeltaForValidator;
}
```

* User net amounts (already reduced by ceil commission) are added to the claimable pool in updateRewardsForValidatorAndToken:

```solidity
if (userRewardDelta > 0) {
    $.userRewards[user][validatorId][token] += userRewardDelta;
    $.totalClaimableByToken[token] += userRewardDelta;
    $.userHasPendingRewards[user][validatorId] = true;
}
```

Example demonstrating the issue:

* Two users, 1-second accrual, tiny rate such that each user’s gross reward = 1 wei.
* Each user’s commission is ceil-rounded and can reduce net to 0.
* Validator’s aggregate commission uses floor rounding; for small totals the floored aggregate may be less than the sum of per-user ceil charges.
* The difference remains uncredited and accumulates in the treasury.

Other affected areas:

* RewardsFacet.claim(address,uint16) and RewardsFacet.claim(address) pay users based on storage that already reflects the ceil reduction — so users receive too little.
* ValidatorFacet.getAccruedCommission exposes only the floored aggregate — validators receive too little as well.

Root cause: asymmetric rounding (per-user ceil vs aggregate floor) with no reconciliation of the leftover.

***

## Impact Details

Systematic under-distribution of rewards:

* Users’ payouts are reduced by ceil-rounded commissions.
* Validators receive floor-rounded aggregate commissions.
* The leftover (ceil\_sum - floor\_aggregate) is never paid out or tracked and accumulates in the treasury.
* Over many segments, validators, tokens, and many users, this can become material and lead to persistent losses.

***

## Reference

* Code reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L184C1-L197C6>

***

## Proof of Concept

Add this test to `PlumeStakingDiamond.t.sol` and run: forge test --match-test test\_CommissionRoundingSurplusAccumulates\_AllPaths -vvvv --via-ir

```solidity
function test_CommissionRoundingSurplusAccumulates_AllPaths() public {
    // Facets
    StakingFacet staking = StakingFacet(address(diamondProxy));
    RewardsFacet rewards = RewardsFacet(address(diamondProxy));
    ValidatorFacet validator = ValidatorFacet(address(diamondProxy));

    // Setup: add a mock reward token at rate=1 so gross per user per 1s = 1 wei
    vm.startPrank(admin);
    MockPUSD rwd = new MockPUSD();
    rwd.transfer(address(treasury), 1e24);
    treasury.addRewardToken(address(rwd));
    rewards.addRewardToken(address(rwd), 1, 1e18); // initialRate=1, max=1e18
    vm.stopPrank();

    // Use validator 1 created in setUp (commission > 0 already set there)
    uint16 vid = 1;

    // Two users stake 1e18 each
    address u1 = makeAddr("round_user1");
    address u2 = makeAddr("round_user2");
    vm.deal(u1, 10 ether);
    vm.deal(u2, 10 ether);

    vm.prank(u1);
    staking.stake{value: 1e18}(vid);
    vm.prank(u2);
    staking.stake{value: 1e18}(vid);

    // Accrue exactly 1 second
    uint256 t0 = block.timestamp;
    vm.warp(t0 + 1);

    // Trigger settlement for both users via both claim functions
    // - u1 uses validator-specific claim
    vm.prank(u1);
    rewards.claim(address(rwd), vid);

    // - u2 uses token-wide claim (covers the other claim signature)
    vm.prank(u2);
    rewards.claim(address(rwd));

    // Read earned (should be ~0 each due to per-user ceil(commission) on gross=1)
    uint256 earnedU1 = rewards.earned(u1, address(rwd));
    uint256 earnedU2 = rewards.earned(u2, address(rwd));

    // Validator accrued commission (aggregate, floored)
    uint256 accrued = validator.getAccruedCommission(vid, address(rwd));

    // Expected gross for the 1-second segment: (2e18 * 1) / 1e18 = 2
    uint256 expectedGross = 2;

    // Accounted amounts the system will ever pay out:
    // users' net (already reduced by ceil) + validator accrued (floored)
    uint256 accounted = earnedU1 + earnedU2 + accrued;

    // Evidence of mismatch: accounted < expectedGross due to asymmetric rounding
    // (ceil per user) vs (floor aggregate).
    assertEq(earnedU1, 0, "user1 net should be zero after ceil commission at gross=1");
    assertEq(earnedU2, 0, "user2 net should be zero after ceil commission at gross=1");
    // accrued can be 0 or 1 depending on the commission configured in setUp; either way, it will be < 2
    assertLt(accounted, expectedGross, "under-distribution due to rounding should be observable");
}
```

***

## Suggested Mitigations (not exhaustive)

* Use consistent rounding strategy for commission accounting:
  * Option A: Use floor rounding consistently both for per-user and aggregate accounting, and reconcile any fractional residuals explicitly.
  * Option B: Sum exact per-user commissions (without ceil) at high precision and do a single rounded allocation to validator (or track residuals).
* Track the remainder explicitly in a dedicated storage field and distribute or allow withdrawal by the correct party (validator or users) during later reconciliation.
* Ensure both the user-facing bookkeeping (userRewards/totalClaimableByToken) and validator accruals use the same rounding path or include a reconciliation step that assigns leftovers.

(Do not add any behavior not present in the original report — above are typical corrective strategies to address asymmetric rounding; implementers should choose the change consistent with intended invariants.)

***

If you want, I can:

* Propose a minimal code patch (diff) to reconcile rounding,
* Or convert the suggested mitigations into concrete Solidity changes and tests.
