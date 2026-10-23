# 59727 sc high double decrease dos on exit permanent unstake revert

**Submitted on Nov 15th 2025 at 09:07:29 UTC by @OxPrince for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59727
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

Requesting exit (packages/contracts/contracts/Stargate.sol (lines 523-569)) and later unstaking after the validator exits (packages/contracts/contracts/Stargate.sol (lines 265-282)) deterministically underflows the second \_updatePeriodEffectiveStake call (packages/contracts/contracts/Stargate.sol (lines 993-1012)), so affected delegators can never withdraw their VET again.

### Vulnerability Details

`requestDelegationExit` always removes a delegator's effective stake immediately after the exit is signalled, regardless of whether the exit is pending or active (see `packages/contracts/contracts/Stargate.sol:547-569`). Later, both `unstake` and the redelegation path call `_updatePeriodEffectiveStake` a *second* time whenever either (a) the validator has exited or (b) the delegation was still pending (`packages/contracts/contracts/Stargate.sol:265-282` and `packages/contracts/contracts/Stargate.sol:398-413`). A user who requests exit while the validator is still active therefore hits two separate "decrease" calls for the same checkpoint as soon as the validator eventually transitions to `VALIDATOR_STATUS_EXITED`.

Trace224.upperLookup`explicitly returns`0 `when no smaller checkpoint exists (`node\_modules/@openzeppelin/contracts/utils/structs/Checkpoints.sol:53-88`), so Solidity 0.8 immediately reverts with an arithmetic underflow.` \_getDelegationStatus`short-circuits to`DelegationStatus.EXITED `as soon as the validator exits (`packages/contracts/contracts/Stargate.sol:652-685`), which forces every later` unstake`/`delegate\` call to hit the underflow path. Nothing in storage gets updated when the revert happens, so the token remains forever stuck in the same state.

{% stepper %} {% step %}

## Sequence demonstrating the issue

1. Alice mints a token and delegates it to validator `V`. `_delegate` increases `delegatorsEffectiveStake[V]` at `currentCompletedPeriod + 2` (`packages/contracts/contracts/Stargate.sol:449-462`). {% endstep %}

{% step %} 2. While `V` is still active, Alice calls `requestDelegationExit`. Because her status is `ACTIVE`, the function signals exit and immediately executes `_updatePeriodEffectiveStake(..., completedPeriods + 2, false)` (`packages/contracts/contracts/Stargate.sol:547-568`). All checkpoints at and after that period now contain `0` for Alice. {% endstep %}

{% step %} 3. Time passes and validator `V` transitions to `VALIDATOR_STATUS_EXITED`. `_getDelegationStatus` now reports Alice's delegation as `EXITED` (`packages/contracts/contracts/Stargate.sol:652-685`). {% endstep %}

{% step %} 4. Alice calls `unstake(_tokenId)`. Lines 265‑282 detect that the validator is exited and invoke `_updatePeriodEffectiveStake` again with `_isIncrease = false`. {% endstep %}

{% step %} 5. Inside `_updatePeriodEffectiveStake`, `upperLookup(oldCompletedPeriods + 2)` returns `0` because the previous call already zeroed the history (`node_modules/@openzeppelin/contracts/utils/structs/Checkpoints.sol:53-88`). Subtracting Alice's positive `effectiveStake` from zero causes Solidity 0.8 to revert with panic code 17, so `unstake` aborts. Redelegating via `_delegate` hits the exact same condition (lines 398‑413) and also reverts.

This sequence does not rely on privileged actors or timing tricks—any validator exit after a prior user exit request suffices to brick the account. {% endstep %} {% endstepper %}

### Design Intent Assessment

* The inline comment at `packages/contracts/contracts/Stargate.sol:547-569` says "decrease the effective stake" once the exit is signalled, indicating the authors intended the reduction to happen immediately when the user requests exit.
* The `unstake` comment at `packages/contracts/contracts/Stargate.sol:265-282` explains that the additional decrease is only meant for pending delegations or validators that *exit without the user signaling*. There is no allowance for the same delegation being decreased twice.
* Together with the fact that `_updatePeriodEffectiveStake` lacks any saturating math, the underflow shows this was not a deliberate design—it is simply an unhandled ordering between "user exit" and "validator exit".

Therefore the observed revert is a genuine bug, not a conscious product choice.

### Impact Details

An honest delegator who requested exit while the validator was active loses the ability to ever call `unstake` or redelegate once the validator later exits. Their VET remains locked inside the protocol until an upgrade manually fixes the checkpoints.

## References

Add any relevant links to documentation or code

## Proof of Concept

Stakes.tests.ts

{% code title="Stakes.tests.ts" %}

```solidity
  it("should revert when unstaking after exiting once the validator has already exited (double decrease)", async () => {
        const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMockContract.getCurrentTokenId();

        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();

        // make delegation active so the exit request path runs
        tx = await protocolStakerMockContract.helper__setValidationCompletedPeriods(
            validator.address,
            10
        );
        await tx.wait();

        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();

        // validator exits after the delegator has already signalled exit
        tx = await protocolStakerMockContract.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();
        tx = await protocolStakerMockContract.helper__setValidationCompletedPeriods(
            validator.address,
            20
        );
        await tx.wait();

        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );

        // second decrease underflows delegatorsEffectiveStake and reverts with a panic
        await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithPanic(
            0x11
        );
    });
```
