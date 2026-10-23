# 50487 sc low cross campaign jackpot denial due to state pollution

**Submitted on Jul 25th 2025 at 10:19:28 UTC by @IronsideSec for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50487
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

The `Spin.sol` contract suffers from a state pollution vulnerability where `lastJackpotClaimWeek` is not reset when a new campaign starts via `setCampaignStartDate()`. This can cause legitimate users in subsequent campaigns to be denied their jackpot prizes if they win a jackpot in a week number that was already used by a previous campaign's jackpot winner. Losses can be up to 100,000 PLUME tokens per affected user (final week 12 jackpot).

See the Recommended Fix below for an easy mitigation.

### Vulnerability Details

{% stepper %}
{% step %}

### Campaign 1

User A wins a jackpot in week 5 → `lastJackpotClaimWeek = 5`. No further jackpots are won until campaign end.
{% endstep %}

{% step %}

### Campaign Transition

Admin calls `setCampaignStartDate(newDate)` → `campaignStartDate` updates but `lastJackpotClaimWeek` remains 5 (stale).
{% endstep %}

{% step %}

### Campaign 2

During Campaign 2, no jackpots for first 4 weeks. On week 5, User B legitimately wins:

* `getCurrentWeek() = 5`
* `currentWeek == lastJackpotClaimWeek` → jackpot denied (reward becomes "Nothing")
  {% endstep %}
  {% endstepper %}

The proof-of-concept test demonstrates this scenario:

* Campaign 1 jackpot in week 5: ✅ 20,000 PLUME awarded
* Campaign 2 jackpot in week 5: ❌ 0 PLUME awarded (denied)

### Root Cause - Incomplete State Reset

The admin function that starts a new campaign updates `campaignStartDate` but does not reset `lastJackpotClaimWeek`, leaving stale state that is used by the jackpot validation logic.

Vulnerable setter:

```solidity
// attackathon-plume-network/plume/src/spin/Spin.sol:479-483
function setCampaignStartDate(
    uint256 start
) external onlyRole(ADMIN_ROLE) {
    campaignStartDate = start == 0 ? block.timestamp : start;
    // ❌ BUG: lastJackpotClaimWeek is NOT reset here
}
```

Vulnerable jackpot validation:

```solidity
// attackathon-plume-network/plume/src/spin/Spin.sol:246-249
if (keccak256(bytes(rewardCategory)) == keccak256("Jackpot")) {
    uint256 currentWeek = getCurrentWeek();
    if (currentWeek == lastJackpotClaimWeek) {  // ❌ Uses stale data from previous campaign
        userDataStorage.nothingCounts += 1;
        rewardCategory = "Nothing";
        rewardAmount = 0;
        emit JackpotAlreadyClaimed("Jackpot already claimed this week");
```

Week calculation:

```solidity
// attackathon-plume-network/plume/src/spin/Spin.sol:207-209
function getCurrentWeek() public view returns (uint256) {
    return (block.timestamp - campaignStartDate) / 7 days;
}
```

## Impact Details

* Direct per-user loss: up to 100,000 PLUME tokens (week 11/12 jackpot).
* Affected users: users with long streaks (13+ day streaks) who legitimately earn jackpots.
* Spin fee loss: 2 PLUME per denied spin (no refund mechanism).
* Systemic: weeks where jackpots were previously claimed become blocked across campaigns if the same week number occurs again; this effectively lets earlier campaigns "reserve" jackpot weeks for future campaigns, reducing availability and harming high-value users disproportionately.

Severity multipliers:

* Users cannot control or predict this.
* Affects legitimately earned jackpots.
* No recovery mechanism for denied jackpots.
* Cumulative across campaigns.

## Recommended Fix

Reset `lastJackpotClaimWeek` when starting a new campaign so prior campaign data cannot block jackpot claims in the new campaign.

Suggested change:

```solidity
function setCampaignStartDate(uint256 start) external onlyRole(ADMIN_ROLE) {
    campaignStartDate = start == 0 ? block.timestamp : start;
    lastJackpotClaimWeek = 999; // Reset to initial state for new campaign
}
```

(Use an appropriate sentinel value consistent with the contract's initialization behavior — e.g., a very large number or the original initial value used in contract deployment.)

## Proof of Concept

Gist with PoC and test steps: <https://gist.github.com/IronsideSec/107fd6c7c13961071c2a7bf52de339d2>

Steps (reproduced from the Gist) shown as a stepper:

{% stepper %}
{% step %}
User1 spins daily for 7 days in Campaign 1 to build streak.
{% endstep %}

{% step %}
User1 hits jackpot in week 5 of Campaign 1 → `lastJackpotClaimWeek = 5`. Campaign 1 ends.
{% endstep %}

{% step %}
Admin starts Campaign 2 with `setCampaignStartDate()` → BUG: `lastJackpotClaimWeek` still = 5.
{% endstep %}

{% step %}
User2 spins daily for 7 days in Campaign 2 to build streak.
{% endstep %}

{% step %}
User2 hits jackpot in week 5 of Campaign 2 (winning RNG).
{% endstep %}

{% step %}
Contract checks: `currentWeek (5) == lastJackpotClaimWeek (5)` → TRUE.
{% endstep %}

{% step %}
Contract emits `JackpotAlreadyClaimed` and converts reward to "Nothing".
{% endstep %}

{% step %}
User2 loses the 2 PLUME spin fee and gets 0 payout despite winning RNG.
{% endstep %}

{% step %}
Fix: Reset `lastJackpotClaimWeek` in `setCampaignStartDate()` function.
{% endstep %}
{% endstepper %}

## References

* Vulnerable Function: [setCampaignStartDate() in Spin.sol:479-483](https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/spin/Spin.sol#L457-L461)
* Jackpot Validation: [handleRandomness() in Spin.sol](https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/spin/Spin.sol#L225-L232)
* Week Calculator: [getCurrentWeek() in Spin.sol:207-209](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L197-L199)

## Link to Proof of Concept

<https://gist.github.com/IronsideSec/107fd6c7c13961071c2a7bf52de339d2>
