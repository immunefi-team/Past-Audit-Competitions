# 52620 sc medium permanently dos to arctokenpurchase contract

* Submitted on Aug 12th 2025 at 03:48:59 UTC by @IronsideSec for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #52620
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* Impacts:
  * Protocol insolvency

## Description

Any factory user can permanently DoS the contract’s purchase-token configuration. An attacker can enable a sale for a token they control (minted via the public factory) before the admin sets `purchaseToken`. After any token is enabled, `setPurchaseToken` always reverts, and only the token’s admin can disable the sale. This permanently blocks the purchase-token configuration and operational use of the storefront.

Short summary:

* `setPurchaseToken` needs to be set in order for users to buy the enabled arc tokens. If an attacker enables a token sale before the admin calls `setPurchaseToken`, the admin can never set the purchase token because `setPurchaseToken` reverts whenever there are enabled sales.
* Calling `setPurchaseToken` in the contract's initialization transaction would avoid this race; relying on a subsequent transaction risks permanent DoS.

## Vulnerability Details

Root causes

* `setPurchaseToken` reverts if any arc token is enabled.
* `enableToken` lacks a guard ensuring a non-zero `purchaseToken`.
* Only token admins can call `disableToken`. The factory grants `ADMIN_ROLE` to the token creator; that creator can revoke roles granted to the factory during token creation, preventing admins from disabling the sale.

Exploit flow (PoC steps)

{% stepper %}
{% step %}

### Step: Deploy and initialize

Deploy a new `ArcTokenPurchase` and initialize it with a valid `factory`, but do not call `setPurchaseToken` in the same transaction.
{% endstep %}

{% step %}

### Step: Create token

Attacker creates a token via `factory.createToken`, becoming that token's admin.
{% endstep %}

{% step %}

### Step: Fund and enable sale

Attacker transfers 1 wei of their token to the purchase contract and calls `enableToken(token, 1, 1)`.
{% endstep %}

{% step %}

### Step: Admin attempt to set purchase token

When the ArcTokenPurchase admin attempts `setPurchaseToken(...)`, the call reverts with `CannotChangePurchaseTokenWithActiveSales`.
{% endstep %}

{% step %}

### Step: Admin cannot disable sale

The admin cannot call `disableToken(...)` because it reverts with `NotTokenAdmin`. Only the token admin (attacker) can disable, so the admin is locked out and the purchase-token configuration is permanently blocked.
{% endstep %}
{% endstepper %}

Notes

* This issue does not occur if `enableToken` validates that `purchaseToken` is set (non-zero) before enabling a sale.

Relevant code snippets

* setPurchaseToken (reverts if any enabled tokens exist):

```solidity
function setPurchaseToken(address purchaseTokenAddress) external onlyRole(DEFAULT_ADMIN_ROLE) {
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

Source: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L293-L295>

* enableToken (no check that purchase token is set):

```solidity
function enableToken(
    address _tokenContract,
    uint256 _numberOfTokens,
    uint256 _tokenPrice
)   external 
    onlyTokenAdmin(_tokenContract) {
    // ...
    ps.tokenInfo[_tokenContract] =
        TokenInfo({ isEnabled: true, tokenPrice: _tokenPrice, totalAmountForSale: _numberOfTokens, amountSold: 0 });

    ps.enabledTokens.add(_tokenContract);
    // ...
}
```

Source: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L142-L175>

* disableToken (only token admin can call):

```solidity
function disableToken(address _tokenContract) 
    external 
    onlyTokenAdmin(_tokenContract) 
{
    PurchaseStorage storage ps = _getPurchaseStorage();
    TokenInfo storage info = ps.tokenInfo[_tokenContract];
    // ...
    info.isEnabled = false;
    ps.enabledTokens.remove(_tokenContract);

    emit TokenSaleDisabled(_tokenContract);
}
```

Source: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L184>

## Impact Details

* Permanent denial-of-service on purchase-token configuration:
  * Admin cannot set or change `purchaseToken` once any sale is enabled by any token admin.
  * Only token admins can disable their sale; the purchase admin is locked out.
* Operational halt of storefront lifecycle:
  * Prevents initial configuration and future rotation of the purchase currency.
  * Requires contract upgrade or external intervention to recover.
* Severity: High. Any factory user can permanently brick a fresh `ArcTokenPurchase` instance before configuration.

## Remediation

{% hint style="info" %}

* In `enableToken`, require purchase token to be set:
  * Add: `if (address(ps.purchaseToken) == address(0)) revert PurchaseTokenNotSet();`
* Optional hardening:
  * Provide an admin override to force-disable all enabled tokens or allow `setPurchaseToken` if all enabled tokens have `amountSold == 0`.
    {% endhint %}

## Proof of Concept

* Exploit gist: <https://gist.github.com/IronsideSec/ea8dc6359e40bae2e323bc0e89d02bc1>
* Follow the steps in the gist for a working PoC.

## References

See code snippets with GitHub links above.
