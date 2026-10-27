# 31555 - \[SC - Low] RewardsDistributoramountToCompound - L The stal...

Submitted on May 21st 2024 at 08:32:49 UTC by @OxSCSamurai for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31555

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

## Summary:

* Due to seemingly accidental oversight the `staleThreshold` variable was accidentally left at a value suitable for testing phase only. The original value was 30 days, but the current value is 60 days.
* 30 days and older would normally be deemed stale prices, but since 60 days is currently used it means only 60 days and older would now be deemed stale prices, so if prices between 30 days and 60 days are used, they will be accepted as valid prices when they are actually stale prices, and the protocol/users wont be aware of the problem.

## Vulnerability Details

## Impact Details

## Impact:

Most likely impact in scope: Contract fails to deliver promised returns, but doesn't lose value

Likelihood: medium Impact: high Severity: low - medium

Potential impacts:

* compounding lower/higher amount than expected versus if the price was not stale, so the user/voter would get either more or less rewards than they should have received
* potential protocol instability due to unstable/invalid pricing

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/9e14da88d8db05794623d8ab5f449451a10c15ac/src/RewardsDistributor.sol#L118>

## Proof of Concept

## PoC:

I used the existing unmodified protocol test:

* `Minter.t.sol::testCompoundRewards()`
* and test run command: `make test_file_debug_test FILE=Minter TEST=testCompoundRewards`

Function under test:

* I made some modifications for PoC testing purposes, to simplify the testing process, see my audit tag comments below:

```solidity
    function amountToCompound(uint256 _alcxAmount) public view returns (uint256, uint256[] memory) {
        // Increased for testing since tests go into future
        uint256 staleThreshold = 60 days; /// @audit-issue probably incorrect due to above comment, and previous value was: `uint256 staleThreshold = 30 days;`
        //uint256 staleThreshold = 30 days; /// @audit added for PoC/testing purposes

        (uint80 roundId, int256 alcxEthPrice, , uint256 priceTimestamp, uint80 answeredInRound) = priceFeed
            .latestRoundData();

        priceTimestamp = block.timestamp - 59 days; /// @audit added for PoC/testing purposes >>> overriding the variable from above for the specific PoC test
        require(answeredInRound >= roundId, "Stale price");
        require(block.timestamp - priceTimestamp < staleThreshold, "Price is stale");
        require(alcxEthPrice > 0, "Chainlink answer reporting 0");

        uint256[] memory normalizedWeights = IManagedPool(address(balancerPool)).getNormalizedWeights();

        uint256 amount = (((_alcxAmount * uint256(alcxEthPrice)) / 1 ether) * normalizedWeights[0]) /
            normalizedWeights[1];

        return (amount, normalizedWeights);
    }
```

### Test 1: running the test with the 60 days stale threshold active, and making sure that the `priceTimestamp` is 59 days old, as can see from above modifications:

```solidity
    │   │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], value: 1485894423105353936225 [1.485e21])
    │   │   │   │   ├─ emit Deposit(caller: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], owner: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], assets: 1485894423105353936225 [1.485e21], shares: 1485894423105353936225 [1.485e21])
    │   │   │   │   ├─ emit Staked(user: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], amount: 1485894423105353936225 [1.485e21])
    │   │   │   │   └─ ← [Return] 1485894423105353936225 [1.485e21]
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit Deposit(provider: RewardsDistributor: [0xB545eE1E4F6d34f4dc9780d9b0291dda0bec0Ba2], tokenId: 1, value: 1485894423105353936225 [1.485e21], locktime: 1747267200 [1.747e9], maxLockEnabled: false, depositType: 0, ts: 1717632001 [1.717e9])
    │   │   ├─ emit Supply(prevSupply: 2000000000000000000 [2e18], supply: 1487894423105353936225 [1.487e21])
    │   │   └─ ← [Stop] 
    │   ├─ [1501] 0x8392F6669292fA56123F71949B52d883aE57e225::fallback{value: 95280000936693794260}()
    │   │   ├─ emit Deposit(param0: RewardsDistributor: [0xB545eE1E4F6d34f4dc9780d9b0291dda0bec0Ba2], param1: 95280000936693794260 [9.528e19])
    │   │   └─ ← [Stop] 
    │   └─ ← [Return] 2728698496087689546026 [2.728e21]
    └─ ← [Stop] 

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 114.09s (95.38s CPU time)

Ran 1 test suite in 116.65s (114.09s CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

* The test completed successfully, so a very stale price of 59 days old was used, and the system accepted it, which should have been an invalid price due to extreme staleness.

### Test 2: now running the test with the correct 30 days stale threshold value, and keeping everything else same as for above test, should now revert as expected:

```solidity
.
.
.
.
.
.
    ├─ [58847] RewardsDistributor::claim(1, true)
    │   ├─ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [611] VotingEscrow::ownerOf(1) [staticcall]
    │   │   └─ ← [Return] 0x8392F6669292fA56123F71949B52d883aE57e225
    │   ├─ [593] VotingEscrow::userPointEpoch(1) [staticcall]
    │   │   └─ ← [Return] 1
    │   ├─ [1337] VotingEscrow::getUserPointHistory(1, 1) [staticcall]
    │   │   └─ ← [Return] Point({ bias: 1966696410435137323 [1.966e18], slope: 63419583967 [6.341e10], ts: 1716256331 [1.716e9], blk: 19915119 [1.991e7] })
    │   ├─ emit Claimed(tokenId: 1, amount: 2728698496087689546026 [2.728e21], claimEpoch: 1, maxEpoch: 1)
    │   ├─ [15735] 0x194a9AaF2e0b67c35915cD01101585A33Fe25CAa::latestRoundData() [staticcall]
    │   │   ├─ [7502] 0x74263dB73076C1389d12e5F8ff0E6a72AE86CA24::latestRoundData() [staticcall]
    │   │   │   └─ ← [Return] 1749, 6919048139724593 [6.919e15], 1716252431 [1.716e9], 1716252431 [1.716e9], 1749
    │   │   └─ ← [Return] 36893488147419104981 [3.689e19], 6919048139724593 [6.919e15], 1716252431 [1.716e9], 1716252431 [1.716e9], 36893488147419104981 [3.689e19]
    │   └─ ← [Revert] revert: Price is stale
    └─ ← [Revert] Error != expected error: Price is stale != insufficient balance to compound

[FAIL. Reason: Error != expected error: Price is stale != insufficient balance to compound] testCompoundRewardsFailureETH() (gas: 2234694)
.
.
.
.
.
.
    ├─ [58847] RewardsDistributor::claim(1, true)
    │   ├─ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [611] VotingEscrow::ownerOf(1) [staticcall]
    │   │   └─ ← [Return] 0x8392F6669292fA56123F71949B52d883aE57e225
    │   ├─ [593] VotingEscrow::userPointEpoch(1) [staticcall]
    │   │   └─ ← [Return] 1
    │   ├─ [1337] VotingEscrow::getUserPointHistory(1, 1) [staticcall]
    │   │   └─ ← [Return] Point({ bias: 1966696410435137323 [1.966e18], slope: 63419583967 [6.341e10], ts: 1716256331 [1.716e9], blk: 19915119 [1.991e7] })
    │   ├─ emit Claimed(tokenId: 1, amount: 2728698496087689546026 [2.728e21], claimEpoch: 1, maxEpoch: 1)
    │   ├─ [15735] 0x194a9AaF2e0b67c35915cD01101585A33Fe25CAa::latestRoundData() [staticcall]
    │   │   ├─ [7502] 0x74263dB73076C1389d12e5F8ff0E6a72AE86CA24::latestRoundData() [staticcall]
    │   │   │   └─ ← [Return] 1749, 6919048139724593 [6.919e15], 1716252431 [1.716e9], 1716252431 [1.716e9], 1749
    │   │   └─ ← [Return] 36893488147419104981 [3.689e19], 6919048139724593 [6.919e15], 1716252431 [1.716e9], 1716252431 [1.716e9], 36893488147419104981 [3.689e19]
    │   └─ ← [Revert] revert: Price is stale
    └─ ← [Revert] Error != expected error: Price is stale != insufficient balance to compound

Suite result: FAILED. 0 passed; 3 failed; 0 skipped; finished in 67.38s (15.50s CPU time)

Ran 1 test suite in 69.92s (67.38s CPU time): 0 tests passed, 3 failed, 0 skipped (3 total tests)

Failing tests:
Encountered 3 failing tests in src/test/Minter.t.sol:MinterTest
[FAIL. Reason: revert: Price is stale] testCompoundRewards() (gas: 2056546)
[FAIL. Reason: Error != expected error: Price is stale != insufficient balance to compound] testCompoundRewardsFailure() (gas: 2234141)
[FAIL. Reason: Error != expected error: Price is stale != insufficient balance to compound] testCompoundRewardsFailureETH() (gas: 2234694)

Encountered a total of 3 failing tests, 0 tests succeeded
make: *** [Makefile:74: test_file_debug_test] Error 1
```

* reverted as expected because of stale price over 30 days

### Suggested bugfix:

```diff
    function amountToCompound(uint256 _alcxAmount) public view returns (uint256, uint256[] memory) {
        // Increased for testing since tests go into future
-       uint256 staleThreshold = 60 days;
+       uint256 staleThreshold = 30 days;
```
