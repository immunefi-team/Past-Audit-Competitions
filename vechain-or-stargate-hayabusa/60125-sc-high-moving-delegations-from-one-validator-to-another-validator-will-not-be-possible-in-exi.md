# 60125 sc high moving delegations from one validator to another validator will not be possible in exit case for validator 1

**Submitted on Nov 19th 2025 at 00:07:57 UTC by @oxrex for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60125
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hour

## Description

### Brief/Intro

The protocol allows moving delegations from one validator to another after first signaling exit so users avoid the maturity wait every time. This is currently implemented incorrectly and fails in an edge case described below, causing a user to be unable to switch validators without unstaking and re-staking.

### Vulnerability Details

{% stepper %}
{% step %}

### Step

User delegates 100k VET to validator 1 at period 0.
{% endstep %}

{% step %}

### Step

At period 1 or 2, the user requests undelegation from validator 1. Validator 1's effective stake at period 4 will be decreased by 100k inside `requestDelegationExit >> _updatePeriodEffectiveStake`.
{% endstep %}

{% step %}

### Step

At period 4 or 5, the validator exits and its status becomes EXITED.
{% endstep %}

{% step %}

### Step

At period 6, the user calls `delegate` to move their delegation from validator 1 (EXITED) to validator 2 (ACTIVE).
{% endstep %}

{% step %}

### Step

Inside `_undelegate`, since the current validator (validator 1) has status EXITED, the following `if` block runs:

```solidity
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    status == DelegationStatus.PENDING
) {
    // get the completed periods of the previous validator
    (, , , uint32 oldCompletedPeriods) = $
        .protocolStakerContract
        .getValidationPeriodDetails(currentValidator);
    // decrease the effective stake of the previous validator
    _updatePeriodEffectiveStake(
        $,
        currentValidator,
        _tokenId,
        oldCompletedPeriods + 2,
        false // decrease
    );
}
```

This attempts to subtract 100k VET again from validator 1's effective stake at period oldCompletedPeriods + 2 (e.g. period 6).
{% endstep %}

{% step %}

### Step

Because the earlier exit request already decreased validator 1's effective stake, attempting to decrease it again causes an underflow (0 - 100k), reverting the transaction.
{% endstep %}

{% step %}

### Step

As a result, the user cannot move their delegation from validator 1 to validator 2. They must call `unstake()` to get their VET back, then `stake()` and wait the maturity period (e.g. 2 days) before delegating to validator 2 — defeating the intended UX of switching validators without unstaking.
{% endstep %}
{% endstepper %}

Relevant code excerpt from the `_delegate` implementation:

```solidity
function _delegate(StargateStorage storage $, uint256 _tokenId, address _validator) private {
    // ensure token is not already delegated
    DelegationStatus status = _getDelegationStatus($, _tokenId);
    if (status == DelegationStatus.ACTIVE) {
        revert TokenAlreadyDelegated(_tokenId);
    }

    // ensure validator is in valid state
    (, , , , uint8 validatorStatus, ) = $.protocolStakerContract.getValidation(_validator); // e.g 2 for active

    (, , uint32 validatorExitBlock, ) = $.protocolStakerContract.getValidationPeriodDetails(
        _validator
    ); // e.g type(uint256).max for no exit

    if (
        (validatorStatus != VALIDATOR_STATUS_ACTIVE &&
            validatorStatus != VALIDATOR_STATUS_QUEUED) ||
        // if the validator has requested to exit, we cannot delegate to it
        validatorExitBlock != type(uint32).max
    ) {
        revert ValidatorNotActiveOrQueued(_validator);
    }

    // Tokens under matutiry period cannot be delegated
    if ($.stargateNFTContract.isUnderMaturityPeriod(_tokenId)) {
        revert TokenUnderMaturityPeriod(_tokenId);
    }

    // get the token details
    DataTypes.Token memory token = $.stargateNFTContract.getToken(_tokenId);
    if (token.levelId == 0) {
        revert InvalidToken(_tokenId);
    }

    uint256 currentDelegationId = $.delegationIdByTokenId[_tokenId];

    // If the token was previously exited or pending it means that the VET is still held in the protocol,
    // so we need to withdraw it and deposit again for the new delegation
    if (status == DelegationStatus.EXITED || status == DelegationStatus.PENDING) {
        // get the current validator
        (address currentValidator, , , ) = $.protocolStakerContract.getDelegation(
            currentDelegationId
        );

        // withdraw the delegation
        $.protocolStakerContract.withdrawDelegation(currentDelegationId);

        // emit the event to signal that the delegation was withdrawn
        emit DelegationWithdrawn(
            _tokenId,
            currentValidator,
            currentDelegationId,
            token.vetAmountStaked,
            token.levelId
        );

        // get the validator status
        (, , , , uint8 currentValidatorStatus, ) = $.protocolStakerContract.getValidation(
            currentValidator
        );
        // if the delegation is pending or the validator is exited or unknown
        // decrease the effective stake of the previous validator
        if (
            currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
            status == DelegationStatus.PENDING
        ) {
            // get the completed periods of the previous validator
            (, , , uint32 oldCompletedPeriods) = $
                .protocolStakerContract
                .getValidationPeriodDetails(currentValidator);
            // decrease the effective stake of the previous validator
            _updatePeriodEffectiveStake(
                $,
                currentValidator,
                _tokenId,
                oldCompletedPeriods + 2,
                false // decrease
            );
        }

        if (status == DelegationStatus.PENDING) {
            // If the current delegation is pending, it means that the owner is changing the validator,
            // without requesting to exit first (which is allowed since the exit is not active yet)
            // so we emit an event to signal this action to the indexers
            emit DelegationExitRequested(
                _tokenId,
                currentValidator,
                currentDelegationId,
                Clock.clock()
            );
        }
    }

    // ... rest of code omitted for brevity ...
}
```

## Impact Details

User will be unable to switch from validator A to validator B after having signaled exit from validator A and will be forced to unstake and restake (and wait the maturity period) instead of using the intended one-stop delegation switch flow. This temporarily freezes the user's funds and prevents the intended smoother UX.

## References

* Vulnerable source region: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L407-L414>

## Proof of Concept

Paste the test case below into the Delegation.test.ts test file after the `it("should delegate a token that was previously delegated and now is exited", async ()` test case:

```js
it.only("poc:", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked token with id:", tokenId);

        // delegate the NFT to the validator
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        log("\n🎉 Correctly delegated the NFT to validator", validator.address);

        // check the delegation status
        const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
        expect(delegationStatus).to.equal(DELEGATION_STATUS_PENDING);

        // advance 1 period
        // so the delegation is active
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 1);
        await tx.wait();
        log("\n Set validator completed periods to 1 so the delegation is active");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );

        // request exit
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        log("\n🎉 Correctly requested to exit the delegation");
        // advance 3 period
        // so the delegation is exited
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 3);
        await tx.wait();
        log("\n Set validator completed periods to 2 so the delegation is exited");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );

        // set validator status to exited
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();
        log("\n Set validator status to exited so the delegation fails");

        // Delegate to another validator
        const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);
        const callTx = stargateContract.connect(user).delegate(tokenId, otherValidator.address);
        await expect(callTx)
            .to.emit(stargateContract, "DelegationWithdrawn")
            .withArgs(
                tokenId,
                validator.address,
                delegationId,
                levelSpec.vetAmountRequiredToStake,
                levelSpec.id
            );
    });
```

Expected test output shows an arithmetic underflow/overflow revert:

```
Error: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block)
    at Stargate._updatePeriodEffectiveStake (contracts/Stargate.sol:1034)
    at Stargate.unstake (contracts/Stargate.sol:284)
    at Stargate.delegate (contracts/Stargate.sol:339)
    at StargateProxy._delegate (@openzeppelin/contracts/proxy/Proxy.sol:31)
```
