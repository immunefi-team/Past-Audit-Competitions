# 52690 sc medium dos of smart contracts on bridging functions

**Submitted on Aug 12th 2025 at 12:55:04 UTC by @funkornaut for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52690
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>

## Brief / Intro

Smart contracts that cannot accept native tokens can be DoS'ed when interacting with the DexAggregatorWrapperWithPredicateProxy deposit and bridge functions.

## Vulnerability Details

DexAggregatorWrapperWithPredicateProxy exposes an open receive function and uses an internal refund helper which forwards the full contract balance to a recipient:

{% code title="DexAggregatorWrapperWithPredicateProxy::\_refundExcessEth" %}

```solidity
function _refundExcessEth(address payable _recipient) internal {
    uint256 balance = address(this).balance;
    if (balance > 0) {
        (bool success,) = _recipient.call{ value: balance }("");
        if (!success) {
            revert DexAggregatorWrapper__EthRefundFailed();
        }
    }
    // If balance is 0, do nothing.
}
```

{% endcode %}

Because refunds use a direct call that requires the recipient to be able to accept native tokens, a malicious actor can send a tiny amount of ETH (e.g., 1 wei) to the DexAggregatorWrapperWithPredicateProxy before a target contract calls the bridge/deposit functions. When the target contract (which lacks a payable receive/fallback) is expected to receive a refund, the refund attempt reverts and causes the overall transaction to fail.

Additionally, native tokens can slowly accumulate in the wrapper contract over normal usage, increasing the chance of incompatibility with contracts that cannot accept native tokens.

{% hint style="danger" %}
This is a Denial-of-Service vector: contracts without payable receive/fallback functions can be prevented from successfully invoking deposit/bridge functions that attempt to refund ETH.
{% endhint %}

## Impact

Denial of service of smart contracts on deposit and bridge functions: interacting contracts that cannot accept native ETH will have their transactions revert when the contract attempts to refund accumulated ETH.

## Proof of Concept

{% stepper %}
{% step %}

### Step

Attacker observes a contract (the victim) without a receive or fallback payable function preparing to call a bridge/deposit function on DexAggregatorWrapperWithPredicateProxy.
{% endstep %}

{% step %}

### Step

Attacker sends 1 wei (or any small amount of native ETH) to DexAggregatorWrapperWithPredicateProxy prior to the victim's transaction.
{% endstep %}

{% step %}

### Step

Victim contract calls the bridge/deposit function. During execution the wrapper calls `_refundExcessEth(victim)` which performs a call{ value: balance }("") to the victim. Since the victim cannot accept native tokens, the call fails, reverting the wrapper and causing the victim's transaction to revert — resulting in DoS.
{% endstep %}
{% endstepper %}

## References

<details>

<summary>View referenced source lines</summary>

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L376-#L385>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L417>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L222>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L148>

</details>
