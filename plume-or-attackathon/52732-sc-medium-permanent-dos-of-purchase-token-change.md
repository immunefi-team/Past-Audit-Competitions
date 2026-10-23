# 52732 sc medium permanent dos of purchase token change

**Submitted on Aug 12th 2025 at 18:18:10 UTC by @Afriauditor for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52732
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:** Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

`ArcTokenPurchase.setPurchaseToken()` cannot be called while any token is in the `enabledTokens` set. Because anyone can create an ArcToken through the factory and becomes that token’s admin, an attacker can enable a “dust” sale (e.g., 1 wei) that only they can later disable. This permanently blocks the Purchase admin from rotating the purchase currency (e.g., during a stablecoin depeg or migration), creating an indefinite denial-of-service.

### Vulnerability Details

The setter for purchase token includes this guard:

```solidity
function setPurchaseToken(address purchaseTokenAddress) external onlyRole(DEFAULT_ADMIN_ROLE) {
    PurchaseStorage storage ps = _getPurchaseStorage();
    if (ps.enabledTokens.length() > 0) {
        revert CannotChangePurchaseTokenWithActiveSales();
    }
    ...
}
```

`setPurchaseToken()` hard-reverts while any token remains in `enabledTokens`. Only token admins can remove themselves from that set:

```solidity
function disableToken(address _tokenContract) external onlyTokenAdmin(_tokenContract) {
    PurchaseStorage storage ps = _getPurchaseStorage();
    TokenInfo storage info = ps.tokenInfo[_tokenContract];
    if (!info.isEnabled) revert TokenNotEnabled();
    info.isEnabled = false;
    ps.enabledTokens.remove(_tokenContract);
    emit TokenSaleDisabled(_tokenContract);
}
```

`disableToken()` requires `onlyTokenAdmin(_tokenContract)`, i.e., the ArcToken’s own `ADMIN_ROLE` holder (not the ArcTokenPurchase admin).

Because `ArcTokenFactory.createToken(...)` is external and permissionless, an attacker can create an ArcToken where they hold `DEFAULT_ADMIN_ROLE`/`ADMIN_ROLE`/`MINTER_ROLE`, transfer 1 wei to `ArcTokenPurchase`, and call `enableToken()` which only checks that the token came from the configured factory and that the contract holds `_numberOfTokens`—to add it to `enabledTokens`. This works even if no purchase token is set yet. From there, the DoS is effectively permanent: there’s no admin override to clear `enabledTokens`; buying out the dust doesn’t help because `isEnabled` stays `true` and the token remains in the set; only the malicious token admin can call `disableToken()`. Changing the factory via `setTokenFactory()` doesn’t remove the already-enabled entry.

### Impact Details

Permanent inability to rotate the purchase currency regardless of whether any legitimate sale exists.

{% hint style="warning" %}
If a purchase currency needs to be rotated (e.g., stablecoin depeg or migration), the admin cannot do so while any token remains enabled. A malicious actor can create and enable a dust sale that only they can disable, resulting in indefinite DoS of the rotation operation.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Attacker creates the malicious token

* Attacker calls `ArcTokenFactory.createToken(...)` (factory already set in `ArcTokenPurchase.initialize`). Attacker becomes that token’s admin/minter.
  {% endstep %}

{% step %}

### Attacker mints and funds the sale contract

* Attacker mints dust: `ArcToken.mint(attacker, 1)`.
* Attacker transfers 1 wei of the token to `ArcTokenPurchase`.
  {% endstep %}

{% step %}

### Attacker enables a dust sale

* Attacker (as token admin) calls: `ArcTokenPurchase.enableToken(tokenAddr, _numberOfTokens=1, _tokenPrice=1)`
* Token is added to `enabledTokens`.
  {% endstep %}

{% step %}

### Admin is blocked from rotating the purchase token

* `ArcTokenPurchase.setPurchaseToken()` now reverts because `enabledTokens.length() > 0`.
* `disableToken(tokenAddr)` is callable only by the token’s admin (the attacker).
* Buying the 1 wei does not clear the entry: `isEnabled` remains true and the token stays in `enabledTokens`.
  {% endstep %}
  {% endstepper %}

Result: Indefinite DoS on purchase-currency rotation, regardless of whether any legitimate sale exists.

## References

(Original code referenced at the target link above.)
