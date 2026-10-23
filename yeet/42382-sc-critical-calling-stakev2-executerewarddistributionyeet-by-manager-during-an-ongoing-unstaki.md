# #42382 \[SC-Critical] Calling \`StakeV2::executeRewardDistributionYeet\` by manager during an ongoing unstaking period for stakers can result in them being unable to unstake permanently

**Submitted on Mar 23rd 2025 at 13:15:22 UTC by @hustling0x for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42382
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `StakeV2::executeRewardDistributionYeet` method uses `StakeV2::accumulatedDeptRewardsYeet` to determine the amount of $YEET to zap in and distribute as excess rewards. However, the `StakeV2::accumulatedDeptRewardsYeet` method falsely returns accumulated rewards in $YEET tokens when there are users who have requested to unstake via `StakeV2::startUnstake`. This means that every time `StakeV2::executeRewardDistributionYeet` is called by a manager, the tokens intended to be unstaked by the users will be sent to the Zapper and distributed as excess rewards, resulting in a permanent freeze and permanent inability for users to withdraw their staked $YEET positions.

## Vulnerability Details

When a user calls `StakeV2::startUnstake`, the `totalSupply` of staked tokens ($YEET) in the contract is reduced by the amount requested for unstake. This way, the calculation in `StakeV2::accumulatedDeptRewardsYeet` will return a false result, not taking into account the amounts pending to be unstaked.

For example:

1. A yeetard stakes 1000 $YEET -> `totalSupply` = 1000 $YEET, `accumulatedDeptRewardsYeet` returns 0;
2. He calls `StakeV2::startUnstake` to request unstake -> `totalSupply` = 0 $YEET, but now `accumulatedDeptRewardsYeet` returns 1000 $YEET, because of the calculation `stakingToken.balanceOf(address(this)) - totalSupply`;

This means at any time there are pending unstaking periods running for users and the manager calls `StakeV2::executeRewardDistributionYeet`, users' tokens will be zapped and distributed as excess rewards, and they will not be able to unstake them via `StakeV2::unstake` or `StakeV2::rageQuit`.

## Impact Details

Although the method `StakeV2::executeRewardDistributionYeet` is only callable by a manager, calling it while there are active unstaking periods ongoing will result in DoS and permanent freezing of users' staked funds.

## Recommended Mitigation

The proposed solution is to keep track of the total pending unstakes for the contract and deduct them in the calculation in `StakeV2::accumulatedDeptRewardsYeet`.

```diff
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
-        return stakingToken.balanceOf(address(this)) - totalSupply;
+        return stakingToken.balanceOf(address(this)) - totalSupply - totalPendingUnstakeAmount;
    }
```

The `totalPendingUnstakeAmount` state should be updated when `StakeV2::startUnstake` is called and when the user actually unstakes either via `StakeV2::unstake` or `StakeV2::rageQuit`.

## References

* <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L148>
* <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L158>
* <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L255>

## Proof of Concept

## Proof of Concept

```javascript
contract UnableToUnstakeTest is Test {
    StakeV2 public stakeV2;
    MockERC20 public token;
    MockWETH public wbera;

    SimpleZapperMock public mockZapper;
    KodiakVaultV1 kodiakVault;

    address staker1 = makeAddr("staker_one");
    address manager = makeAddr("manager");
    address owner = makeAddr("owner");

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();

        mockZapper = new SimpleZapperMock(token, wbera);
        mockZapper.setReturnValues(1, 1);

        stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));        
        kodiakVault = new KodiakVaultV1(token, wbera);
    }

    function test_UnableToUnstake() public {
        // Prank staker1
        vm.startPrank(staker1);

        // Mint 1000 $YEET to staker1
        token.mint(address(staker1), 1000 ether);

        // Approve stakeV2 contract to spend 1000 $YEET on behalf of staker1
        token.approve(address(stakeV2), 1000 ether);

        // Stake 1000 $YEET
        stakeV2.stake(1000 ether);

        // Cache the accumulated rewards, total supply and stakeV2 balance
        uint256 _accRewardsAfterStake = stakeV2.accumulatedRewards();
        uint256 _totalSupplyAfterStake = stakeV2.totalSupply();
        uint256 _stakeV2BalanceAfterStake = token.balanceOf(address(stakeV2));

        // Start the unstake process for 1000 $YEET
        stakeV2.startUnstake(1000 ether);

        // Cache the accumulated rewards and total supply after starting the unstake process
        uint256 _accRewardsAfterStartUnstake = stakeV2.accumulatedDeptRewardsYeet();
        uint256 _totalSupplyAfterStartUnstake = stakeV2.totalSupply();

        // Stop pranking staker1
        vm.stopPrank();

        // Prank manager
        vm.startPrank(manager);

        // Execute reward distribution yeet during the unstake process for users
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(1000 ether, 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(
                address(kodiakVault),
                0,
                0,
                0,
                0,
                0,
                address(0)
            ),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );

        // Cache the accumulated rewards, total supply and stakeV2 balance after reward distribution yeet
        uint256 _accRewardsAfterRewardDistributionYeet = stakeV2.accumulatedDeptRewardsYeet();
        uint256 _totalSupplyAfterRewardDistributionYeet = stakeV2.totalSupply();
        uint256 _stakeV2BalanceAfterRewardDistributionYeet = token.balanceOf(address(stakeV2));

        // Stop pranking manager
        vm.stopPrank();

        // Fast forward 10 days
        skip(10 days);

        // Expect revert when trying to unstake 1000 $YEET
        vm.expectRevert(abi.encodeWithSelector(IERC20Errors.ERC20InsufficientBalance.selector, address(stakeV2), 0, 1000 ether));     

        // Prank staker1
        vm.startPrank(staker1);

        // Unstake 1000 $YEET
        stakeV2.unstake(0);

        // Stop pranking staker1
        vm.stopPrank();

        console.log("Accumulated rewards after stake: ", _accRewardsAfterStake); // 0
        console.log("Total supply after stake: ", _totalSupplyAfterStake); // 1000000000000000000000
        console.log("StakeV2 balance after stake: ", _stakeV2BalanceAfterStake); //1000000000000000000000

        console.log("Accumulated rewards after start unstake: ", _accRewardsAfterStartUnstake); // 1000000000000000000000
        console.log("Total supply after start unstake: ", _totalSupplyAfterStartUnstake); // 0

        console.log("Accumulated rewards after reward distribution yeet: ", _accRewardsAfterRewardDistributionYeet); // 0
        console.log("Total supply after reward distribution yeet: ", _totalSupplyAfterRewardDistributionYeet); // 0
        console.log("StakeV2 balance after reward distribution yeet: ", _stakeV2BalanceAfterRewardDistributionYeet); // 0
    }
}
```
