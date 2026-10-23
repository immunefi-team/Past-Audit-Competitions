# 59386 sc high fund freeze from double stake subtraction when validator exits&#x20;

**Submitted on Nov 11th 2025 at 18:23:40 UTC by @humanitia for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59386
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief / Intro

The delegator signals exit and later the validator moves to an `EXITED` state. Stargate tries to subtract that NFT’s stake twice: the first subtraction already zeroed the validator’s checkpoint, so the second subtraction underflows and reverts. Affected NFTs can never unstake or redelegate.

## Vulnerability Details

Relevant code paths:

unstake path

```solidity
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    delegation.status == DelegationStatus.PENDING
) {
    _updatePeriodEffectiveStake(..., false);
}
```

delegate path

```solidity
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    status == DelegationStatus.PENDING
) {
    _updatePeriodEffectiveStake(..., false);
}
```

unchecked subtraction inside `_updatePeriodEffectiveStake`

```solidity
uint256 updatedValue = _isIncrease
    ? currentValue + effectiveStake
    : currentValue - effectiveStake;
```

Both `unstake` and `_delegate` call `_updatePeriodEffectiveStake` whenever the validator is `EXITED` or the delegation is still pending.

When a user already requested exit, the first call to `_updatePeriodEffectiveStake` during that flow can push the checkpoint to zero for the next period. Later, if the validator itself becomes `EXITED`, the same subtraction path runs again even though `delegatorsEffectiveStake[_validator].upperLookup(_period)` already returns `0`. `_updatePeriodEffectiveStake` subtracts without a floor so the second subtraction underflows, reverting with panic `0x11`.

## Impact Details

Any NFT that requests exit is stuck once its validator exits, halting unstaking or redelegation and causing fund lock for all affected users.

{% hint style="danger" %}
Permanent freezing of funds for affected NFTs (cannot unstake or redelegate).
{% endhint %}

## Proof of Concept

Add to `packages/contracts/test/unit/Stargate/Stake.test.ts`

New import

```ts
import { PANIC_CODES } from "@nomicfoundation/hardhat-chai-matchers/panic";
```

Test:

```ts
it("should revert when validator exits after user already decreased effective stake", async () => {
    const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);
    await stargateContract.connect(user).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    const tokenId = await stargateNFTMockContract.getCurrentTokenId();

    await stargateContract.connect(user).delegate(tokenId, validator.address);

    await protocolStakerMockContract.helper__setValidationCompletedPeriods(
        validator.address,
        5
    );

    await stargateContract.connect(user).requestDelegationExit(tokenId);

    await protocolStakerMockContract.helper__setValidationCompletedPeriods(
        validator.address,
        15
    );

    await protocolStakerMockContract.helper__setValidatorStatus(
        validator.address,
        VALIDATOR_STATUS_EXITED
    );

    await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithPanic(
        PANIC_CODES.ARITHMETIC_OVERFLOW_UNDERFLOW
    );
});
```

Reproduction steps:

{% stepper %}
{% step %}

### Stake and delegate

Stake and delegate an NFT.
{% endstep %}

{% step %}

### Request delegation exit

Call `requestDelegationExit(tokenId)`.
{% endstep %}

{% step %}

### Advance completed periods

Advance the validator’s completed periods so delegation becomes `EXITED`.
{% endstep %}

{% step %}

### Exit validator

Flip validator to `VALIDATOR_STATUS_EXITED`.
{% endstep %}

{% step %}

### Attempt unstake — revert

Call `unstake(tokenId)`. The call reverts with panic code `0x11` (arithmetic under/overflow) because `_updatePeriodEffectiveStake` subtracts from zero.
{% endstep %}
{% endstepper %}
