# 51596 sc low unsafe uint256 to uint8 downcast causes integer overflow leading to unauthorized jackpot payouts after week 255

**Submitted on Aug 4th 2025 at 10:03:55 UTC by @vivekd for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51596
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

The `determineReward` function unsafely downcasts the week number from `uint256` to `uint8` without validation, causing an integer overflow after week 255.

This overflow results in the week number wrapping back to 0, which re-enables jackpot payouts that should have ended after the 12-week campaign.

This vulnerability allows users to claim jackpots worth up to 100,000 PLUME tokens indefinitely every 256 weeks, potentially leading to complete protocol insolvency.

### Vulnerability Details

The vulnerability exists in the `determineReward` function at line \~263:

```solidity
function determineReward(
    uint256 randomness,
    uint256 streakForReward
) internal view returns (string memory, uint256) {
    uint256 probability = randomness % 1_000_000;
    
    // Calculate current week
    uint256 daysSinceStart = (block.timestamp - campaignStartDate) / 1 days;
    uint8 weekNumber = uint8(getCurrentWeek()); // UNSAFE DOWNCAST
    
    // ... jackpot logic uses weekNumber
    if (probability < jackpotThreshold) {
        return ("Jackpot", jackpotPrizes[weekNumber]); // Uses overflowed value!
    }
}
```

The `getCurrentWeek()` function returns a `uint256`:

```solidity
function getCurrentWeek() public view returns (uint256) {
    return (block.timestamp - campaignStartDate) / 7 days;
}
```

The issue occurs because:

* `uint8` can only store values 0-255.
* When `getCurrentWeek()` returns 256 or higher, the downcast overflows.
* `uint8(256) = 0`, `uint8(257) = 1`, etc.

The `jackpotPrizes` mapping is only populated for weeks 0-11:

```solidity
jackpotPrizes[0] = 5000;    // Week 0: 5,000 PLUME
jackpotPrizes[1] = 5000;    // Week 1: 5,000 PLUME
// ...
jackpotPrizes[10] = 50_000; // Week 10: 50,000 PLUME
jackpotPrizes[11] = 100_000; // Week 11: 100,000 PLUME
// Weeks 12+ have no prizes (default 0)
```

After the intended 12-week campaign ends, the contract should stop awarding jackpots. However, due to the overflow:

* Week 256 → `uint8(256) = 0` → Awards 5,000 PLUME
* Week 267 → `uint8(267) = 11` → Awards 100,000 PLUME

This pattern repeats every 256 weeks indefinitely.

## Impact Details

This vulnerability directly causes Protocol Insolvency through unauthorized drainage of the contract's PLUME token reserves.

After week 255 (\~4.9 years), the unsafe downcast causes the week number to overflow and wrap around to 0, re-enabling jackpot payouts that should have permanently ended after the 12-week campaign.

This creates an infinite exploitation loop where malicious or lucky users can claim up to 435,000 PLUME tokens (worth \~$870,000 at $2/PLUME) every 256-week cycle.

The protocol will become insolvent when accumulated unauthorized payouts exceed the contract's PLUME balance.

The deterministic nature of this overflow (guaranteed to occur at week 256) combined with the high-value unauthorized payouts makes this a critical threat to the protocol's financial viability and continued operation.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L286>

## Proof of Concept

{% stepper %}
{% step %}

### Initial Setup

Deploy the `Spin` contract and set `campaignStartDate` to current timestamp:

```solidity
setCampaignStartDate(block.timestamp);
```

{% endstep %}

{% step %}

### Demonstrate Normal Campaign Behavior

During weeks 0-11, jackpots are awarded as intended:

* Week 0: 5,000 PLUME jackpot available
* Week 11: 100,000 PLUME jackpot available
* Week 12+: No jackpot prizes (returns 0)
  {% endstep %}

{% step %}

### Fast Forward to Week 256

Calculate timestamp for week 256:

```solidity
uint256 week256Timestamp = campaignStartDate + (256 * 7 days);
// Move block.timestamp to week 256
```

{% endstep %}

{% step %}

### Observe Integer Overflow

```solidity
uint256 actualWeek = getCurrentWeek(); // Returns 256
uint8 weekNumber = uint8(actualWeek);  // Overflows to 0
uint256 prize = jackpotPrizes[0];      // Returns 5,000 PLUME!
```

{% endstep %}

{% step %}

### Exploit Maximum Payout

Fast forward to week 267:

```solidity
uint256 week267Timestamp = campaignStartDate + (267 * 7 days);
uint256 actualWeek = getCurrentWeek(); // Returns 267
uint8 weekNumber = uint8(actualWeek);  // Overflows to 11
uint256 prize = jackpotPrizes[11];     // Returns 100,000 PLUME!
```

{% endstep %}

{% step %}

### Confirm Vulnerability Pattern

The overflow pattern repeats:

* Weeks 256-267: First unauthorized jackpot cycle
* Weeks 512-523: Second unauthorized jackpot cycle

Pattern continues every 256 weeks
{% endstep %}

{% step %}

### Calculate Total Unauthorized Payouts

Per 256-week cycle, available jackpots:

```
Week 256 (0): 5,000 PLUME
Week 257 (1): 5,000 PLUME
Week 258 (2): 10,000 PLUME
Week 259 (3): 10,000 PLUME
Week 260 (4): 20,000 PLUME
Week 261 (5): 20,000 PLUME
Week 262 (6): 30,000 PLUME
Week 263 (7): 30,000 PLUME
Week 264 (8): 40,000 PLUME
Week 265 (9): 40,000 PLUME
Week 266 (10): 50,000 PLUME
Week 267 (11): 100,000 PLUME
Total: 435,000 PLUME per cycle
```

{% endstep %}
{% endstepper %}
