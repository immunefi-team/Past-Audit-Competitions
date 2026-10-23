# #42345 \[SC-Critical] Theft of User Funds in executeRewardDistributionYeet During Vesting Period

**Submitted on Mar 23rd 2025 at 06:16:35 UTC by @testnate for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42345
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `executeRewardDistributionYeet` function allows managers to distribute tokens that belong to users who have initiated the unstaking process but are still in the vesting period. This can lead to theft of users' funds, as demonstrated in the test case `test_fundsLostDuringVestingPeriod`. When users call `startUnstake`, their tokens are put into a vesting period but remain in the contract, where they're incorrectly counted as available rewards that managers can distribute.

## Vulnerability Details

The issue occurs due to how `accumulatedDeptRewardsYeet` calculates "accumulated rewards":

```sol
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return stakingToken.balanceOf(address(this)) - totalSupply;
}
```

This calculation assumes that any tokens in the contract beyond the `totalSupply` are excess rewards. However, this is incorrect because it also includes tokens that are in vesting periods from users who have called `startUnstake`.

The vulnerability sequence:

1. When a user calls `startUnstake`:

```sol
function startUnstake(uint256 unStakeAmount) external {
    // ...
    balanceOf[msg.sender] -= unStakeAmount;
    totalSupply -= unStakeAmount;
    // ...
    vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
    // ...
}
```

Their tokens are removed from `balanceOf` and `totalSupply`, but remain in the contract, marked only by a vestings struct.

2. `executeRewardDistributionYeet` can then distribute these tokens:

```sol
function executeRewardDistributionYeet(
    IZapper.SingleTokenSwap calldata swap,
    IZapper.KodiakVaultStakingParams calldata stakingParams,
    IZapper.VaultDepositParams calldata vaultParams
) external onlyManager nonReentrant {
    uint256 accRevToken0 = accumulatedDeptRewardsYeet();
    // ...
    stakingToken.approve(address(zapper), accRevToken0);
    // ...
    // Calls to zapper that send the tokens
}
```

A malicious or careless manager can distribute tokens that actually belong to users in the vesting period, causing them to permanently lose these funds.

## Impact Details

This is a critical vulnerability that allows managers to steal or accidentally distribute users' funds:

* Loss of User Funds: Users who have called `startUnstake` can lose 100% of their tokens in the vesting period if a manager calls `executeRewardDistributionYeet`.
* Scale of Impact: The issue affects all users who enter the vesting period through `startUnstake`. Since the vesting period is 10 days long, this creates a large window where users' funds are at risk.
* No Recovery Mechanism: Once tokens are sent to the zapper, there's no way for users to recover them. When they attempt to call `unstake` after the vesting period, the transaction fails due to insufficient token balance in the contract.

This is particularly problematic because users reasonably expect their tokens to be returned after the vesting period, but this vulnerability breaks this fundamental contract guarantee.

## Proof of Concept

## Proof of Concept

Add this contract in `StakeV2.test.sol`

```sol
contract StartUnstakeVulnerabilityTest is Test, StakeV2_BaseTest {
    address user;
    KodiakVaultV1 kodiakVault;

    function setUp() public override {
        super.setUp();

        // Setup test user
        user = makeAddr("User");
        token.mint(user, 100 ether);

        // Setup KodiakVault for zapper
        kodiakVault = new KodiakVaultV1(token, wbera);

        // Configure zapper
        mockZapper.setReturnValues(1 ether, 1 ether);
    }

    function test_fundsLostDuringVestingPeriod() public {
        // 1. User stakes tokens
        vm.startPrank(user);
        token.approve(address(stakeV2), 50 ether);
        stakeV2.stake(50 ether);
        vm.stopPrank();

        // 2. User starts unstaking (enters vesting period)
        vm.prank(user);
        stakeV2.startUnstake(50 ether);

        // 3. Verify contract state after startUnstake
        assertEq(stakeV2.balanceOf(user), 0 ether, "User should have 0 ether still staked");
        assertEq(stakeV2.totalSupply(), 0 ether, "Total supply should be 0 ether");
        assertEq(token.balanceOf(address(stakeV2)), 50 ether, "Contract should hold all 50 ether of tokens");

        // 4. Check that unstaking tokens are counted as "rewards"
        assertEq(stakeV2.accumulatedDeptRewardsYeet(), 50 ether, "Unstaking tokens incorrectly counted as rewards");

        // 5. Manager executes reward distribution using the unstaking tokens
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(50 ether, 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );

        // 6. Verify tokens were sent to zapper (permanently lost to the user)
        assertEq(token.balanceOf(address(stakeV2)), 0 ether, "Contract should now hold only 0 ether");
        assertEq(token.balanceOf(address(mockZapper)), 50 ether, "Zapper should have received 50 ether");

        // 7. Fast forward past vesting period
        vm.warp(block.timestamp + 10 days + 1);

        // 8. User attempts to unstake but it fails due to missing tokens
        vm.startPrank(user);
        vm.expectRevert();
        stakeV2.unstake(0);
        vm.stopPrank();
    }
}
```

Run `forge test --use 0.8.20 -vvv --mt test_fundsLostDuringVestingPeriod`
