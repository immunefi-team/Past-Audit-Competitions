# #41559 \[SC-Critical] Incorrect Calculation of Accumulated Rewards Due to Unstaked Tokens

**Submitted on Mar 16th 2025 at 14:53:39 UTC by @aksoy for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41559
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The accumulatedDeptRewardsYeet function incorrectly calculates the accumulated rewards by not accounting for unstaked tokens that are still in the contract. This discrepancy arises because totalSupply is reduced when users initiate unstaking, but the tokens remain in the contract until the vesting period ends. As a result, the function may return an inflated reward amount, leading to excessive distribution in executeRewardDistributionYeet, which could ultimately deplete user funds and prevent some users from unstaking their tokens.

## Vulnerability Details

The vulnerability lies in the accumulatedDeptRewardsYeet function. stakingToken.balanceOf(address(this)) returns the total balance of staking tokens held by the contract, while totalSupply represents the total amount of staked tokens. However, totalSupply is reduced immediately when a user initiates unstaking via the startUnstake function, but the tokens are not transferred out of the contract until the vesting period ends. This means that stakingToken.balanceOf(address(this)) will still include the unstaked tokens, leading to an overestimation of the accumulated rewards.

```solidity
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }

    function startUnstake(uint256 unStakeAmount) external {
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");
        balanceOf[msg.sender] -= unStakeAmount;
        totalSupply -= unStakeAmount;
    }

```

## Impact Details

The contract could distribute more rewards than it actually has, leading to a depletion of funds. Users who unstake later may find that there are insufficient tokens in the contract to cover their unstaking requests.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L149>

## Proof of Concept

## Proof of Concept

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

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
        address owner = address(this);
        address manager = address(this);
        mockZapper = new SimpleZapperMock(token, wbera);
        stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));
    }

    function test() public {}
}

contract StakeV2_HandleExcessDebt is Test {
    MockERC20 public token;
    MockWETH public wbera;

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
    }

    // make sure we can handle excess yeet when its token0
    function test_accumulatedDeptRewardsYeet() public {
        address owner = address(this);
        address manager = address(this);
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

        token.mint(address(this), 100 ether);
        token.approve(address(stakeV2), 50 ether);
        stakeV2.stake(50 ether);

        // simulate debt by adding excess token0
        token.transfer(address(stakeV2), 50 ether);
        //zapper
        mockZapper.setReturnValues(1, 1); // does not matter

        assertEq(50 ether, stakeV2.accumulatedDeptRewardsYeet());

        //@audit user unstake some amounts
        stakeV2.startUnstake(40 ether);

        //@audit accumulatedDeptRewardsYeet is now higher than it should be includes unstaked tokens
        assertEq(90 ether, stakeV2.accumulatedDeptRewardsYeet());

        stakeV2.depositReward{value: 1 ether}();

        assertEq(100 ether, token.balanceOf(address(stakeV2)));

        //@audit manager distributes all accumulatedDeptRewardsYeet which also sends unstaked tokens
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(90 ether, 0, 0, address(0), ""),
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
        
        //@audit there should be 10+40 tokens left for user but only 10 tokens left
        assertEq(10 ether, token.balanceOf(address(stakeV2)));
        assertEq(90 ether, token.balanceOf(address(mockZapper)));

        vm.warp(block.timestamp + 20 days);
        //@audit there is not enough stake tokens left 
        vm.expectRevert();
        stakeV2.unstake(0);
    }
}
```
