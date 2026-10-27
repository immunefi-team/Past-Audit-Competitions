# 51951 sc low a global blocking check in claimprize prevents individual winner claims until all winners are drawn

**Submitted on Aug 6th 2025 at 19:57:08 UTC by @XDZIBECX for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51951
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

The `claimPrize` function contains a validation logic flaw that creates a global blocking mechanism preventing any winner from claiming their prize until all winners for that specific prize have been drawn.

The function checks two conditions together:

* whether the prize is still active: `prizes[prizeId].isActive`
* whether the number of winners drawn is less than the total quantity: `winnersDrawn[prizeId] < prizes[prizeId].quantity`

If both conditions are true, the function reverts and blocks the claim attempt entirely. This global check occurs before any individual winner validation, meaning a legitimately selected winner is blocked if other winners for the same prize haven't been drawn yet.

The bug stems from a misunderstanding of multi-winner behavior: treating all winners as a single unit rather than allowing individual winners to claim their prizes independently once they have been selected.

### Vulnerability Details

The issue in `claimPrize` prevents any winner from claiming their prize until all winners for a given prize have been drawn. This violates the intended multi-winner design documented for the raffle:

"Each winner must call claimPrize(prizeId, winnerIndex) to claim their specific prize. Since a prize can have multiple winners, the winnerIndex (starting from 0) is used to identify which winning slot is being claimed. Each winner's claim status is tracked independently. One user claiming their prize has no effect on the ability of other winners to claim theirs."

Vulnerable snippet (source pointer): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Raffle.sol#L299C5-L304C1>

Example problematic code:

```solidity
function claimPrize(uint256 prizeId, uint256 winnerIndex) external {
    if (prizes[prizeId].isActive && winnersDrawn[prizeId] < prizes[prizeId].quantity) {
        revert WinnerNotDrawn(); // <-- this global blocking check is vulnerable
    }
    
    Winner storage individualWin = prizeWinners[prizeId][winnerIndex]; // <-- individual validation continues
}
```

Because the contract reverts the claim for any winner if the prize is still active (meaning more winners are yet to be drawn), as long as any winner remains undrawn for a prize, no winner can claim—regardless of whether their own win has been selected and recorded in `prizeWinners`. There is no on-chain timeout, expiry, or bypass to allow individual winners to claim independently.

### Impact Details

This bug can permanently prevent legitimately selected winners from claiming their prize unless all winners for the prize are drawn. If the admin fails to draw all winners, prizes for that round could be frozen and unclaimable. Winners recorded in the contract would be blocked from accessing their rewards, potentially resulting in a permanent freeze of funds unless resolved via external intervention (e.g., contract upgrade).

{% hint style="warning" %}
Permanent freezing of funds is possible: a single missed/failed draw for a multi-winner prize can block all winners from claiming their recorded prizes.
{% endhint %}

## Proof of Concept

{% code title="test\_GlobalBlockingCheck\_PreventsEarlyClaims.sol" %}

```solidity
function test_GlobalBlockingCheck_PreventsEarlyClaims() public {
    // Setup: Create a prize with 3 winners
    vm.prank(ADMIN);
    raffle.addPrize("Multi-Winner Prize", "Desc", 100, 3);

    // Add tickets for multiple users
    spinStub.setBalance(USER, 5);
    vm.prank(USER);
    raffle.spendRaffle(1, 3); // USER: tickets 1-3
    
    spinStub.setBalance(USER2, 5);
    vm.prank(USER2);
    raffle.spendRaffle(1, 2); // USER2: tickets 4-5

    // Draw ONLY the first winner (USER2)
    uint256 req1 = requestWinnerForPrize(1);
    uint256[] memory rng1 = new uint256[](1);
    rng1[0] = 4; // Ticket #5 -> USER2
    vm.prank(SUPRA_ORACLE);
    raffle.handleWinnerSelection(req1, rng1);

    // Verify first winner was drawn correctly
    assertEq(raffle.getWinner(1, 0), USER2, "First winner should be USER2");
    
    // Check prize status - should still be active since only 1/3 winners drawn
    (,,, bool isActive,,,,,, uint256 drawn) = raffle.getPrizeDetails(1);
    assertTrue(isActive, "Prize should still be active");
    assertEq(drawn, 1, "Should have 1 winner drawn");

    // BUG: USER2 (the drawn winner) tries to claim their prize
    // This should work according to docs, but will fail due to global check
    vm.prank(USER2);
    vm.expectRevert(abi.encodeWithSelector(Raffle.WinnerNotDrawn.selector));
    raffle.claimPrize(1, 0);

    // Verify the winner exists and is not claimed
    Raffle.Winner[] memory winners = raffle.getPrizeWinners(1);
    assertEq(winners.length, 1, "Should have 1 winner");
    assertEq(winners[0].winnerAddress, USER2, "Winner should be USER2");
    assertFalse(winners[0].claimed, "Winner should not be claimed yet");

    // Draw the second winner (USER)
    uint256 req2 = requestWinnerForPrize(1);
    uint256[] memory rng2 = new uint256[](1);
    rng2[0] = 1; // Ticket #2 -> USER
    vm.prank(SUPRA_ORACLE);
    raffle.handleWinnerSelection(req2, rng2);

    // BUG: Even with 2/3 winners drawn, USER2 still cannot claim
    vm.prank(USER2);
    vm.expectRevert(abi.encodeWithSelector(Raffle.WinnerNotDrawn.selector));
    raffle.claimPrize(1, 0);

    // Draw the third winner (USER again)
    uint256 req3 = requestWinnerForPrize(1);
    uint256[] memory rng3 = new uint256[](1);
    rng3[0] = 2; // Ticket #3 -> USER
    vm.prank(SUPRA_ORACLE);
    raffle.handleWinnerSelection(req3, rng3);

    // NOW the prize is inactive and USER2 can finally claim
    (,,, bool isActiveFinal,,,,,, uint256 drawnFinal) = raffle.getPrizeDetails(1);
    assertFalse(isActiveFinal, "Prize should be inactive after all winners drawn");
    assertEq(drawnFinal, 3, "Should have 3 winners drawn");

    // This should work now
    vm.prank(USER2);
    raffle.claimPrize(1, 0);

    // Verify claim was successful
    winners = raffle.getPrizeWinners(1);
    assertTrue(winners[0].claimed, "Winner should now be claimed");
}
```

{% endcode %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Raffle.sol#L299C5-L304C1>
