# 51754 sc high double yield distribution via token transfers between distributeyieldwithlimit calls

**Submitted on Aug 5th 2025 at 14:38:04 UTC by @KlosMitSoss for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51754
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

For tokens with a large number of holders, `ArcToken::distributeYieldWithLimit()` allows for paginated distribution. An off-chain script or keeper can call this function repeatedly in batches, using the `startIndex` and `maxHolders` parameters to process a subset of holders in each transaction. This can be exploited by an attacker who transfers ArcTokens from a holder in one batch to a holder in the next batch. As a result, some ArcTokens will be eligible to receive yield multiple times, which qualifies as theft of yield.

### Vulnerability Details

When tokens have a large number of holders, `ArcToken::distributeYield()` may revert due to exceeding the block gas limit. In such cases, `ArcToken::distributeYieldWithLimit()` can be used instead. This function distributes yield across all holders in batches, sending the entire `totalAmount` only in the first batch.

The share for each holder is calculated using the following formula, where `holderBalance` is the ArcToken balance of the holder at the current timestamp and `effectiveTotalSupply` is the sum of all balances of holders that are allowed to receive yield:

```solidity
uint256 share = (amount * holderBalance) / effectiveTotalSupply
```

The vulnerability occurs when holders transfer ArcTokens to an address that is not yield-restricted after they have received their yield in an earlier batch. These tokens will then be included in the `holderBalance` of that address in a later batch, effectively receiving yield twice.

Consider the following scenario (with only 5 holders for simplicity):

totalAmount of yieldTokens: 1000\
holderBalance of A: 100\
holderBalance of B: 100\
holderBalance of C: 0\
holderBalance of D: 100\
holderBalance of E: 700\
Hence, effectiveTotalSupply: 1000

When the first batch, including A and B, is executed, both will receive `(totalAmount * holderBalance) / effectiveTotalSupply = 1000 * 100 / 1000 = 100` yield tokens. The yield token amount remaining in the contract is `1000 - 200 = 800`.

Next, A transfers their ArcTokens to C (this also works for any arbitrary address that is not yet a holder, as the overridden `_update()` function would add that address to the holders: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L701-L703>).

The second batch is executed, including C and D. Since the `holderBalance` of C now equals 100 due to the ArcTokens that A transferred, both C and D will also receive `1000 * 100 / 1000 = 100` yield tokens. The yield token amount remaining in the contract is `800 - 200 = 600`.

Finally, the last batch including E is executed. E should receive `1000 * 700 / 1000 = 700` yield tokens. However, the contract only holds `600` yield tokens, causing the transaction to revert. This happens because some ArcTokens were eligible to receive yield twice while the `effectiveTotalSupply` never changed. As a result, E will not be able to receive yield since the addresses involved in the attack have stolen it.

### Impact Details

Yield can be stolen from other holders between different `ArcToken::distributeYieldWithLimit()` calls. Affected holders will not be able to receive any yield, as the transaction will revert due to insufficient yield tokens remaining in the contract.

### References

Code references are provided throughout the report.

## Proof of Concept

{% stepper %}
{% step %}

### Step

`ArcToken::distributeYieldWithLimit()` is called to distribute yield to a subset of holders, including Alice (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466-L555>).
{% endstep %}

{% step %}

### Step

Alice transfers her tokens to Bob, who has not yet been included in any batch.
{% endstep %}

{% step %}

### Step

`ArcToken::distributeYieldWithLimit()` is called again to distribute yield to a different subset of holders, including Bob. The `holderBalance` of Bob now includes ArcTokens that were already included in an earlier batch (the tokens from Alice) (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L540>). Therefore, those ArcTokens will be eligible to receive yield multiple times.
{% endstep %}

{% step %}

### Step

When `ArcToken::distributeYieldWithLimit()` is called for the last batch, this transaction reverts because the contract does not hold enough tokens. This occurs due to the ArcTokens that were originally owned by Alice being eligible to receive yield twice. Hence, when a transfer amount exceeds the current contract balance, the transaction reverts (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L542>).
{% endstep %}
{% endstepper %}
