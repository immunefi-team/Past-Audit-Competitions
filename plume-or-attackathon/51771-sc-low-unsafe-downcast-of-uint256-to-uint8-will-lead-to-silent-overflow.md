# 51771 sc low unsafe downcast of uint256 to uint8 will lead to silent overflow

* **Submitted on:** Aug 5th 2025 at 17:09:02 UTC by @vielite for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #51771
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `determineReward()` function in `Spin.sol` unsafely downcasts the `uint256` week counter into a `uint8`:

```solidity
function determineReward(
    uint256 randomness,
    uint256 streakForReward
) internal view returns (string memory, uint256) {
    uint256 probability = randomness % 1_000_000; // Normalize VRF range to 1M

    // Determine the current week in the 12-week campaign
    uint256 daysSinceStart = (block.timestamp - campaignStartDate) / 1 days;
    uint8 weekNumber = uint8(getCurrentWeek());
    uint8 dayOfWeek = uint8(daysSinceStart % 7);
```

While the developer's intent was to enforce a 12-week campaign, the smart contract does not enforce this limit. If the campaign runs longer than 255 weeks, the downcast will truncate the value and cause a silent overflow: `weekNumber` will wrap and may be `0`.

## Vulnerability Details

The vulnerability stems from the unsafe downcast of a `uint256` value returned by `getCurrentWeek()` into a `uint8` in `determineReward()`. `getCurrentWeek()` returns:

```solidity
function getCurrentWeek() public view returns (uint256) {
    return (block.timestamp - campaignStartDate) / 7 days;
}
```

Because `getCurrentWeek()` returns a `uint256` (number of weeks since campaign start), forcing this into `uint8` (`uint8 weekNumber = uint8(getCurrentWeek());`) will silently truncate for values > 255. When the week count reaches 256, `weekNumber` becomes 0, causing reward lookups to use the wrong week index.

## Impact Details

* Severity: Low (contract fails to deliver promised returns but does not lose value)
* If the campaign runs past 255 weeks (\~5 years), reward calculation can reference incorrect week indices (e.g., week 256 truncated to 0), causing users to receive incorrect (typically lower) rewards. No funds are directly lost, but promised returns are not delivered.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L278>

## Proof of Concept

{% stepper %}
{% step %}

### Step — Prerequisite

The `Spin.sol` contract has been deployed and is actively running for ≥255 weeks.
{% endstep %}

{% step %}

### Step — State

Admin sets jackpot prize for week 256 as `10_000`. Assume `jackpotPrizes[0] = 100`.

```solidity
function setJackpotPrizes(uint8 week, uint256 prize) external onlyRole(ADMIN_ROLE) {
    jackpotPrizes[week] = prize;
}
```

{% endstep %}

{% step %}

### Step — User Action

A user calls `startSpin()` and the oracle callback provides randomness that results in a "Jackpot" win.
{% endstep %}

{% step %}

### Step — Flawed execution

* `getCurrentWeek()` returns `256`.
* `uint8 weekNumber = uint8(256)` truncates to `0` due to downcast.
* The function uses `jackpotPrizes[0]` instead of `jackpotPrizes[256]`.
  {% endstep %}

{% step %}

### Step — Result

The user receives `100` tokens (the value for week index `0`) instead of the intended `10_000` tokens for week `256`.
{% endstep %}
{% endstepper %}
