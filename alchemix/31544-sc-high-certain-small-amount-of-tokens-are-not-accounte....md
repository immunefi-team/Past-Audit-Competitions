# 31544 - \[SC - High] Certain small amount of tokens are not accounte...

Submitted on May 21st 2024 at 04:49:19 UTC by @jecikpo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31544

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Temporary freezing of funds for 12 hours

## Description

## Brief/Intro

When a sufficiently small amount of tokens is deposited and locked as veALCX it is not included into the bias due to division by `iMAXTIME`

## Vulnerability Details

When a new slope is calculated at `_checkpoint()` the difference in bias is divided through timestamp. If the value is smaller than the timestamp the result of division is zero, hence no bias increase is added to the checkpoint.

## Impact Details

The Voting power of a user is not increased despite increased deposit. While the impact is minuscule as the amount of tokens that can be lost is so small. It still affects the correctness of accounting. It should be fairly easy to prevent that from happening by establishing a minimum deposit requirement.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1157>

## Proof of Concept

Paste the following code into `Voting.t.sol`:

```
    function testVEVotePower4() public {
        //uint256 tokenId1 = createVeAlcx(admin, 15768000 - 1, 40 weeks, false);
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, 40 weeks, false);

        // Add BAL bribes to sushiGauge
        address bribeAddress = voter.bribes(address(sushiGauge));
        createThirdPartyBribe(bribeAddress, bal, TOKEN_1);

        // epoch 1
        hevm.warp(newEpoch());
        voter.distribute();
        

        console.log("Votes for admin before deposit before: %d", veALCX.getVotes(admin));
        depositFor(tokenId1, 1_000_000);
        console.log("Votes for admin after deposit after:  %d", veALCX.getVotes(admin));

        (uint256 amount,,,) = veALCX.locked(tokenId1);
        console.log("Locked amount: %d:", amount);
    }
```

We can see that that the amount of votes didn't change after additional tokens were deposited into a veALCX:

```
Ran 1 test for src/test/Voting.t.sol:VotingTest
[PASS] testVEVotePower4() (gas: 3940483)
Logs:
  Votes for admin before deposit 3 before: 1419178018760355233
  Votes for admin after deposit 3 after:  1419178018760355233
  Locked amount: 1000000000001000000:

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.32ms (9.64ms CPU time)
```
