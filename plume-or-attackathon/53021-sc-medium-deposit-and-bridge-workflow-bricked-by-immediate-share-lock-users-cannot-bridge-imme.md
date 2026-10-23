# 53021 sc medium deposit and bridge workflow bricked by immediate share lock users cannot bridge immediately after deposit

**Submitted on Aug 14th 2025 at 17:09:44 UTC by @farman1094 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53021
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

There is a critical logic flaw in all the `depositAndBridge*` functions of the `TellerWithMultiAssetSupportPredicateProxy` and `DexAggregatorWrapperWithPredicateProxy` contracts. When a user calls the `deposit` function, their newly minted shares are immediately locked by the share-lock mechanism, preventing their subsequent transfer. Later, when the bridge logic checks the lock time (via `beforeTransfer`), the check triggers and causes a revert — making it impossible to complete the deposit-and-bridge operation in a single transaction.

### Vulnerability Details

The vulnerable code path is in `DexAggregatorWrapperWithPredicateProxy.depositAndBridgeOneInch`, which calls:

```solidity
teller.depositAndBridge{ value: msg.value - nativeValueToWrap }(
    supportedAsset, supportedAssetAmount, minimumMint, bridgeData
);
```

Internally, this runs `_erc20Deposit` to mint shares for the user, followed immediately by `_afterPublicDeposit`, which executes:

```solidity
shareUnlockTime[user] = block.timestamp + currentShareLockPeriod;
```

This lock is set before the bridge transfer is attempted. The bridge logic then calls `beforeTransfer(from)`, which checks:

```solidity
if (shareUnlockTime[from] > block.timestamp) revert TellerWithMultiAssetSupport__SharesAreLocked();
```

Since the shares were just locked, the transfer will always revert, making it impossible to complete the deposit-and-bridge operation in a single transaction. This logic error effectively disables the combined deposit-and-bridge feature.

## Impact Details

Users will be unable to use Deposit-and-Bridge flows that expect to mint shares and immediately bridge them in the same transaction, breaking a core workflow of the protocol. The contract does not lose funds, but the feature is unusable as designed.

## Proof of Concept

{% stepper %}
{% step %}

### Step: User intends to deposit and bridge in one transaction

* Alice calls `TellerWithMultiAssetSupportPredicateProxy::depositAndBridge` (or the dex wrapper variant) expecting:
  * Deposit accepted
  * Shares minted to Alice
  * Newly minted shares bridged immediately, completing a cross-chain transfer in a single transaction
    {% endstep %}

{% step %}

### Deposit Phase

* The wrapper calls `teller.depositAndBridge`.
* Inside `depositAndBridge`:
  * `_erc20Deposit` mints shares to Alice.
  * `_afterPublicDeposit` is immediately called:

```solidity
shareUnlockTime[Alice] = block.timestamp + currentShareLockPeriod;
```

* This sets a lock on Alice's shares, preventing their transfer until the lock period expires.
  {% endstep %}

{% step %}

### Bridge Phase

* The next step calls `TellerWithMultiAssetSupport::bridge`.
* Inside bridge flow, `beforeTransfer(Alice)` is called:

```solidity
if (shareUnlockTime[Alice] > block.timestamp) revert TellerWithMultiAssetSupport__SharesAreLocked();
```

* Because the lock was just set, this condition is true and the call reverts.
* The entire transaction fails, so deposit-and-bridge cannot complete in one transaction.
  {% endstep %}
  {% endstepper %}

## Conclusion

Immediate locking of freshly minted shares before the bridge transfer causes `beforeTransfer` to always revert, breaking the intended deposit-and-bridge workflow. The issue prevents the combined operation from being completed atomically; users cannot bridge immediately after deposit.
