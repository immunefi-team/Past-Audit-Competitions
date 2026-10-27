# 52845 sc high distributeyieldwithlimit lacks snapshot between batches allowing state changes to break distribution and lock yield

* Reported on: Aug 13th 2025 at 15:28:59 UTC by @lirezarazavi
* Report ID: #52845
* Report Type: Smart Contract
* Severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>

Impact: Permanent freezing of funds

## Description

### Brief / Intro

The `distributeYieldWithLimit` function recalculates `effectiveTotalSupply` on each batch execution without storing a snapshot of balances or total supply from the first batch. This allows state changes (transfers, burns, toggling eligibility) between batches to alter supply used in share calculations mid-distribution. As a result, the function may attempt to overpay holders, causing transaction reverts, locking yield in the contract indefinitely, and producing unfair distributions.

### Vulnerability Details

In `ArcToken.sol`:

```solidity
uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
```

* `totalAmount` remains constant for all batches.
* `effectiveTotalSupply` is recalculated at every batch call by iterating over all holders.
* No snapshot is taken at the start of distribution.
* No tracking exists for:
  * How much yield has been distributed so far
  * Remaining amount
  * Original total supply of eligible holders

Because `effectiveTotalSupply` can change between batches (e.g., token transfers, mint/burn, or `_isYieldAllowed` changes), the calculation can become inconsistent. This can cause the function to attempt to send more tokens than the contract holds (revert), or to distribute yield unfairly across holders.

### Impact Details

* Funds Locked: Remaining yield cannot be distributed after revert.
* DoS: Distribution can be blocked by a single malicious holder modifying balance/eligibility mid-process.
* Fairness Violation: Early-processed holders may receive more than their fair share while others receive none.
* Gas Waste: Unbounded iteration increases cost of retries.

Severity justification: An attacker can intentionally lock yield distribution for all holders, freezing funds and breaking intended protocol functionality.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L510-L516>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L540>

## Proof of Concept

Assume 3 holders:

| Holder | Balance | Eligible |
| ------ | ------- | -------- |
| H1     | 50      | Yes      |
| H2     | 30      | Yes      |
| H3     | 20      | Yes      |

* `effectiveTotalSupply` = 100
* `totalAmount` = 100 tokens yield
* `maxHolders` = 1 (process 1 holder per batch)

{% stepper %}
{% step %}

### Batch 1 (startIndex = 0)

* `startIndex == 0` → `transferFrom` pulls 100 tokens to the contract
* `effectiveTotalSupply` = 100
* H1 share = `(100 * 50) / 100 = 50`
* Contract balance after: 50 tokens remain
  {% endstep %}

{% step %}

### State Change Before Batch 2

* H1 transfers tokens away or becomes restricted (`_isYieldAllowed` returns false)
* New `effectiveTotalSupply` = 30 + 20 = 50
  {% endstep %}

{% step %}

### Batch 2 (startIndex = 1)

* `effectiveTotalSupply` = 50 (recalculated)
* H2 share = `(100 * 30) / 50 = 60`
* Contract only has 50 → `safeTransfer` fails → revert
* Distribution halts permanently, locking yield in the contract
  {% endstep %}
  {% endstepper %}

### Notes on mitigation attempt

A possible mitigation is to adjust `totalAmount` before each batch to reflect only the remaining yield. However:

* The caller must be aware of on-chain interactions (transfers, burns, blacklists) and adjust correctly.
* The caller must calculate remaining yield perfectly.
* State changes between batches still invalidate the original plan and break fairness.
* This approach relies on trusted off-chain logic and is not trustless.

Conclusion: Without an on-chain snapshot or robust tracking of distributed amount and original eligible supply, the design cannot guarantee correct, permissionless yield distribution.

## Severity: High / Critical

(As reported)

***
