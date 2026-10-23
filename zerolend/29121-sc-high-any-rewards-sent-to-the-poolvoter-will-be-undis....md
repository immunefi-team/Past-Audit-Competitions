# 29121 - \[SC - High] Any rewards sent to the PoolVoter will be undis...

Submitted on Mar 7th 2024 at 18:36:15 UTC by @Trust for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29121

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The PoolVoter distributes reward tokens to different gauges. Gauges are registered below:

```
function registerGauge(
    address _asset,
    address _gauge
) external onlyOwner returns (address) {
    if (isPool[_asset]) {
        _pools.push(_asset);
        isPool[_asset] = true;
    }
    bribes[_gauge] = address(0);
    gauges[_asset] = _gauge;
    poolForGauge[_gauge] = _asset;
    _updateFor(_gauge);
    return _gauge;
}
```

Distribution of rewards other than `reward` token are done through `distributeEx()`:

```
function distributeEx(
    address token,
    uint256 start,
    uint256 finish
) public nonReentrant {
    uint256 _balance = IERC20(token).balanceOf(address(this));
    if (_balance > 0 && totalWeight > 0) {
        uint256 _totalWeight = totalWeight;
        for (uint256 x = start; x < finish; x++) {
            uint256 _reward = (_balance * weights[_pools[x]]) /
                _totalWeight;
            if (_reward > 0) {
                address _gauge = gauges[_pools[x]];
                IERC20(token).approve(_gauge, 0); // first set to 0, this helps reset some non-standard tokens
                IERC20(token).approve(_gauge, _reward);
                IGauge(_gauge).notifyRewardAmount(token, _reward); // can return false, will simply not distribute tokens
            }
        }
    }
}
```

The weights are accessed according to the `_pools[]` entry populated by `registerGauge()`

## Vulnerability Details

Distribution will always fail because there's wrong logic when registering gauges. Specifically this part will never be executed:

```
if (isPool[_asset]) {
    _pools.push(_asset);
    isPool[_asset] = true;
}
```

The intention is to use `!isPool[_asset]`.

This means \_pools will never be populated. After funds are sent to the PoolVoter for dispatch, they can never be claimed by any gauge. There is no escape hatch to unfreeze the rewards.

## Impact Details

Rewards sent to the PoolVoter will be forever stuck.

## Proof of Concept

The POC is implemented in a single file. Simply run DistributorPOC's `PoolFailPOC()` which shows that `_pools` stays empty.

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
        owner = msg.sender;
    }

    modifier onlyOwner() {
        if (msg.sender != owner)
            revert("Not owner");
        _;
    }

    mapping(address => uint256) public claimable;
    IERC20 public reward;
    address owner;

    address[] internal _pools; // all pools viable for incentives
    mapping(address => address) public gauges; // pool => gauge
    mapping(address => bool) public isPool; // pool => bool
    mapping(address => address) public poolForGauge; // pool => gauge
    mapping(address => address) public bribes; // gauge => bribe
    mapping(address => uint256) public weights; // pool => weight
    mapping(address => mapping(address => uint256)) public votes; // nft => votes
    mapping(address => address[]) public poolVote; // nft => pools
    mapping(address => uint256) public usedWeights; // nft => total voting weight of user
    uint256 public index;
    mapping(address => uint256) public supplyIndex;
    
    function pools() external view returns (address[] memory) {
        return _pools;
    }


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

    function registerGauge(
        address _asset,
        address _gauge
    ) external onlyOwner returns (address) {
        if (isPool[_asset]) {
            _pools.push(_asset);
            isPool[_asset] = true;
        }
        bribes[_gauge] = address(0);
        gauges[_asset] = _gauge;
        poolForGauge[_gauge] = _asset;
        //_updateFor(_gauge);
        return _gauge;
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

    function PoolFailPOC() external {
        require(d.pools().length == 0);
        d.registerGauge(address(0), address(1));
        require(d.pools().length == 0);

    }
}
```
