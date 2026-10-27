# 52285 sc high incorrect dust handling in yield distribution leads to permanent fund lock

**Submitted on Aug 9th 2025 at 13:39:56 UTC by @vivekd for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52285
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief/Intro

The `distributeYield` and `previewYieldDistribution` functions in `ArcToken` fail to properly redistribute rounding dust when the last holder in the `holders` array is restricted from receiving yield.

This causes yield tokens to become permanently locked in the `ArcToken` contract with no mechanism for recovery, leading to an accumulation of stuck funds over time.

### Vulnerability Details

The vulnerability exists in the dust redistribution logic within both yield distribution functions.

The contract uses a common pattern to handle rounding errors where the last eligible holder receives `totalAmount - sumOfPreviousDistributions` to capture any dust from integer division:

```solidity
// In distributeYield() - lines 448-457
  if (holderCount > 0) {
      address lastHolder = $.holders.at(lastProcessedIndex);
      if (_isYieldAllowed(lastHolder)) {
          uint256 lastShare = amount - distributedSum;
          if (lastShare > 0) {
              yToken.safeTransfer(lastHolder, lastShare);
              distributedSum += lastShare;
          }
      }
      // BUG: If lastHolder is restricted, lastShare is never distributed
  }
```

When the last holder in the array is blacklisted or sanctioned (fails `_isYieldAllowed` check), the dust amount (`amount - distributedSum`) is never transferred to anyone.

The yield tokens remain in the `ArcToken` contract permanently as there is no recovery mechanism.

The same issue exists in `previewYieldDistribution()`:

```solidity
// Lines 297-301
  if (!_isYieldAllowed(lastHolder)) {
      amounts[lastProcessedIndex] = 0;  // Dust is not accounted for
  } else {
      amounts[lastProcessedIndex] = amount - totalPreviewAmount;
  }
```

### Impact Details

* Permanent Fund Lock: With realistic RWA parameters, significant value accumulates.
* Accounting Discrepancy: The `YieldDistributed` event emits `distributedSum` which is less than the actual amount transferred into the contract, creating accounting inconsistencies that affect analytics and auditing.
* Preview Function Inaccuracy: `previewYieldDistribution` returns incorrect totals (example: shows 490,200 satoshis distributed when 500,000 were allocated).
* Scalability Impact: With multiple RWA tokens on the platform, locked funds multiply.
* No Recovery Mechanism: The `ArcToken` contract has no function to sweep stuck yield tokens, making the lock permanent and growing indefinitely.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L233-L305>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L384-L460>

## Proof of Concept

{% stepper %}
{% step %}

### Setup

Deploy `ArcToken` with a realistic RWA tokenization scenario:

* TVL: $120,000
* 10,000 holders each holding 12 tokens (120,000 total supply)
* 196 holders are yield-restricted (sanctioned/blacklisted)
* One restricted holder is last in the `holders` array
* APY: 6% with monthly distributions
* Yield paid in WBTC at $120,000/BTC
  {% endstep %}

{% step %}

### Monthly Yield Calculation

* Monthly yield: ($120,000 \* 6%) / 12 = $600
* In WBTC: $600 / $120,000 = 0.005 WBTC = 500,000 satoshis
  {% endstep %}

{% step %}

### Calculate Effective Total Supply

* Total supply: 120,000 tokens
* Restricted holders' balance: 196 holders \* 12 tokens = 2,352 tokens
* Effective total supply (eligible for yield): 120,000 - 2,352 = 117,648 tokens
  {% endstep %}

{% step %}

### Call previewYieldDistribution(500000)

* 9,804 eligible holders (10,000 - 196)
* Each eligible holder calculation:

holderShare = (500,000 \* 12) / 117,648 = 50.999592003264... satoshis

* Integer division truncates to 50 satoshis
* Total distributed to first 9,804 eligible holders: 9,804 \* 50 = 490,200 satoshis
* Dust that should go to last eligible holder: 500,000 - 490,200 = 9,800 satoshis
* But if last holder is restricted: 9,800 satoshis not distributed
  {% endstep %}

{% step %}

### Execute distributeYield(500000)

* Contract receives 500,000 satoshis via `transferFrom`
* Process holders loop:
  * First 9,804 eligible holders each receive 50 satoshis
  * `distributedSum = 9,804 * 50 = 490,200` satoshis
  * Last holder (restricted):

    ```solidity
    if (_isYieldAllowed(lastHolder)) { // Returns false
        // Skipped - 9,800 satoshis remain undistributed
    }
    ```
* Event emitted: `YieldDistributed(490200, WBTC)` — incorrect amount
* 9,800 satoshis permanently locked in contract
  {% endstep %}

{% step %}

### Check Contract Balance Over Time

* After first month: `WBTC.balanceOf(arcToken) = 9,800 satoshis`
* After one year: `WBTC.balanceOf(arcToken) = 117,600 satoshis` (accumulated monthly)

No function exists to recover these funds; they accumulate indefinitely.
{% endstep %}
{% endstepper %}

## Conclusion

The distribution logic assumes the last holder is eligible to receive the dust. When that holder is restricted, the dust is neither redistributed nor recoverable, causing permanent lock of yield tokens and incorrect accounting/reporting. The issue affects both actual distribution and preview calculations and scales with the number of tokens and distributions over time.
