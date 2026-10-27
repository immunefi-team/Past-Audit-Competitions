# #41374 \[SC-Insight] Incorrect NFT Boost Value in Lookup Array

**Submitted on Mar 14th 2025 at 13:21:51 UTC by @DoD4uFN for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41374
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The nftBoostLookup array in the Yeet contract contains an incorrect boost value for users holding 11 Yeetard NFTs. Instead of the intended boost of 11.1% (represented as 1110 in a 10000 scale), the array sets the value at 1100. This discrepancy can lead to slightly lower rewards for affected users, undermining the fairness of the reward distribution mechanism.

## Vulnerability Details

The contract uses the nftBoostLookup array to determine the boost percentage based on the number of NFTs a user holds. The array is defined as follows:

```solidity
uint256[26] public nftBoostLookup = [
    0,
    345,
    540,
    675,
    765,
    840,
    900,
    960,
    1005,
    1050,
    1080,
    1100,   // This value should be 1110 for 11 NFTs as per the documentation.
    1155,
    1185,
    1215,
    1245,
    1275,
    1305,
    1335,
    1365,
    1380,
    1400,
    1440,
    1455,
    1470,
    1500
];
```

The intended boost mechanism, as described in the documentation, applies a concave curve to limit the rewards of large NFT holders, capping the maximum boost at 15% for 25 NFTs. For 11 NFTs, the expected boost is 11.1% (1110), not 11.0% (1100).

## Impact Details

The immediate impact is that users holding 11 NFTs will receive a 0.1% lower boost on their yeet rewards than expected, potentially reducing their reward share over multiple rounds. Although the absolute difference might appear minimal, in aggregate over many epochs and among a large user base, this miscalculation can lead to discrepancies in reward distribution.

## References

Here is the documentation about the percentages of boosts depending on the number of NFTs: <https://docs.yeetit.xyz/yeet/yeet-game/nftboostweight>

and here is the relevant line of code:\
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L199>

## Proof of Concept

## Proof of Concept

### Step 1: Understanding the nftBoostLookup Mechanism

The `nftBoostLookup` array in the Yeet contract determines the boost percentage applied to rewards based on the number of NFTs a user holds. Each index in the array corresponds to the number of NFTs, and the stored value represents the percentage boost multiplied by 100 (e.g., 11.1% is stored as 1110).

### Step 2: Identifying the Incorrect Value

According to the project documentation, a user holding 11 NFTs should receive a boost of 11.1% (1110). However, the contract incorrectly assigns a value of 1100, which corresponds to an 11.0% boost.

**Incorrect implementation:**

```solidity
uint256[26] public nftBoostLookup = [
    0, 345, 540, 675, 765, 840, 900, 960, 1005, 1050, 1080,
    1100,   // This value should be 1110 for 11 NFTs.
    1155, 1185, 1215, 1245, 1275, 1305, 1335, 1365, 1380, 1400, 1440, 1455, 1470, 1500
];
```

### Step 3: Reproducing the Issue

1. **Set up the environment:** Deploy the Yeet contract and ensure it contains the `nftBoostLookup` array as specified.
2. **Hold 11 NFTs:** A test account should acquire exactly 11 Yeetard NFTs.
3. **Check the applied boost:** Retrieve the boost value using the contract function that references `nftBoostLookup`.
4. **Compare expected vs. actual output:** The contract will return `1100` instead of `1110`, indicating a 0.1% discrepancy.

### Step 4: Impact Analysis

Users holding exactly 11 NFTs will receive 11.0% instead of 11.1% boost on their rewards, leading to slightly lower returns.

### Step 5: Suggested Fix

The fix involves updating the `nftBoostLookup` array to reflect the correct value at index `11`:

```solidity
uint256[26] public nftBoostLookup = [
    0, 345, 540, 675, 765, 840, 900, 960, 1005, 1050, 1080,
    1110,   // Corrected value for 11 NFTs
    1155, 1185, 1215, 1245, 1275, 1305, 1335, 1365, 1380, 1400, 1440, 1455, 1470, 1500
];
```

This ensures the contract applies the correct 11.1% boost, maintaining consistency with the documented behavior and ensuring fair rewards distribution.
