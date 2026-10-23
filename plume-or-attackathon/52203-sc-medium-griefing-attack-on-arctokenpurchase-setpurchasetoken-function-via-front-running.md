# 52203 sc medium griefing attack on arctokenpurchase setpurchasetoken function via front running

**Submitted on Aug 8th 2025 at 17:24:09 UTC by @thesvn for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52203
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Permanent freezing of funds

## Description

### Brief/Intro

The `setPurchaseToken` function in the ArcTokenPurchase contract can be permanently blocked by malicious actors through a griefing attack. Once a purchase token is set and any token is enabled for sale, an attacker can prevent the `DEFAULT_ADMIN_ROLE` from ever changing the purchase token by maintaining at least one enabled token. This creates a denial-of-service condition that prevents the protocol from upgrading or migrating to a different purchase token, even when necessary due to technical issues, regulatory requirements, or protocol improvements.

### Vulnerability Details

The vulnerability exists in the `setPurchaseToken` function of `ArcTokenPurchase.sol`. The function includes a check that prevents changing the purchase token when there are active sales:

```solidity
function setPurchaseToken(
    address purchaseTokenAddress
) external onlyRole(DEFAULT_ADMIN_ROLE) {
    PurchaseStorage storage ps = _getPurchaseStorage();
    if (ps.enabledTokens.length() > 0) {
        revert CannotChangePurchaseTokenWithActiveSales();
    }
    if (purchaseTokenAddress == address(0)) {
        revert InvalidPurchaseTokenAddress();
    }
    ps.purchaseToken = IERC20(purchaseTokenAddress);
    emit PurchaseTokenUpdated(purchaseTokenAddress);
}
```

The fundamental problem is that once any token is enabled for sale, the contract permanently blocks the DEFAULT\_ADMIN\_ROLE from changing the purchase token. This means that if the protocol ever needs to upgrade, migrate, or recover from a misconfiguration of the purchase token (for example, due to a bug, a depegged stablecoin, or a regulatory requirement), it is impossible to do so without disabling all active sales. A malicious token creator can enable a sale and the DEFAULT\_ADMIN\_ROLE is forever prevented from updating the purchase token.

### Impact Details

{% hint style="warning" %}
High Likelihood, High Impact - Permanent Protocol Control Loss

Likelihood: High

* Low Barrier to Entry: Any user can create tokens via the public `createToken()` function
* Minimal Cost: Attack requires only gas fees for token creation and enabling
* Simple Execution: No technical expertise required beyond basic contract interaction
* Immediate Effect: Attack succeeds instantly once a token is enabled

Impact Level: High

* Permanent Protocol Lock: Once exploited, the protocol is permanently locked to the current purchase token
* Governance Failure: Admin loses control over critical protocol parameter (purchase token)
* Upgrade Prevention: Protocol cannot respond to technical issues, regulatory changes, or security concerns
* Economic Disruption: Inability to migrate to better or more stable purchase tokens
  {% endhint %}

## References

<details>

<summary>Relevant code links</summary>

* [ArcTokenPurchase.sol setPurchaseToken function](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L289-L301)
* [ArcTokenPurchase.sol enableToken function](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L142-L178)

</details>

## Proof of Concept

{% stepper %}
{% step %}

### Initial Setup

* ArcTokenPurchase contract is deployed and initialized
* Admin sets USDC as the purchase token
* Contract is operational with USDC as purchase currency
  {% endstep %}

{% step %}

### Attacker Preparation

* Attacker calls `ArcTokenFactory.createToken()` to create a new ArcToken
* Attacker becomes the token admin with full privileges
* Attacker transfers some tokens to ArcTokenPurchase contract
  {% endstep %}

{% step %}

### Griefing Attack

* Attacker calls `ArcTokenPurchase.enableToken(attackerToken, amount, price)`
* Function succeeds; `enabledTokens.length()` becomes 1
  {% endstep %}

{% step %}

### Admin Attempt to Change Purchase Token

* Admin tries to call `setPurchaseToken(USDT_ADDRESS)` to migrate from USDC to USDT
* Function reverts with `CannotChangePurchaseTokenWithActiveSales()`
* Admin is permanently blocked from changing the purchase token while any token remains enabled
  {% endstep %}

{% step %}

### Protocol State

* Purchase token remains locked to USDC
* Protocol cannot upgrade to USDT or any other token while the enabled token exists
* All future purchase token changes are blocked while any token is enabled
  {% endstep %}
  {% endstepper %}
