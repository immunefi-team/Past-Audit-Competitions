# #41911 \[SC-Critical] Unstake amount can be zapped before user withdrawal

**Submitted on Mar 19th 2025 at 10:06:16 UTC by @Ragnarok for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41911
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `StakeV2` contract has a flaw in the `startUnstake` function that can lead to a permanent loss of user funds. When users initiate an unstake request, the contract’s `totalSupply` decreases, but its `stakingToken` balance remains unchanged. This discrepancy causes the contract’s `accumulatedDeptRewardsYeet` to increase, creating a potential risk where a manager can use these funds before users can withdraw them.

## Vulnerability Details

When a user initiates an unstake, they must first call the `StakeV2::startUnstake` function. However, this function only decreases `totalSupply` without affecting the contract’s `stakingToken` balance.

`StakeV2::startUnstake` function:

```solidity
function startUnstake(uint256 unStakeAmount) external {
    ...
    // @note totalSupply decreases, but the YEET balance of this contract does not change
    // @note As a consequence, accumulatedDeptRewardsYeet increases (balance - totalSupply)
    // @audit Users lose funds if the manager calls executeRewardDistributionYeet.
    totalSupply -= unStakeAmount;
    ...
}
```

Because of this, the `StakeV2::accumulatedDeptRewardsYeet()` will increase by the same amount the user intends to unstake. The unstaked amount remains locked in the contract, users are allowed to withdraw all after 10 days.

However, what if, before the lock period ends, the manager calls `StakeV2::executeRewardDistributionYeet`, utilizing all `accumulatedDeptRewardsYeet()` funds. Then the unstake amount is zapped into the `Zapper` contract, permanently removing it from `StakeV2`. When the user attempts to withdraw their unstaked tokens after the waiting period, the contract no longer has the funds, leading to irrecoverable user losses.

## Impact Details

* The `StakeV2` contract will permanently lose funds allocated for unstaking.
* Some users may be unable to withdraw their full staked amount due to missing funds.

## Proof of Concept

## Proof of Concept

Put the following code into the `test/StakeV2.test.sol` file and run `forge test --mt test_unableToUnstake`.

```solidity
contract StakeV2_Unstake is Test {
    MockERC20 public token;
    MockWETH public wbera;
    SimpleZapperMock mockZapper;
    KodiakVaultV1 kodiakVault;
    StakeV2 stakeV2;

    address owner = makeAddr("owner");
    address manager = makeAddr("manager");
    address user1 = makeAddr("user1");
    address user2 = makeAddr("user2");

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
       
        kodiakVault = new KodiakVaultV1(token, wbera);
        mockZapper = new SimpleZapperMock(kodiakVault.token0(), kodiakVault.token1());
        stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));
    }

    function test_unableToUnstake() public {
        token.mint(user1, 100 ether);
        token.mint(user2, 50 ether);
        // Stake
        vm.startPrank(user1);
        token.approve(address(stakeV2), 100 ether);
        stakeV2.stake(100 ether);
        vm.stopPrank();

        vm.startPrank(user2);
        token.approve(address(stakeV2), 50 ether);
        stakeV2.stake(50 ether);
        vm.stopPrank();

        assertEq(token.balanceOf(address(stakeV2)), 150 ether);
        assertEq(stakeV2.accumulatedDeptRewardsYeet(), 0);

        // User1 starts unstake
        vm.startPrank(user1);
        stakeV2.startUnstake(100 ether);
        vm.stopPrank();

        assertEq(token.balanceOf(address(stakeV2)), 150 ether);
        assertEq(stakeV2.accumulatedDeptRewardsYeet(), 100 ether);

        // Distribute YEET
        vm.startPrank(manager);
        mockZapper.setReturnValues(1, 1); // does not matter
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(stakeV2.accumulatedDeptRewardsYeet(), 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );
        vm.stopPrank();

        assertEq(token.balanceOf(address(stakeV2)), 50 ether);

        // Unable to unstake
        vm.warp(block.timestamp + 11 days);
        vm.startPrank(user1);
        vm.expectRevert();
        stakeV2.unstake(0);
        vm.stopPrank();
    }
}
```
