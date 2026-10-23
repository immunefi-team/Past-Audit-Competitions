# 60553 sc high the delegator and the validator both exiting consecutively could lead to underflow in the unstake and delegate and stuck staked vet&#x20;

**Submitted on Nov 24th 2025 at 03:09:54 UTC by @frolic for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60553
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

If a user requests exit and later the validator itself exits before the user calls unstake() or delegate(), the code applies the same decrease twice for the total delegator staked in the validator. This double decrement can lead to an underflow and a DOS of the users staked VET.

## Vulnerability Details

The contract tracks delegators’ total stake per validator and period via checkpoints. Decreasing a user’s contribution is done by:

```solidity
// Stargate.sol
function _updatePeriodEffectiveStake(..., uint32 _period, bool _isIncrease) private {
     uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);
     uint256 currentValue   = $.delegatorsEffectiveStake[_validator].upperLookup(_period);
@>   uint256 updatedValue   = _isIncrease ? currentValue + effectiveStake : 
                                            currentValue - effectiveStake;
     $.delegatorsEffectiveStake[_validator].push(_period, SafeCast.toUint224(updatedValue));
}
```

There are two externally reachable flows that both call \_updatePeriodEffectiveStake(..., false):

1. User-initiated exit — requestDelegationExit()

```solidity
// decrease when the user signals exit
(, , , uint32 completedPeriods) = protocolStaker.getValidationPeriodDetails(delegation.validator);
_updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

2. Validator exited — later in unstake() (and also in delegate() when switching away)

```solidity
// in unstake(): if validator exited or delegation pending, decrease again
(, , , uint32 oldCompletedPeriods) = protocolStaker.getValidationPeriodDetails(delegation.validator);
_updatePeriodEffectiveStake($, delegation.validator, _tokenId, oldCompletedPeriods + 2, false);

```

### Attack path

* User delegates → delegation becomes active. Some time pass
* User calls requestDelegationExit(tokenId) → first decrease is applied for period completedPeriods + 2. some time pass
* Validator status is then set to EXITED. (validator calls signalExit()) some time pass
* User calls unstake(tokenId)/delegate() → code sees VALIDATOR\_STATUS\_EXITED and applies the second decrease to the validator's total delegated stake
* Because the first decrease may have taken the checkpointed value to exactly 0, subtracting again underflows and causes panic 0x11. Test: “should revert when unstaking if both the validator and delegation are exited (POC for double effective stake decrease)”.

## Impact Details

The users funds are stuck as the underflow will lead to a revert of the unstake() and the delegate() transaction calls. And hence the user is unable to acess their staked VET.

## References

* Contract: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol>
* requestDelegationExit() — <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L523C5-L571C6>
* unstake() — <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L266C9-L283C10>
* \_updatePeriodEffectiveStake() — performs unchecked subtraction on decreases. <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L993C4-L1013C6>

## Proof of Concept

## Proof of Concept

```javascript
it("should revert when unstaking if both the validator and delegation are exited (POC for double effective stake decrease)", async () => {
        const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);
        await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        const tokenId = await stargateNFTMockContract.getCurrentTokenId();

        // delegate to the validator
        await stargateContract.connect(user).delegate(tokenId, validator.address);

        // fast-forward validator periods so the delegation becomes active
        tx = await protocolStakerMockContract.helper__setValidationCompletedPeriods(
            validator.address,
            5
        );
        await tx.wait();

        // user requests delegation exit while validator is still active (first decrease)
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();

        // force the validator to exit as well which triggers the second decrease later on
        tx = await protocolStakerMockContract.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();

        // The second decrease happens inside unstake producing an underflow (bug)
        await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithPanic(0x11);
    });
```
