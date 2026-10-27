# 49710 sc high cross batch state manipulation in yield distribution allows double dipping of yield funds

**Submitted on Jul 18th 2025 at 16:30:56 UTC by @DSbeX for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49710
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Theft of unclaimed yield

{% hint style="danger" %}
High severity: attackers can manipulate cross-batch state to claim yield multiple times, resulting in theft of funds intended for other holders.
{% endhint %}

## Description

### Brief / Intro

The batched yield distribution processes holders in segments but reads live token balances and does not track which addresses or tokens have already received yield for a given distribution. An attacker can transfer tokens between addresses in different, unprocessed batches so the same tokens are included multiple times across batches and claim yield repeatedly. This allows theft of yield funds and breaks the intended proportional distribution.

### Vulnerability Details

The vulnerability is in `distributeYieldWithLimit` and arises from three main design flaws:

* Real-time balance checks: balances are read live during each batch processing (`balanceOf(holder)`)
* No claim tracking: no storage of which addresses/tokens have received yield for the current distribution
* Global amount distribution: a fixed `totalAmount` is divided across a mutable effective total supply computed from live balances

Vulnerable code excerpt:

```javascript
uint256 effectiveTotalSupply = 0;
for (uint256 i = 0; i < totalHolders; i++) {
    address holder = $.holders.at(i);
    if (_isYieldAllowed(holder)) {
        effectiveTotalSupply += balanceOf(holder); //live balance
    }
}

// Per-holder calculation uses current balance
uint256 holderBalance = balanceOf(holder);
uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
```

The contract does not implement:

* Balance snapshots (e.g., ERC20Snapshot) for a distribution epoch
* Per-distribution claim tracking or locks for processed tokens/addresses
* Locking or reserving the distributed `totalAmount` against reallocation across batches

## Impact Details

* Attackers can claim yield proportional to their holdings multiple times by moving tokens between unprocessed batches.
* This directly results in theft of unclaimed yield intended for other token holders.
* Losses scale with the number of batches; maximum theoretical loss ≈ (batches - 1) \* attacker\_token\_balance \* yield\_per\_token.

## References

* distributeYieldWithLimit function - <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466>
* effectiveTotalSupply calculation - <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L514>
* Balance-based yield share per holder - <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L538>

## Proof of Concept

{% stepper %}
{% step %}

### Scenario setup

* Total holders: 300 (indices 0–299)
* Batches: 3 batches of 100 holders each
  {% endstep %}

{% step %}

### Attacker preparation

* Attacker controls:
  * Address A: Index 50 (Batch 1)
  * Address B: Index 250 (Batch 3) — second wallet controlled by the attacker
* Attacker holds 1,000 tokens (10% of total supply)
  {% endstep %}

{% step %}

### Attack execution

1. First batch (Batch 1) is executed; Address A (index 50) receives its yield share.
2. Immediately after Batch 1, attacker transfers the 1,000 tokens from Address A to Address B.
3. When Batch 3 is processed, Address B (index 250) receives yield for the same 1,000 tokens again.
   {% endstep %}

{% step %}

### Mathematical proof (example)

* Total yield distributed: 100,000 USDC
* Legitimate share for 10% holdings: 10,000 USDC
* Attacker receives: 10,000 USDC in Batch 1 + 10,000 USDC in Batch 3 = 20,000 USDC
* Result: Attacker steals 10,000 USDC more than fair share; other holders are proportionally shorted.
  {% endstep %}
  {% endstepper %}

## Mitigation suggestions (not exhaustive)

* Use snapshots (e.g., ERC20Snapshot) to compute effective total supply and per-holder balances at a single point in time for the entire distribution epoch.
* Track per-distribution claims (mapping of distribution ID → address → claimed) to prevent double claims.
* Lock or reserve the distributed amount (or mark tokens as processed) when starting a multi-batch distribution to prevent reallocation across batches.
* Alternatively, compute and store all shares off-chain and enforce single-claim on-chain via a distribution ID and per-address claim flag.
