# 52241 sc low unexposed pauseable functionality

**Submitted on Aug 8th 2025 at 22:47:05 UTC by @funkornaut for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52241
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

{% hint style="info" %}
Brief/Intro

The `TellerWithMultiAssetSupportPredicateProxy` contract inherits from `Pausable` and includes pause checks, but cannot be paused because no public functions expose the internal `_pause()` and `_unpause()` methods.
{% endhint %}

## Vulnerability Details

The contract inherits `Pausable` and uses `paused()` guards in `deposit()` and `depositAndBridge()`. However, there are no externally callable `pause()` / `unpause()` functions (only the internal `_pause()` / `_unpause()` exist in `Pausable`).

Result: `paused()` is effectively always `false`, so the `if (paused()) revert` checks are unnecessary and currently wasteful. This diverges from the regular `TellerWithMultiAssetSupport` contract, which does expose `pause()` / `unpause()` and behaves correctly.

{% hint style="warning" %}
Impact

The contract can never be paused in emergencies.
{% endhint %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol?utm\\_source=immunefi#L78>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol?utm\\_source=immunefi#L78>

## Link to Proof of Concept

* <https://gist.github.com/Funkornaut/5e0d2397fea37670615c9b559d9853dd>

## Proof of Concept

{% stepper %}
{% step %}

### Steps demonstrating the issue

* The contract inherits `Pausable`.
* It never exposes the internal `_pause()` or `_unpause()` functions.
* Pause functionality is therefore broken: `paused()` remains `false`.
  {% endstep %}
  {% endstepper %}
