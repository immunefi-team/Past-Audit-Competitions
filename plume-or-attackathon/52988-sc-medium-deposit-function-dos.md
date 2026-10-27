# 52988 sc medium deposit function dos

**Submitted on Aug 14th 2025 at 15:13:01 UTC by @frolic for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52988
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `deposit` function in the `TellerWithMultiAssetSupportPredicateProxy` contract attempts to transfer shares to the depositor after minting them via the underlying vault. However, due to the vault's share lock mechanism, the minted shares are locked to the proxy contract and cannot be transferred, causing the transfer to fail and preventing users from receiving their shares. This results in deposits being stuck and users unable to access their funds.

## Vulnerability Details

The issue arises from the interaction between `TellerWithMultiAssetSupportPredicateProxy` and the underlying vault's share lock mechanism.

In the proxy's `deposit` flow:

```solidity
shares = teller.deposit(depositAsset, depositAmount, minimumMint);
// ...
vault.safeTransfer(recipient, shares); // transfers the shares to the recipient
```

When `teller.deposit` is executed, shares are minted to the proxy contract, but the vault sets a share lock period for the recipient (the proxy). The vault tracks locks with `shareUnlockTime`:

```solidity
function _afterPublicDeposit(
        address user,
        ERC20 depositAsset,
        uint256 depositAmount,
        uint256 shares,
        uint256 currentShareLockPeriod
    )
        internal
    {
    shareUnlockTime[user] = block.timestamp + currentShareLockPeriod; 
    uint256 nonce = depositNonce;
    publicDepositHistory[nonce] =
        keccak256(abi.encode(user, depositAsset, depositAmount, shares, block.timestamp, currentShareLockPeriod));
    // ...
}
```

Because the proxy is recorded as the user for the minting step, `shareUnlockTime[proxy]` is set and prevents transfers from the proxy until the lock expires. The vault enforces this in `beforeTransfer`:

```solidity
function beforeTransfer(address from) public view {
    if (shareUnlockTime[from] > block.timestamp) revert TellerWithMultiAssetSupport__SharesAreLocked();
}
```

Therefore, when the proxy attempts `vault.safeTransfer(recipient, shares)` immediately after minting, the transfer reverts due to the active lock on the proxy, causing the entire `deposit` transaction to revert and resulting in a denial of service for depositors via the proxy.

## Impact Details

Any user attempting to deposit assets through the proxy will have their transaction reverted when the contract attempts to send the minted shares. The vault's `beforeTransfer` hook causes the revert because the proxy contract is subject to the newly set share lock. This leads to a denial of service for deposits made through the proxy — users cannot receive their minted shares and deposits fail.

## References

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L94>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupport.sol#L370>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupport.sol#L189>

## Proof of Concept

{% stepper %}
{% step %}

### Reproduce the issue

* User calls `deposit` on `TellerWithMultiAssetSupportPredicateProxy` with valid parameters.
* The proxy mints shares to itself via the underlying teller/vault.
* The proxy then attempts to transfer the minted shares to the user.
* The vault's `beforeTransfer` detects `shareUnlockTime[proxy] > block.timestamp` and reverts.
* The user does not receive shares and the transaction reverts.
  {% endstep %}
  {% endstepper %}
