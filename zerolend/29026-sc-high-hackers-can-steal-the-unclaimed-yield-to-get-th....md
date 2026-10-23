# 29026 - \[SC - High] Hackers can steal the unclaimed yield to get th...

## Hackers can steal the unclaimed yield to get the double reward from the Poolvoter contract by calling distributeEx() 2 times in the same block

Submitted on Mar 5th 2024 at 00:43:27 UTC by @perseverance for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29026

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Theft of unclaimed yield

### Description

## Description

### Brief/Intro

Hackers can steal the unclaimed yield from the Poolvoter contract using the function distributeEx() 2 times in the same block. By doing this, the hacker can get double the reward from the protocol. Thus, steal the yield from other users.

### Vulnerability Details

#### Background Information

The Zerolend provide the Incentive program as described in the Documentation of the protocol: <https://docs.zerolend.xyz/zeronomics/token-overview>

```
3. Incentive Programs: The protocol will conduct various incentive programs to encourage users to engage with the platform actively. These programs can offer token rewards or other benefits to participants who perform certain actions, such as providing liquidity, referring new users, or completing specific tasks within the ecosystem.

```

According to the Zerolend Boost Technical Walkthrough, this is done via the Poolvoter contract <https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol>

The contract will receive the reward via the function notifyRewardAmount

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L154-L158>

```solidity
function notifyRewardAmount(uint256 amount) public nonReentrant {
        reward.safeTransferFrom(msg.sender, address(this), amount); // transfer the distro in
        uint256 _ratio = (amount * 1e18) / totalWeight; // 1e18 adjustment is removed during claim
        if (_ratio > 0) index += _ratio;
    }

```

So the program will give the rewards to participants who perform actions such as providing liquidity. The contract will provide the rewards for all the pools in the list:

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L22>

```solidity
address[] internal _pools; // all pools viable for incentives

```

The reward is distributed based on the weights\[pool]/totalWeight. The formula

```solidity

uint256 _reward = (_balance * weights[_pools[x]]) /
                    _totalWeight;
```

this can be seen in the function distributeEx

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L214-L235>

```solidity
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

#### The vulnerability

So the token is distributed to all the pools in the list based on the portion weights\[\_pools\[pool]]) / \_totalWeight. But the bug in the function distributeEx allows a pool to receive the reward of other pools by calling distributeEx repeatedly at least 2 times. How many times depend on the supply and borrow gauge contract. I analyzed with the current code, it is possible to call 2 times.

If a hacker deposit or borrow into a pool (POOL\_A) and can get 50% of reward of POOL\_A. According to the design of the protocol, if POOL\_A have weights is 10% of the total reward then the reward should be

REWARD\_AMOUNT \* 0.1 \* 0.5 = 0.05 \* REWARD\_AMOUNT

But by calling distributeEx 2 times, the hacker can get

REWARD\_AMOUT \* 0.2 \* 0.5 = 0.1 \* REWARD\_AMOUT = 2 times

So the profit can be 2 times

How:

Assume that there is 5 pools in the list of "\_pools" and the POOL\_A has index of 2

the Hacker call

```solidity

PoolVoter.distributeEx(reward_token,2,3); 
```

Balance reward\_token of this contract is REWARD\_AMOUNT. Since POOL\_A has weight of 10% then \_reward = 0.1 \* REWARD\_AMOUNT

The pool is the contract LendingPoolGauge <https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/LendingPoolGauge.sol#L20-L35>

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

So the reward is sent to supplyGauge and borrowGauge.

I analyzed the [LendingPoolGaugeFactory](https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/LendingPoolGaugeFactory.sol) , so the BorrowGauge and SupplyGauge is [GaugeIncentiveController contract](https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/GaugeIncentiveController.sol)

<https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/RewardBase.sol#L104-L131>

```solidity
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

On the first call to this function, the transaction call will go to branch

```solidity
 if (block.timestamp >= periodFinish[token]) {
            token.safeTransferFrom(msg.sender, address(this), amount);
            rewardRate[token] = amount / DURATION; // Supposed amount = x = 0.1*REWARD_AMOUNT*0.25 for supply or 0.1*REWARD_AMOUNT*0.75 for borrow
        }

    lastUpdateTime[token] = block.timestamp;
     periodFinish[token] = block.timestamp + DURATION;
```

When the hacker call distributeEx() the second time, the the transaction call will go to the second branch

```solidity

else {
            uint256 _remaining = periodFinish[token] - block.timestamp;
            uint256 _left = _remaining * rewardRate[token];
            if (amount < _left) {
                return false; // don't revert to help distribute run through its tokens
            }
            token.safeTransferFrom(msg.sender, address(this), amount); // Supposed amount = x = 0.1*REWARD_AMOUNT*0.25 for supply or 0.1*REWARD_AMOUNT*0.75 for borrow
            rewardRate[token] = (amount + _left) / DURATION; // Supposed rewardRate[token] = 2* amount /DURATION ; //  ( x = 0.1*REWARD_AMOUNT*0.25 for supply or 0.1*REWARD_AMOUNT*0.75 for borrow) 
        }

        lastUpdateTime[token] = block.timestamp;
        periodFinish[token] = block.timestamp + DURATION;

```

So we can see that this bug allow the hackers to double the reward get from the protocol.

## Impacts

## About the severity assessment

This bug allow the hackers to double the reward get from the protocol and thus steal yield from other users.

This bug is high category "Theft of unclaimed yield"

### Proof of concept

## Proof of concept

Step 1: Wait the reward token is distributed to the PoolVoter contract Step 2: Deploy a contract. In this contract have the attack function

```solidity
function attack() public {


// Step 2: Hackers call with POOL_A has the index of 2 in pool array in PoolVoter contract

PoolVoter.distributeEx(reward_token,2,3); 

// Step 3: Hacker call the function again in the same block. 

PoolVoter.distributeEx(reward_token,2,3); 


} 

```

Step 3: Call getReward to take the reward <https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/RewardBase.sol#L79>

```solidity
function getReward(
        address account,
        IERC20 token
    ) 
```

It seems that the contract PoolVoter not yet deployed, so I provide the above POC to demonstrate the bug. Please let me know if you need more clarification.
