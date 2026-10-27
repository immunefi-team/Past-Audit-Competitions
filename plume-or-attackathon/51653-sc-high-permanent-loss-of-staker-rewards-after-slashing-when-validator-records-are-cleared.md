# 51653 sc high permanent loss of staker rewards after slashing when validator records are cleared

**Submitted on Aug 4th 2025 at 16:42:22 UTC by @wellbyt3 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51653
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

Clearing a slashed validator’s records sets each user’s stake to 0, so when the user later calls `claim()`, `updateRewardsForValidatorAndToken` exits early and marks rewards as paid without incrementing the user's reward related state. This results in a permanent loss of rewards earned between the last time the user's rewards state was updated and the slash.

### Vulnerability Details

Staker's rewards are claimable after slashing, which is a design choice (only staked funds and commissions are impacted by slashing).

After a slash occurs, to cleanup state, the admin calls `adminBatchClearValidatorRecords` or `adminClearValidatorRecord`, which removes the amount slashed from the user's staking related state. Importantly, now:

$.userValidatorStakes\[user]\[validatorId].staked == 0

When a staker calls `claim()` to claim their rewards from the slashed validator, `_processValidatorRewards` calls `updateRewardsForValidatorAndToken`, which should update the user's reward state, but because their stake amount was set to 0, this is skipped.

Key snippet (from report):

```solidity
function updateRewardsForValidatorAndToken(
        PlumeStakingStorage.Layout storage $,
        address user,
        uint16 validatorId,
        address token
    ) internal {
        // @audit adminClearValidatorRecord() set this to 0
        uint256 userStakedAmount = $.userValidatorStakes[user][validatorId].staked;

        // @audit since userStakeAmount == 0, get to the return part of the function without updating the user's reward related state.
        if (userStakedAmount == 0) {

            if (!$.validators[validatorId].slashed) {
                updateRewardPerTokenForValidator($, token, validatorId);
            }

            $.userValidatorRewardPerTokenPaid[user][validatorId][token] =
                $.validatorRewardPerTokenCumulative[validatorId][token];

            $.userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token] = block.timestamp;
            return; // @audit return here without getting to the reward related state that follows.
        }
...SNIP...
          // @audit this is skipped!
           if (userRewardDelta > 0) {
                    $.userRewards[user][validatorId][token] += userRewardDelta;
                    $.totalClaimableByToken[token] += userRewardDelta;
                    $.userHasPendingRewards[user][validatorId] = true;
        }
...SNIP....
}
```

Because the early return happens before adding the calculated reward delta into the user's rewards, any accumulated rewards between the last reward-state update and the slash are permanently lost for that user.

### Impact Details

Both `adminClearValidatorRecord` and `adminBatchClearValidatorRecords` are intended to be called after slashing occurs. Even though these are admin functions, there's no danger noted in the code or documentation on when these can be called (like many of the other admin functions intended to be used during updates), making it perfectly reasonable to assume this would have been called before all users have claimed their rewards.

It's also important to consider that often stakers are passive. It's reasonable to assume a long time can go by before a staker realizes the validator they've staked with has been slashed and that they should claim their rewards.

Because this is permanent freezing of user rewards, the scope suggests this is critical severity.

The mitigation for this is to ensure user reward related state is updated before allowing the calls to `adminClearValidatorRecord` and `adminBatchClearValidatorRecords` to succeed.

Also note, all 3 claim flows are impacted by this bug.

### References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L316>

## Proof of Concept

Summary of the PoC steps:

{% stepper %}
{% step %}

### Step: Setup and environment

Create a new `.t.sol` file in /plume/test/ and paste the provided solidity test file (below). The test deploys the diamond, facets, treasury, reward tokens, and validators.
{% endstep %}

{% step %}

### Step: Stake and accrue rewards

* user1 stakes 1e18 PLUME with validatorId = 1.
* Advance time by 1 day to accumulate 1 day's worth of rewards.
  {% endstep %}

{% step %}

### Step: Slash validator

* validatorId 1 is slashed using `voteToSlashValidator`.
  {% endstep %}

{% step %}

### Step: Admin clears validator record

* Admin calls `adminClearValidatorRecord(user1, 1)` to clean up state after slashing.
* This sets user1's staked amount for validator 1 to 0.
  {% endstep %}

{% step %}

### Step: Claim and observe loss

* user1 calls `claim(PLUME_NATIVE, 1)`.
* Due to the early return in `updateRewardsForValidatorAndToken` (userStakedAmount == 0), the user's pending rewards are not added to their rewards state.
* The claim returns 0 instead of the expected 1 day's worth of rewards.
  {% endstep %}
  {% endstepper %}

Full test file used in the PoC:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import {Script} from "forge-std/Script.sol";
import "forge-std/Test.sol";

import {PlumeStaking} from "../src/PlumeStaking.sol";
import {PlumeStakingStorage} from "../src/lib/PlumeStakingStorage.sol";
import {PlumeRewardLogic} from "../src/lib/PlumeRewardLogic.sol";
import {AccessControlFacet} from "../src/facets/AccessControlFacet.sol";
import {PlumeStakingRewardTreasury} from "../src/PlumeStakingRewardTreasury.sol";
import {ManagementFacet} from "../src/facets/ManagementFacet.sol";
import {RewardsFacet} from "../src/facets/RewardsFacet.sol";
import {StakingFacet} from "../src/facets/StakingFacet.sol";
import {ValidatorFacet} from "../src/facets/ValidatorFacet.sol";
import {IAccessControl} from "../src/interfaces/IAccessControl.sol";
import {IPlumeStakingRewardTreasury} from "../src/interfaces/IPlumeStakingRewardTreasury.sol";
import {IERC2535DiamondCutInternal} from "@solidstate/interfaces/IERC2535DiamondCutInternal.sol";
import {ISolidStateDiamond} from "@solidstate/proxy/diamond/ISolidStateDiamond.sol";
import {PlumeRoles} from "../src/lib/PlumeRoles.sol";
import {ERC20Mock} from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {PlumeStakingRewardTreasuryProxy} from "../src/proxy/PlumeStakingRewardTreasuryProxy.sol";

contract LostRewardsOnClaim is Test {
    PlumeStaking public diamondProxy;
    PlumeStakingRewardTreasury public treasury;
    
    AccessControlFacet public accessControlFacet;
    StakingFacet public stakingFacet;
    RewardsFacet public rewardsFacet;
    ValidatorFacet public validatorFacet;
    ManagementFacet public managementFacet;

    ERC20Mock public pUSD;
    address public constant PLUME_NATIVE = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

    address public admin = makeAddr("admin");
    address public validatorAdmin = makeAddr("validatorAdmin");
    address public user1 = makeAddr("user1");    
    address public user2 = makeAddr("user2");    

    uint256 public constant MIN_STAKE = 1e18;
    uint256 public constant INITIAL_COOLDOWN = 7 days;
    uint256 public constant MAX_SLASH_VOTE_DURATION = 1 days;
    uint256 public constant MAX_ALLOWED_COMMISSION = .5e18; 

    uint256 public constant PLUME_REWARD_RATE = 1_587_301_587;
    uint256 public constant PLUME_MAX_REWARD_RATE = 1e18;

    function setUp() public {
         
        // 1. Deploy Diamond Proxy
        vm.startPrank(admin);
        diamondProxy = new PlumeStaking();

        // 2. Deploy Custom Facets
        accessControlFacet = new AccessControlFacet();
        stakingFacet = new StakingFacet();
        rewardsFacet = new RewardsFacet();
        validatorFacet = new ValidatorFacet();
        managementFacet = new ManagementFacet();

        // 3. Prepare Diamond Cut
        IERC2535DiamondCutInternal.FacetCut[] memory cut = _prepareDiamondCut();

        // 4. Deploy Diamond Proxy and initialize
        ISolidStateDiamond(payable(address(diamondProxy))).diamondCut(cut, address(0), "");
        diamondProxy.initializePlume(
            address(0),
            MIN_STAKE,
            INITIAL_COOLDOWN,
            MAX_SLASH_VOTE_DURATION,
            MAX_ALLOWED_COMMISSION
        );

        // 5. Initialize Access Control Facet
        AccessControlFacet(address(diamondProxy)).initializeAccessControl();
        AccessControlFacet(address(diamondProxy)).grantRole(PlumeRoles.ADMIN_ROLE,admin);
        AccessControlFacet(address(diamondProxy)).grantRole(PlumeRoles.VALIDATOR_ROLE,admin);
        AccessControlFacet(address(diamondProxy)).grantRole(PlumeRoles.TIMELOCK_ROLE,admin);

        // 6. Create a commission checkpoint and set the max validator percentage
        ManagementFacet(address(diamondProxy)).setMaxAllowedValidatorCommission(
            PlumeStakingStorage.REWARD_PRECISION / 2
        );

        // 7. Deploy reward treasury
        PlumeStakingRewardTreasury treasuryImpl = new PlumeStakingRewardTreasury();
        bytes memory initData = abi.encodeWithSelector(
            PlumeStakingRewardTreasury.initialize.selector,
            admin,
            address(diamondProxy)
        );
        PlumeStakingRewardTreasuryProxy treasuryProxy = new PlumeStakingRewardTreasuryProxy(
                address(treasuryImpl),
                initData
            );
        treasury = PlumeStakingRewardTreasury(payable(address(treasuryProxy)));

        // 8. Set the reward treasury
        RewardsFacet(address(diamondProxy)).setTreasury(address(treasury));

        // 9. Deploy pUSD and add as the reward token
        pUSD = new ERC20Mock();

        // 10. Add pUSD and Native Plume as reward tokens
        treasury.addRewardToken(address(pUSD));
        RewardsFacet(address(diamondProxy)).addRewardToken(
            PLUME_NATIVE,
            PLUME_REWARD_RATE,
            PLUME_MAX_REWARD_RATE
        );
        treasury.addRewardToken(PLUME_NATIVE);

        // 11. Deploy validators
        _deployValidators();

        vm.stopPrank();
        
    }

    function test_lostRewardsOnClaim() public {
        deal(user1, 1000e18);
        deal(address(treasury), 10000e18);

        // 1. user1 stakes 1e18 PLUME with validatorId = 1
        vm.prank(user1);
        StakingFacet(address(diamondProxy)).stake{value: 1e18}(1);
        
        // 2. 1 days worth of rewards accumulates for user1
        skip(1 days);

        // 3. validatorId 1 is slashed
        vm.prank(validatorAdmin);
        ValidatorFacet(address(diamondProxy)).voteToSlashValidator(1, block.timestamp + 1 days);

        // 4. admin clears validator record
        vm.prank(admin);
        ManagementFacet(address(diamondProxy)).adminClearValidatorRecord(user1, 1);
        

        // 5. user1 goes to claim their 1 days worth of rewards, but receives 0 rewards.
        vm.prank(user1);
        uint256 rewardsClaimedPlume = RewardsFacet(address(diamondProxy)).claim(PLUME_NATIVE, 1);
        console2.log("rewardsClaimedPlume: %s", rewardsClaimedPlume);
        assertEq(rewardsClaimedPlume, 0);
    }


//////////////////////////////////////////////////////////////////////////////////////////
//////////////////////////////////////////////////////////////////////////////////////////
//////////////////////////////////////////////////////////////////////////////////////////
//////////////////////////////////////////////////////////////////////////////////////////
//////////////////////////////////////////////////////////////////////////////////////////

    function _deployValidators() internal {
         ValidatorFacet(address(diamondProxy)).addValidator(
            0,
            .05e18,
            validatorAdmin,
            validatorAdmin,
            "0x123",
            "0x456",
            address(0x1234),
            1_000_000e18
        );

        // Add validator 1
        ValidatorFacet(address(diamondProxy)).addValidator(
            1,
            .08e18, // 8% commission
            user2,
            user2,
            "0x789",
            "0xabc",
            address(0x2345),
            1_000_000e18
        );
    }

    function _prepareDiamondCut() internal returns (IERC2535DiamondCutInternal.FacetCut[] memory) {
         IERC2535DiamondCutInternal.FacetCut[]
            memory cut = new IERC2535DiamondCutInternal.FacetCut[](5);

        // AccessControl Facet Selectors
        bytes4[] memory accessControlSigs_Manual = new bytes4[](7);
        accessControlSigs_Manual[0] = bytes4(keccak256(bytes("initializeAccessControl()")));
        accessControlSigs_Manual[1] = bytes4(keccak256(bytes("hasRole(bytes32,address)")));
        accessControlSigs_Manual[2] = bytes4(keccak256(bytes("getRoleAdmin(bytes32)")));
        accessControlSigs_Manual[3] = bytes4(keccak256(bytes("grantRole(bytes32,address)")));
        accessControlSigs_Manual[4] = bytes4(keccak256(bytes("revokeRole(bytes32,address)")));
        accessControlSigs_Manual[5] = bytes4(keccak256(bytes("renounceRole(bytes32,address)")));
        accessControlSigs_Manual[6] = bytes4(keccak256(bytes("setRoleAdmin(bytes32,bytes32)")));

        // Staking Facet Selectors
        bytes4[] memory stakingSigs_Manual = new bytes4[](15);
        stakingSigs_Manual[0] = bytes4(keccak256(bytes("stake(uint16)")));
        stakingSigs_Manual[1] = bytes4(keccak256(bytes("restake(uint16,uint256)")));
        stakingSigs_Manual[2] = bytes4(keccak256(bytes("unstake(uint16)")));
        stakingSigs_Manual[3] = bytes4(keccak256(bytes("unstake(uint16,uint256)")));
        stakingSigs_Manual[4] = bytes4(keccak256(bytes("withdraw()")));
        stakingSigs_Manual[5] = bytes4(keccak256(bytes("stakeOnBehalf(uint16,address)")));
        stakingSigs_Manual[6] = bytes4(keccak256(bytes("stakeInfo(address)")));
        stakingSigs_Manual[7] = bytes4(keccak256(bytes("amountStaked()")));
        stakingSigs_Manual[8] = bytes4(keccak256(bytes("amountCooling()")));
        stakingSigs_Manual[9] = bytes4(keccak256(bytes("amountWithdrawable()")));
        stakingSigs_Manual[10] = bytes4(keccak256(bytes("getUserCooldowns(address)")));
        stakingSigs_Manual[11] = bytes4(keccak256(bytes("getUserValidatorStake(address,uint16)")));
        stakingSigs_Manual[12] = bytes4(keccak256(bytes("restakeRewards(uint16)")));
        stakingSigs_Manual[13] = bytes4(keccak256(bytes("totalAmountStaked()")));
        stakingSigs_Manual[14] = bytes4(keccak256(bytes("totalAmountClaimable(address)")));

        // Rewards Facet Selectors
        bytes4[] memory rewardsSigs_Manual = new bytes4[](23);
        rewardsSigs_Manual[0] = bytes4(keccak256(bytes("addRewardToken(address,uint256,uint256)")));
        rewardsSigs_Manual[1] = bytes4(keccak256(bytes("removeRewardToken(address)")));
        rewardsSigs_Manual[2] = bytes4(keccak256(bytes("setRewardRates(address[],uint256[])")));
        rewardsSigs_Manual[3] = bytes4(keccak256(bytes("setMaxRewardRate(address,uint256)")));
        rewardsSigs_Manual[4] = bytes4(keccak256(bytes("addRewards(address,uint256)")));
        rewardsSigs_Manual[5] = bytes4(keccak256(bytes("claim(address)")));
        rewardsSigs_Manual[6] = bytes4(keccak256(bytes("claim(address,uint16)")));
        rewardsSigs_Manual[7] = bytes4(keccak256(bytes("claimAll()")));
        rewardsSigs_Manual[8] = bytes4(keccak256(bytes("earned(address,address)")));
        rewardsSigs_Manual[9] = bytes4(keccak256(bytes("getClaimableReward(address,address)")));
        rewardsSigs_Manual[10] = bytes4(keccak256(bytes("getRewardTokens()")));
        rewardsSigs_Manual[11] = bytes4(keccak256(bytes("getMaxRewardRate(address)")));
        rewardsSigs_Manual[12] = bytes4(keccak256(bytes("tokenRewardInfo(address)")));
        rewardsSigs_Manual[13] = bytes4(keccak256(bytes("getRewardRateCheckpointCount(address)")));
        rewardsSigs_Manual[14] = bytes4(keccak256(bytes("getValidatorRewardRateCheckpointCount(uint16,address)")));
        rewardsSigs_Manual[15] = bytes4(keccak256(bytes("getUserLastCheckpointIndex(address,uint16,address)")));
        rewardsSigs_Manual[16] = bytes4(keccak256(bytes("getRewardRateCheckpoint(address,uint256)")));
        rewardsSigs_Manual[17] = bytes4(keccak256(bytes("getValidatorRewardRateCheckpoint(uint16,address,uint256)")));
        rewardsSigs_Manual[18] = bytes4(keccak256(bytes("setTreasury(address)")));
        rewardsSigs_Manual[19] = bytes4(keccak256(bytes("getTreasury()")));
        rewardsSigs_Manual[20] = bytes4(keccak256(bytes("getPendingRewardForValidator(address,uint16,address)")));
        rewardsSigs_Manual[21] = bytes4(keccak256(bytes("getRewardRate(address)")));
        rewardsSigs_Manual[22] = bytes4(keccak256(bytes("isRewardToken(address)")));

        // Validator Facet Selectors
        bytes4[] memory validatorSigs_Manual = new bytes4[](20); // Size updated to 19
        validatorSigs_Manual[0] = 
            bytes4(keccak256(bytes("addValidator(uint16,uint256,address,address,string,string,address,uint256)")));
        validatorSigs_Manual[1] = bytes4(keccak256(bytes("setValidatorCapacity(uint16,uint256)")));
        validatorSigs_Manual[2] = bytes4(keccak256(bytes("setValidatorCommission(uint16,uint256)")));
        validatorSigs_Manual[3] = 
            bytes4(keccak256(bytes("setValidatorAddresses(uint16,address,address,string,string,address)")));
        validatorSigs_Manual[4] = bytes4(keccak256(bytes("setValidatorStatus(uint16,bool)")));
        validatorSigs_Manual[5] = bytes4(keccak256(bytes("getValidatorInfo(uint16)")));
        validatorSigs_Manual[6] = bytes4(keccak256(bytes("getValidatorStats(uint16)")));
        validatorSigs_Manual[7] = bytes4(keccak256(bytes("getUserValidators(address)")));
        validatorSigs_Manual[8] = bytes4(keccak256(bytes("getAccruedCommission(uint16,address)")));
        validatorSigs_Manual[9] = bytes4(keccak256(bytes("getValidatorsList()")));
        validatorSigs_Manual[10] = bytes4(keccak256(bytes("getActiveValidatorCount()")));
        validatorSigs_Manual[11] = bytes4(keccak256(bytes("requestCommissionClaim(uint16,address)")));
        validatorSigs_Manual[12] = bytes4(keccak256(bytes("finalizeCommissionClaim(uint16,address)")));
        validatorSigs_Manual[13] = bytes4(keccak256(bytes("voteToSlashValidator(uint16,uint256)")));
        validatorSigs_Manual[14] = bytes4(keccak256(bytes("slashValidator(uint16)")));
        validatorSigs_Manual[15] = bytes4(keccak256(bytes("forceSettleValidatorCommission(uint16)")));
        validatorSigs_Manual[16] = bytes4(keccak256(bytes("getSlashVoteCount(uint16)"))); // <<< NEW SELECTOR
        validatorSigs_Manual[17] = bytes4(keccak256(bytes("cleanupExpiredVotes(uint16)"))); // <<< NEW CLEANUP FUNCTION
        validatorSigs_Manual[18] = bytes4(keccak256(bytes("getValidatorCommissionCheckpoints(uint16)"))); // <<< NEW
        // getValidatorCommissionCheckpoints FUNCTION
        validatorSigs_Manual[19] = bytes4(keccak256(bytes("acceptAdmin(uint16)"))); // <<< NEW acceptAdmin FUNCTION

        // Management Facet Selectors
        bytes4[] memory managementSigs_Manual = new bytes4[](16); // Size updated
        managementSigs_Manual[0] = bytes4(keccak256(bytes("setMinStakeAmount(uint256)")));
        managementSigs_Manual[1] = bytes4(keccak256(bytes("setCooldownInterval(uint256)")));
        managementSigs_Manual[2] = bytes4(keccak256(bytes("adminWithdraw(address,uint256,address)")));
        // updateTotalAmounts was at index 3, it's removed.
        managementSigs_Manual[3] = bytes4(keccak256(bytes("getMinStakeAmount()"))); // Index shifted
        managementSigs_Manual[4] = bytes4(keccak256(bytes("getCooldownInterval()"))); // Index shifted
        managementSigs_Manual[5] = bytes4(keccak256(bytes("setMaxSlashVoteDuration(uint256)"))); // Index shifted
        managementSigs_Manual[6] = bytes4(keccak256(bytes("setMaxAllowedValidatorCommission(uint256)"))); // Index
        // shifted
        managementSigs_Manual[7] = bytes4(keccak256(bytes("adminClearValidatorRecord(address,uint16)"))); // New
        managementSigs_Manual[8] = bytes4(keccak256(bytes("adminBatchClearValidatorRecords(address[],uint16)"))); // New
        managementSigs_Manual[9] = bytes4(keccak256(bytes("pruneCommissionCheckpoints(uint16,uint256)"))); // New
        managementSigs_Manual[10] = bytes4(keccak256(bytes("pruneRewardRateCheckpoints(uint16,address,uint256)"))); // New
        managementSigs_Manual[11] = bytes4(keccak256(bytes("setMaxValidatorPercentage(uint256)"))); // New
        managementSigs_Manual[12] = bytes4(keccak256(bytes("addHistoricalRewardToken(address)"))); // New
        managementSigs_Manual[13] = bytes4(keccak256(bytes("removeHistoricalRewardToken(address)"))); // New
        managementSigs_Manual[14] = bytes4(keccak256(bytes("isHistoricalRewardToken(address)"))); // New
        managementSigs_Manual[15] = bytes4(keccak256(bytes("getHistoricalRewardTokens()"))); // New

        cut[0] = IERC2535DiamondCutInternal.FacetCut({
            target: address(accessControlFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: accessControlSigs_Manual
        });

        cut[1] = IERC2535DiamondCutInternal.FacetCut({
            target: address(managementFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: managementSigs_Manual
        });

        cut[2] = IERC2535DiamondCutInternal.FacetCut({
            target: address(stakingFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: stakingSigs_Manual
        });

        cut[3] = IERC2535DiamondCutInternal.FacetCut({
            target: address(validatorFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: validatorSigs_Manual
        });

        cut[4] = IERC2535DiamondCutInternal.FacetCut({
            target: address(rewardsFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: rewardsSigs_Manual
        });

        return cut;
    }
}
```

Logs from the test run:

<details>

<summary>Test logs</summary>

```
Ran 1 test for test/zLostRewardsAfterAdminCall.t.sol:LostRewardsOnClaim
[PASS] test_lostRewardsOnClaim() (gas: 420784)
Logs:
  rewardsClaimedPlume: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.61ms (210.50µs CPU time)

Ran 1 test suite in 136.61ms (2.61ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

</details>

***

Note: The recommended mitigation in the original report is to ensure user reward-related state is updated before allowing `adminClearValidatorRecord` / `adminBatchClearValidatorRecords` to succeed (so the user's earned rewards are captured before clearing stake amounts).
