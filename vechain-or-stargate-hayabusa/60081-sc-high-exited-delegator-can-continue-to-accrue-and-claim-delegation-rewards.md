# 60081 sc high exited delegator can continue to accrue and claim delegation rewards&#x20;

**Submitted on Nov 18th 2025 at 10:05:04 UTC by @Diavol0 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60081
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

In the Hayabusa Stargate staking protocol, each delegator’s staking position is represented by an NFT. When a user exits delegation, their NFT should stop accruing delegation rewards after the exit period, and further rewards should be distributed only to remaining active delegators.

Due to an off‑by‑one logic bug in the delegation rewards window calculation, an exited delegator’s NFT continues to have its claimable rewards increase over time as validator periods complete, as long as there are still other active delegators on that validator. This effectively allows the exited delegator to keep “participating” in future reward periods and to siphon a portion of the yield that should have belonged to active delegators.

This behaviour fits Immunefi’s “Theft of unclaimed yield” category: the attacker does not directly steal at‑rest principal, but steals yield that should accrue to other users. From the victims’ perspective, the protocol fails to deliver the promised share of rewards proportional to their stake.

### Root cause

The bug is located in the reward window calculation logic:

* File: `packages/contracts/contracts/Stargate.sol`
* Function: `_claimableDelegationPeriods` (private view)

This function returns the pair `(firstClaimablePeriod, lastClaimablePeriod)` for a given NFT (`_tokenId`). It is used both by the read‑only `claimableDelegationPeriods` / `claimableRewards` helpers and by the state‑changing `claimRewards` function.

Relevant excerpt (simplified):

```solidity
function _claimableDelegationPeriods(
    StargateStorage storage $,
    uint256 _tokenId
) private view returns (uint32, uint32) {
    uint256 delegationId = $.delegationIdByTokenId[_tokenId];
    if (delegationId == 0) {
        return (0, 0);
    }

    (address validator, , , ) = $.protocolStakerContract.getDelegation(delegationId);
    if (validator == address(0)) {
        return (0, 0);
    }

    (uint32 startPeriod, uint32 endPeriod) =
        $.protocolStakerContract.getDelegationPeriodDetails(delegationId);
    (, , , uint32 completedPeriods) =
        $.protocolStakerContract.getValidationPeriodDetails(validator);

    uint32 currentValidatorPeriod = completedPeriods + 1;

    uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
    if (nextClaimablePeriod < startPeriod) {
        nextClaimablePeriod = startPeriod;
    }

    // handle ended delegations
    if (
        endPeriod != type(uint32).max &&
        endPeriod < currentValidatorPeriod &&
        endPeriod > nextClaimablePeriod
    ) {
        return (nextClaimablePeriod, endPeriod);
    }

    // handle active/pending delegations
    if (nextClaimablePeriod < currentValidatorPeriod) {
        return (nextClaimablePeriod, completedPeriods);
    }

    return (0, 0);
}
```

Two key points:

1. When a delegation exit is signalled in the `ProtocolStaker` mock, its `endPeriod` is set to `completedPeriods + 1` (i.e. first period after the last completed period).
2. The condition that attempts to handle “ended delegations” uses `endPeriod > nextClaimablePeriod` instead of `>=`:

   ```solidity
   endPeriod < currentValidatorPeriod &&
   endPeriod > nextClaimablePeriod
   ```

This creates a corner case when the delegator has claimed rewards such that `nextClaimablePeriod == endPeriod`. In that situation:

* The “ended delegation” branch is **not** taken, because `endPeriod > nextClaimablePeriod` fails when `endPeriod == nextClaimablePeriod`.
* The code falls through to the “active/pending” branch, which returns `(nextClaimablePeriod, completedPeriods)`. In other words, the NFT is treated as if the delegation were still active up to the latest `completedPeriods` reported by the validator, even though the delegation has logically ended.

As the validator continues to complete more periods over time (with other delegators still active), `completedPeriods` increases and thus `lastClaimablePeriod` for the exited NFT also keeps increasing. Consequently, `claimableRewards(tokenId)` for this exited NFT continues to grow even after exit, stealing a slice of the delegators’ rewards in those later periods.

### Why this is a vulnerability and not just “dust” or rounding

* The protocol’s documentation and code comments state that rewards for a delegation should only accrue while the delegation is active and until its exit period. After that, the NFT’s owner should only be entitled to rewards for completed periods between the delegation’s start and end.
* In the affected corner case, rewards from later periods — in which the NFT was no longer actively delegated — are still counted into the exited NFT’s claimable rewards.
* Those rewards necessarily come at the expense of other delegators on the same validator, because the total validator rewards for a period are fixed (`getDelegatorsRewards`), and distribution is based on relative effective stake. The exited NFT’s effective stake has been removed from the denominator for those periods, but its owner still receives a numerator share based on their historical stake, causing other delegators’ shares to be reduced.

This is not just a UX or dust issue. It is a systematic misallocation of yield in favour of the exited NFT holder, i.e. “Theft of unclaimed yield” from other delegators.

## Link to Proof of Concept

<https://gist.github.com/6newbie/84368ec9da05363585286636a48d15ca>

## Proof of Concept

Below is a step‑by‑step PoC based on a Hardhat unit test added to the repository:\
`packages/contracts/test/unit/Stargate/RewardsExploit_H01.test.ts`

{% stepper %}
{% step %}

### Setup — Environment & Contracts

* Network: Hardhat in‑memory network.
* Contracts deployed via existing helper `getOrDeployContracts({ forceDeploy: true, config })` with a local config (`createLocalConfig()`), as used by the project’s own `Rewards.test.ts`.
* `ProtocolStakerMock` is used as the staking protocol backend.
  {% endstep %}

{% step %}

### Setup — Actors

* `user` (User A): first delegator, will exit later (attacker).
* `otherUser` (User B): second delegator, remains active (victim).
* `validator`: a validator address configured in `ProtocolStakerMock` as ACTIVE.
  {% endstep %}

{% step %}

### Setup — NFT configuration

* One level with `vetAmountRequiredToStake = 1 VET` and a simple `scaledRewardFactor`, so both users’ NFTs have identical effective stake.
  {% endstep %}

{% step %}

### Exploit scenario — Both users stake and delegate

* User A:
  * `stargate.stake(LEVEL_ID)` → mints `tokenIdA`.
  * `stargate.delegate(tokenIdA, validator)` → creates delegation A.
* User B:
  * Same steps → mints `tokenIdB` and creates delegation B.

Now there are two equal delegators on the validator.
{% endstep %}

{% step %}

### Exploit scenario — Fast‑forward completed periods

* Set validator completed periods to C:

  ```ts
  const C = 8;
  await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, C);
  ```
* This simulates the validator having completed C reward periods.
  {% endstep %}

{% step %}

### Exploit scenario — Both users claim to reset lastClaimedPeriod

* User A: `stargate.claimRewards(tokenIdA)`.
* User B: `stargate.claimRewards(tokenIdB)`.
* After this, both NFTs have `lastClaimedPeriod ≈ C` (caught up to period C).
  {% endstep %}

{% step %}

### Exploit scenario — User A requests delegation exit

* User A: `stargate.requestDelegationExit(tokenIdA)`.
* In `ProtocolStakerMock`, `signalDelegationExit` sets:

  ```solidity
  endPeriod = validations[validator].completedPeriods + 1; // C + 1
  ```
* User A’s delegation is now logically exiting and should not accrue rewards beyond `endPeriod`.
  {% endstep %}

{% step %}

### Exploit scenario — Shortly after exit: small progression

* Advance completed periods by 1:

  ```ts
  const shortAfterExitCompleted = C + 1;
  await protocolStakerMock.helper__setValidationCompletedPeriods(
      validator.address,
      shortAfterExitCompleted
  );
  ```
* Measure claimable rewards for A:

  ```ts
  const claimableSoonAfterExit =
      await stargateContract["claimableRewards(uint256)"](tokenIdA);
  ```
* This captures the small legitimate window around exit.
  {% endstep %}

{% step %}

### Exploit scenario — Long after exit: many more periods while B remains delegated

* Advance completed periods further:

  ```ts
  const longAfterExitCompleted = shortAfterExitCompleted + 5;
  await protocolStakerMock.helper__setValidationCompletedPeriods(
      validator.address,
      longAfterExitCompleted
  );
  ```
* Measure claimable rewards for same exited NFT:

  ```ts
  const claimableLongAfterExit =
      await stargateContract["claimableRewards(uint256)"](tokenIdA);
  ```

{% endstep %}

{% step %}

### Observed buggy behaviour

* The test asserts:

  ```ts
  expect(claimableLongAfterExit).to.be.gt(claimableSoonAfterExit);
  ```
* Under current implementation this **passes**: `claimableLongAfterExit > claimableSoonAfterExit`.
* Interpretation:
  * An NFT whose delegation has exited should be bounded by the exit window; further increases in `completedPeriods` should not increase its rewards.
  * Instead, as more periods elapse while other delegators remain active, the exited NFT’s `claimableRewards` keeps increasing as if still active, showing it improperly shares in future periods’ rewards.
    {% endstep %}
    {% endstepper %}

## Impact on other delegators

Because `getDelegatorsRewards(validator, period)` per period is fixed, any extra rewards granted to the exited NFT for a period must come at the expense of remaining delegators’ shares. The exited NFT is treated as if it can claim a proportional slice based on its historical stake, reducing other delegators’ received rewards — effectively theft of unclaimed yield.

***

If you want, I can:

* Suggest a minimal code fix to the `_claimableDelegationPeriods` logic (e.g., change the `>` to `>=` and add tests), or
* Produce a proposed unit test that reproduces the PoC in the repo’s test style.
