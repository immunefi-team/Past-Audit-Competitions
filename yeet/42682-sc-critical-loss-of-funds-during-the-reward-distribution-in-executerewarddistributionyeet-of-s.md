# #42682 \[SC-Critical] Loss of funds during the reward distribution in executeRewardDistributionYeet() of StakeV2 contract

**Submitted on Mar 25th 2025 at 11:00:23 UTC by @x60scs for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42682
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Not taking into account the tokens in the vesting process will cause the distribution not to occur as intended.

## Vulnerability Details

The startUnstake process starts a 10-day vesting period. Then unstaking becomes possible. Alternatively, unstaking is possible before the vesting period ends with rageQuit().

However, this reward distribution (executeRewardDistributionYeet function) does not take into account the tokens in the vesting process, and causes the tokens in the vesting process to be distributed.

This causes the "stakingToken.balanceOf(address(this))" value to be lower than totalSupply after users unstake. In other words, it causes a loss of funds.

## Impact Details

Tokens in the vesting period are included in the reward distribution, which will result in a loss of funds.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L149\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L158>

## Recommendation

In the StakeV2 contract, a variable named totalStartUnstakeToken should be defined. This variable should be incremented by the unStakeAmount during the startUnstake process, and decremented when an unstake is performed. Additionally, when calculating accumulatedDeptRewardsYeet, the totalStartUnstakeToken value should be taken into account.

```diff
    /// @notice The total supply of the staking token
    uint256 public totalSupply;
    
+   uint256 public totalStartUnstakeToken;
```

```diff
	function accumulatedDeptRewardsYeet() public view returns (uint256) {
-        return stakingToken.balanceOf(address(this)) - totalSupply;
+        return stakingToken.balanceOf(address(this)) - totalSupply - totalStartUnstakeToken;
    }
```

```diff
	function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); // DOS protection https://github.com/Enigma-Dark/Yeet/issues/12
        _updateRewards(msg.sender);
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");

        balanceOf[msg.sender] -= unStakeAmount;
        totalSupply -= unStakeAmount;
+       totalStartUnstakeToken += unStakeAmount;
    ...
```

```diff
   function _unstake(uint256 index) private {
        Vesting memory vesting = vestings[msg.sender][index];
+		totalStartUnstakeToken -= vesting.amount;
        (uint256 unlockedAmount, uint256 lockedAmount) = calculateVesting(vesting);
        require(unlockedAmount != 0, "No unlocked amount");
    ...
```

## Proof of Concept

## Proof of Concept

1. Staker1 and Staker2 users stake 50 tokens.
2. Staker1 user calls the startUnstake function to unstake 50 tokens and the 10-day vesting period begins.
3. The Manager performs the distribution with the executeRewardDistributionYeet() function according to the result obtained from the accumulatedDeptRewardsYeet() function.
4. After 10 days, Staker1 unstakes its tokens. The StakeV2 balance is reset.
5. Staker2 user calls the startUnstake function to unstake 50 tokens and the 10-day period begins.
6. After 10 days, Staker2 user cannot unstake his tokens. Tokens are no longer available in StakeV2.

```js
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

contract StakeV2_POC is Test {
    MockERC20 public token;
    MockWETH public wbera;

    error ERC20InsufficientBalance(
        address sender,
        uint256 balance,
        uint256 needed
    );

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
    }

    function test_poc() public {
        address owner = makeAddr("owner");
        address manager = makeAddr("manager");
        address staker1 = makeAddr("staker_one");
        address staker2 = makeAddr("staker_two");

        KodiakVaultV1 kodiakVault = new KodiakVaultV1(token, wbera);
        SimpleZapperMock mockZapper = new SimpleZapperMock(
            kodiakVault.token0(),
            kodiakVault.token1()
        );
        StakeV2 stakeV2 = new StakeV2(
            token,
            mockZapper,
            owner,
            manager,
            IWETH(wbera)
        );

        vm.startPrank(staker1);
        token.mint(address(staker1), 50 ether);
        token.approve(address(stakeV2), 50 ether);
        stakeV2.stake(50 ether);
        vm.stopPrank();

        vm.startPrank(staker2);
        token.mint(address(staker2), 50 ether);
        token.approve(address(stakeV2), 50 ether);
        stakeV2.stake(50 ether);
        vm.stopPrank();

        mockZapper.setReturnValues(1, 1);

        assertEq(100 ether, token.balanceOf(address(stakeV2)));

        console.log(
            "accumulatedDeptRewardsYeet before unstake= %s",
            stakeV2.accumulatedDeptRewardsYeet()
        );
        console.log("totalSupply before unstake= %s", stakeV2.totalSupply());

        vm.startPrank(staker1);
        stakeV2.startUnstake(50 ether);
        vm.stopPrank();

        console.log(
            "accumulatedDeptRewardsYeet after unstake= %s",
            stakeV2.accumulatedDeptRewardsYeet()
        );
        console.log("totalSupply after unstake= %s", stakeV2.totalSupply());

        vm.prank(manager);
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(50 ether, 0, 0, address(0), ""),
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

        console.log(
            "accumulatedDeptRewardsYeet after executeRewardDistributionYeet= %s",
            stakeV2.accumulatedDeptRewardsYeet()
        );
        console.log(
            "totalSupply after executeRewardDistributionYeet= %s",
            stakeV2.totalSupply()
        );

        vm.warp(block.timestamp + 11 days);

        vm.prank(staker1);
        stakeV2.unstake(0);

        console.log(
            "stakeV2 balance after unstake=%s",
            token.balanceOf(address(stakeV2))
        );
        console.log("totalSupply after unstake= %s", stakeV2.totalSupply());

        vm.startPrank(staker2);
        stakeV2.startUnstake(50 ether);

        vm.warp(block.timestamp + 11 days);

        bytes memory _error = abi.encodeWithSelector(
            ERC20InsufficientBalance.selector,
            address(stakeV2),
            0 ether,
            50 ether
        );
        vm.expectRevert(_error);
        stakeV2.unstake(0);
        vm.stopPrank();
    }
}
```
