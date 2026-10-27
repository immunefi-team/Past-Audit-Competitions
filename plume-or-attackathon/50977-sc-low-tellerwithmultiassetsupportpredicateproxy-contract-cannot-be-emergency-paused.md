# 50977 sc low tellerwithmultiassetsupportpredicateproxy contract cannot be emergency paused&#x20;

* Report ID: #50977
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

TellerWithMultiAssetSupportPredicateProxy contract cannot be paused even though it implements the Pausable module.

### Vulnerability Details

The TellerWithMultiAssetSupportPredicateProxy contract implements the Pausable contract module. The `deposit()` and `depositAndBridge()` functions check whether the contract is in a paused state before proceeding. However, the contract does not expose any mechanism to pause or unpause it externally, as it lacks public or external `pause()` and `unpause()` functions. The Pausable module only provides internal `_pause()` and `_unpause()` functions, which must be explicitly invoked by the implementing contract.

### Impact Details

In case of an emergency, the contract cannot be paused, and all operations can proceed as normal.

## References

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L20-L222>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L78-L80>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L133-L135>

## Proof of Concept

{% stepper %}
{% step %}
There is an emergency situation where the contract needs to be paused.
{% endstep %}

{% step %}
No deposit or bridging should be allowed.
{% endstep %}

{% step %}
The owner/admin attempts to pause the contract.
{% endstep %}

{% step %}
The pause functionality is not present in the contract; therefore, the admin is unable to pause it.
{% endstep %}

{% step %}
Deposit and bridging operations continue to work as usual, even in the emergency situation.
{% endstep %}
{% endstepper %}
