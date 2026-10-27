# 60210 sc high during a validator exit users will be unable to unstake due to underflow

**Submitted on Nov 20th 2025 at 00:08:26 UTC by @oxrex for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60210
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

During a validator EXIT, users will be unable to unstake due to an underflow issue inside the `unstake()` function.

### Vulnerability Details

The unstaking process in the current implementation of Vechain staking is implemented in two main user actions:

{% stepper %}
{% step %}

### User requests delegation exit

1. A user requests to exit delegation validator X.\
   In `requestDelegationExit` the validator's effective stake for the next period is reduced:

```solidity
function requestDelegationExit(
        uint256 _tokenId
    ) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    ...

    // decrease the effective stake
    _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);

    emit DelegationExitRequested(_tokenId, delegation.validator, delegationId, exitBlock);
}
```

{% endstep %}

{% step %}

### User calls unstake()

2. The user then calls `unstake()` to get their staked VET tokens back and claim earned rewards.
   {% endstep %}
   {% endstepper %}

If the validator has not been exited before the user calls `unstake()`, the user receives their VET back as expected.

However, a problematic sequence creates an underflow:

{% stepper %}
{% step %}

### Problematic sequence — part 1

* User requests exit from validator X (reducing the validator effective stake for the upcoming period).
  {% endstep %}

{% step %}

### Problematic sequence — part 2

* Validator X is exited (validator becomes status EXITED).
  {% endstep %}

{% step %}

### Problematic sequence — part 3

* User calls `unstake()`. Because the validator status is EXITED, `unstake()` attempts to call `_updatePeriodEffectiveStake` again — even though the effective stake was already reduced when the user requested exit.
  {% endstep %}
  {% endstepper %}

This double deduction creates two failure modes:

* If the user is the sole delegator to that validator, the second deduction causes an underflow, permanently locking the user's tokens.
* If there are multiple delegators, the second deduction corrupts the validator's effective stake and will cause other delegators' future unstake calls to underflow as well, propagating the issue.

Relevant excerpt from `unstake()`:

```solidity
function unstake(
        uint256 _tokenId
    ) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    ...
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
    ...
}
```

The PoC attached with this report demonstrates the bug. The fix is to avoid decreasing effective stake twice for the same delegation/validator (i.e., ensure `_updatePeriodEffectiveStake` isn't called redundantly when the exit was already processed in `requestDelegationExit`).

## Impact Details

{% hint style="danger" %}
Complete lock of the user's VET tokens (the `token.vetAmountStaked` associated with the NFT). The only contract function that sends out those VET tokens is `unstake()`, so this underflow can permanently prevent users from reclaiming funds.
{% endhint %}

## References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L276-L283>

## Proof of Concept

<details>

<summary>PoC test (paste into Rewards.test.ts and run)</summary>

```js
it.only("poc for locked stake:", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked token with id:", tokenId);

        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        log("\n🎉 Delegated token to validator", validator.address);

        // fast forward some periods
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 5); //
        await tx.wait();
        log("\n🎉 Set validator completed periods to 5");
        // request to exit the delegation
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        log("\n🎉 Requested to exit the delegation");

        // validator exits
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();

        tx1 = await stargateContract.connect(user).unstake(tokenId);
        await tx1.wait();
    });
```

Expected revert seen when underflow occurs:

```
Error: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block)
at Stargate._updatePeriodEffectiveStake (contracts/Stargate.sol:1039)
at Stargate.unstake (contracts/Stargate.sol:285)
```

</details>
