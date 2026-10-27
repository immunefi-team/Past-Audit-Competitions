# #41132 \[SC-Insight] NFT Boost Lookup values not adhere to docs

**Submitted on Mar 11th 2025 at 14:18:17 UTC by @peppef for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41132
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol>
* **Impacts:**

## Description

## Brief/Intro

A mismatch exists between the values in `NFTBoostWeight` page of Yeet Documentation and the `nftBoostLookup` array in `Yeet.sol`

## Vulnerability Details

On the docs, the page NFTBoostWeight describes the boosts in percentage that a player will receive in $YEET tokens depending on the number of NFTs.\
In `Yeet.sol` the array `nftBoostLookup` holds all the boost values from 0 to 1500, which corresponds to 15%, namely the maximum boost a user can receive by holding 25 NFTs.\
Particularly if a user holds 21 NFTs the boost should be 14.1% while `nftBoostLookup` at the related position 1400 is stored instead of 1410.

## Impact Details

No vulnerability can be found regarding the issue, but it shows a mismatch between docs and code

## Proof of Concept

## Proof of Concept

N/A
