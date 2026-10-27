# 49732 sc medium malicious token admin can permanently block setpurchasetoken

**Submitted on Jul 18th 2025 at 20:31:50 UTC by @magtentic for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49732
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

{% hint style="info" %}
Brief/Intro

A malicious user can implement a custom `ArcToken` contract, assign themselves the `ADMIN_ROLE`, and use it to call `enableToken` on `ArcTokenPurchase`. This adds the malicious token to `enabledTokens`, which permanently blocks the `setPurchaseToken` function from being called by legitimate admins.
{% endhint %}

## Vulnerability Details

Currently within `ArcTokenPurchase`, any user can call `enableToken` by having the `ADMIN_ROLE` on the `ArcToken` contract and transferring some tokens to make it seem like they have started a sale. This process effectively adds the provided `_tokenContract` to `enabledTokens`:

```solidity
function enableToken(
    address _tokenContract,
    uint256 _numberOfTokens,
    uint256 _tokenPrice
) external onlyTokenAdmin(_tokenContract) {
    ...
    ps.enabledTokens.add(_tokenContract);

    emit TokenSaleEnabled(_tokenContract, _numberOfTokens, _tokenPrice);
}
```

The `onlyTokenAdmin` modifier simply checks if the caller has `ADMIN_ROLE` on the `_tokenContract` contract:

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

Once added, the token is considered an active sale and included in the storage check for `setPurchaseToken`:

```solidity
function setPurchaseToken(
    address purchaseTokenAddress
) external onlyRole(DEFAULT_ADMIN_ROLE) {
    PurchaseStorage storage ps = _getPurchaseStorage();
    if (ps.enabledTokens.length() > 0) {
        revert CannotChangePurchaseTokenWithActiveSales();
    }
    ...
}
```

The only way to remove the `_tokenContract` from the `enabledTokens` list is by calling `disableToken` which can only be done by the `_tokenContract` admin. A malicious actor will not do this, intentionally leaving the contract in a blocked state:

```solidity
function disableToken(address _tokenContract) external onlyTokenAdmin(_tokenContract) {
    ...
    ps.enabledTokens.remove(_tokenContract);
    ...
}
```

## Impact Details

* A malicious token admin can permanently lock the `setPurchaseToken` function, blocking protocol upgrades or configuration changes. Since only the admin of the enabled token can disable it, there is no recovery path once the malicious token is added.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>

<details>

<summary>Proof of Concept</summary>

Details provided in bounty report.

</details>
