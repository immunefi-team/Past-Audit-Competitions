# 31258 - \[SC - High] Loss of Unclaimed Bribes After Burning veALCX T...

Submitted on May 15th 2024 at 21:09:32 UTC by @Limbooo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31258

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Introduction

This report details a vulnerability discovered in the VotingEscrow\.sol contract of the Alchemix V2 DAO. The issue arises when users withdraw their locked veALCX tokens. During this process, unclaimed rewards are intended to be claimed, but the system fails to account for the potential bribes earned through voting interactions. Consequently, users who withdraw their veALCX tokens lose their right to claim these bribes, leading to permanent loss of rewards.

## Vulnerability Details

When a user withdraws their locked veALCX tokens (interacting with `VotingEscrow::withdraw`), the contract ensures that any unclaimed ALCX rewards and FLUX are claimed before the token is burned, resulting in the user losing their control over the token as they are no longer its owner.

```solidity
src/VotingEscrow.sol:
  741:     function withdraw(uint256 _tokenId) public nonreentrant {
..SNIP..
@>767:         // Claim any unclaimed ALCX rewards and FLUX
  768:         IRewardsDistributor(distributor).claim(_tokenId, false);
  769:         IFluxToken(FLUX).claimFlux(_tokenId, IFluxToken(FLUX).getUnclaimedFlux(_tokenId));
  770: 
  771:         // Burn the token
@>772:         _burn(_tokenId, value);
  773: 
  774:         emit Withdraw(msg.sender, _tokenId, value, block.timestamp);
  775:     }
..SNIP..
  1558:     function _burn(uint256 _tokenId, uint256 _value) internal {
  1559:         address owner = ownerOf(_tokenId);
..SNIP..
  1570:         // Remove token
@>1571:         _removeTokenFrom(owner, _tokenId);
  1572:         emit Transfer(owner, address(0), _tokenId);
  1573:         emit Supply(supplyBefore, supplyAfter);
  1574:     }
```

While this procedure is generally acceptable, an issue arises when the user has interacted with the `Voter` contract and voted for pools (the user may have used their FLUX to boost their votes). The bribes earned from these votes will be lost if the user withdraws their token and subsequently attempts to claim their bribes. This is because `Voter::claimBribes` checks the ownership status of the token, and after the token is burned, the user is no longer considered its owner, preventing them from claiming their rewards.

```solidity
src/Voter.sol:
  332:     function claimBribes(address[] memory _bribes, address[][] memory _tokens, uint256 _tokenId) external {
@>333:         require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId));
  334: 
  335:         for (uint256 i = 0; i < _bribes.length; i++) {
  336:             IBribe(_bribes[i]).getRewardForOwner(_tokenId, _tokens[i]);
  337:         }
  338:     }
```

## Impact Details

The primary impact of this vulnerability is the permanent loss of bribes for users who withdraw their veALCX tokens. This occurs because the ownership check in the `Voter::claimBribes` function fails after the token is burned. Consequently, users are unable to claim rewards they have rightfully earned, leading to dissatisfaction and potential loss of trust in the protocol.

## Mitigation Analysis

To mitigate this issue, it is recommended to enhance the withdrawal process to ensure that there are no unclaimed bribes before allowing the token to be burned. Here are a few suggested approaches:

1. **Prevent Withdrawal if Bribes are Unclaimed**: Implement a check in the `VotingEscrow::withdraw` function to prevent the withdrawal if there are unclaimed bribes. This ensures that users must claim their bribes before they can withdraw and burn their veALCX tokens.
2. **Force Bribe Claiming During Withdrawal**: Similar to how unclaimed ALCX rewards and FLUX are claimed during withdrawal, modify the withdrawal process to enforce the claiming of any unclaimed bribes. This would involve adding logic to claim bribes within the `withdraw` function, ensuring users receive all due rewards before their token is burned.
3. **New Restriction of ClaimBribes Function**: A new layer of security replaces the current check, could involve restricting who can call the `Voter::claimBribes` function to ensure that only valid claims are processed. However, this might be less effective than ensuring bribes are claimed during the withdrawal process. Also, it may has some drawbacks and establish a new way to manipulate the flow of voter contract specialty for cases like this issue where the veALCX is burned or ended (I remember proofing a vulnerability and it was prevent by the check of ownabilty of the token in `claimBribes`).

## References

* VotingEscrow\.sol#L737-L775: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L737-L775>
* VotingEscrow\.sol#L1558-L1575: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1558-L1575>
* Voter.sol#L331-L339: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L331-L339>

## Proof of concept

### Test Case (Foundry)

The test can be added to a new file under the current test suite `src/test/VotingPoC.t.sol`, then specify the file name in `FILE` flag under `Makefile` configuration. Run using `make test_file`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract VotingPoCTest is BaseTest {
    address public alice;
    address public bob;

    function setUp() public {
        setupContracts(block.timestamp);

        // Setup Alice address
        alice = hevm.addr(uint256(keccak256(abi.encodePacked('Alice'))));
        vm.label(alice, 'Alice');
    }

    function testParmentFreesingOfBribesAfterWithdrowLocks() public {
        uint256 firstPeriodStart = minter.activePeriod();

        // Forwards 1 day from the begining of the current epoch.
        hevm.warp(firstPeriodStart + 1 days);

        // Mint new veALCX token for Alice with 1e18 amount and 3 weeks locks time.
        uint256 tokenId = createVeAlcx(alice, TOKEN_1, 3 weeks, false);

        address bribeAddress = voter.bribes(address(sushiGauge));
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 10000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        // Alice Vote
        hevm.prank(alice);
        voter.vote(tokenId, pools, weights, 0);

        // Reward amount
        uint256 rewardAmount = TOKEN_100K;
        // Notify bribe for reward amount
        createThirdPartyBribe(bribeAddress, bal, rewardAmount);

        // Next epoch started
        hevm.warp(firstPeriodStart + 2 weeks +  1 seconds );
        voter.distribute();

        // Check Alice has earned a reward
        assertEq(IBribe(bribeAddress).earned(bal, tokenId), rewardAmount);

        // Forwards to the end of Alice's veALCX locks end
        hevm.warp(firstPeriodStart + 3 weeks + 1 seconds );
        // Check that Alice's veALCX had ended
        assertGt(block.timestamp, veALCX.lockEnd(tokenId));

        // Alice decided to withdraw his locks
        // First he need to reset his vote statues
        hevm.prank(alice);
        voter.reset(tokenId);
        // Then, start cooldown of his veALCX
        hevm.prank(alice);
        veALCX.startCooldown(tokenId);
        // Now he should wait for 1 week untill cooldowd ends to be able to withdraw
        hevm.warp(firstPeriodStart + 4 weeks + 1 seconds);
        // Since we enter the next epoch, distribute (not needed but it will called in real world scenario)
        voter.distribute();
        // Save the earned bribes before withdrawing
        uint256 bribesErnedBeforeWithdraw = IBribe(bribeAddress).earned(bal, tokenId);
        // Check that the rewards equal to the first epoch reward amount.
        assertEq(bribesErnedBeforeWithdraw, rewardAmount);

        // Now Alice's veALCX is withdrawable at current moment.
        hevm.prank(alice);
        veALCX.withdraw(tokenId);

        // Compare the reward earned before and after withdrawing.
        uint256 bribesErnedAfterWithdraw = IBribe(bribeAddress).earned(bal, tokenId);
        assertEq(bribesErnedBeforeWithdraw, bribesErnedAfterWithdraw);
        // Make sure that the current reward earnd is more than zero
        assertGt(bribesErnedAfterWithdraw, 0);

        // Now if Alice try to claim his bribes, it will revert.
        hevm.prank(alice);
        hevm.expectRevert();
        voter.claimBribes(bribes, tokens, tokenId);
        // This happning because after withdrawing he lost his rights on the veALCX token, since it will be burned.
        // Here we check that he is no longer the owner of the token.
        assertFalse(veALCX.isApprovedOrOwner(alice, tokenId));
        // While this is not an issue, he should lose the rights of controling the token after withdrawing his locks,
        // but the issue is that he lost his bribes.
    }
}
```

#### Test Output

```bash
alchemix-v2-dao main 1m44s
❯ make test_file
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.g.alchemy.com/v2/*** --match-path src/test/VotingPoC.t.sol -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/VotingPoC.t.sol:VotingPoCTest
[PASS] testParmentFreesingOfBribesAfterWithdrowLocks() (gas: 5702775)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 94.97s (82.32s CPU time)

Ran 1 test suite in 96.74s (94.97s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
