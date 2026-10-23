# #42123 \[SC-Critical] Insufficient Token Reservation in \`startUnstake\` Leads to Permanent Freezing of Vested Funds

**Submitted on Mar 20th 2025 at 23:24:15 UTC by @Ekko for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42123
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In StakeV2.sol, the `startUnstake` function reduces `totalSupply` immediately, but the `stakingToken balance` isn’t decreased until `unstake` is called later. This creates a temporary "excess" in `accumulatedDeptRewardsYeet (stakingToken.balanceOf(this) - totalSupply)`, which can be distributed as rewards via `executeRewardDistributionYeet`. If these tokens are distributed before the vesting period ends, the contract lacks sufficient stakingToken to fulfill vesting withdrawals, permanently freezing user funds in production.

## Vulnerability Details

1. `startUnstake` reduces `totalSupply` but leaves staking token contract balance - `stakingToken.balanceOf(this)` unchanged

```solidity
balanceOf[msg.sender] -= unStakeAmount;
totalSupply -= unStakeAmount;
```

2. `accumulatedDeptRewardsYeet` calculates excess tokens as

```solidity
return stakingToken.balanceOf(address(this)) - totalSupply;
```

* This "excess" includes tokens still needed for vesting, which are not reserved.

3. `executeRewardDistributionYeet` uses this excess to transfer stakingToken to the zapper for reward distribution:

```solidity
uint256 accRevToken0 = accumulatedDeptRewardsYeet();
require(accRevToken0 > 0, "No rewards to distribute");
require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");

stakingToken.approve(address(zapper), accRevToken0);
```

4. These tokens are permanently converted to LP positions and vault shares, with no mechanism to convert them back for unstaking:

```solidity
if (address(token0) == address(stakingToken)) {
    (, vaultSharesMinted) = zapper.zapInToken0(swap, stakingParams, vaultParams);
} else {
    (, vaultSharesMinted) = zapper.zapInToken1(swap, stakingParams, vaultParams);
}

_handleVaultShares(vaultSharesMinted);
```

5. When users try to unstake after vesting, the transaction reverts because the contract has insufficient balance:

```solidity
function _unstake(uint256 index) private {
    Vesting memory vesting = vestings[msg.sender][index];
    (uint256 unlockedAmount, uint256 lockedAmount) = calculateVesting(vesting);
    // This transfer will revert if tokens were converted to rewards
    stakingToken.transfer(msg.sender, unlockedAmount);
    // ...
}
```

## Impact Details

If executeRewardDistributionYeet distributes tokens committed to vesting entries as rewards, the contract’s stakingToken balance may become insufficient to fulfill withdrawals during unstake. This causes the vested tokens to be permanently frozen, as there’s no mechanism to recover the funds or cancel vesting entries.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L149>

## Proof of Concept

## Proof of Concept

copy this to StakeV2.test.sol

```solidity
     function test_POC_Unstake_Funds_Lost() public {

         
        KodiakVaultV1 kodiakVault = new KodiakVaultV1(token, wbera);
        address alice = makeAddr("alice");
        token.mint(alice, 100 ether);
        

        // Initial setup - Alice stakes tokens
        vm.startPrank(alice);
        token.approve(address(stakeV2), 100 ether);
        stakeV2.stake(100 ether);
        vm.stopPrank();
        
        // Verify initial state
        assertEq(stakeV2.balanceOf(alice), 100 ether, "Initial stake balance incorrect");
        assertEq(stakeV2.totalSupply(), 100 ether, "Initial total supply incorrect");
        assertEq(token.balanceOf(address(stakeV2)), 100 ether, "Initial contract token balance incorrect");
        
        // Alice starts unstaking half their tokens
        vm.prank(alice);
        stakeV2.startUnstake(50 ether);
        
        // Verify state after startUnstake
        assertEq(stakeV2.balanceOf(alice), 50 ether, "Balance after startUnstake incorrect");
        assertEq(stakeV2.totalSupply(), 50 ether, "Total supply after startUnstake incorrect");
        assertEq(token.balanceOf(address(stakeV2)), 100 ether, "Contract balance should remain unchanged");
        
        // Verify excess tokens are detected
        uint256 excessTokens = stakeV2.accumulatedDeptRewardsYeet();
        assertEq(excessTokens, 50 ether, "Excess tokens calculation incorrect");
        
        // Manager executes reward distribution with the "excess" tokens
        vm.startPrank(address(this)); 
        
        // Setup mock zapper return values
        mockZapper.setReturnValues(1 ether, 1 ether); // Mock some return values
        
        // Execute reward distribution with the tokens that should be reserved for unstaking
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(50 ether, 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );
        vm.stopPrank();

        // Verify contract balance is now 50 ether after distribution
        assertEq(token.balanceOf(address(stakeV2)), 50 ether, "Contract should have 50 ether left");
    
        // Fast forward past vesting period
        vm.warp(block.timestamp + 10 days);

        // Step 4: Alice tries to unstake her first 50 tokens
        vm.startPrank(alice);
        stakeV2.unstake(0);  // Try to unstake first vesting entry

        // Verify Token Balance
        assertEq(token.balanceOf(address(stakeV2)), 0 ether, "Contract should now have 0 ether");
        assertEq(stakeV2.balanceOf(alice), 50 ether, "Balance should be 50");
        assertEq(stakeV2.totalSupply(), 50 ether, "Total supply should be 50");

        
        // Alice starts unstaking her remaining 50 tokens 
        vm.startPrank(alice);
        stakeV2.startUnstake(50 ether);
        
        // Verify new state
        assertEq(stakeV2.balanceOf(alice), 0, "Balance should be 0");
        assertEq(stakeV2.totalSupply(), 0, "Total supply should be 0");


        // Fast forward past vesting period
        vm.warp(block.timestamp + 10 days);
        
    
        // Alice tries to unstake the remaining
        vm.startPrank(alice);
        uint256 aliceBalanceBefore = token.balanceOf(alice);
        vm.expectRevert();
        stakeV2.unstake(0);
        

        // Verify the unstake failed (tokens weren't transferred)
        assertEq(token.balanceOf(alice), aliceBalanceBefore, "Alice shouldn't have received tokens");

    }
```
