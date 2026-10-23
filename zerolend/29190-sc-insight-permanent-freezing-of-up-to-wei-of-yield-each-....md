# 29190 - \[SC - Insight] Permanent freezing of up to wei of yield each ...

Submitted on Mar 10th 2024 at 01:46:37 UTC by @nethoxa for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29190

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/zerolend/governance>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Due to a rounding error when notifying a reward to `LendingPoolGauge`, up to `3 wei` of the used token will be locked forever in the contract.

## Vulnerability Details

It's well known Solidity rounds down on integer division. Because of that, in `LendingPoolGauge::notifyRewardAmount`, if the given `amount` is not divisible by `4`, up to `3 wei` of yield will be permanently locked in the contract as there is no way to take them back and the contract sends `amount / 4` to the `supplyGauge` and `amount / 4 * 3` to the `borrowGauge`:

```solidity
    function notifyRewardAmount(
        address token,
        uint256 amount
    ) external returns (bool) {
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        // send 1/4 to the supply side
        IERC20(token).approve(address(supplyGauge), amount);
        bool a = supplyGauge.notifyRewardAmount(token, amount / 4);

        // send 3/4th to the borrow side
        IERC20(token).approve(address(borrowGauge), amount);
        bool b = borrowGauge.notifyRewardAmount(token, (amount / 4) * 3); // @audit rounding, yield lost forever

        return a && b;
    }
```

It may not be a high amount for tokens with high decimals, but for other tokens like USDC (`6` decimals) or a variant of EURO which I do not remember, but had `2` decimals, it can be a significant loss of yield.

## Impact Details

Vanilla loss of yield, permanent as there is no way to take them back.

## Proof of Concept

The runnable POC is the next one:

```solidity
pragma solidity 0.8.20;

import {Test, console2} from "forge-std/Test.sol";

import {StakingBonus} from "src/vesting/StakingBonus.sol";
import {VestedZeroNFT} from "src/vesting/VestedZeroNFT.sol";

import {LockerToken} from "src/locker/LockerToken.sol";
import {LockerLP} from "src/locker/LockerLP.sol";
import {OmnichainStaking} from "src/locker/OmnichainStaking.sol";

import {PoolVoter} from "src/voter/PoolVoter.sol";
import {LendingPoolGauge} from "src/voter/gauge/LendingPoolGauge.sol";
import {RewardBase} from "src/voter/gauge/RewardBase.sol";

import {ZeroLend} from "src/ZeroLendToken.sol";

import {IVestedZeroNFT} from "src/interfaces/IVestedZeroNFT.sol";

import {IERC20} from "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract DummyGauge is RewardBase {
    function init(address _zero, address _vesting) external {
        __RewardBase_init(_zero, _vesting);
    }

    function rewardPerToken(IERC20 token) public view override returns (uint256) {
        return 0;
    }

    function earned(
        IERC20 token,
        address account
    ) public view override returns (uint256) {
        return 0;
    }

    modifier updateReward(IERC20 token, address account) override {
        _;
    }
}

contract POC is Test {

    function test_POC() external {
        address bob = makeAddr("bob");

        vm.startPrank(bob);

        StakingBonus bonus = new StakingBonus();
        VestedZeroNFT vZero = new VestedZeroNFT();
        LockerToken locker = new LockerToken();
        LockerLP lockerLP = new LockerLP();
        ZeroLend zero = new ZeroLend();
        OmnichainStaking staking = new OmnichainStaking();
        PoolVoter voter = new PoolVoter();
        DummyGauge dummyGauge1 = new DummyGauge();
        DummyGauge dummyGauge2 = new DummyGauge();
        LendingPoolGauge gauge = new LendingPoolGauge(address(dummyGauge1), address(dummyGauge2));

        bonus.init(address(zero), address(locker), address(vZero), 5); // 5% bonus, as StakingBonus::calculateBonus does the maths /100 instead of /10000
        vZero.init(address(zero), address(bonus));
        locker.init(address(zero), address(staking), address(bonus));
        staking.init(address(0), address(locker), address(lockerLP));
        lockerLP.init(address(zero), address(staking), address(bonus));
        voter.init(address(staking), address(zero));
        dummyGauge1.init(address(zero), address(vZero));
        dummyGauge2.init(address(zero), address(vZero));

        zero.togglePause(false);
        zero.approve(address(gauge), 1e18 + 3); // so that % 4 != 0 and the lost yield is 3 wei
        
        console2.log("\n\n");
        console2.log("[\x1b[32m+\x1b[0m] ZERO balance of Bob before the notify =\x1b[31m", zero.balanceOf(bob), "\x1b[0m");
        console2.log("[\x1b[32m+\x1b[0m] ZERO balance of LendingPoolGauge before the notify =\x1b[31m", zero.balanceOf(address(gauge)), "\x1b[0m");

        console2.log("");
        console2.log(" \x1b[33m-\x1b[0m Calling \x1b[32mLendingPoolGauge::notifyRewardAmount\x1b[0m with amount being \x1b[31m1e18 + 3\x1b[0m...");
        gauge.notifyRewardAmount(address(zero), 1e18 + 3);
        console2.log("");

        console2.log("[\x1b[32m+\x1b[0m] ZERO balance of Bob after the notify =\x1b[31m", zero.balanceOf(bob), "\x1b[0m");
        console2.log("[\x1b[32m+\x1b[0m] ZERO balance of LendingPoolGauge after the notify =\x1b[31m", zero.balanceOf(address(gauge)), "\x1b[0m");
        console2.log("\n\n");

        require(zero.balanceOf(address(gauge)) != 0, "POC");

        vm.stopPrank();
    }
}
```
