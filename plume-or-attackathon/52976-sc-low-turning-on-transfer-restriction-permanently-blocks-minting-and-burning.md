# 52976 sc low turning on transfer restriction permanently blocks minting and burning

**Submitted on Aug 14th 2025 at 14:36:18 UTC by @silver\_eth for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52976
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

The `WhitelistRestrictions` contract requires both the sender and the receiver to be whitelisted. This causes a problem for minting and burning flows, where the `from` or `to` addresses are `address(0)`.

### Vulnerability Details

ArcToken::\_update is called on every balance change and uses the whitelist restriction to check whether the transfer is permitted. Two important facts:

{% stepper %}
{% step %}

### Whitelisting `address(0)` is impossible

The whitelist implementation does not allow adding `address(0)` to the whitelist.
{% endstep %}

{% step %}

### isTransferAllowed requires both parties be whitelisted

`WhitelistRestrictions::isTransferAllowed` always checks that both `from` and `to` are whitelisted, even when one of them is `address(0)`.
{% endstep %}
{% endstepper %}

Because of these two facts, minting (where `from == address(0)`) or burning (where `to == address(0)`) will cause `isTransferAllowed` to revert.

### Impact Details

{% hint style="warning" %}
If the whitelist-based transfer restriction is enabled and never disabled, the protocol will be unable to mint or burn tokens. This prevents delivering expected token issuance/burning functionality, though it does not directly cause loss of token value.
{% endhint %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/WhitelistRestrictions.sol#L135-L137>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/WhitelistRestrictions.sol#L110>

## Proof Of Concept

{% stepper %}
{% step %}

### Deploy token with whitelist restriction enabled

Deploy a new `ArcToken` with some initial supply and have the whitelist restriction active.
{% endstep %}

{% step %}

### Admin cannot/should not turn off restriction

Assume the admin contract does not include logic to disable `transferAllowed` because it was never intended to be turned off.
{% endstep %}

{% step %}

### Minting / burning attempts revert

Attempts by admin (or the mint/burn logic) to mint new tokens or burn tokens will call `_update`, which invokes the whitelist check. Since `from` or `to` will be `address(0)` and cannot be whitelisted, the check reverts and mint/burn operations fail.
{% endstep %}
{% endstepper %}
