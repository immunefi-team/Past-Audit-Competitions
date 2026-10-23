# #41511 \[SC-Low] The contract calculates the \`minimumYeetPoint\` using the Pot going to the winner instead of the whole Pot.

**Submitted on Mar 16th 2025 at 00:09:11 UTC by @robin\_bl4z3 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41511
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeetback.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

As the docs say (See the Reference or the screenshot), the `minimumYeetPoint` is determined using the Pot Value (after the bootstrap has ended), but only 1 Pot Value is used to calculate this.

## Vulnerability Details

There are two types of Pots in this system: `potToWinner` and `potToYeetback`, but the contract only uses the `potToWinner` to calculate what the next user is required to yeet

## Impact Details

* The first impact is that as the docs promised, "minimum yeet size increases with the pot size" But only 1 of 2 pot sizes is being used to calculate the minimum yeet, which is `failing to deliver promised returns`.
* The second one is that the protocol loses the tax they would have been able to get as if the actual pool value was used to increase the `minimum yeet` required more tax would be collected. So the Protocol loses some of their tax that would accumulate faster.

## References

* <https://docs.yeetit.xyz/yeet/yeet-game> (Step 6 b.)

## Recommendation

Change the `minimumYeetPoint` to account for both pots

```solidity
- uint256 minimumYeetPoint = _minimumYeetPoint(potToWinner);

+ uint256 minimumYeetPoint = _minimumYeetPoint(potToWinner + potToYeetback);
```

## Proof of Concept

## Proof of Concept

1. As you can see when the user yeets, only the `PotToWinner` is taken to consideration when calculating the `minimumYeetPoint` and not both Pots

* <https://github.com/yeet-protocol/contracts/blob/main/src/Yeet.sol#L279>

```solidity
uint256 minimumYeetPoint = _minimumYeetPoint(potToWinner);
```

2. This is how the `minimumYeetPoint` is calculated when Bootstrap has ended; it takes the `totalPot` as the param value (But only the PotToWinner is passed)

* <https://github.com/yeet-protocol/contracts/blob/main/src/Yeet.sol#L447C4-L461C6>

```solidity
 function _minimumYeetPoint(uint256 totalPot) private view returns (uint256) {
        if (totalPot == 0) {
            return MINIMUM_YEET_POINT;
        }

        if (isBoostrapPhase()) {
            return MINIMUM_YEET_POINT;
        }

        uint256 min = totalPot / POT_DIVISION;
        if (min < MINIMUM_YEET_POINT) {
            return MINIMUM_YEET_POINT;
        }
        return min;
    }
```

3. On the Tax side, it relies on the `msg.value`, while `msg.value` relies on the returned `minimumYeetPoint`, which is only for the `PotToWinner`.

* <https://github.com/yeet-protocol/contracts/blob/main/src/Yeet.sol#L278C1-L285C38>

```solidity
        uint256 minimumYeetPoint = _minimumYeetPoint(potToWinner);
        if (msg.value < minimumYeetPoint) {
            revert InsufficientYeet(msg.value, minimumYeetPoint);
        }

        (uint256 valueToPot, uint256 valueToYeetback, uint256 valueToStakers, uint256 publicGoods, uint256 teamRevenue)
        = getDistribution(msg.value);
```
