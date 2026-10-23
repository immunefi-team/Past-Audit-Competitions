# 60004 sc high double decrease effective stake bug in unstake&#x20;

**Submitted on Nov 17th 2025 at 14:09:08 UTC by @jo13 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60004
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield

## Description

### Brief/Intro

A vulnerability in the `Stargate` contract’s `unstake()` function causes the protocol to **double-decrease a delegator’s effective stake** for a given validator and NFT. In realistic conditions (a user requests delegation exit, the validator later exits, and the user then calls `unstake()`), this double-decrease causes an arithmetic underflow inside `_updatePeriodEffectiveStake`, leading to a **permanent denial-of-service on unstaking**. Affected users are unable to withdraw their staked VET or fully realize their staking rewards, causing **permanent freezing of funds and unclaimed yield** unless the contract is upgraded or otherwise remediated off-chain.

## Vulnerability Details

### High-level description

The `Stargate` contract tracks delegators’ "effective stake" per validator and per period using a checkpointed mapping `delegatorsEffectiveStake[validator]`. This mapping is updated via the internal function `_updatePeriodEffectiveStake`, which (simplified):

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

When `_isIncrease == false`, this function performs `currentValue - effectiveStake` without any explicit guard that `currentValue >= effectiveStake`. In normal flows, the checkpointed value is expected to be large enough, but under certain sequences of calls the protocol **applies this decrease twice** for the same NFT/validator combination at future periods where the total effective stake is already 0. In Solidity 0.8.x, `0 - effectiveStake` reverts with a panic due to arithmetic underflow, causing `unstake()` to revert.

### Call flow and conditions

Two key public functions contribute to the double-decrease:

1. `requestDelegationExit(uint256 _tokenId)`
2. `unstake(uint256 _tokenId)`

#### `requestDelegationExit()` path

When a delegation is **ACTIVE**, `requestDelegationExit` signals an exit in the external `ProtocolStaker` and **immediately decreases** the delegator's effective stake at a future period:

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

    if (delegation.status == DelegationStatus.ACTIVE) {
        // Signal exit in the protocol
        // ...
        $.protocolStakerContract.signalDelegationExit(delegationId);
    } else {
        revert InvalidDelegationStatus(_tokenId, delegation.status);
    }

    // Get the latest completed period of the validator
    (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
        delegation.validator
    );
    (, uint32 exitBlock) = $.protocolStakerContract.getDelegationPeriodDetails(delegationId);

    // **Decrease** the effective stake
    _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);

    emit DelegationExitRequested(_tokenId, delegation.validator, delegationId, exitBlock);
}
```

At this point, the effective stake for that NFT/validator pair is decreased for `completedPeriods + 2`, which is a future period relative to the current checkpoint.

#### `unstake()` path

Later, after the validator has completed additional periods and may have exited, the user calls `unstake(_tokenId)`:

```solidity
function unstake(
    uint256 _tokenId
) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    StargateStorage storage $ = _getStargateStorage();
    Delegation memory delegation = _getDelegationDetails($, _tokenId);
    DataTypes.Token memory token = $.stargateNFTContract.getToken(_tokenId);

    // ... maturity and status checks omitted for brevity ...

    // if the delegation is pending or exited
    if (delegation.status != DelegationStatus.NONE) {
        $.protocolStakerContract.withdrawDelegation(delegation.delegationId);
        emit DelegationWithdrawn(
            _tokenId,
            delegation.validator,
            delegation.delegationId,
            delegation.stake,
            token.levelId
        );
    }

    // get the current validator status
    (, , , , uint8 currentValidatorStatus, ) = $.protocolStakerContract.getValidation(
        delegation.validator
    );

    // if the delegation is pending or the validator is exited or unknown
    // **decrease** the effective stake of the previous validator
    if (
        currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
        delegation.status == DelegationStatus.PENDING
    ) {
        (, , , uint32 oldCompletedPeriods) = $
            .protocolStakerContract
            .getValidationPeriodDetails(delegation.validator);

        _updatePeriodEffectiveStake(
            $,
            delegation.validator,
            _tokenId,
            oldCompletedPeriods + 2,
            false // decrease
        );
    }

    // ... rewards claiming, burn, and fund transfer ...
}
```

The important point is that **`unstake()` may call `_updatePeriodEffectiveStake(..., false)` a second time** for the same NFT/validator pair at a similar future period (`oldCompletedPeriods + 2`). Depending on how periods are advanced between `requestDelegationExit` and `unstake`, this can target a period where the delegators’ effective stake is already fully drawn down to 0.

In that scenario, inside `_updatePeriodEffectiveStake`:

* `currentValue = $.delegatorsEffectiveStake[_validator].upperLookup(_period)` returns `0`.
* `effectiveStake = _calculateEffectiveStake($, _tokenId)` is positive.
* `updatedValue = currentValue - effectiveStake` becomes `0 - effectiveStake`, which reverts with a panic due to underflow.

Because this arithmetic occurs inside `unstake()`, the **entire unstake operation reverts**, even though the user followed a valid lifecycle.

{% stepper %}
{% step %}

### Typical valid lifecycle for a delegator (relevant to the bug)

* Stake
* Delegate
* Request delegation exit
* Wait for validator/delegation exit
* Call `unstake()` to reclaim principal
  {% endstep %}
  {% endstepper %}

## Impact Details

In the affected flow, a user stakes VET, delegates their NFT to a validator, requests delegation exit while the delegation is active, waits until the delegation and validator are considered exited, and then calls `unstake()` to reclaim their principal. Due to the double-decrease of effective stake, `_updatePeriodEffectiveStake` underflows and causes `unstake()` to revert every time, resulting in:

* Permanent freezing of staked funds and any unclaimed yield for that position.
* A broken staking lifecycle where the protocol fails to deliver the promised ability to exit.
* A safety/availability issue that can impact any regular delegator following a normal exit flow.
* Potential for griefing: attackers can deliberately create conditions that push users into the unrecoverable state.

## References

* **Core contract:**
  * `contracts/Stargate.sol`
    * `unstake(uint256 _tokenId)` logic and call to `_updatePeriodEffectiveStake` when validator or delegation is exited.
    * `requestDelegationExit(uint256 _tokenId)` logic and initial decrease of effective stake at `completedPeriods + 2`.
    * `_updatePeriodEffectiveStake(StargateStorage storage $, address _validator, uint256 _tokenId, uint32 _period, bool _isIncrease)` implementation.
* **Mocks:**
  * `contracts/mocks/ProtocolStakerMock.sol`
    * Used in tests to simulate validator statuses and completed periods.
* **Tests:**
  * `test/unit/Stargate/Delegation.test.ts`
    * New regression test: `"should revert when unstaking after requesting delegation exit and validator exit"`.
* **VeChain Thor Staker implementation (for behavior reference):**
  * `builtin/staker.go` and `builtin/staker_native.go` in the VeChain Thor repository (confirming behaviors of `GetValidation`, `GetDelegation`, period handling, and exit semantics).

## Proof of Concept

A dedicated unit test was added to `test/unit/Stargate/Delegation.test.ts` to reproduce this behavior using the `ProtocolStakerMock`:

```typescript
it("should revert when unstaking after requesting delegation exit and validator exit", async () => {
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

        // make the delegation active by advancing completed periods
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            2
        );
        await tx.wait();
        log("\n🎉 Set validator completed periods to 2 so the delegation is active");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );

        // user requests delegation exit while validator is active
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        log("\n🎉 Requested delegation exit");

        // advance periods so delegation becomes exited in the mock
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            3
        );
        await tx.wait();
        log("\n🎉 Set validator completed periods to 3 so the delegation is exited");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );

        // mark validator as exited in the protocol mock
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();
        log("\n🎉 Set validator status to exited");

        // unstake should revert due to double decrease of effective stake
        await expect(stargateContract.connect(user).unstake(tokenId)).to.be.reverted;
    });
```

Run the test:

```bash
export VITE_APP_ENV=local
npx hardhat test test/unit/Stargate/Delegation.test.ts \
  --grep "should revert when unstaking after requesting delegation exit and validator exit"
```

Observed output:

```
shard-u2: Stargate: Delegation
  ✔ should revert when unstaking after requesting delegation exit and validator exit (41ms)

1 passing (1s)
```

This confirms that in the described scenario, `unstake()` **always reverts**, demonstrating the existence of the vulnerability.
