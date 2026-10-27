# 51090 sc high malicious user can steal yields when arctoken distributeyieldwithlimit is used&#x20;

Submitted on Jul 31st 2025 at 03:45:37 UTC by @jasonxiale for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #51090
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* Impacts:
  * Theft of unclaimed yield

## Description

### Brief / Intro

In the current implementation, `ArcToken.distributeYieldWithLimit` is used for paginated distribution. There is a flaw in the function that can allow a malicious user to steal yields.

### Vulnerability Details

1. In [ArcToken.distributeYieldWithLimit's implementation](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466-L555), the function calculates the `effectiveTotalSupply` in [ArcToken.sol#L514](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L514), and distributes the yields based on `balanceOf(holder) / effectiveTotalSupply` in [ArcToken.sol#L538-L545](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L538-L545)

```solidity
466     function distributeYieldWithLimit(
467         uint256 totalAmount,
468         uint256 startIndex,
469         uint256 maxHolders
470     )
...
510         uint256 effectiveTotalSupply = 0;
511         for (uint256 i = 0; i < totalHolders; i++) {
512             address holder = $.holders.at(i);
513             if (_isYieldAllowed(holder)) {
514                 effectiveTotalSupply += balanceOf(holder); <<<<<<<<<<<< calculate the `effectiveTotalSupply`
515             }
516         }
...
530         for (uint256 i = 0; i < batchSize; i++) {
531             uint256 holderIndex = startIndex + i;
532             address holder = $.holders.at(holderIndex);
533
534             if (!_isYieldAllowed(holder)) {
535                 continue;
536             }
537
538             uint256 holderBalance = balanceOf(holder);
539             if (holderBalance > 0) {
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< distribute yields based on balanceOf(holder)
540                 uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
541                 if (share > 0) {
542                     yToken.safeTransfer(holder, share);
543                     amountDistributed += share;
544                 }
545             }
546         }
...
554         return (nextIndex, totalHolders, amountDistributed);
555     }
```

2. ArcToken inherits from [ERC20Upgradeable](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L23), so `transfer` and `transferFrom` are supported.
3. While `transfer` or `transferFrom` is called, [ArcToken.\_update](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L653-L712) will be called, and in its implementation there are a few issues:
   * Issue 1: The function doesn't check if `from` has received the yields.
   * Issue 2: If the `to` address is not in `$.holders`, the function will add `to` into `$.holders` ([ArcToken.sol#L701-L703](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L701-L703)).

Because of the calculation being based on the balances at the time of distribution but the holder list and balances can change between paginated distribution calls, a malicious user can manipulate balances and holder positions to receive more yield than deserved across distribution batches.

### Impact Details

A malicious user can steal yields (receive more than their fair share across paginated distributions).

## Proof of Concept

{% stepper %}
{% step %}

### Setup

* Attacker Alice controls two wallets: Addr1 and Addr2.
* Total ArcToken holders: 20.
* Addr1 is at `$.holders.at(1)`, Addr2 is at `$.holders.at(11)`.
* Alice controls X tokens split as:
  * Addr1: (X - 1)
  * Addr2: 1 wei
    {% endstep %}

{% step %}

### Step 1 — First distribution (holders 0–10)

* `distributeYieldWithLimit` is called for holders between indices 0–10.
* effectiveTotalSupply is calculated using current balances.
* Addr1 receives yield proportional to (X - 1) / effectiveTotalSupply.
  {% endstep %}

{% step %}

### Step 2 — Manipulate balances

* After receiving the yield, Alice transfers (X - 2) tokens from Addr1 to Addr2.
* Balances now:
  * Addr1: 1 wei
  * Addr2: (X - 1)
* Because \_update may add Addr2 to the holders list if not present, and transfers changed balances after the first batch, the second batch's distribution calculation still used the earlier effectiveTotalSupply, or otherwise allows Addr2 to be paid based on the previous large balance allocation.
  {% endstep %}

{% step %}

### Step 3 — Second distribution (holders 10–20)

* `distributeYieldWithLimit` is called for holders between indices 10–20.
* Addr2 receives yield proportional to (X - 1) / effectiveTotalSupply (the same large share).
* As a result, Alice's combined receipts across the two batches exceed the fair share; yield was effectively stolen.
  {% endstep %}
  {% endstepper %}

## References

* ArcToken.sol distribution implementation: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol>

(Links in the document point to the relevant lines in the referenced file.)
