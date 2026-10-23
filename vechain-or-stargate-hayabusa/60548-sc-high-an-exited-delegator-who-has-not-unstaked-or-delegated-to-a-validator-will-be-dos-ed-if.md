# 60548 sc high an exited delegator who has not unstaked or delegated to a validator will be dos ed if a validator exits&#x20;

**Submitted on Nov 24th 2025 at 00:47:20 UTC by @HalalAudits for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60548
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

The checks in `unstake` and `delegate` will subtract a delegator's effective stake from the `delegatorsEffectiveStake[_validator]` if the delegation is still pending or the validator has exited.

An issue arises incase a delegator has exited and before `unstaking` or `redelegating to a new validator`, the validator also exits, which will lead to subtracting an already subtracted value leading to a panic due to underflow hence preventing the delegator from recovering their amount.

## Vulnerability Details

Before `unstaking` or `delegating to a new validator`, a delegator must first call `requestDelegationExit` to show his exit intent and his effective stake is subtracted from the validtaor's `delegatorsEffectiveStake[_validator]` for the next period.

```solidity
   // decrease the effective stake
   _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L567C9-L568C101>

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

* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L993C5-L1013C6>

But if this previous validator exits( which is possible since a validator can exit anytime) before the delegator `unstakes` or redelegates (a delegator is not given any period in which to unstake or redelegate so it can also be a longer period), the effectiveStake will be subtracted again from the validator's `delegatorsEffectiveStake[_validator]` due to these checks in `unstake` and `delegate`:

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
        }
```

* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L398C13-L414C14>

As you can see in the above check will be `true` if the `validator has exited` regardless of whether the delegator had exited before the validator's exit hence guaranteeing a re-subtraction of the validator's `delegatorsEffectiveStake[_validator]`.

Yet this amount was already subtracted in the call to `requestDelegationExit` before.

### VulnerabiltyPath

* Delegator A has delegated his stake to validator B, and he is the only delegator to this validator.
* After sometime Delegator A wants to unstake so he calls `requestDelegationExit`.
* But before Delegator A calls `unstake`, validator B also exits.
* So when Delegator A tries to call unstake, the function just reverts with a panic due to underflow.
* Delegator A will lose his funds forever if validator B does not become a validator again.

## Impact Details

This will prevent users from being able to `unstake` or redelegate their funds due to the underflow panic hence a total loss of funds.

Some such exited delegators will be able to collect their funds if the funds are still huge but it will be inevitable for the last unstakers from this `exited validator` to lose their funds as with each `unstake` the `delegatorsEffectiveStake[_validator]` reduces upto the last one where it will underflow.

This can also be done by a malicious validator to prevent delegators from unstaking their funds.

## References

* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L398C13-L414C14>
* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L993C5-L1013C6>
* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L567C9-L568C101>

## Proof of Concept

## Proof of Concept

Paste the test below and run `yarn contracts:test:unit -- -- --grep "For an exited delegaotor, cannot unstake if validator exited"`

```javascript
it("For an exited delegaotor, cannot unstake if validator exited", async () => {
        // stake the token
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked token with id:", tokenId);

        // delegate the token to the validator
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        log("\n🎉 Delegated token to validator", validator.address);

        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
        await tx.wait();
        log("\n🎉 Set validator completed periods to 2 so the delegation is active");
        
        //@audit user requests an exit
        const pendingTx = stargateContract.connect(user).requestDelegationExit(tokenId);

        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, (2 + 2));
        await tx.wait();
        log("\n🎉 Set validator completed periods to 2 + 2 so the delegation should be able to unstake");

        //@audit but before completing the exit the validator also exits
        await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );

        //@audit the user will not be able to unstake or even delegate, due to a panic in resubtacting,
        //effective stake which is zero hence an underflow
        //Panic code 0x11 represents overflow or underflo
        await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithPanic(0x11);

    });

```
