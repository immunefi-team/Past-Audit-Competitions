# 52397 sc medium repeated approve without zero reset can revert on nonstandard erc20s blocking deposits

**Submitted on Aug 10th 2025 at 12:13:50 UTC by @Khay3 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52397
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Summary

The contract calls `safeApprove(spender, amount)` repeatedly without first setting the allowance to zero. Some ERC20 tokens (e.g., USDT) require resetting allowance to zero before changing a non-zero allowance; repeated deposits can therefore revert and cause a DoS of the deposit flow.

### Vulnerability Detail

\[TellerWithMultiAssetSupportPredicateProxy::deposit()]:

```solidity
ERC20 vault = ERC20(teller.vault());
depositAsset.safeApprove(address(vault), depositAmount);
depositAsset.safeTransferFrom(msg.sender, address(this), depositAmount);
shares = teller.deposit(depositAsset, depositAmount, minimumMint);
```

\[TellerWithMultiAssetSupportPredicateProxy::depositAndBridge()]:

```solidity
ERC20 vault = ERC20(teller.vault());
depositAsset.safeApprove(address(vault), depositAmount);
depositAsset.safeTransferFrom(msg.sender, address(this), depositAmount);
teller.depositAndBridge{ value: msg.value }(depositAsset, depositAmount, minimumMint, data);
```

## Impact

Users can experience transaction reverts and be unable to deposit or bridge assets that enforce zero-reset allowance semantics, resulting in effective DoS for affected assets until manual intervention or a code change fixes the allowance handling.

## Recommendation

{% hint style="info" %}

* Use a safe "reset-then-set" flow: if current allowance != 0, call `safeApprove(spender, 0)`, then `safeApprove(spender, depositAmount)`.
* Alternatively, use `safeIncreaseAllowance` to increase the allowance by the exact needed delta and ensure the vault fully spends allowances.
  {% endhint %}

## Proof of Concept

<details>

<summary>Conditions and steps</summary>

The issue can be reproduced under these conditions:

* A token that requires zero-first allowance semantics (for example, USDT).
* A residual allowance remains non-zero between deposits, or the vault underspends the approved allowance.

First depositFirst deposit approves depositAmount. Vault spends less than depositAmount (or none), leaving a non-zero residual allowance.Second depositSecond deposit again calls safeApprove(spender, depositAmount) without zero-reset.RevertUSDT-like tokens revert on non-zero-to-non-zero allowance change → the second deposit call reverts.

</details>
