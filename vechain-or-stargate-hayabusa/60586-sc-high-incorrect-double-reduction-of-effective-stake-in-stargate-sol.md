# 60586 sc high incorrect double reduction of effective stake in stargate sol

**Submitted on Nov 24th 2025 at 08:05:31 UTC by @T0nraq for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60586
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Description:

When a user requests a delegation exit and subsequently unstakes their NFT after the validator has exited, the contract logic can trigger a double reduction of the validator's effective stake for the same token. This occurs because both `requestDelegationExit()` and `unstake()` call `_updatePeriodEffectiveStake()` to decrease the validator's effective stake, but for overlapping periods and the same token.

**Relevant Code Snippets:**

#### requestDelegationExit()

```solidity
// decrease the effective stake
// Get the latest completed period of the validator
(, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
    delegation.validator
);
// decrease the effective stake
_updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

#### unstake()

```solidity
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    delegation.status == DelegationStatus.PENDING
) {
    // get the completed periods of the previous validator
    (, , , uint32 oldCompletedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(delegation.validator);

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

#### \_updatePeriodEffectiveStake()

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

**Steps to Reproduce/Attack path:**

1. Stake and delegate an NFT to a validator.
2. Call `requestDelegationExit(tokenId)` for the NFT.
3. Advance the validator to `VALIDATOR_STATUS_EXITED`.
4. Call `unstake(tokenId)` for the same NFT.
5. Observe that the validator's effective stake is reduced twice for the same token in future periods.

**Expected Behavior:**\
The validator's effective stake should only be reduced once per token exit event, regardless of whether the user unstakes after requesting a delegation exit.

**Actual Behavior:**\
The effective stake is reduced twice: once during `requestDelegationExit()` and again during `unstake()`, resulting in an incorrect accounting of the validator's effective stake.

**Impact:**

* Will affect reward calculations.
* Over-penalizes the validator's effective stake.
* May lead to inconsistencies in protocol accounting and user expectations.
* Some users may not be able to exit delegations as it could cause an underflow

**Suggested Fix:**

* Track whether the effective stake has already been reduced for a given token/period, and prevent redundant reductions in `unstake()`.
* Refactor the logic so that only one reduction occurs per exit/unstake lifecycle.

## Proof of Concept

## Proof of Concept

> Add the following unit test to stake.test.ts

```solidity
    it.only("should show double reduction of effective stake when unstaking after validator exit", async () => {
        // stake an NFT for user1
        const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);
        
        await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        const tokenId1 = await stargateNFTMockContract.getCurrentTokenId();
        await stargateContract.connect(user).delegate(tokenId1, validator.address);

        // stake an NFT for user2 (second staker/delegator)
        const user2 = otherAccounts[2];
        await stargateContract.connect(user2).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        const tokenId2 = await stargateNFTMockContract.getCurrentTokenId();
        await stargateContract.connect(user2).delegate(tokenId2, validator.address);

        // set validator completed periods to 10 so is active
        await protocolStakerMockContract.helper__setValidationCompletedPeriods(validator.address, 10);

        // get effective stake before exit
        let period = 12; // oldCompletedPeriods + 2 after exit
        const stakeBefore = await stargateContract.getDelegatorsEffectiveStake(validator.address, period);

        // request delegation exit for user1
        await stargateContract.connect(user).requestDelegationExit(tokenId1);

        // get effective stake after exit
        const stakeAfterExit = await stargateContract.getDelegatorsEffectiveStake(validator.address, period + 2);

        await protocolStakerMockContract.helper__setValidationCompletedPeriods(validator.address, 15);

        // Exit Validator
        await protocolStakerMockContract.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );

        // advance some periods so is exited
        await protocolStakerMockContract.helper__setValidationCompletedPeriods(validator.address, 20);
        
        // unstake the NFT for user1
        await stargateContract.connect(user).unstake(tokenId1);
        
        // get effective stake after unstake
        period = 22; // oldCompletedPeriods + 2
        const stakeAfterUnstake = await stargateContract.getDelegatorsEffectiveStake(validator.address, period);
        
        await protocolStakerMockContract.helper__setValidationCompletedPeriods(validator.address, 23);
        // await stargateContract.connect(user2).requestDelegationExit(tokenId2);

        // Print the values to show double reduction
        console.log("Stake before exit:", stakeBefore.toString());
        console.log("Stake after exit:", stakeAfterExit.toString());
        console.log("Stake after unstake:", stakeAfterUnstake.toString());

        // The bug: stakeAfterUnstake should equal stakeAfterExit, but will be reduced again
        expect(stakeAfterUnstake).to.be.lt(stakeAfterExit);
    });
```
