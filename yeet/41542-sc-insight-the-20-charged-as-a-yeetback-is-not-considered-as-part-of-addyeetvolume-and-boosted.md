# #41542 \[SC-Insight] The 20% charged as a \`yeetback\` is not considered as part of \`addYeetVolume\` and \`boostedValue\`

**Submitted on Mar 16th 2025 at 12:27:35 UTC by @robin\_bl4z3 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41542
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Documentation says the pool's value of 100% is distributed like this

80% of the pot goes to the lastYeet (Winner)\
20% goes to the 10 random users\
But the 20% is not considered when it comes to `addYeetVolume`, only the 80% of the deposited amount after tax is. It should be 100% after-tax amount that determines the $Yeet Reward a user should get.

## Vulnerability Details

As the docs said (Check reference or screenshot #2 in Attachments), the mechanism is that `15%` of every yeet is taxed, then the `ValueAftertax` is distributed as 80-20. 80% to the winner and 20% to other random winners.

But as you know, when a user participates, they provide `$BERA` and receive `$Yeet` as a reward according to the `$BERA` amount they provided. Aside from tax, the contract only takes `80%` as the volume provided by the user, and that determines the `$Yeet` and `Boosted Value` they will get. But that seems to be incorrect as the `20%` that would go to the other 10 winners is not considered for `$Yeet` rewards, as the docs said even if the user does not wish to participate but only wants to earn `$Yeet` rewards they can do so, but they will only be entitled to `80%` of their YeetVolume

## Impact Details

Users do not get their promised returns as it was stated in the docs (Scrrenshot #1) that calculating `BoostedValue` depends on how much $BERA an address has yeeted in an epoch (assuming it excludes tax).

Users will be entitled to only 80%`of their`YeetVolume`and leaving the`20%`, which will result in the low amount of` $Yeet`Reward and`BoostedValue\` they will get.

## References

<https://docs.yeetit.xyz/yeet/yeet-game/mechanics> (YeetingRewards section and Lottery Section)

## Recommendation

Consider the amount of $BERA an address has yeeted to calculate the $Yeet and `BoostedValue` they will receive.

## Proof of Concept

## Proof of Concept

Let's assume that Finding 5 is fixed from the pre-audit (for simplicity)

1. When we look at the \_yeet function, it calls `getDistribution` which will calculate tax (10% of the total amount), the yeetback amount (20% of the taxed amount), and the PotValue (80% of the taxed amount). and return values

* <https://github.com/yeet-protocol/contracts/blob/main/src/Yeet.sol#L284C2-L285C38>

```solidity
        (uint256 valueToPot, uint256 valueToYeetback, uint256 valueToStakers, uint256 publicGoods, uint256 teamRevenue)
        = getDistribution(msg.value);
```

2. That means the `valueToPot` represents the 80% of one winner and valueToYeetback represents 20% of 10 winners, which makes up 100%. But when we go get the `addYeetVolume` (or `BoostedValue`). The 80% `valueToPot` is used to reward the user $Yeet rewards.

* <https://github.com/yeet-protocol/contracts/blob/main/src/Yeet.sol#L329C8-L330C65>

```solidity
        uint256 boostedValue = getBoostedValue(msg.sender, valueToPot, tokenIds);
        rewardsContract.addYeetVolume(msg.sender, boostedValue);
```
