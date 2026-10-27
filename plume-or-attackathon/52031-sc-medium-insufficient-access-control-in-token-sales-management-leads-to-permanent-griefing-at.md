# 52031 sc medium insufficient access control in token sales management leads to permanent griefing attack

**Submitted on Aug 7th 2025 at 12:43:59 UTC by @OxPrince for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52031
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

The `ArcTokenPurchase` contract contains an access control vulnerability where token admins can permanently lock the contract admin's ability to update the purchase token configuration. A malicious token admin can enable a token sale with just 1 wei, which adds the token to the enabledTokens set and prevents the contract admin from calling `setPurchaseToken()` due to the `CannotChangePurchaseTokenWithActiveSales` error. Because only token admins can disable tokens, this can create an irreversible parameter freeze attack that permanently compromises core functionality.

## Vulnerability Details

* Roles involved:
  * Contract Admin: has `DEFAULT_ADMIN_ROLE` on `ArcTokenPurchase` and can call `setPurchaseToken()` (ArcTokenPurchase.sol:289-301).
  * Token Admin: has `ADMIN_ROLE` on individual ArcToken contracts and can call `enableToken()` and `disableToken()` (ArcTokenPurchase.sol:125-134).
* Root cause and mechanics:
  * `enableToken()` only requires `_numberOfTokens > 0`, allowing a token admin to enable a sale with just 1 wei (1 base unit) (ArcTokenPurchase.sol:150-152).
  * When a token is enabled, it is added to the `enabledTokens` set (ArcTokenPurchase.sol:175).
  * `setPurchaseToken()` reverts if any tokens are enabled by checking that the `enabledTokens` set is empty (ArcTokenPurchase.sol:293-295).
  * Only token admins can call `disableToken()` to remove tokens from `enabledTokens` (ArcTokenPurchase.sol:184-196).
* Consequence:
  * A token admin can enable a token with minimal cost and permanently prevent the contract admin from updating the purchase token. This constitutes an irreversible parameter freeze, since the contract admin cannot disable the token.

## Impact Details

* Permanent denial of service on a critical contract parameter:
  * Parameter Freeze: Contract admin cannot update the purchase token address, blocking:
    * Migration to new stablecoins or payment tokens
    * Responses to token deprecations or security issues
    * Adaptation to changing business requirements
* Low-cost execution: attack can be executed with minimal funds (e.g., 1 wei).
* Recovery requires deploying a new contract or upgrading the implementation.

## References

* ArcTokenPurchase contract: arc/src/ArcTokenPurchase.sol
* Test demonstrating the vulnerability: arc/test/ArcTokenPurchase.t.sol:322-348
* Protocol documentation: arc/README.md:200-267

## Proof of Concept

{% stepper %}
{% step %}

### Setup

Contract admin deploys `ArcTokenPurchase` with an initial purchase token (e.g., USDC).
{% endstep %}

{% step %}

### Token Creation

A token is created via `ArcTokenFactory` with a separate token admin.
{% endstep %}

{% step %}

### Minimal Token Transfer

Token admin transfers just 1 wei of their token to the `ArcTokenPurchase` contract.
{% endstep %}

{% step %}

### Griefing Attack

Token admin calls `enableToken(tokenAddress, 1, anyPrice)` to enable a sale with 1 wei.
{% endstep %}

{% step %}

### Parameter Lock

Contract admin attempts to call `setPurchaseToken(newTokenAddress)` but the transaction reverts with `CannotChangePurchaseTokenWithActiveSales`.
{% endstep %}

{% step %}

### Permanent Effect

Contract admin cannot disable the token because `disableToken()` requires token admin privileges, creating a permanent lock on purchase token updates.

The test case in arc/test/ArcTokenPurchase.t.sol:322-348 demonstrates this scenario, showing how an enabled token prevents purchase token updates until the token is disabled by its admin.
{% endstep %}
{% endstepper %}
