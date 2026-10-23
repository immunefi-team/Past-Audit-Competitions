# #41695 \[SC-Critical] StakeV2 leaks user tokens as rewards and eventually will become insolvent.

**Submitted on Mar 17th 2025 at 15:58:59 UTC by @whitehatanon1 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41695
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Vulnerability Summary

In StakeV2 contract, we assume that zapper swaps are not 100% efficient, so we created a function accumulatedDeptRewardsYeet to calculate yeet rewards which are not distributed yet. The assumption is that difference between stakingToken.balanceOf(address(this)) and totalSupply are all because of zapper inefficiency and all this difference can be freely distributed to users as rewards. However this assumption is flawed which leads to a critical vulnerability in which user staked tokens are being used as rewards. StakeV2.startUnstake function reduces the totalSupply (probably to stop paying rewards) and starts a 10 day Vesting period. During these 10 days the tokens are with the Contract. Because of which now stakingToken.balanceOf(StakeV2) is now out of sync with totalSupply variable on the contract. These tokens will be claimed back by user, so when we use these Vesting tokens as rewards, contract is leaking staked tokens as rewards and eventually will not be able to pay user their staked tokens.

## Vulnerability Details

To understand the vulnerability, let us start with function accumulatedDeptRewardsYeet, which makes the assumption that all the difference is because of zapper and hence can be freely distributed. In this function below we are getting the balance from staking token for contract address and subtracting the totalSupply from it.

accumulatedDeptRewardsYeet function from StakeV2.sol StartLine: 145, EndLine: 150

```solidity
    /// @notice The function used to calculate the accumulated rewards that gets return by the zapper since swaps are not 100% efficient
    // @dev The function is used to calculate the rewards that are not distributed yet
    /// @return The accumulated rewards in YEET tokens
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }
```

Now let us look at other functions which manipulate totalSupply.

stake function from StakeV2.sol StartLine: 230, EndLine: 242

```solidity
/// @notice The function used to stake tokens
    /// @param amount The amount of tokens to stake
    /// @dev updates the rewards of the account
    function stake(uint256 amount) external {
        require(amount > 0, "Amount must be greater than 0");
        _updateRewards(msg.sender);

        stakingToken.transferFrom(msg.sender, address(this), amount);

        balanceOf[msg.sender] += amount;
        totalSupply += amount;
        emit Stake(msg.sender, amount);
    }
```

In stake function we add staked amount to totalSupply

startUnstake function from StakeV2.sol StartLine: 244, EndLine: 262

```solidity
    /// @notice The function used to start the unstake process
    /// @param unStakeAmount The amount of tokens to unstake
    /// @dev The tokens are locked for the VESTING_PERIOD
    function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); // DOS protection https://github.com/Enigma-Dark/Yeet/issues/12
        _updateRewards(msg.sender);
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");

        balanceOf[msg.sender] -= unStakeAmount;
        totalSupply -= unStakeAmount;

        uint256 start = block.timestamp;
        uint256 end = start + VESTING_PERIOD;
        vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
        stakedTimes[msg.sender]++;
        emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
    }
```

startUnstake function reduces the totalSupply with unStakeAmount and starts a Vesting Period. It does other things as well but they are not relevant for this discussion. However one thing to note it that startUnstake DOES NOT transfer any tokens, tokens are still kept by the contract. Tokens are actually transfered when user calls unstake if the Vesting period is over or user calls rageQuit in which we give 50% + linear amount based on the time spend in Vesting period. The locked amount in rageQuit is burned by sending it to a dead address. So all the Vested tokens are transfered out, some to user some to burn address.

This in itself is a logical issue, but not enough to put user tokens at risk. We need to look at another function, which actually moves the tokens out to valut assuming they are reward tokens.

executeRewardDistributionYeet function from StakeV2.sol StartLine: 152, EndLine: 180

```solidity
    /// @notice The function used to distribute excess rewards to the vault.
    function executeRewardDistributionYeet(
        IZapper.SingleTokenSwap calldata swap,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultParams
    ) external onlyManager nonReentrant {
        uint256 accRevToken0 = accumulatedDeptRewardsYeet();
        require(accRevToken0 > 0, "No rewards to distribute");
        require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");

        stakingToken.approve(address(zapper), accRevToken0);
        IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0();
        IERC20 token1 = IKodiakVaultV1(stakingParams.kodiakVault).token1();

        uint256 vaultSharesMinted;
        require(
            address(token0) == address(stakingToken) || address(token1) == address(stakingToken),
            "Neither token0 nor token1 match staking token"
        );

        if (address(token0) == address(stakingToken)) {
            (, vaultSharesMinted) = zapper.zapInToken0(swap, stakingParams, vaultParams);
        } else {
            (, vaultSharesMinted) = zapper.zapInToken1(swap, stakingParams, vaultParams);
        }

        _handleVaultShares(vaultSharesMinted);
        emit RewardsDistributedToken0(accRevToken0, rewardIndex);
    }
```

A manager will invoke this function periodically to transfer excessive rewards to the vault and that in turn will make it available for the users. In function executeRewardDistributionYeet we call accumulatedDeptRewardsYeet to see how many access rewards we can distribute and then take manager inputAmount as long as it is less than what we have as excess in stakingToken contract.

Once we look at these two functions together it can produce critical vulnerability. As there could always be someone which has startUnstake when manager invoked executeRewardDistributionYeet.

Let us understand this process with an example.

1. Let us assume that stakerA stakes 1000 tokens.
2. After the staking, Contract token balance: 1000 totalSupply: 1000
3. Now assume that stakerB stakes 500 tokens.
4. After the staking, Contract token balance: 1500 totalSupply:1500
5. Now assume that stakerA wants to unStake 200 tokens. So stakerA will send a startUnstake transaction with amount as 200 tokens.
6. Once this transaction is completed, Contract token balance: 1500 totalSupply: 1300
7. Now assume manager sends a transaction to distribute Yeet rewards with 200 as amount (manager has no way of knowing whether this 200 difference is because of Vesting or because of zapper swaps, so manager will always try to distribute the max).
8. This will awaard stakerA and stakerB with portion of 200 tokens a rewards.
9. Once this transaction is completed, Contract token balance: 1300 totalSupply: 1300
10. Both stakerA and stakerB now can claim these 200 tokens as rewards.
11. Now assume stakerB wants to unStake all tokens, so it will send a transaction of startUnstake with 500 as amount
12. Once this transaction is completed, Contract token balance: 1300 totalSupply: 800
13. Let us assume that stakerA also wants to get remaining token out, so stakerA will also send a transaction of startUnstake with 800 as amount. stakerA has already initiated 200, so now it can initiate for remaining 800.
14. Once this transaction is completed, Contract token balance:1300 totalSupply: 0
15. After 10 days, stakerB calls unstake with index as 0. This will transfer 500 tokens to stakerB
16. After this transaction, Contract token balance: 800 totalSupply: 0
17. stakerA calls unstake with index as 0. This will transfer 200 to stakerA
18. After this transaction, Contract token balance: 600 totalSupply: 0
19. stakerA cals unstake with index as 0 ( previous transaction removed index 0 and made 1 to 0). This should transfer 800 token but since Contract token balance is only 600, the transaction will fail.
20. Someone now have to stake for stakerA to unstake, but the new person will not be able to unstake successfully.
21. Eventually the contract will become Insolvent

## Impact Details

Vunerability will keep on leaking the user tokens as rewards, whenever executeRewardDistributionYeet function is called with one/many active vesting period for any user/users. These leaks will keep on accumulating over time and eventually the contract will become Insolvent.

## Proposed Solution

The most logical way to solve this would be to ignore current Active vesting amounts when we are calculating accumulatedDeptRewardsYeet. However in current implemenation it will be hard to know which vesting are currently active. Since its a mapping of address with Vesting\[]. Also we need to maintain totalSupply logic as it is, as we need to stop the rewards bein awarded in Vesting Period. So easy solution would be to add a new state level varilable totalVestingAmount, and add vesting amount on startUnstake and subtract in \_unstake. Then in accumulatedDeptRewardsYeet subtract this totalVestingAmount, as we still need to pay this amount back to users and it is really not a reward.

So the new functions will be:

```solidity



    // @audit: new state variable in StakeV2 contract
    uint256 public totalVestingAmount;




    /// @notice The function used to calculate the accumulated rewards that gets return by the zapper since swaps are not 100% efficient
    // @dev The function is used to calculate the rewards that are not distributed yet
    /// @return The accumulated rewards in YEET tokens
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        // @Audit: replaced return statement
        // return stakingToken.balanceOf(address(this)) - totalSupply;
        return stakingToken.balanceOf(address(this)) - totalSupply - totalVestingAmount;

    }

    /// @notice The function used to start the unstake process
    /// @param unStakeAmount The amount of tokens to unstake
    /// @dev The tokens are locked for the VESTING_PERIOD
    function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); // DOS protection https://github.com/Enigma-Dark/Yeet/issues/12
        _updateRewards(msg.sender);
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");

        balanceOf[msg.sender] -= unStakeAmount;
        totalSupply -= unStakeAmount;

        // @audit: Added below line
        totalVestingAmount += unStakeAmount;

        uint256 start = block.timestamp;
        uint256 end = start + VESTING_PERIOD;
        vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
        stakedTimes[msg.sender]++;
        emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
    }

    /// @notice The function used to unstake the tokens
    /// @param index The index of the vesting to unstake
    function _unstake(uint256 index) private {
        Vesting memory vesting = vestings[msg.sender][index];

        (uint256 unlockedAmount, uint256 lockedAmount) = calculateVesting(vesting);
        require(unlockedAmount != 0, "No unlocked amount");

        // @audit: Added below line
        totalVestingAmount -= vesting.amount;

        stakingToken.transfer(msg.sender, unlockedAmount);
        stakingToken.transfer(address(0x000000dead), lockedAmount);
        _remove(msg.sender, index);
        if (lockedAmount > 0) {
            emit RageQuit(msg.sender, unlockedAmount, lockedAmount, index);
        } else {
            emit Unstake(msg.sender, unlockedAmount, index);
        }
        stakedTimes[msg.sender]--;
    }

```

## Proof of Concept

## Proof of Concept## Proof of Concept

To demonstrate the vulnerability a runable PoC has been created. In this PoC we do stake and startUnstake and then we do executeRewardDistributionYeet to show how these tokens are leaked as reward and in end we show, how the contract is not able to payback the stakes of a given user.

## PoC details

A new test was added in the existing codebase of audit-comp-yeet. In PoC we add a new test in audit-comp-yeet repo from github to demonstrate the vulnerability.

Github Repo URL: <https://github.com/immunefi-team/audit-comp-yeet.git>

Test File: StakeV2VulnerabilityLostTokensPoC.test.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "../src/StakeV2.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {MockWETH} from "./mocks/MockWBERA.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./mocks/SimpleZapperMock.sol";

contract KodiakVaultV1 {
    IERC20 public token0;
    IERC20 public token1;

    constructor(IERC20 _token0, IERC20 _token1) {
        token0 = _token0;
        token1 = _token1;
    }
}

abstract contract StakeV2_BaseTest {
    StakeV2 public stakeV2;
    MockERC20 public token;
    MockWETH public wbera;
    SimpleZapperMock public mockZapper;
    KodiakVaultV1 public kodiakVault;

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
        address owner = address(this);
        address manager = address(this);
        kodiakVault = new KodiakVaultV1(token, wbera);
        mockZapper = new SimpleZapperMock(token, wbera);
        stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));
    }

    function test() public {}
}

interface IVault {
    function deposit(uint256, address) external returns (uint256);

    function withdraw(uint256, address, address) external returns (uint256);
}
contract StakeV2_VestingLostTokens is Test, StakeV2_BaseTest {
    function test_lostTokensAsRewards() public {
        assertEq(stakeV2.rewardIndex(), 0, "Reward index should be initialized to 0");
        address staker1 = makeAddr("staker_one");
        address staker2 = makeAddr("staker_two");
        uint256 initialRewards = stakeV2.calculateRewardsEarned(staker1);
        assertEq(initialRewards, 0, "initial rewards should be 0");

        /*
            Normal Flow, 2 stakers one staking 900 ether and other staking 100 ether
        */
        vm.startPrank(staker1);
        token.mint(address(staker1), 900 ether);
        token.approve(address(stakeV2), 900 ether);
        stakeV2.stake(900 ether);
        vm.stopPrank();

        vm.startPrank(staker2);
        token.mint(address(staker2), 100 ether);
        token.approve(address(stakeV2), 100 ether);
        stakeV2.stake(100 ether);
        vm.stopPrank();

        // Normal Flow, verify balance of stakeV2, accumulatedDeptRewardsYeet and total supply
        assertEq(1000 ether, token.balanceOf(address(stakeV2)));
        assertEq(stakeV2.totalSupply(), 1000 ether);
        assertEq(0, stakeV2.accumulatedDeptRewardsYeet());

        /*
            Vulnerable Code
                When we startUnstake it will lower the total supply but keep the token in the contract for Vesting period
                This introduces the vulnerability as accumulatedDeptRewardsYeet is calculated as balance of contract on staking token - totalSupply
                Even though we just unstaked, it should not change accumulatedDeptRewardsYeet. But it changes as we are reducing total supply.
        */
        vm.startPrank(staker1);
        stakeV2.startUnstake(200 ether);
        vm.stopPrank();

        assertEq(1000 ether, token.balanceOf(address(stakeV2)));
        assertEq(stakeV2.totalSupply(), 800 ether);
        assertEq(200 ether, stakeV2.accumulatedDeptRewardsYeet());

        uint256 expectedIslandTokens = 0 ether;
        uint256 rewardAmount = 1 ether;

        mockZapper.setReturnValues(200 ether, 200 ether);
        assertEq(stakeV2.rewardIndex(), 0);
        assertEq(stakeV2.totalVaultShares(), 0);
        
        // executing RewardDistributionYeet will transfer the DebtRewards to vault
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(200 ether, 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );
        // Rewards awarded based on DebtYeet
        assertEq(stakeV2.totalVaultShares(), 200 ether);
        assertEq(stakeV2.calculateRewardsEarned(staker1), 175 ether, "Rewards1 earned should be updated correctly");
        assertEq(stakeV2.calculateRewardsEarned(staker2), 25 ether, "Rewards2 earned should be updated correctly");
        assertEq(800 ether, token.balanceOf(address(stakeV2)));

        // we can collect the rewards
        vm.startPrank(staker1);
        stakeV2.claimRewardsInNative(
            stakeV2.calculateRewardsEarned(staker1),
            IZapper.SingleTokenSwap(0, 0, 0, address(0), ""),
            IZapper.SingleTokenSwap(0, 0, 0, address(0), ""),
            IZapper.KodiakVaultUnstakingParams(address(0), 0, 0, address(0)),
            IZapper.VaultRedeemParams(address(token), address(0), 0, 0)
        );
        vm.stopPrank();

        assertEq(stakeV2.calculateRewardsEarned(staker1), 0, "staker1 rewards should be 0");

        vm.startPrank(staker2);
        stakeV2.startUnstake(100 ether);
        stakeV2.rageQuit(0);
        vm.stopPrank();

        // RageQuit only give 50% back to user hence 50 ether
        assertEq(50 ether, token.balanceOf(staker2));
        assertEq(700 ether, token.balanceOf(address(stakeV2)));

        vm.startPrank(staker1);
        stakeV2.rageQuit(0);
        // RageQuit only give 50% back to user hence 100 ether
        assertEq(100 ether, token.balanceOf(staker1));
        
        // Our Unstake is 700, but only 500 are in contract .....
        // staker1 has already unstake 200, so out of 900, 700 are left to claim
        stakeV2.startUnstake(700 ether);
        
        assertEq(100 ether, token.balanceOf(staker1));
        assertEq(500 ether, token.balanceOf(address(stakeV2)));

        // Now user is not able to withdraw as the balance on the contract is less than Vesting amount.
        //vm.expectRevert();
        stakeV2.rageQuit(0);
        
        vm.stopPrank();
        
    }
}


```

1. In this test, first we setup a MockERC20 staking token, a KodiakVault contract, a zapper contract with staking token and MockWETH token.
2. Then in test test\_lostTokensAsRewards we demonstrate the actuall vulnerability.
3. In earlier part of test we demonstrate normal functioning of StakeV2 contract, we demonstrate that users can stake and their initial rewards are still 0 and we assert token balance of contract, totalSupply and accumulatedDeptRewardsYeet
4. From Line 81 we startUnstake for staker1
5. From line 89 to 106 we call executeRewardDistributionYeet to take those Vesting tokens as rewards.
6. We assert that rewards are earned by staker1 and staker2, contract token balance has dropped by 200 tokens from 1000 to 800.
7. Line 109 to 119 demonstrates the ability to claim these rewards.
8. Line 121 to 142 demonstrates that contract can not transfer tokens as it has less token then the vested amount. we used rageQuit so half of the amount is supposed to credited back to user and other half to be burned by sending it to a dead address.
9. Test fails at line 140 with ERC20InsufficientBalance exception and staker1 cannot clain 700 ether tokens, as contract only has 500 ether tokens.

## Steps to Run the PoC

1. Clone audit-comp-yeet repo

```
    git clone https://github.com/immunefi-team/audit-comp-yeet.git
```

2. Add a new Test File in Test folder with name RewardVulnerability.Test.sol
3. Copy paste the contract code shown in PoC detail section. Save the file
4. Run command

   ```
   npm i
   ```
5. Run command

   ```
   forge test RewardVulnerability.Test.sol
   ```
6. Verify the test ran successfully and user was able to calim rewards of past 4 days in single transaction
