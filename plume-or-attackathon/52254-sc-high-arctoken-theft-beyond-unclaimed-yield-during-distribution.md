# 52254 sc high arctoken theft beyond unclaimed yield during distribution

**Submitted on Aug 9th 2025 at 05:30:38 UTC by @Blobism for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52254
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

An attacker controlling multiple accounts with ArcToken holdings can transfer tokens between their accounts in between calls to `distributeYieldWithLimit`. Because the distribution function uses current balances (no snapshot) and processes holders in chunks, the attacker can be paid twice for the same underlying tokens — once in an earlier chunk from account A, then again in a later chunk after moving the tokens to account B — causing yield distributed to exceed the intended `totalAmount`.

### Vulnerability Details

The `distributeYieldWithLimit` method distributes yield in chunks to avoid gas exhaustion. It accepts a start index and a max number of holders (batchSize) to process for a given chunk. The implementation queries each holder's balance at the time of processing and uses that to compute their share, but it does not snapshot balances at the start of the distribution run or otherwise prevent balance changes between chunks.

Relevant excerpt:

{% code title="ArcToken.sol (excerpt)" %}

```solidity
function distributeYieldWithLimit(
    uint256 totalAmount,
    uint256 startIndex,
    uint256 maxHolders
)
    external
    onlyRole(YIELD_DISTRIBUTOR_ROLE)
    nonReentrant
    returns (uint256 nextIndex, uint256 totalHolders, uint256 amountDistributed)
{
    // ...

    for (uint256 i = 0; i < batchSize; i++) {
        uint256 holderIndex = startIndex + i;
        address holder = $.holders.at(holderIndex);

        if (!_isYieldAllowed(holder)) {
            continue;
        }

        uint256 holderBalance = balanceOf(holder); // <----- no balances snapshot
        if (holderBalance > 0) {
            uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
            if (share > 0) {
                yToken.safeTransfer(holder, share);
                amountDistributed += share;
            }
        }
    }

    // ...
}
```

{% endcode %}

Because effectiveTotalSupply and holder balances can change between chunks, an attacker can manipulate holdings between chunk calls to be counted multiple times.

### Impact Details

* Expected behavior: distribute up to `totalAmount` among holders, proportional to their holdings at the moment of distribution.
* Actual issue: by transferring tokens between attacker-controlled accounts that are processed in different chunks, the attacker can be counted multiple times and receive more yield than intended. This results in direct theft of tokens beyond the configured distribution amount.

## Proof of Concept

{% stepper %}
{% step %}

### Setup: Two attacker accounts at different holder indices

* Attacker controls Account A (early index, e.g., index 50) and Account B (later index, e.g., index 150).
* The attacker places most of their ArcToken holdings in Account A initially.
  {% endstep %}

{% step %}

### Distributor processes first chunk

* The yield distributor calls `distributeYieldWithLimit` with `maxHolders = 100` for holders 0–99.
* Account A (index 50) receives its calculated share based on its large balance at that time.
  {% endstep %}

{% step %}

### Attacker moves tokens between chunks

* Immediately after the first chunk is processed, the attacker transfers most tokens from Account A to Account B.
* No snapshot or guard prevents this transfer from affecting later chunks.
  {% endstep %}

{% step %}

### Distributor processes second chunk

* The yield distributor calls `distributeYieldWithLimit` again for holders 100–199.
* Account B (index 150) now has the large balance and receives its share, effectively double-counting the same underlying tokens.
  {% endstep %}
  {% endstepper %}

As a result, more yield is distributed than the `totalAmount` provided to the distribution call sequence, enabling theft beyond intended limits.

## References

* Contract: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>

{% hint style="danger" %}
Severity: High — an attacker controlling multiple accounts can directly extract funds beyond the configured distribution by transferring tokens between chunks during a multi-call distribution.
{% endhint %}
