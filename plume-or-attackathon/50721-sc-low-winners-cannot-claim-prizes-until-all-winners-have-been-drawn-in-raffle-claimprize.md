# 50721 sc low winners cannot claim prizes until all winners have been drawn in raffle claimprize&#x20;

* Report ID: #50721
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* Submitted on: Jul 27th 2025 at 21:14:40 UTC by @blackgrease for Attackathon | Plume Network (<https://immunefi.com/audit-competition/plume-network-attackathon>)

## #50721 \[SC-Low] Winners cannot claim Prizes until all winners have been drawn in `Raffle::claimPrize`

Summary

* The `Raffle::claimPrize` function contains a check that prevents individual winners from claiming their prize when a prize has more than one winner. This makes individual claims dependent on all winners being drawn, which contradicts the documented invariant that each winner’s claim status is independent.

Hint: severity/impact note

{% hint style="info" %}
The reporter states the official severity as Low but argues the practical impact and broken invariant justify a Medium severity rating. This note is preserved from the original report.
{% endhint %}

Description

The Raffle documentation states the contract is "Multi-winner aware" and tracks winner claims independently:

"Independent Status: Each winner's claim status is tracked independently. One user claiming their prize has no effect on the ability of other winners to claim theirs." — Plume Raffle Documentation

However, `Raffle::claimPrize` contains the following check that partially breaks this promise:

Problematic check in `claimPrize`

```solidity
function claimPrize(uint256 prizeId, uint256 winnerIndex) external {
    if (prizes[prizeId].isActive && winnersDrawn[prizeId] < prizes[prizeId].quantity) {
          revert WinnerNotDrawn(); //@audit-medium: winners cannot claim until all winners have been selected
    //--snip---
}
```

This requires the total number of winners drawn to equal the number of possible winners (prize quantity) before any winner can claim. For prizes with multiple winners, earlier winners cannot claim until all winners have been drawn, making claims dependent on the admin-driven drawing process and introducing unpredictable waiting times.

Impact

The reporter preserved their severity note but recommends Medium; the original report labels the finding Low. The practical impacts are summarized below as individual items:

{% stepper %}
{% step %}

### Impact item 1

Partially broken invariant: The contract can behave outside documented expectations, creating a conflict between protocol promises and executed logic.
{% endstep %}

{% step %}

### Impact item 2

Inconvenience: For prizes with many winners, early winners must wait until all winners are drawn (an admin-initiated action) to claim, causing unpredictable delays.
{% endstep %}

{% step %}

### Impact item 3

User experience: Long waiting times may cause negative feedback and decreased engagement in the Raffle game.
{% endstep %}
{% endstepper %}

Likelihood: High — this will occur whenever a prize has more than one winner.

Mitigation

Change the check so it only reverts when no winners have yet been drawn for that prize (i.e., `winnersDrawn[prizeId] == 0`), instead of requiring that all winners have already been drawn.

Suggested fix (diff)

```diff
// User claims their prize, we mark it as claimed and deactivate the prize
function claimPrize(uint256 prizeId, uint256 winnerIndex) external {
-    if (prizes[prizeId].isActive && winnersDrawn[prizeId] < prizes[prizeId].quantity) {
+    if (prizes[prizeId].isActive && winnersDrawn[prizeId] == 0) { //Only check if there are 0 winners meaning `requestWinner` has yet to be called. 
        revert WinnerNotDrawn(); 
    }

    Winner storage individualWin = prizeWinners[prizeId][winnerIndex];
    //---snip---
```

Proof of Concept

Private Gist: <https://gist.github.com/blackgrease/3af095c20d2604f14141bb5a74b80a3a>

Run with:

```
forge test --mt testCannotClaimUntillAllWinnersDrawn --via-ir -vvvv
```

Walk-through (PoC)

{% stepper %}
{% step %}

### Step 1

An admin creates a Prize with 4 possible winners.
{% endstep %}

{% step %}

### Step 2

6 different players join the raffle pool with different amounts of raffle tickets.
{% endstep %}

{% step %}

### Step 3

The Admin initiates the winner selection and the Supra Router calls back and selects winners.
{% endstep %}

{% step %}

### Step 4

One winner is announced and multiple (3) remaining winners are pending.
{% endstep %}

{% step %}

### Step 5

The announced winner tries to claim their prize by calling `Raffle::claimPrize` but the call reverts with `WinnerNotDrawn`.
{% endstep %}

{% step %}

### Step 6

The winner must wait until all winners have been drawn before being able to successfully claim their prize.
{% endstep %}
{% endstepper %}

Notes

* The reporter validated that applying the suggested check (`winnersDrawn[prizeId] == 0`) allows winners to claim once they are drawn even if not all winners for the prize have been selected, thus resolving the issue described.
* All links and the Gist URL are kept as provided.
