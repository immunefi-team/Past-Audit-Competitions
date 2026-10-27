# #41524 \[SC-Critical] Incorrect Reward Calculation in accumulatedDeptRewardsYeet() Function Leads to Loss of User Funds During Vesting Period

**Submitted on Mar 16th 2025 at 08:34:54 UTC by @InquisitorScythe for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41524
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The accumulatedDeptRewardsYeet() function calculates rewards incorrectly by using a simple formula (stakingToken.balanceOf(address(this)) - totalSupply) that fails to account for tokens in vesting periods. This critical vulnerability allows the protocol to mistakenly identify users' vesting tokens as distributable rewards. When a manager calls executeRewardDistributionYeet(), these misidentified tokens can be permanently redirected to the vault, resulting in direct theft of users' funds that are in the unstaking process. If exploited in production, this could lead to substantial financial losses for users with active vestings, undermining protocol solvency and user trust.

## Vulnerability Details

The vulnerability stems from a fundamental accounting error in the `accumulatedDeptRewardsYeet()` function, which calculates distributable rewards incorrectly:

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return stakingToken.balanceOf(address(this)) - totalSupply;
}
```

This function assumes that any balance of `stakingToken` held by the contract beyond the `totalSupply` represents distributable rewards. However, this fails to account for tokens that are in the vesting process after users have called `startUnstake()`.

When a user calls `startUnstake()`, the following occurs:

1. Their balance in `balanceOf` mapping is reduced
2. The contract's `totalSupply` is reduced
3. A new `Vesting` entry is created and tokens enter a vesting period (typically 10 days)
4. **Critically, the tokens remain in the contract until the vesting period ends**

The vulnerability exists because tokens in vesting are no longer counted in `totalSupply` but still contribute to `stakingToken.balanceOf(address(this))`. This discrepancy causes `accumulatedDeptRewardsYeet()` to incorrectly include vesting tokens as distributable rewards.

The vulnerable flow is triggered when:

1. Users stake tokens, increasing `totalSupply`
2. Users call `startUnstake()`, which decreases `totalSupply` but keeps tokens in the contract
3. `accumulatedDeptRewardsYeet()` incorrectly reports these vesting tokens as rewards
4. A manager calls `executeRewardDistributionYeet()`, which:

   ```solidity
   function executeRewardDistributionYeet(...) external onlyManager nonReentrant {
       uint256 accRevToken0 = accumulatedDeptRewardsYeet();
       require(accRevToken0 > 0, "No rewards to distribute");
       require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");
       
       stakingToken.approve(address(zapper), accRevToken0);
       // ... swap operations using users' vesting tokens
   }
   ```
5. The function incorrectly approves zapper to spend tokens that actually belong to users in vesting
6. The zapper converts these tokens to LP positions in the vault, permanently removing them from the contract

This vulnerability is particularly severe because:

1. It allows direct theft of user funds that are in vesting
2. It can be exploited by a malicious or misinformed manager
3. Users have no means to prevent their vesting funds from being taken
4. The larger the volume of unstaking, the larger the potential funds misdirected

The absence of proper accounting for vesting tokens represents a critical design flaw in the contract's reward calculation logic.

## Impact Details

This vulnerability poses a severe risk to user assets and protocol integrity, with the following specific impacts:

1. **Direct Loss of User Funds**:
   * When a manager calls `executeRewardDistributionYeet()`, tokens belonging to users in the vesting process can be permanently converted to vault shares
   * 100% of vesting tokens could be misappropriated if distributed
   * Example: If users collectively have 1,000,000 tokens in vesting and a manager distributes all "rewards," users would lose their entire vesting amount
2. **Protocol Insolvency**:
   * As users complete their vesting period and attempt to claim tokens via the `unstake()` function, the contract will lack sufficient tokens to fulfill these claims
   * The contract would eventually become insolvent, unable to honor user withdrawals
   * This creates a "bank run" scenario where users who unstake early receive their tokens, while later users receive nothing

The vulnerability represents a Critical severity issue as it leads to direct theft of user funds, creates systemic insolvency, and undermines the core economic guarantees of the protocol. The impact is immediate and requires no complex conditions to exploit - simply normal operation of the contract by managers who believe they are distributing legitimate rewards.

## References

none

## Proof of Concept

## Proof of Concept

add following test in `test/StakeV2.test.sol`

```solidity
contract StakeV2_RewardDistributionYeetVulnerability is Test, StakeV2_BaseTest {
    function test_executeRewardDistributionYeet_vulnerability() public {
        address attacker = makeAddr("attacker");
        address victim = makeAddr("victim");
        address kodiakVault = makeAddr("kodiakVault");
        
        vm.mockCall(
            kodiakVault,
            abi.encodeWithSignature("token0()"),
            abi.encode(address(token))
        );
        
        vm.mockCall(
            kodiakVault,
            abi.encodeWithSignature("token1()"),
            abi.encode(address(wbera))
        );
        
        vm.startPrank(victim);
        token.mint(victim, 100 ether);
        token.approve(address(stakeV2), 100 ether);
        stakeV2.stake(100 ether);
        vm.stopPrank();
        
        assertEq(stakeV2.balanceOf(victim), 100 ether);
        assertEq(stakeV2.totalSupply(), 100 ether);
        
        vm.startPrank(victim);
        stakeV2.startUnstake(100 ether);
        vm.stopPrank();
        
        // Verify victim's tokens are now in vesting period
        StakeV2.Vesting[] memory victimVestings = stakeV2.getVestings(victim);
        assertEq(victimVestings.length, 1);
        assertEq(victimVestings[0].amount, 100 ether);
        
        // Key point: accumulatedDeptRewardsYeet() incorrectly treats vesting tokens as "excess rewards"
        uint256 wronglyCalculatedRewards = stakeV2.accumulatedDeptRewardsYeet();
        assertEq(wronglyCalculatedRewards, 100 ether, "Vulnerability confirmed: vesting tokens incorrectly treated as distributable rewards");
        
        // Set mockZapper return values to simulate successful distribution
        mockZapper.setReturnValues(0, 100 ether);
        
        // Manager executes reward distribution, incorrectly using victim's vesting tokens as rewards
        vm.prank(address(this)); // Using test contract as manager
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(100 ether, 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(kodiakVault, 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );
        
        // At this point, tokens in the contract should have been transferred to mockZapper
        assertEq(token.balanceOf(address(stakeV2)), 0, "All tokens (including vesting tokens) have been transferred out");
        
        vm.warp(block.timestamp + 11 days); // Fast forward past vesting period
        vm.startPrank(victim);
        
        // This should fail because there are no tokens left in the contract
        vm.expectRevert(); // Expect any type of revert
        stakeV2.unstake(0);
        
        vm.stopPrank();
        
        victimVestings = stakeV2.getVestings(victim);
        assertEq(victimVestings.length, 1, "Vesting records still exist");
        assertEq(token.balanceOf(address(stakeV2)), 0, "But there are no tokens in the contract to return");
    }
}
```

run the test with `forge test ./test/StakeV2.test.sol --match-test test_executeRewardDistributionYeet_vulnerability -vv`

output:

```
Ran 1 test for test/StakeV2.test.sol:StakeV2_RewardDistributionYeetVulnerability
[PASS] test_executeRewardDistributionYeet_vulnerability() (gas: 343375)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.49ms (1.26ms CPU time)

Ran 1 test suite in 10.10ms (3.49ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
