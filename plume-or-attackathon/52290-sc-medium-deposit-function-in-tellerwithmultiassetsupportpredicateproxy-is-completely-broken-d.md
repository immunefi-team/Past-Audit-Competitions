# 52290 sc medium deposit function in tellerwithmultiassetsupportpredicateproxy is completely broken due to wrong share lock

**Submitted on Aug 9th 2025 at 14:22:52 UTC by @avoloder for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52290
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

The `deposit` function will not work correctly, as the shares will be locked for the proxy contract rather than the original user, making it impossible to transfer those funds back to the user, thus reverting the whole transaction every time.

### Vulnerability Details

The `deposit` function in the **TellerWithMultiAssetSupportPredicateProxy** transfers funds from the user to itself, then calls the `deposit` function in the **TellerWithMultiAssetSupport** contract, which transfers the funds to the vault and mints shares. As per official documentation:

"After deposits all of the depositors shares are locked to their account for the `shareLockPeriod`, which makes flashloan arbitrages impossible".

This step occurs as part of the `deposit` function in the same contract, specifically in the `_afterPublicDeposit()` function where the `shareUnlockTime` mapping is set:

```solidity
shareUnlockTime[user] = block.timestamp + currentShareLockPeriod;
```

However, the problem is that the `user` parameter passed to the `_afterPublicDeposit()` function is `msg.sender` which is in this case **TellerWithMultiAssetSupportPredicateProxy** and not the original sender, because the call is not made using delegateCall. This locks the shares minted to the **TellerWithMultiAssetSupportProxy** which cannot be transferred to the original user in the original `deposit` function because of the `beforeTransferHook` where it is checked:

```solidity
if (shareUnlockTime[from] > block.timestamp) revert TellerWithMultiAssetSupport__SharesAreLocked();
```

Therefore, the call `vault.safeTransfer(recipient, shares);` in `deposit` function of **TellerWithMultiAssetSupportPredicateProxy** will always revert, as the shares are locked. This renders the `deposit` function unusable.

### Recommendation

Forward the original `msg.sender` to the **TellerWithMultiAssetSupport** to be able to correctly lock the funds. You could still keep the logic of minting the shares to the proxy contract, but you can lock the funds as intended.

### Impact Details

Medium — the smart contract's function is not usable (not able to operate) due to wrongly locked (insufficient) funds.

### References

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L91-L94>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupport.sol#L257>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupport.sol#L361-L370>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupport.sol#L189-L191>

## Proof of Concept

This call chain shows how funds are locked for the proxy contract instead of the original user.

{% stepper %}
{% step %}

### Step 1

User X calls `deposit` function in the proxy contract
{% endstep %}

{% step %}

### Step 2

Proxy contract transfers the funds from the user to itself
{% endstep %}

{% step %}

### Step 3

Proxy contract calls `deposit` function in the Teller contract
{% endstep %}

{% step %}

### Step 4

Inside the `deposit` function, `msg.sender` is proxy contract
{% endstep %}

{% step %}

### Step 5

Funds are transferred to the vault, shares are minted to the `msg.sender` -> to the proxy contract
{% endstep %}

{% step %}

### Step 6

`_afterPublicDeposit()` is called passing `msg.sender` as `user` parameter which is still proxy contract
{% endstep %}

{% step %}

### Step 7

Shares are locked and cannot be transferred for the passed user (proxy)
{% endstep %}

{% step %}

### Step 8

Execution continues in the proxy contract where vault shares are supposed to be transferred to User X (proxy contract wants to transfer shares minted to it)
{% endstep %}

{% step %}

### Step 9

Teller's `beforeTransfer()` hook gets triggered where the `shareUnlockTime` mapping for the `from` user is checked against the `block.timestamp`.
{% endstep %}

{% step %}

### Step 10

`from` user here is the proxy contract, as proxy tries to transfer its shares to User X
{% endstep %}

{% step %}

### Step 11

The function reverts, since the `shareUnlockTime` is greater than the `block.timestamp` for the proxy contract
{% endstep %}
{% endstepper %}
