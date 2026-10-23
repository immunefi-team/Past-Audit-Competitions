# 60151 sc high double reduction of effective stake can lead to stuck delegations&#x20;

**Submitted on Nov 19th 2025 at 09:28:35 UTC by @Bizarro for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60151
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

Double reduction of the effective stake of a validator can lead to users not being able to withdraw their staked amount.

### Vulnerability Details

In the `_delegation` and `_unstake` functions, the effective stake of the validator is reduced for the next period when the validator has exited their position in the `Staker` contract. The problem arises when a user, who has already exited their delegation in the same or before period as the validator's exit, proceeds to unstake or delegate to another validator. This sequence of actions causes the contract to reduce the effective stake twice for the same user's stake.

{% stepper %}
{% step %}

### Sequence example — initial state

Four users each stake 500e18, making the total effective stake 2000e18.
{% endstep %}

{% step %}

### User exit without unstake/re-delegate

Two users exit their delegation (but do not yet unstake or re-delegate). The effective stake is reduced to 1000e18 — this is correct for now.
{% endstep %}

{% step %}

### Validator signals exit

The validator signals exit. In the next validation period, the validator's status becomes `exited`.
{% endstep %}

{% step %}

### User unstake / re-delegate causes double reduction

A user who previously requested delegation exit now calls `unstake` (or delegates elsewhere). Because the validator status is now `exited`, the contract logic reduces the effective stake again for the next period. This results in an additional decrease, incorrectly bringing effective stake to 0e18 in this example. Remaining users then cannot unstake because the system believes there is no effective stake left — their funds become stuck.
{% endstep %}
{% endstepper %}

Below are the relevant contract excerpts showing where the effective stake is decreased.

requestDelegationExit snippet (decreases effective stake at @1):

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
@1>        _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);

        emit DelegationExitRequested(_tokenId, delegation.validator, delegationId, exitBlock);
    }
```

unstake snippet (may decrease effective stake again at @3 when validator is exited — check at @2):

```solidity
    function unstake(
        uint256 _tokenId
    ) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
        StargateStorage storage $ = _getStargateStorage();
        Delegation memory delegation = _getDelegationDetails($, _tokenId);
        DataTypes.Token memory token = $.stargateNFTContract.getToken(_tokenId);

        // if the token is under the maturity period, we cannot unstake it
        if ($.stargateNFTContract.isUnderMaturityPeriod(_tokenId)) {
            revert TokenUnderMaturityPeriod(_tokenId);
        }

        // check the delegation status
        // if the delegation is active, then NFT cannot be unstaked, since the VET is locked in the protocol
        if (delegation.status == DelegationStatus.ACTIVE) {
            revert InvalidDelegationStatus(_tokenId, DelegationStatus.ACTIVE);
        } else if (delegation.status != DelegationStatus.NONE) {
            // if the delegation is pending or exited
            // withdraw the VET from the protocol so we can transfer it back to the caller (which is also the owner of the NFT)
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
        // decrease the effective stake of the previous validator
@2>        if (
            currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
            delegation.status == DelegationStatus.PENDING
        ) {
            // get the completed periods of the previous validator
            (, , , uint32 oldCompletedPeriods) = $
                .protocolStakerContract
                .getValidationPeriodDetails(delegation.validator);

            // decrease the effective stake of the previous validator
@3>            _updatePeriodEffectiveStake(
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

        // If the NFT has reached the max number of claimable periods, we revert to avoid any loss of rewards
        // In this case, the owner should separatly call the claimRewards() function multiple times to claim all rewards,
        // then call the unstake() function again.
        if (_exceedsMaxClaimablePeriods($, _tokenId)) {
            revert MaxClaimablePeriodsExceeded();
        }
        // ensure that the rewards are claimed
        _claimRewards($, _tokenId);

        // reset the mappings in storage regarding this delegation
        _resetDelegationDetails($, _tokenId);

        // burn the token
        $.stargateNFTContract.burn(_tokenId);

        // validate the contract has enough VET to transfer to the caller
        if (address(this).balance < token.vetAmountStaked) {
            revert InsufficientContractBalance(address(this).balance, token.vetAmountStaked);
        }

        // transfer the VET to the caller (which is also the owner of the NFT since only the owner can unstake)
        (bool success, ) = msg.sender.call{ value: token.vetAmountStaked }("");
        if (!success) {
            revert VetTransferFailed(msg.sender, token.vetAmountStaked);
        }
    }
```

Steps summarized:

* Step-1: A validator exits by calling `signalExit`.
* Step-2: A user calls `requestDelegationExit`, which decreases the `effectiveStake` for the next period (`@1`).
* Step-3: In the next period, validator status becomes `VALIDATOR_STATUS_EXITED`. The same user calls `unstake`. Because of the check at `@2`, `effectiveStake` is decreased again (`@3`).
* Step-4: The double reduction leads to incorrect total stake accounting and can cause further unstake operations to fail.

### Impact Details

Due to the double reduction of the effective stake, users legitimately staked with a validator that has exited may be unable to unstake their funds. This results in funds becoming permanently stuck in the protocol.

## References

* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L276>
* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L407>

## Proof of Concept

Add this console.log to track the new effective values: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L1010>

```solidity
import "hardhat/console.sol";

console.log("updatedValue: ", updatedValue, "for period: ", _period);
```

Add this test to `test/unit/Stargate/Rewards.test.ts` and run `yarn contracts:test:unit`:

```js
it.only("Double counting of effective stake when validator exits", async () => {
        tx = await stargateContract.connect(deployer).setMaxClaimablePeriods(100000);
        await tx.wait();
        
        const levelId = 1;
        const levelSpec = await stargateNFTMock.getLevel(levelId);

        // Get three users
        const user1 = user;
        const user2 = (await ethers.getSigners())[6];
        const user3 = (await ethers.getSigners())[7];

        log("\n📝 Three users will stake and delegate to validator:", validator.address);

        // All three users stake and delegate NFTs
        const tokenIds: bigint[] = [];
        const users = [user1, user2, user3];

        for (let i = 0; i < users.length; i++) {
            const userAccount = users[i];

            // Stake NFT
            const stakeTx = await stargateContract.connect(userAccount).stake(levelId, {
                value: levelSpec.vetAmountRequiredToStake
            });
            await stakeTx.wait();

            const tokenId = await stargateNFTMock.getCurrentTokenId();
            tokenIds.push(tokenId);

            // Delegate to validator
            const delegateTx = await stargateContract
                .connect(userAccount)
                .delegate(tokenId, validator.address);
            await delegateTx.wait();

            log(`\n🎉 User${i + 1} staked and delegated NFT with tokenId:`, tokenId.toString());
        }

        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 5); // 3'5 max claimable periods
        await tx.wait();
        log("\n🎉 Set validator completed periods to 2912");


        // User1 requests delegation exit
        const exitTx = await stargateContract.connect(user1).requestDelegationExit(tokenIds[0]);
        await exitTx.wait();
        log("\n🚪 User1 requested delegation exit for tokenId:", tokenIds[0].toString());

        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 7);
        await tx.wait();
        log("\n🎉 Set validator completed periods to 2913");

        tx = await protocolStakerMock.connect(validator).signalExit(validator.address);
        await tx.wait();
        log("\n🚪 Validator signaled exit");

        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();


        await stargateContract.connect(user1).unstake(tokenIds[0]);
        log("\n🎉 user1 unstaked tokenId:", tokenIds[0].toString());
        
    });
```

Expected console/test output demonstrates the double accounting (updatedValue decreased twice for the same stake):

```
updatedValue:  1500000000000000000 for period:  2
updatedValue:  3000000000000000000 for period:  2
updatedValue:  4500000000000000000 for period:  2
...
updatedValue:  3000000000000000000 for period:  7
...
updatedValue:  1500000000000000000 for period:  9   -> double decrease observed
```

## Mitigation / Notes

(Left as-is — no changes/additions to remediation beyond the original report content per import rules.)
