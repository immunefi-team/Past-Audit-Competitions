# #46108 \[SC-Medium] Minting Cap can by bypassed while self minting

**Submitted on May 25th 2025 at 00:34:50 UTC by @Oxgritty for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46108
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Minting.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

* Minting Cap can be bypassed when Agent Owner does self minting, due to lack of proper checks in `Minting.sol::selfMint`.

## Vulnerability Details

* When minting is done for a normal user or agent does self minting, two amounts are minted [mintValueUBA and \_poolFeeUBA](https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L204-#L205).
* So its important that, before minting we account for both the amounts while checking minting cap, but in `Minting.sol::selfMint` this [check](https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L90) is only done for `mintValue`, meaning `poolFee` will be minted over the limit.

## Impact Details

* Minting Cap plays an important role in ensuring proper functioning of the protocol and this bug allows the agent vault owner to do bypass it.

## References

* Correct checking when a user [mints](https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CollateralReservations.sol#L51)
* Flawed minting cap check when agent owner [self mints](https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L90)

## Proof of Concept

## POC \[Note: All units are in AMG for simplicity]

### Step 1: Assumptions we are making:

1. mintingCapAMG = 10000000000
2. mintValueAMG = 10000000000
3. Agent Owner has sufficient collateral in the system to self mint.
4. mintFee = 1000000000 (10% of mintValueAMG)
5. poolFee = 400000000 (40% of mintFee)

### Step 2: Agent Owner calls `MintingFacet.sol::selfMint`

* When control flow will reach `Minting.sol::selfMint`, minting cap check will be done for `mintValue` and not `mintValue + poolFee`.
* mintValue worth of fassets will be minted to minter address and poolFee worth of fassets will be minted to collateral pool, effectively bypassing the minting cap.
