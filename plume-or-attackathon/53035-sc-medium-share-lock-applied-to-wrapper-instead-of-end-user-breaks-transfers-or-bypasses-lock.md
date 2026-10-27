# 53035 sc medium share lock applied to wrapper instead of end user breaks transfers or bypasses lock

**Submitted on Aug 14th 2025 at 17:48:31 UTC by @Afriauditor for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53035
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** Protocol insolvency

## Description

### Brief / Intro

The vault’s share lock mechanism is applied to the wrapper contract’s address rather than the actual end user in both the bridge (`depositAndBridge`) and non-bridge (`depositOneInch`) deposit flows. Depending on configuration this causes one of two critical outcomes:

* If the vault enforces the `BeforeTransferHook`, immediate post-deposit transfers revert (flows break).
* If the hook is disabled, users receive transferable shares instantly, effectively bypassing the intended lock.

Either outcome undermines the intended security model and can brick core functionality.

### Vulnerability Details

When the wrapper calls into the teller (TellerWithMultiAssetSupport) it is the wrapper that is `msg.sender`. This results in:

* Shares minted to the wrapper
* The share lock recorded against the wrapper's address rather than the end-user

Example behavior in the teller called by `DexAggregatorWrapperWithPredicateProxy`:

```solidity
// shares minted to msg.sender (the wrapper)
shares = _erc20Deposit(depositAsset, depositAmount, minimumMint, msg.sender);

// lock recorded for msg.sender (the wrapper), not the user
_afterPublicDeposit(msg.sender, depositAsset, depositAmount, shares, shareLockPeriod);
```

Consequences:

* If the vault enforces `BeforeTransferHook`, any immediate transfer from the wrapper (to the user or onward into the bridge flow) invokes `beforeTransfer(wrapper)` and triggers: `if (shareUnlockTime[from] > block.timestamp) revert SharesAreLocked();` — causing the transfer to revert.
* If the hook is disabled, the user receives transferable shares immediately (lock bypassed).

## Impact Details

{% hint style="warning" %}
All wrapper-based deposit paths (both `depositOneInch` and `depositAndBridge`) are blocked when the vault enforces `BeforeTransferHook`, because moving the freshly minted shares out of the wrapper immediately triggers `SharesAreLocked`. If the hook is not enforced, the intended share lock is effectively bypassed and users obtain transferable shares immediately, breaking invariants.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Setup

* Configure the vault to invoke `BeforeTransferHook.beforeTransfer(from)` on every transfer.
* Set `shareLockPeriod = 2 days` in the teller.
  {% endstep %}

{% step %}

### Trigger

* Call either `depositOneInch` (non-bridge) or `depositAndBridge` (bridge) via the wrapper.
  {% endstep %}

{% step %}

### Observed behavior

* Inside the teller, shares are minted to the wrapper; `_afterPublicDeposit` sets `shareUnlockTime[wrapper] = now + 2 days`.
* The wrapper immediately attempts to move those shares (to the user or into the bridge flow).
* Vault calls `beforeTransfer(wrapper)` → sees lock active → reverts with `SharesAreLocked`.
  {% endstep %}
  {% endstepper %}

<details>

<summary>Minimal reproduction summary</summary>

* Wrapper calls teller deposit path.
* Teller treats wrapper as `msg.sender`, mints shares to wrapper and records lock against wrapper.
* Immediate transfer from wrapper hits `BeforeTransferHook` and reverts due to lock.

</details>

## References

* Target repository: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
