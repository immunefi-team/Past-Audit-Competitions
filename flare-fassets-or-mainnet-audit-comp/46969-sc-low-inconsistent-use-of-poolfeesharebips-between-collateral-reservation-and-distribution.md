# #46969 \[SC-Low] Inconsistent Use of poolFeeShareBIPS Between Collateral Reservation and Distribution

**Submitted on Jun 6th 2025 at 23:00:55 UTC by @Josh4324 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46969
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The protocol allows a minter to reserve collateral using the reserveCollateral function. During this process, the poolFeeShareBIPS which determines how much of the reservation fee is sent to the pool is recorded and stored as part of the collateral reservation (cr.poolFeeShareBIPS). However, during the actual minting step, the function distributeCollateralReservationFee calculates the fee split using the current value of \_agent.poolFeeShareBIPS from storage instead of using the previously stored cr.poolFeeShareBIPS.

## Vulnerability Details

The stored cr.poolFeeShareBIPS is intended to represent the fee split agreement at the time of reservation. However, because the actual distribution later relies on the mutable Agent.State.poolFeeShareBIPS, a malicious or opportunistic agent can change the value after the reservation but before minting. This allows the agent to redirect a larger share of the fee to themselves, violating the implicit fee agreement made at the time of reservation.

This inconsistency breaks the assumption that the parameters agreed upon during reservation will be respected during fee distribution.

## Impact Details

Minter Exploitation: Minters are subjected to fee changes after reservation, leading to potential overpayment to the agent.

Agent Manipulation: Dishonest agents can exploit this timing window to divert more fees from the pool to their own vaults.

Loss of Trust: Undermines the reliability of the fee mechanics and may discourage protocol participants from interacting with certain agents.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Proof of Concept (PoC) Agent sets poolFeeShareBIPS to 80% (to make the pool look attractive).

Minter calls reserveCollateral, and cr.poolFeeShareBIPS = 81 (value + 1 offset) is stored.

Agent immediately updates poolFeeShareBIPS to 10%.

During minting, distributeCollateralReservationFee() is called.

The new 10% value is used instead of the agreed-upon 80%, and the agent keeps most of the reservation fee.
