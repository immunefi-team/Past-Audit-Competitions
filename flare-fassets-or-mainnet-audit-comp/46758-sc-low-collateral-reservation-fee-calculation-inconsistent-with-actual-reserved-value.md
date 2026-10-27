# #46758 \[SC-Low] Collateral Reservation Fee Calculation Inconsistent with Actual Reserved Value

**Submitted on Jun 4th 2025 at 09:31:48 UTC by @light279 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46758
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The function `CollateralReservations::reserveCollateral` is responsible for reserving collateral in preparation for minting f-assets. It reserves collateral based on both the requested minting amount (`valueAMG`) and an additional amount minted as a pool fee (`poolFeeAMG`). However, when calculating the required reservation fee, only `valueAMG` is considered, potentially leading to under-collection of fees compared to the actual value reserved.

## Vulnerability Details

When a user reserves collateral to mint f-assets through `CollateralReservations::reserveCollateral` , the following logic is used:

```
_reserveCollateral(agent, valueAMG + _currentPoolFeeAMG(agent, valueAMG));
```

This means collateral is reserved for both the user's requested amount and the agent's pool fee. However, the reservation fee charged to the user is calculated using only `valueAMG`:

```
uint256 reservationFee = agent.availableAgentsPos != 0
    ? _reservationFee(collateralData.poolCollateral.amgToTokenWeiPrice, valueAMG)
    : 0;
```

This results in an inconsistency: the fee collected does not correspond to the total value of assets being reserved. This could create a mismatch between the economic intent of the system and actual cost imposed on users.

## Impact Details

* Fee Undercalculation: The protocol collects reservation fees based on only a portion of the actual reserved collateral.
* An agent can configure a higher minting fee and a larger share of the pool fee, which increases the total collateral reserved during a mint. However, since the reservation fee is calculated only on the user's mint amount and not the full reserved value (including the pool fee), the agent effectively receives less fee coverage for more collateral being locked.

## Proof of Concept

## Proof of Concept

1. Assume the agent fee is non-zero and \_lots = 1. Let valueAMG = 100. 2.Call reserveCollateral(...) for this 1 lot through CollateralReservationFacet.
2. Internally, collateral reserved will be:

```
_reserveCollateral(agent, 100 + poolFeeAMG);
```

Let's say poolFeeAMG = 2, so total reserved = 102.

4. However, reservation fee is computed only using 100:

```
uint256 reservationFee = agent.availableAgentsPos != 0
            ? _reservationFee(collateralData.poolCollateral.amgToTokenWeiPrice, 100)
            : 0;
```

**Mismatch:** The user is paying fee for 100 AMG reservation while 102 AMG worth is reserved.

**Result:** Fee is undercharged for what is effectively being locked in the system.
