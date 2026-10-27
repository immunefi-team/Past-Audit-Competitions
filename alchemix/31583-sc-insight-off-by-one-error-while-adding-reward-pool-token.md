# 31583 - \[SC - Insight] Off by one error while adding reward pool token

Submitted on May 21st 2024 at 14:45:01 UTC by @sss for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31583

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardPoolManager.sol>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

The provided code snippet is from a Solidity function `RewardPoolManager::_addRewardPoolToken()`, which adds reward pool tokens to a contract. It contains an off-by-one error in the check for the maximum number of reward pool tokens.

## Vulnerability Details

* The code checks if the number of reward pool tokens is less than `MAX_REWARD_POOL_TOKENS`.
* However, the condition should be `<=` rather than `<`, as the maximum count should be inclusive.

## Impact Details

* Due to the off-by-one error, the contract allows adding one more reward pool token than intended.
* This may lead to unexpected behavior such as exceeding storage limits or unexpected gas costs.
* It could potentially disrupt the functionality of the contract or introduce vulnerabilities in token management.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardPoolManager.sol#L13> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardPoolManager.sol#L145>

```solidity
    uint256 internal constant MAX_REWARD_POOL_TOKENS = 10;

    function _addRewardPoolToken(address token) internal {
        if (!isRewardPoolToken[token] && token != address(0)) {
            require(rewardPoolTokens.length < MAX_REWARD_POOL_TOKENS, "too many reward pool tokens");// @audit  off by one error

            isRewardPoolToken[token] = true;
            rewardPoolTokens.push(token);
        }
    }
```

**fix:**

```diff
    uint256 internal constant MAX_REWARD_POOL_TOKENS = 10;

    function _addRewardPoolToken(address token) internal {
        if (!isRewardPoolToken[token] && token != address(0)) {
--            require(rewardPoolTokens.length < MAX_REWARD_POOL_TOKENS, "too many reward pool tokens");
++            require(rewardPoolTokens.length <= MAX_REWARD_POOL_TOKENS, "too many reward pool tokens");

            isRewardPoolToken[token] = true;
            rewardPoolTokens.push(token);
        }
    }
```

## Proof of Concept

on test file `RewardPoolManagerTest.t.sol` the test for max token is done but it is incomplete add these two lines to add two more tokens which makes total of 10 tokens which is max but it reverts and run poc

```diff
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract RewardPoolManagerTest is BaseTest {
    function setUp() public {
        setupContracts(block.timestamp);
    }

    function testAdminFunctions() public {
        address admin = rewardPoolManager.admin();

        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.setAdmin(devmsig);

        hevm.prank(admin);
        hevm.expectRevert(abi.encodePacked("not pending admin"));
        rewardPoolManager.acceptAdmin();

        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.setTreasury(devmsig);

        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.setRewardPool(devmsig);

        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.setPoolToken(devmsig);

        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.setVeALCX(devmsig);

        hevm.prank(admin);
        rewardPoolManager.setAdmin(devmsig);

        hevm.startPrank(devmsig);
        rewardPoolManager.acceptAdmin();

        rewardPoolManager.setTreasury(devmsig);

        rewardPoolManager.setRewardPool(devmsig);

        rewardPoolManager.setPoolToken(address(usdc));

        rewardPoolManager.setVeALCX(address(minter));

        hevm.stopPrank();
    }

    function testDepositIntoRewardPoolError() public {
        hevm.expectRevert(abi.encodePacked("must be veALCX"));
        rewardPoolManager.depositIntoRewardPool(TOKEN_1);
    }

    function testWithdrawFromRewardPool() public {
        hevm.expectRevert(abi.encodePacked("must be veALCX"));
        rewardPoolManager.withdrawFromRewardPool(1000);
    }

    // Test depositing, withdrawing from a rewardPool (Aura pool)
    function testRewardPool() public {
        // Reward pool should be set
        assertEq(rewardPool, rewardPoolManager.rewardPool());

        deal(bpt, address(rewardPoolManager), TOKEN_1);

        // Initial amount of bal and aura rewards earned
        uint256 rewardBalanceBefore1 = IERC20(bal).balanceOf(admin);
        uint256 rewardBalanceBefore2 = IERC20(aura).balanceOf(admin);
        assertEq(rewardBalanceBefore1, 0, "rewardBalanceBefore1 should be 0");
        assertEq(rewardBalanceBefore2, 0, "rewardBalanceBefore2 should be 0");

        // Initial BPT balance of rewardPoolManager
        uint256 amount = IERC20(bpt).balanceOf(address(rewardPoolManager));
        assertEq(amount, TOKEN_1);

        // Deposit BPT balance into rewardPool
        hevm.prank(address(veALCX));
        rewardPoolManager.depositIntoRewardPool(amount);

        uint256 amountAfterDeposit = IERC20(bpt).balanceOf(address(rewardPoolManager));
        assertEq(amountAfterDeposit, 0, "full balance should be deposited");

        uint256 rewardPoolBalance = IRewardPool4626(rewardPool).balanceOf(address(rewardPoolManager));
        assertEq(rewardPoolBalance, amount, "rewardPool balance should equal amount deposited");

        // Fast forward to accumulate rewards
        hevm.warp(block.timestamp + 2 weeks);

        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.claimRewardPoolRewards();

        hevm.prank(admin);
        rewardPoolManager.claimRewardPoolRewards();
        uint256 rewardBalanceAfter1 = IERC20(bal).balanceOf(address(admin));
        uint256 rewardBalanceAfter2 = IERC20(aura).balanceOf(address(admin));

        // After claiming rewards admin bal balance should increase
        assertGt(rewardBalanceAfter1, rewardBalanceBefore1, "should accumulate bal rewards");
        assertGt(rewardBalanceAfter2, rewardBalanceBefore2, "should accumulate aura rewards");

        hevm.prank(address(veALCX));
        rewardPoolManager.withdrawFromRewardPool(amount);

        // veALCX BPT balance should equal original amount after withdrawing from rewardPool
        uint256 amountAfterWithdraw = IERC20(bpt).balanceOf(address(veALCX));
        assertEq(amountAfterWithdraw, amount, "should equal original amount");

        // Only rewardPoolManager admin can update rewardPool
        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.setRewardPool(sushiPoolAddress);

        hevm.prank(admin);
        rewardPoolManager.setRewardPool(sushiPoolAddress);

        // Reward pool should update
        assertEq(sushiPoolAddress, rewardPoolManager.rewardPool(), "rewardPool not updated");
    }

    function testUpdatingRewardPoolTokens() public {
        address admin = rewardPoolManager.admin();

        address[] memory tokens = new address[](2);
        tokens[0] = dai;
        tokens[1] = usdt;

        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.swapOutRewardPoolToken(0, bal, usdc);

        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.addRewardPoolTokens(tokens);

        hevm.expectRevert(abi.encodePacked("not admin"));
        rewardPoolManager.addRewardPoolToken(dai);

        hevm.startPrank(admin);

        hevm.expectRevert(abi.encodePacked("incorrect token"));
        rewardPoolManager.swapOutRewardPoolToken(0, dai, usdc);

        rewardPoolManager.swapOutRewardPoolToken(0, bal, usdc);
        assertEq(rewardPoolManager.rewardPoolTokens(0), usdc, "rewardPoolTokens[0] should be usdc");

        rewardPoolManager.addRewardPoolTokens(tokens);
        assertEq(rewardPoolManager.rewardPoolTokens(2), dai, "rewardPoolTokens[2] should be dai");
        assertEq(rewardPoolManager.rewardPoolTokens(3), usdt, "rewardPoolTokens[3] should be usdt");
    }

    function testMaxRewardPoolTokens() public {
        address[] memory tokens = new address[](10);
        tokens[0] = dai;
        tokens[1] = usdt;
        tokens[2] = usdc;
        tokens[3] = bpt;
        tokens[4] = time;
        tokens[5] = aleth;
        tokens[6] = alusd3crv;
        tokens[7] = alusd;
++        tokens[8] = address(0x11);
++        tokens[9] = address(0x12);
        hevm.prank(admin);
        rewardPoolManager.addRewardPoolTokens(tokens);

        hevm.prank(admin);
        hevm.expectRevert(abi.encodePacked("too many reward pool tokens"));
        rewardPoolManager.addRewardPoolToken(beef);
    }
}

```
