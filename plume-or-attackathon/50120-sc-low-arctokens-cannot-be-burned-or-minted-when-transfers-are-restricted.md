# 50120 sc low arctokens cannot be burned or minted when transfers are restricted

**Submitted on Jul 21st 2025 at 21:00:21 UTC by @KlosMitSoss for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50120
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

When restricting transfers, it is ensured that both the `to` address and the `from` address are whitelisted. However, this causes calls to `ArcToken::mint()` and `ArcToken::burn()` to revert since `address(0)` cannot be whitelisted.

## Vulnerability Details

When minting ArcTokens, they are transferred from `address(0)`. When burning ArcTokens, they are transferred to `address(0)`. Furthermore, it is possible to restrict transfers, which means that whenever an ArcToken is transferred, `WhiteListRestrictions::isTransferAllowed()` is called (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/WhitelistRestrictions.sol#L101-L111>).

When transfers are restricted, this function only returns true when both the `from` address and the `to` address are whitelisted. However, when minting or burning, one of the two addresses will be `address(0)`. Since `address(0)` cannot be whitelisted (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/WhitelistRestrictions.sol#L135-L137>), minting or burning operations will always revert.

To mitigate this issue, either allow whitelisting `address(0)` or modify `isTransferAllowed()` to return true when only one of the `to` or `from` addresses is whitelisted, as long as the other is `address(0)`.

## Impact Details

ArcTokens can neither be minted nor burned when transfers are restricted.

## References

Code references are provided throughout the report.

## Proof of Concept

{% stepper %}
{% step %}

### Step

Call `WhitelistRestrictions::setTransfersAllowed()` to toggle transfer restrictions. In this case, set it to `false` (restrict transfers).

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/WhitelistRestrictions.sol#L207-L212>
{% endstep %}

{% step %}

### Step

Now, any calls to `ArcToken::mint()` or `ArcToken::burn()` will revert on one of the following lines, because `address(0)` cannot be whitelisted, nor is there any exception to the restriction for transferring from `address(0)` (minting) or transferring to `address(0)` (burning):

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L666>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L671>
  {% endstep %}
  {% endstepper %}
