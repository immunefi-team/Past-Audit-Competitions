# 51882 sc low unnecessary claiming restriction in raffle contract prevents winners from claiming prizes until all winners are drawn

* **Submitted on:** Aug 6th 2025 at 12:46:33 UTC by @vivekd for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #51882
* **Report Type:** Smart Contract
* **Severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impact Summary:** Contract fails to deliver promised returns in a timely manner (no direct fund loss)

## Description

### Brief / Intro

The `Raffle` contract's `claimPrize` function contains an overly restrictive check that prevents legitimate winners from claiming their prizes until all winner slots for a multi-winner prize have been drawn. This creates an unnecessary dependency on administrative actions and forces winners to wait indefinitely to claim their prizes, harming user experience.

### Vulnerability Details

The problematic logic is in `claimPrize` (lines \~298-301):

```solidity
// Lines 298-301 in Raffle.sol
function claimPrize(uint256 prizeId, uint256 winnerIndex) external {
    if (prizes[prizeId].isActive && winnersDrawn[prizeId] < prizes[prizeId].quantity) {
        revert WinnerNotDrawn();
    }
    // ... rest of claiming logic
}
```

What this does:

* It checks `prizes[prizeId].isActive && winnersDrawn[prizeId] < prizes[prizeId].quantity`
* If true, it reverts with `WinnerNotDrawn()`
* Thus ANY winner cannot claim until ALL winners for that prize have been selected

How prize state is managed elsewhere (relevant excerpt):

```solidity
// Lines 278-281 in Raffle.sol
// Deactivate prize if all winners have been drawn
if (winnersDrawn[prizeId] == prizes[prizeId].quantity) {
    prizes[prizeId].isActive = false;
}
```

Flaw summary:

* For prizes with `quantity > 1`, once an individual winner is drawn and recorded in `prizeWinners[prizeId]`, that winner still cannot call `claimPrize` until `winnersDrawn[prizeId] == quantity` (i.e., all winners drawn and the prize deactivated). The ability to claim is incorrectly tied to completion of all drawings rather than to the caller being a recorded winner.

### Impact Details

* No direct fund loss, but winners cannot claim their prizes until all winners are drawn. This can cause indefinite delays and breaks the promised timely delivery of prizes.

## References

* Raffle.sol lines referenced: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Raffle.sol#L297-L312>

<details>

<summary>Show referenced code snippet link</summary>

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Raffle.sol#L297-L312>

</details>

## Proof of Concept

{% stepper %}
{% step %}

### Setup

* Admin creates a raffle prize with `quantity = 5` (5 winners total).
* Users enter the raffle by spending tickets via `spendRaffle()`.
  {% endstep %}

{% step %}

### First Winner Drawing

* Admin calls `requestWinner(prizeId)`.
* VRF callback executes `handleWinnerSelection()`.
* Winner #1 is selected and stored at `prizeWinners[prizeId][0]`.
* `winnersDrawn[prizeId] = 1`.
* The prize remains active: `prizes[prizeId].isActive = true`.
  {% endstep %}

{% step %}

### Winner Attempts to Claim

* Winner #1 calls `claimPrize(prizeId, 0)`.
* Contract evaluates: `prizes[prizeId].isActive && winnersDrawn[prizeId] < prizes[prizeId].quantity`
* Evaluates to: `true && (1 < 5) = true`
* Transaction reverts with `WinnerNotDrawn()`.
  {% endstep %}

{% step %}

### Verification

* `getWinner(prizeId, 0)` returns Winner #1's address.
* `getPrizeWinners(prizeId)` shows Winner #1 in the array.
* Winner #1 is confirmed but cannot claim.
  {% endstep %}

{% step %}

### Waiting Period & Resolution

* If admin does not draw remaining winners, Winner #1 continues to be unable to claim.
* Only after admin draws winners 2–5, when `winnersDrawn[prizeId] == quantity`, does `prizes[prizeId].isActive` become `false` and Winner #1 can finally call `claimPrize(prizeId, 0)` successfully.
  {% endstep %}
  {% endstepper %}

## Suggested Fix (conceptual)

* The claim check should verify that the caller is a recorded winner (e.g., matches `prizeWinners[prizeId][winnerIndex]`) and that the specific winner slot has been drawn, rather than requiring that all winner slots for the prize have been drawn and the prize deactivated. Ensure existing invariants and replay/claim protections remain intact.

(Note: The exact code change was not provided in the original report — keep behavior and state invariants in mind when implementing the fix.)
