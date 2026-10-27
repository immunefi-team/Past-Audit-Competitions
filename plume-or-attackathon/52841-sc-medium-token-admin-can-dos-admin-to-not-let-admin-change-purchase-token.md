# 52841 sc medium token admin can dos admin to not let admin change purchase token

* Submitted on Aug 13th 2025 at 14:57:15 UTC by @ladboy233 for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #52841
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

(see Vulnerability Details)

## Vulnerability Details

In `ArchTokenPurchase.sol`, the admin should be able to `setPurchaseToken` and change the purchase token.

Relevant code:

```solidity
   function setPurchaseToken(
        address purchaseTokenAddress
    ) external onlyRole(DEFAULT_ADMIN_ROLE) {
        PurchaseStorage storage ps = _getPurchaseStorage();
        if (ps.enabledTokens.length() > 0) { // @audit token owner can block admin from change the purchase token for sure.
            revert CannotChangePurchaseTokenWithActiveSales();
        }
        if (purchaseTokenAddress == address(0)) {
            revert InvalidPurchaseTokenAddress();
        }
        ps.purchaseToken = IERC20(purchaseTokenAddress);
        emit PurchaseTokenUpdated(purchaseTokenAddress);
    }
```

However, the code requires that no token is enabled:

```solidity
 if (ps.enabledTokens.length() > 0) { // @audit token owner can block admin from change the purchase token for sure.
   revert CannotChangePurchaseTokenWithActiveSales(); 
 }
```

But the token admin can permissionlessly [enable a token](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L146) and never disable a token:

```solidity
   modifier onlyTokenAdmin(
        address _tokenContract
    ) {
        address adminRoleHolder = msg.sender;
        bytes32 adminRole = ArcToken(_tokenContract).ADMIN_ROLE();
        if (!ArcToken(_tokenContract).hasRole(adminRole, adminRoleHolder)) {
            revert NotTokenAdmin(adminRoleHolder, _tokenContract);
        }
        _;
    }
```

Therefore the token admin can DOS the protocol admin from changing the purchase token by simply not disabling the token or by enabling a token before the protocol `setPurchaseToken` call.

## Impact Details

{% hint style="warning" %}
The token admin can permissionlessly DOS the protocol admin to prevent the protocol admin from changing the purchase token by leaving any token enabled (or enabling one before `setPurchaseToken`).
{% endhint %}

## Recommendation

Admin should be able to disable token

(Ensure protocol admin has a way to clear/disable enabled tokens or that `setPurchaseToken` can be executed despite enabled tokens under controlled conditions.)

## Proof of Concept

Please see above.
