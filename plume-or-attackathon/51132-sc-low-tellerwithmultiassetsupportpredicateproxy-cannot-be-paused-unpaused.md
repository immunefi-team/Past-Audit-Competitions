# 51132 sc low tellerwithmultiassetsupportpredicateproxy cannot be paused unpaused

**Submitted on Jul 31st 2025 at 12:32:34 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51132
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Vulnerability Details

`TellerWithMultiAssetSupportPredicateProxy` inherits from OZ `Pausable` contract but does not expose any function to allow pause/unpause the contract. See OpenZeppelin Pausable implementation:

<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/c3961a45380831135b37e55bc3ed441f678a4f5e/contracts/utils/Pausable.sol#L96-L111>

However, `TellerWithMultiAssetSupportPredicateProxy` uses the `paused` check to prevent deposits in case of emergency:

<https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L78-L80>

```solidity
@> import { Pausable } from "@openzeppelin/contracts/utils/Pausable.sol";
...
@> contract TellerWithMultiAssetSupportPredicateProxy is Pausable {
...
    function deposit(
        ERC20 depositAsset,
        uint256 depositAmount,
        uint256 minimumMint,
        address recipient,
        CrossChainTellerBase teller,
        PredicateMessage calldata predicateMessage
    )
        external
        nonReentrant
        returns (uint256 shares)
    {
@>        if (paused()) {
            revert TellerWithMultiAssetSupportPredicateProxy__Paused();
        }
...
```

## Impact Details

Because the contract does not expose pause/unpause functions, the `paused` check can never be activated by governance or an authorized party. Deposits will continue to be accepted in situations where pausing the contract is desired (e.g., compromise or emergency), preventing mitigation via pausing.

## Recommendation

{% hint style="warning" %}
Add `pause` and `unpause` functions to the contract and restrict them (for example with `onlyOwner` or appropriate access control) so governance or the designated admin can pause deposits in emergencies.
{% endhint %}

## Proof of Concept

**Context**

Governance needs to pause the contract because the vault has been compromised.

{% stepper %}
{% step %}

### Attempt to pause

Governance tries to call the `pause` function, but it does not exist in the `TellerWithMultiAssetSupportPredicateProxy` contract.
{% endstep %}

{% step %}

### Unable to pause

Governance cannot pause the contract because no `pause`/`unpause` functions are exposed.
{% endstep %}

{% step %}

### Continued deposits

Funds continue to flow into the compromised vault (also impacting other contracts that use this one, like `DexAggregatorWrapperWithPredicateProxy`).
{% endstep %}
{% endstepper %}

## References

<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/c3961a45380831135b37e55bc3ed441f678a4f5e/contracts/utils/Pausable.sol#L96-L111>

***
