# 60282 sc high last delegators for an exited validator may be dosed from re delegating or unstaking due to incorrect accounting of period effective stake

**Submitted on Nov 20th 2025 at 22:46:09 UTC by @prk0 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60282
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The period effective stake, which represents the total sum of users' weighted delegations, can be subtracted twice for the same delegator in an edge case. This results in insolvency for the affected validator, as some delegators will be unable to re-delegate or unstake due to DoS.

## Vulnerability Details

```solidity
function requestDelegationExit(
    uint256 _tokenId
) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    StargateStorage storage $ = _getStargateStorage();
    uint256 delegationId = $.delegationIdByTokenId[_tokenId];
    if (delegationId == 0) {
        revert DelegationNotFound(_tokenId);
    }

    Delegation memory delegation = _getDelegationDetails($, _tokenId);

    if (delegation.status == DelegationStatus.PENDING) {
        // if the delegation is pending, we can exit it immediately
        // by withdrawing the VET from the protocol
        $.protocolStakerContract.withdrawDelegation(delegationId); 
        emit DelegationWithdrawn(
            _tokenId,
            delegation.validator,
            delegationId,
            delegation.stake,
            $.stargateNFTContract.getTokenLevel(_tokenId)
        );
        // and reset the mappings in storage regarding this delegation
        _resetDelegationDetails($, _tokenId);
    } else if (delegation.status == DelegationStatus.ACTIVE) {
        // If the delegation is active, we need to signal the exit to the protocol and wait for the end of the period
        // We do not allow the user to request an exit multiple times
        if (delegation.endPeriod != type(uint32).max) {
            revert DelegationExitAlreadyRequested();
        } 
        $.protocolStakerContract.signalDelegationExit(delegationId);
    } else { 
        revert InvalidDelegationStatus(_tokenId, delegation.status);
    }

    // decrease the effective stake
    // Get the latest completed period of the validator
    (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
        delegation.validator
    );
    (, uint32 exitBlock) = $.protocolStakerContract.getDelegationPeriodDetails(delegationId);

    // decrease the effective stake
>   _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);

    emit DelegationExitRequested(_tokenId, delegation.validator, delegationId, exitBlock);
}

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
>   uint256 updatedValue = _isIncrease
>       ? currentValue + effectiveStake
>       : currentValue - effectiveStake;

    // push the updated effective stake
    $.delegatorsEffectiveStake[_validator].push(_period, SafeCast.toUint224(updatedValue));
}
```

Users can call `Stargate::requestDelegationExit()` when in the `PENDING` state to cancel their delegation, or in the `ACTIVE` state to exit the current delegation.

In `Stargate::requestDelegationExit()`, the period effective stake is decremented to remove the requesting user’s share starting from the next period (`completedPeriods + 2`).

```solidity
function _delegate(StargateStorage storage $, uint256 _tokenId, address _validator) private {
    // SNIP

    uint256 currentDelegationId = $.delegationIdByTokenId[_tokenId];

    // If the token was previously exited or pending it means that the VET is still held in the protocol,
    // so we need to withdraw it and deposit again for the new delegation
    if (status == DelegationStatus.EXITED || status == DelegationStatus.PENDING) {
        // SNIP

>       if (currentValidatorStatus == VALIDATOR_STATUS_EXITED || status == DelegationStatus.PENDING) {
>           // get the completed periods of the previous validator
>           (, , , uint32 oldCompletedPeriods) = $
>               .protocolStakerContract
>               .getValidationPeriodDetails(currentValidator);
>           // decrease the effective stake of the previous validator
>           _updatePeriodEffectiveStake(
>               $,
>               currentValidator,
>               _tokenId,
>               oldCompletedPeriods + 2,
>               false // decrease
>           );
>       }

        // SNIP
    }

    // SNIP
}
```

A validator can enter the `EXITED` status via voluntarily signaling to exit - Starting from the next period (`completedPeriods + 2`), the validator will enter a 24 hour cooldown period and will stop producing blocks / accepting delegations.

A validator can also enter the `EXITED` status if a validator is force removed by not producing blocks for 7 consecutive days.

When this occurs, users can either re-delegate to another active / queued validator, or unstake.

In both delegate and unstake flows, period effective stake is decremented, if the validator that the `tokenId` was delegating to has exited.

If a user requests to exit delegation in the same period that a validator signals exit as well, the effective stake for this user may be decremented twice - initially when `requestDelegationExit()` is called, then again when `delegate()` or `unstake()` is called.

The last user(s) may be unable to re-delegate or unstake because the effective stake for the concluded period will eventually return 0 and will result in an underflow revert in `_updatePeriodEffectiveStake()`

For example, if a Mjolnir tier user requests to exit delegation, the number of users who are DoSed may be larger than if a Dawn tier user requests to exit delegation.

## Impact Details

The last delegators for an exited validator may be DoSed from re-delegating or unstaking, which permanently locks their staked `VET` in `Stargate`.

The impact of this issue can increase depending on the number of users who request delegation exit and the tiers of their NFTs.

For example, if more users request delegation exit, then more users will be affected by DoS.

In addition, if users with higher tier NFTs request delegation exit, then more users will be affected by DoS.

## Recommendation

Consider skipping the update to period effective stake when a delegation exit has been requested in both unstake and delegate flows.

```solidity
function _delegate(StargateStorage storage $, uint256 _tokenId, address _validator) private {
    // SNIP

    uint256 currentDelegationId = $.delegationIdByTokenId[_tokenId];

    // If the token was previously exited or pending it means that the VET is still held in the protocol,
    // so we need to withdraw it and deposit again for the new delegation
    if (status == DelegationStatus.EXITED || status == DelegationStatus.PENDING) {
        // SNIP

       if (currentValidatorStatus == VALIDATOR_STATUS_EXITED || status == DelegationStatus.PENDING) {
-           // get the completed periods of the previous validator
-           (, , , uint32 oldCompletedPeriods) = $
-               .protocolStakerContract
-               .getValidationPeriodDetails(currentValidator);
-           // decrease the effective stake of the previous validator
-           _updatePeriodEffectiveStake(
-               $,
-               currentValidator,
-               _tokenId,
-               oldCompletedPeriods + 2,
-               false // decrease
-           );      
+           // get the completed periods of the previous validator
+           // skips if tokenId has previously requested to exit delegation
+           Delegation memory delegation = _getDelegationDetails($, _tokenId);
+           if (delegation.endPeriod == type(uint32).max || status == DelegationStatus.PENDING) {
+               (, , , uint32 oldCompletedPeriods) = $
+                   .protocolStakerContract
+                   .getValidationPeriodDetails(currentValidator);
+               // decrease the effective stake of the previous validator
+               _updatePeriodEffectiveStake(
+                   $,
+                   currentValidator,
+                   _tokenId,
+                   oldCompletedPeriods + 2,
+                   false // decrease
+               );
           }            

        }

        // SNIP
    }

    // SNIP
}
```

## References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L993-L1013>

## Proof of Concept

Set up: Copy and paste the code below into `Rewards.test.ts`

Run with the following command: `yarn contracts:test:unit`

```solidity
it("Delegation exits that occur in the same period that a validator voluntarily exits may cause effectiveStake checkpoints to go out of sync", async () => {
    // user stakes + delegates
    const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
    tx = await stargateContract.connect(user).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await tx.wait();
    const tokenId = await stargateNFTMock.getCurrentTokenId();
    log("\n🎉 user TokenId:", tokenId);

    tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
    await tx.wait();
    
    // otherUser stakes and delegates -> delegates to same validator as user
    tx = await stargateContract.connect(otherUser).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await tx.wait();
    
    const tokenIdOther = await stargateNFTMock.getCurrentTokenId();
    log("\n🎉 otherUser TokenId:", tokenId);

    tx = await stargateContract.connect(otherUser).delegate(tokenIdOther, validator.address);
    await tx.wait();

    // @audit-info Period 6 begins
    //   completedPeriods = 5
    //   currentPeriod = 6
    //   nextPeriod = 7
    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 5);
    await tx.wait();
    log("\n🎉 Set validator completed periods to 5");

    // @audit-info Validator must commit to 7 day, 15 day, or 30 day periods.
    //   Assume validator's period is 7 days (minimum period commitment)
    //   On the first day of the Period 6, user requests delegation exit
    //   <1> The effectiveStake checkpoint for Period 7 is decremented
    await stargateContract.connect(user).requestDelegationExit(tokenId);

    // @audit-info Validator voluntarily exits on day 2 of Period 6
    //   Validator exit takes effect starting next period (Period 7)
    //   That is, starting from Period 7, this validator will not produce any blocks 
    //     and will be unable to accept any new delegations.
    tx = await protocolStakerMock.helper__setValidatorStatus(
        validator.address,
        VALIDATOR_STATUS_EXITED
    );
    await tx.wait();
    log("\n🎉 Set validator status to exited");

    // Period 6 ends
    // currentPeriod = 7
    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 6);
    await tx.wait();

    // @audit-info user (1) claims rewards (auto) & (2) re-delegates after Period 6
    const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
    expect(delegationStatus).to.be.equal(DELEGATION_STATUS_EXITED);

    // @audit-info <2> The most recent effectiveStake checkpoint for validator is decremented again for user
    tx = await stargateContract.connect(user).delegate(tokenId, otherValidator.address);
    await tx.wait();
    log("\n🎉 Delegated token to validator", validator.address);

    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
        otherValidator.address,
        1
    );
    await tx.wait();
    log("\n🎉 Set validator completed periods to 1");

    // @audit-info otherUser is the last remaining delegator of validator 
    //   otherUser attempts to re-delegate as well from validator -> otherValidator
    //   otherUser will be permanently DoSed from unstake() / delegate() due to underflow when decreasing effectiveStake,
    //     which has been decremented twice now for the same user.
    //     1. effectiveStake was decremented when user requested delegation exit in Period 6
    //     2. effectiveStake for user was decremented a second time when user re-delegates in Period 7
    const delegationStatus2 = await stargateContract.getDelegationStatus(tokenId);
    expect(delegationStatus2).to.be.equal(DELEGATION_STATUS_ACTIVE);

    // @audit Reverts due to underflow in currentValue (= 0) - effectiveStake (!= 0) computation [_updatePeriodEffectiveStake()]
    tx = await stargateContract.connect(otherUser).delegate(tokenIdOther, otherValidator.address);
    await tx.wait();
});
```
