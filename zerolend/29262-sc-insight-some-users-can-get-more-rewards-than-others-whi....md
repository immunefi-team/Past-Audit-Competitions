# 29262 - \[SC - Insight] Some users can get more rewards than others whi...

Submitted on Mar 12th 2024 at 12:49:20 UTC by @oxumarkhatab for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29262

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/zerolend/governance>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

The notifyReward Allows the reward amount for a specific token for multiple times in 14 days Duration. This opens up a weakness of the system using which an attacker can earn more rewards for exact same time as others.

## Why this impact?

Essentially the attacker can get more of the yield than other users. So Attacker is Thefting the yield ( Just in my opinion ) This is the most relevant impact I found so consider it.

## Vulnerability Details

This Vulnerability details section contains the PoC itself too because the only way to show this weakness is to crunch the numbers before you.

In short , if a user has tokens for earning rewards at the time the notifyReward is called , they will get less tokens than a person who stakes tokens at the time after the second notifyReward function call is made to the GuageIncentiveController contract within 14 days and not after 14 days of initial call because second call in the interval \[0,14 days) allows setting greater rewardRate for a token ( same amount passed in function call ) that will allow the attacker to gain more rewards `after the second notifyReward function call` than other people who has staked `before the second notifyReward function call`

The main lines of concern are setting the rewardRate lines

```
 rewardRate[token] = amount  / DURATION;
```

and

```
 rewardRate[token] = (amount + _left) / DURATION;
```

Let's see this in action

## PoC

The rewards calculation for GuageIncentiveController is unfair

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

LEt's say just after the notifyReward function call , Alice has get's her 100e18 of tokens from somewhere and willing to stake for one day.

LEt's say the notifyReward amount is 10e18

```
  if (block.timestamp >= periodFinish[token]) {
            token.safeTransferFrom(msg.sender, address(this), amount);
            rewardRate[token] = amount / DURATION;
        }
```

```
         rewardRate[token] = amount / DURATION;

```

And the periodFinish is set to `block.timestamp+Deadline`

She calculates the reward and gets some tokens on `A` amount of tokens on the rate of 10e18/14 days = 8.2671958e+12

\=> rewardRate = 8e12

Now some time passes , as the Duration is 14 days long , a malicious actor Bob can monitor the mempool for notifyReward for the next notifyReward call within next 13.9 days.

When the notifyReward is called ( Let's say for the amount is same as 10e18 ) , block.timestamp < PeriodFinish clearly , so first condition is falsified

```
 if (block.timestamp >= periodFinish[token]) {
            token.safeTransferFrom(msg.sender, address(this), amount);
            rewardRate[token] = amount / DURATION;
        }
```

The second condition is executed

```
else {
            uint256 _remaining = periodFinish[token] - block.timestamp;
            uint256 _left = _remaining * rewardRate[token];
            if (amount < _left) {
                return false; // don't revert to help distribute run through its tokens
            }
            token.safeTransferFrom(msg.sender, address(this), amount);
            rewardRate[token] = (amount + _left) / DURATION;
        }
```

Let's say notifyReward is called after one day , \_remaining = 13 days = 86400\*13 = 1,123,200 \_left = 1,123,200 \* 8e12 = 9e18

clearly , amount > \_left , 10e18 > 9e18 so following condition does not execute and new rate is set

```
   if (amount < _left) {
                return false; // don't revert to help distribute run through its tokens
            }

```

Carefully see how the rate is being set:

```
rewardRate[token] = (amount + _left) / DURATION;

```

rewardRate\[token] = (10e18+9e18) / DURATION = 19e18/14 days = 2.1990741e+14 = 2e14

Now when Bob stakes tokens , he gets far more tokens than alice

See 8e12 < 2e14 , This amount is substantially large when accumulate with each passing moment.

so using a simpler formula of elapsedTime and rewardPerToken ( the protocol does slightly different but essense is same)

### Alice keeps tokens for 1 day = 86400 seconds :

Alice's rewards = (86400\*8e12) = 6.912e+17 = Solidity integer rounding down = 6e17

### Bob's keeps tokens for 1 day = 86400 seconds :

Bob's rewards = (86400\*2e14) = 1.728e+19 = Solidity integer rounding down = 1e19

Thus Bob is able to get more rewards for the same time as Alice's. This is a source of unfair distribution.

## Impact Details

* Unfair rewards distribution between users
* Theft of rewards

## Proof of concept

Disclaimer: This is essentially the same PoC in Vulnerability Details section. If you've read that, you can safely ignore this one

### PoC

The rewards calculation for GuageIncentiveController is unfair

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

LEt's say just after the notifyReward function call , Alice has get's her 100e18 of tokens from somewhere and willing to stake for one day.

LEt's say the notifyReward amount is 10e18

```
  if (block.timestamp >= periodFinish[token]) {
            token.safeTransferFrom(msg.sender, address(this), amount);
            rewardRate[token] = amount / DURATION;
        }
```

```
         rewardRate[token] = amount / DURATION;

```

And the periodFinish is set to `block.timestamp+Deadline`

She calculates the reward and gets some tokens on `A` amount of tokens on the rate of 10e18/14 days = 8.2671958e+12

\=> rewardRate = 8e12

Now some time passes , as the Duration is 14 days long , a malicious actor Bob can monitor the mempool for notifyReward for the next notifyReward call within next 13.9 days.

When the notifyReward is called ( Let's say for the amount is same as 10e18 ) , block.timestamp < PeriodFinish clearly , so first condition is falsified

```
 if (block.timestamp >= periodFinish[token]) {
            token.safeTransferFrom(msg.sender, address(this), amount);
            rewardRate[token] = amount / DURATION;
        }
```

The second condition is executed

```
else {
            uint256 _remaining = periodFinish[token] - block.timestamp;
            uint256 _left = _remaining * rewardRate[token];
            if (amount < _left) {
                return false; // don't revert to help distribute run through its tokens
            }
            token.safeTransferFrom(msg.sender, address(this), amount);
            rewardRate[token] = (amount + _left) / DURATION;
        }
```

Let's say notifyReward is called after one day , \_remaining = 13 days = 86400\*13 = 1,123,200 \_left = 1,123,200 \* 8e12 = 9e18

clearly , amount > \_left , 10e18 > 9e18 so following condition does not execute and new rate is set

```
   if (amount < _left) {
                return false; // don't revert to help distribute run through its tokens
            }

```

Carefully see how the rate is being set:

```
rewardRate[token] = (amount + _left) / DURATION;

```

rewardRate\[token] = (10e18+9e18) / DURATION = 19e18/14 days = 2.1990741e+14 = 2e14

Now when Bob stakes tokens , he gets far more tokens than alice

See 8e12 < 2e14 , This amount is substantially large when accumulate with each passing moment.

so using a simpler formula of elapsedTime and rewardPerToken ( the protocol does slightly different but essense is same)

### Alice keeps tokens for 1 day = 86400 seconds :

Alice's rewards = (86400\*8e12) = 6.912e+17 = Solidity integer rounding down = 6e17

### Bob's keeps tokens for 1 day = 86400 seconds :

Bob's rewards = (86400\*2e14) = 1.728e+19 = Solidity integer rounding down = 1e19

Thus Bob is able to get more rewards for the same time as Alice's. This is a source of unfair distribution.
