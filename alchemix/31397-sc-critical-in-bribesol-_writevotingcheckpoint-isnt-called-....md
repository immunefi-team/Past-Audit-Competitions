# 31397 - \[SC - Critical] In Bribesol \_writeVotingCheckpoint isnt called ...

Submitted on May 18th 2024 at 01:42:23 UTC by @Praise for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31397

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* necessary updates aren't done

## Description

## Brief/Intro

in Bribe.sol, `_writeVotingCheckpoint()` isn't called to update `votingCheckpoints` and `votingNumCheckpoints` whenever votes are withdrawn or there's a reset

## Vulnerability Details

In `Bribe.deposit()`, whenever votes are allocated to a given guage `totalVoting` is updated with the amount and `_writeVotingCheckpoint()` is called to update `votingCheckpoints` and `votingNumCheckpoints`.

```solidity
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

Now the issue lies in `Bribe.withdraw()` & `Bribe.resetVoting()`, where necessary updates aren't done.

1. in `Bribe.withdraw()` whenever votes are withdrawn from a given guage, the withdrawn votes aren't deducted from `totalVoting` and `_writeVotingCheckpoint()` isn't called to update `votingCheckpoints` and `votingNumCheckpoints`

```solidity
    function withdraw(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply -= amount;
        balanceOf[tokenId] -= amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();
       

        emit Withdraw(msg.sender, tokenId, amount);
    }
```

2. In `Bribe.resetVoting()` when `totalVoting` is reset by making it 0, `_writeVotingCheckpoint()` isn't called to update `votingCheckpoints` and `votingNumCheckpoints`

```solidity
   function resetVoting() external {
        require(msg.sender == voter);
        totalVoting = 0;
    }
```

So whenever `Bribe.withdraw()` / `Bribe.resetVoting()` is done, record of balance checkpoints for voting period is not updated.

## Impact Details

1. After `Bribe.withdraw()` is done, withdrawn votes doesn't reflect on `totalVoting`. This is wrong
2. resetting of votes is never updated in `votingCheckpoints` and `votingNumCheckpoints`

Necessary updates aren't done after such trivial operations.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L319>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L332>

## Proof of Concept

```solidity
    function testBribe_sol() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);

        address bribeAddress = voter.bribes(address(sushiGauge));

        // voter deposits votes to bribe.
        hevm.startPrank(address(voter));

        Bribe(bribeAddress).deposit(1000e18, tokenId1);

        uint256 amt1 = Bribe(bribeAddress).totalVoting();

        console.log("total Voting after Deposit", amt1);


        // voter withdraws votes from bribe BUT it doesn't reflect in totalVoting
        Bribe(bribeAddress).withdraw(1000e18, tokenId1);

        uint256 amt2 = Bribe(bribeAddress).totalVoting();

        console.log("total Voting after withdraw", amt2);
        hevm.stopPrank();

        ///scenario to show totalVoting still exists in `votingCheckpoints` even after resetVoting()

        hevm.startPrank(address(voter));

        Bribe(bribeAddress).deposit(1000e18, tokenId1);

        uint256 amt1a = Bribe(bribeAddress).totalVoting();

        console.log("total Voting after Deposit", amt1a);

        Bribe(bribeAddress).resetVoting();

        (uint256 timeHere, uint256 amt2a) = Bribe(bribeAddress).votingCheckpoints(0);

        console.log("total Voting still existing in votingCheckpoints", amt2a);


    }
```
