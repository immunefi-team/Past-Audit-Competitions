# #41286 \[SC-Critical] \`accumulatedDeptRewardsYeet()\` accounts for tokens under unstaking process

**Submitted on Mar 13th 2025 at 10:45:12 UTC by @peppef for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41286
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Smart contract unable to operate due to lack of token funds

## Description

The function `accumulatedDeptRewardsYeet()` computes the surplus in $YEET token to be later distributed with `executeRewardDistributionYeet()` as the difference between the contract $YEET balance and `totalSupply`, namely the sum of the $YEET current staked in StakeV2 contract.

````
```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return stakingToken.balanceOf(address(this)) - totalSupply;
}
```
````

However if a user calls `startUnstake()` when he wants to start the unstaking process, his principal amount is removed from `totalSupply` straight away before the vesting period ends but that amount is still in the contract $YEET balance.

This means that `accumulatedDeptRewardsYeet()` returns an higher value than it should be and will assign it to `accRevToken0`. Then `executeRewardDistributionYeet()` may distribute part of user funds that should be returned to them after the vesting ends.

Under this circumstance that a manager calls `executeRewardDistributionYeet()` with wrong parameters that passes validations, both `rageQuit()` and `unstake()` will be unable to transfer back user stakes due to a lack of funds until someone (treasury or dev team) refund stakeV2 of the necessary $YEET.

## Proof of Concept

## Proof of Concept

A test is provided for that scenario. Run with `forge test --match-test test_issue8 -vvv`:

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
  
  contract issue8 is  Test {
      StakeV2 public stakeV2;
      MockERC20 public token;
      MockWETH public wbera;
      SimpleZapperMock public mockZapper;
  
      function setUp() public virtual {
          token = new MockERC20("MockERC20", "MockERC20", 18);
          wbera = new MockWETH();
          address owner = makeAddr("owner");
          address manager = makeAddr("manager");
          mockZapper = new SimpleZapperMock(token, wbera);
          stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));
      }
  
      function test_issue8() public {
          // stake 50e18 out of the minted 100e18
          address userA = makeAddr("userA");
          token.mint(userA, 100 ether);
          vm.startPrank(userA);
          token.approve(address(stakeV2), 50 ether);
          stakeV2.stake(50 ether);
          vm.stopPrank();
  
          // fund stakeV2 with some more YEET simulating the scenario of surplus $YEET in StakeV2 caused by zapper imprecisions
          address zapper = makeAddr("zapper");
          token.mint(zapper, 0.1 ether);
          vm.prank(zapper);
          token.transfer(address(stakeV2), 0.1 ether);
  
          /* asserting that:
              - stakeV2 has 50e18 token in stake
              - stakeV2 has 50.1e18 token in its balance
              - accumulatedDeptRewardsYeet returns 50.1e18 - 50e18 */
          assertEq(stakeV2.totalSupply(), 50 ether);
          assertEq(token.balanceOf(address(stakeV2)), 50.1 ether);
          assertEq(stakeV2.accumulatedDeptRewardsYeet(), 0.1 ether);
          
          // simulating that userA start the unstake process for 30e18 tokens out of 50e18 he deposited
          vm.prank(userA);
          stakeV2.startUnstake(30 ether);
  
          // asserting that totalSupply was decreased by startUnstake()
          assertEq(stakeV2.totalSupply(), 20 ether);
  
          // asserting that accumulatedDeptRewardsYeet returns a wrong value rather than 0.1 surplus from zapper
          assertEq(stakeV2.accumulatedDeptRewardsYeet(), 0.1 ether);
  
  
      }
  }
```
