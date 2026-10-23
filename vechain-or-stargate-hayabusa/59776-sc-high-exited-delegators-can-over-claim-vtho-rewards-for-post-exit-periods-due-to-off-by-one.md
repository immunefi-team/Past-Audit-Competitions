# 59776 sc high exited delegators can over claim vtho rewards for post exit periods due to off by one error in claimabledelegationperiods

**Submitted on Nov 15th 2025 at 17:38:20 UTC by @jayx for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59776
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

The Stargate contract contains an off‑by‑one error in `_claimableDelegationPeriods` that allows users who have exited a delegation to continue claiming VTHO rewards for validator periods that occurred after their delegation ended.

When a delegation's `endPeriod` equals the user's next claimable period, the "ended" branch condition (`endPeriod > nextClaimablePeriod`) evaluates to false and the function falls through to the "active" branch, incorrectly returning the validator's latest completed period as the last claimable period. This lets ex‑delegators drain rewards that should remain in the pool for currently active delegators, resulting in theft of unclaimed yield and unfair distribution of protocol rewards.

## Vulnerability Details

The root cause lies in the `_claimableDelegationPeriods` function of `Stargate.sol`.

Current implementation (relevant excerpt):

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod  // ❌ Bug: Excludes equality case
) {
    return (nextClaimablePeriod, endPeriod);  // ✅ Correct cap at endPeriod
}

// Falls through to "active" branch
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);  // ❌ Inflated: Includes post-exit periods
}
```

The problem is the strict inequality `endPeriod > nextClaimablePeriod`. Consider this scenario:

{% stepper %}
{% step %}

### Step 1 — Initial state and first claim

A user delegates to a validator and claims rewards once, which advances `lastClaimedPeriod` to, say, period 3.
{% endstep %}

{% step %}

### Step 2 — User requests exit

The user requests an exit, setting `endPeriod` to 4 (the period during which the exit was requested).
{% endstep %}

{% step %}

### Step 3 — Validator continues validating

The validator completes periods 4, 5, 6, etc.
{% endstep %}

{% step %}

### Step 4 — Next claim after exit

When `claimRewards` is called again:

* `nextClaimablePeriod = lastClaimedPeriod + 1 = 4`
* `endPeriod = 4`

The condition `endPeriod > nextClaimablePeriod` evaluates to `4 > 4 = false`, so the "ended" branch is skipped.
{% endstep %}

{% step %}

### Step 5 — Active branch incorrectly used

The function falls through to the "active" branch and returns `lastClaimablePeriod = completedPeriods` (e.g., 6). As a result, `claimableDelegationPeriods` returns `(4, 6)`, exposing periods 4, 5, and 6 as claimable even though the delegation ended at period 4.
{% endstep %}
{% endstepper %}

The correct condition should be `endPeriod >= nextClaimablePeriod` to ensure that when `endPeriod` equals `nextClaimablePeriod`, the function caps the window at `endPeriod` and does not expose later periods.

This bug is amplified because `claimRewards` calls `_claimableDelegationPeriods` internally, and the calculated window is used directly to compute VTHO transfers:

```solidity
(uint32 firstClaimablePeriod, uint32 lastClaimablePeriod) = _claimableDelegationPeriods(tokenId);
...
for (uint32 period = firstClaimablePeriod; period <= lastClaimablePeriod; ++period) {
    rewards += _calculateRewardsForPeriod(...);
}
```

Since the loop iterates inclusive of `lastClaimablePeriod`, the inflated `lastClaimablePeriod` directly results in excess VTHO being transferred to the ex‑delegator.

## Impact Details

This vulnerability enables theft of unclaimed yield from the protocol's VTHO reward pool, with these consequences:

* Direct financial loss: Ex‑delegators can claim VTHO rewards for periods during which they were no longer delegating, stealing yield that should either remain in the pool or be distributed to currently active delegators. Over multiple periods and users, the drain can be significant.
* Unfair reward distribution: Active delegators receive proportionally less reward because a portion of each period's rewards is being siphoned by ex‑delegators who are no longer participating. This breaks the protocol's economic guarantees.

## References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L916C8-L930C10>

## Proof of Concept

The test below reproduces the current (buggy) behavior. It performs delegation, claims once, requests exit, advances validator completed periods, and then observes that an ex‑delegator can still claim for periods after their `endPeriod`.

```typescript
it("should allow an exited delegator to claim rewards for periods after endPeriod (BUG)", async () => {
    // 1. Stake and delegate
    const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
    tx = await stargateContract.connect(user).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await tx.wait();

    const tokenId = await stargateNFTMock.getCurrentTokenId();

    // Delegate to validator
    tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
    await tx.wait();

    // 2. Make delegation active: validator has completed some periods
    //    so that rewards start accumulating
    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
        validator.address,
        3 // completedPeriods = 3 => current validator period = 4
    );
    await tx.wait();

    // 3. Claim rewards once while delegation is still active.
    //    This advances lastClaimedPeriod inside Stargate for this token.
    tx = await stargateContract.connect(user).claimRewards(tokenId);
    await tx.wait();

    // 4. Request exit; this sets a finite endPeriod for the delegation.
    tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
    await tx.wait();

    // 5. Advance validator further so completedPeriods > endPeriod,
    //    simulating the validator continuing to validate after user exit.
    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
        validator.address,
        6 // completedPeriods = 6 => current validator period = 7
    );
    await tx.wait();

    // 6. Read claimable periods for the ex-delegator.
    const [firstClaimablePeriod, lastClaimablePeriod] =
        await stargateContract.claimableDelegationPeriods(tokenId);

    // The *intended* behavior (per protocol team) is that an ex-delegator
    // should only be able to claim up to endPeriod and nothing after it.
    // However, the current implementation returns a window where
    // lastClaimablePeriod > firstClaimablePeriod (e.g. 6 > 4),
    // meaning the user can still claim rewards for periods after exit.
    //
    // This assertion encodes the CURRENT (buggy) behavior so the test passes
    // while clearly flagging it as a bug.
    expect(lastClaimablePeriod).to.be.gt(
        firstClaimablePeriod,
        "BUG: ex-delegator is allowed to claim rewards for periods after endPeriod"
    );
});
```

Suggested fix: change the condition to `endPeriod >= nextClaimablePeriod` so the delegation's `endPeriod` properly caps the claimable window when equal to `nextClaimablePeriod`.
