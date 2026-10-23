# 52706 sc low multi quantity prize claims revert until all winners are drawn freezing early winners

**Submitted on Aug 12th 2025 at 14:36:49 UTC by @wellbyt3 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52706
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:** Temporary freezing of funds for at least 1 hour

## Description

### Brief/Intro

`claimPrize()` reverts for multi-quantity prizes until all winners are drawn, so early winners (e.g., week 1 of a 4-week raffle) can’t claim until the final draw, temporarily freezing their prize.

### Vulnerability Details

When a raffle participant wins a prize, they call `claimPrize()` to claim the prize they won.

However, this call will revert if the quantity of the prize is > 1 and all the winners haven't been drawn:

```solidity
function claimPrize(uint256 prizeId, uint256 winnerIndex) external {
     if (prizes[prizeId].isActive && winnersDrawn[prizeId] < prizes[prizeId].quantity) {
            revert WinnerNotDrawn();
        }
...SNIP...
}
```

This causes an issue if the same prize is intended to be distributed (for example once a week for 4 weeks).

A user who wins the prize in week 1 won't be able to claim until after the week 4 prize has been drawn, temporarily freezing the prize they're entitled to.

### Impact Details

Temporary freezing of funds (i.e., raffle prizes).

### References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Raffle.sol#L298-L301>

## Proof of Concept

<details>

<summary>Repro test (Forge)</summary>

Add a new `.t.sol` file to /plume/test/, paste the code below, then run:

forge test --mt test\_userCannotClaimPrizeIfNotAllWinnersDrawn -vv

{% code title="UserCannotClaimPrizeIfNotAllWinnersDrawn.t.sol" %}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "../src/spin/Spin.sol";
import "../src/spin/Raffle.sol";
import "../src/spin/DateTime.sol";
import "../src/interfaces/ISupraRouterContract.sol";
import "forge-std/Test.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract MockSupraRouter  {

    uint256 public nonce;

    function generateRequest(string memory callbackSignature, uint8 rngCount, uint256 numConfirmations, uint256 clientSeed, address admin) external returns (uint256) {
        nonce++;
        return nonce;
    }
}


contract UserCannotClaimPrizeIfNotAllWinnersDrawn is Test {
    address public admin = makeAddr("admin");
    address public user = makeAddr("user");
    Spin spin;
    DateTime dateTime;
    MockSupraRouter supraRouter;
    Raffle raffle;

    function setUp() public {
        vm.startPrank(admin);
        supraRouter = new MockSupraRouter();
        spin = new Spin();
        dateTime = new DateTime();
        raffle = new Raffle();
        raffle.initialize(address(spin), address(supraRouter));

        spin.initialize(address(supraRouter), address(dateTime));

        uint256[3] memory newPlumeAmounts = [uint256(100), uint256(200), uint256(300)];
        spin.setPlumeAmounts(newPlumeAmounts);
        
        spin.setRaffleContract(address(raffle));

        uint16 year = 2025;
        uint8 month = 8;
        uint8 day = 8;
        uint8 hour = 10;
        uint8 minute = 0;
        uint8 second = 0;
        // Set up spin with date March 8, 2025 10:00:00
        vm.warp(dateTime.toTimestamp(year, month, day, hour, minute, second));

        spin.setCampaignStartDate(block.timestamp);

        spin.setEnableSpin(true);
        vm.stopPrank();

    }

    function test_userCannotClaimPrizeIfNotAllWinnersDrawn() public {
        deal(user, 100e18);

        // 1. User spins
        vm.prank(user);
        spin.startSpin{value: 2 ether}();

        // 2. User wins raffle tickets
        vm.prank(address(supraRouter));
        uint256[] memory randomNumberJackpot = new uint256[](1);
        randomNumberJackpot[0] = 500_000; 
        spin.handleRandomness(1, randomNumberJackpot);

        // 3. Admin adds a prize with a quantity of 2
        vm.startPrank(admin);
        raffle.addPrize("Prize", "Prize", 100, 2);
        vm.stopPrank();

        // 4. User spends raffle tickets on prize
        vm.prank(user);
        raffle.spendRaffle(1, 8);

        // 5. Admin Request winner
        vm.prank(admin);
        raffle.requestWinner(1);

        // 6. Callback logic handles winner, which is user
        vm.startPrank(address(supraRouter));
        uint256[] memory randomNumberPrize = new uint256[](1);
        randomNumberPrize[0] = 500_000; 
        raffle.handleWinnerSelection(2, randomNumberPrize);
        Raffle.Winner[] memory winners = raffle.getPrizeWinners(1);
        address winnerAddress = winners[0].winnerAddress;
        assertEq(winnerAddress, user);
        vm.stopPrank();

        // 7. User tried to claim prize but can't because the 2nd winner hasn't been drawn for the prize
        vm.prank(user);
        vm.expectRevert(Raffle.WinnerNotDrawn.selector);
        raffle.claimPrize(1, 0);
    }
}
```

{% endcode %}

</details>

## Reproduction Steps

{% stepper %}
{% step %}

### Step 1 — Setup and spin

* Fund user with test ETH and have them call `spin.startSpin{value: 2 ether}()`.
* Simulate randomness callback so the user acquires raffle tickets.
  {% endstep %}

{% step %}

### Step 2 — Add multi-quantity prize

* Admin adds a prize with `quantity = 2` via `raffle.addPrize(...)`.
  {% endstep %}

{% step %}

### Step 3 — Spend raffle tickets

* The user spends raffle tickets on the prize (`raffle.spendRaffle(prizeId, amount)`).
  {% endstep %}

{% step %}

### Step 4 — Request and draw first winner

* Admin requests a winner for the prize (`raffle.requestWinner(prizeId)`).
* Simulate the RNG callback and ensure the first winner is the user.
  {% endstep %}

{% step %}

### Step 5 — Attempt to claim

* The user calls `raffle.claimPrize(prizeId, winnerIndex)` and the call reverts with `WinnerNotDrawn` because `winnersDrawn[prizeId] < prizes[prizeId].quantity`.
  {% endstep %}
  {% endstepper %}

## Notes

* This is a logical bug that causes early winners of multi-quantity prizes to be unable to claim their prize until the final winner has been drawn (temporarily freezing the prize).
* Severity assessed as Low, impact is temporary freezing of prizes until draws complete.
