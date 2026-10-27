# 49939 sc high initial timestamp mismatch might lead to users being able to spin twice in the same day

Submitted on Jul 20th 2025 at 16:45:07 UTC by @a16 for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #49939
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

Non-whitelisted users should only be able to "spin" once per day, which is significant because winning odds vary by day of the week. Due to a potential mismatch between the initial timestamp used to calculate the day in the `canSpin()` modifier and the `getCurrentWeek()` function (and other reward/jackpot computations), users could potentially circumvent the once-per-day restriction.

### Vulnerability Details

Two different parts of the code determine the day differently:

* The restriction that prevents users from spinning more than once a day relies on `DateTime.sol` via `dateTime.getDay()`, used by the `canSpin()` modifier. `getDay()` is based on the integer division of the current timestamp by DAY\_IN\_SECONDS (86400).
* Rewards and jackpot day calculations are based on `campaignStartDate`. For example, `jackpotThreshold` uses:

```solidity
uint256 daysSinceStart = (block.timestamp - campaignStartDate) / 1 days;
uint8 dayOfWeek = uint8(daysSinceStart % 7);
```

The setter for `campaignStartDate`:

```solidity
function setCampaignStartDate(
    uint256 start
) external onlyRole(ADMIN_ROLE) {
    campaignStartDate = start == 0 ? block.timestamp : start;
}
```

Because `campaignStartDate` does not have to be divisible by 86400 (it may be set to an arbitrary timestamp like `block.timestamp` when the admin calls `setCampaignStartDate()`), it is possible that the two day calculations disagree. Concretely, `canSpin()` might consider two timestamps to be different calendar days (using timestamp // 86400), while `(block.timestamp - campaignStartDate) / 1 days` yields the same day index for rewards/jackpot. This discrepancy can let a user effectively "spin twice" for the same reward-day bucket while passing the `canSpin()` check.

### Impact Details

Savvy users could exploit this mismatch to increase their effective chances of winning or obtaining better rewards. For example, the jackpot probabilities per day could be:

jackpotProbabilities = \[1, 2, 3, 5, 7, 10, 20];

If one day is much more favorable (e.g., 20x chance on a particular day), a user could make two spins that the modifier allows as two different calendar days, but the reward logic treats them as the same day relative to `campaignStartDate`, thereby unfairly amplifying their odds.

## Suggestion

{% hint style="info" %}
Enforce in code that `campaignStartDate` is divisible by 86400 (1 day in seconds). For example, when setting `campaignStartDate`, round or floor the provided timestamp to a multiple of 86400 (UTC midnight), or require that the admin provides a timestamp aligned to day boundaries.
{% endhint %}

## Proof of Concept

<details>

<summary>Expand PoC</summary>

Example timeline:

* Assume `campaignStartDate` is set via `setCampaignStartDate(0)` at timestamp 1753056060 (one minute after midnight UTC).
* A non-whitelisted user calls `startSpin()` at 1753140000. This is their first spin and is allowed.
* The same user calls `startSpin()` again at 1753142430.
  * `canSpin()` uses `dateTime.getDay()` which is based on `timestamp // 86400`:
    * 1753140000 // 86400 = 20290
    * 1753142430 // 86400 = 20291
    * So `canSpin()` sees two different days and allows the second spin.
  * Reward/jackpot day calculation uses `(block.timestamp - campaignStartDate) / 1 days`:
    * Assuming `handleRandomness()` runs immediately after (e.g., at 1753142440):
    * (1753140000 - 1753056060) // 86400 = 0
    * (1753142440 - 1753056060) // 86400 = 0
    * Both spins are treated as the same day for rewards/jackpot.

Result: `canSpin()` allows a second spin because of calendar-day division, but reward logic treats both spins as occurring on the same campaign-relative day — producing a mismatch that can be exploited.

</details>
